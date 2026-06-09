# Design: Ledger Sequence + Repository Cleanup (#256, #255, #262)

**Branch:** `issue-256-ledger-sequence-allocator`  
**Covers:** #256 (sequence to save()), #255 (delete LedgerEntryJpaRepository), #262 (batch findByMessageIds)  
**Date:** 2026-06-09

---

## Problem Statement

`LedgerWriteService.record()` and `ReactiveLedgerWriteService.record()` manually compute sequence numbers via `findLatestBySubjectId()` before persisting a `MessageLedgerEntry`. This was a necessary intermediate state (#253) because `JpaLedgerEntryRepository.save()` (the library class) calls `LedgerSequenceAllocator.nextSequenceNumber()` internally — which would conflict with qhorus pre-setting the field. The fix is a two-step migration: move sequence to `save()` (#256), then delete the qhorus-owned JPA implementation (#255).

Additionally, `getChannelTimeline()` issues one `findByMessageId()` query per EVENT message in the result window — up to 200 individual queries for a full page (#262).

---

## Approach

Use the real JPA repositories in both production and tests (not `InMemoryLedgerEntryRepository`). This is required because qhorus has a split persistence model: `LedgerWriteService` writes `MessageLedgerEntry` via `LedgerEntryRepository.save()`, while `MessageLedgerEntryRepository.findByMessageId()` queries the same rows via JPQL. If `LedgerEntryRepository` routes to an in-memory store and `MessageLedgerEntryRepository` queries H2, timeline tests will silently lose EVENT telemetry. Approach rejected: `casehub-ledger-memory` / `InMemoryLedgerEntryRepository` in tests.

The only non-entity table that `drop-and-create` misses is `ledger_subject_sequence` (used by `LedgerSequenceAllocator`). This is solved with a SQL init script for the qhorus PU in H2 tests.

---

## #256 — Sequence assignment moves to `save()`

### `LedgerEntryJpaRepository.save()` (qhorus-owned, deleted in #255)

Add sequence assignment using the same MERGE SQL as `LedgerSequenceAllocator`. Uses the existing `@PersistenceUnit("qhorus") EntityManager` — no cross-PU dependency:

```java
@Override
public LedgerEntry save(final LedgerEntry entry, final String tenancyId) {
    entry.tenancyId = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    em.createNativeQuery(
        "MERGE INTO ledger_subject_sequence AS t " +
        "USING (SELECT CAST(?1 AS UUID) AS sid) AS s ON t.subject_id = s.sid " +
        "WHEN MATCHED THEN UPDATE SET next_seq = t.next_seq + 1 " +
        "WHEN NOT MATCHED THEN INSERT (subject_id, next_seq) VALUES (s.sid, 2)")
        .setParameter(1, entry.subjectId)
        .executeUpdate();
    Number nextSeq = (Number) em.createNativeQuery(
        "SELECT next_seq - 1 FROM ledger_subject_sequence WHERE subject_id = ?1")
        .setParameter(1, entry.subjectId)
        .getSingleResult();
    entry.sequenceNumber = nextSeq.intValue();
    em.persist(entry);
    return entry;
}
```

### `ReactiveLedgerEntryJpaRepository.save()` (reactive path)

Add reactive sequence via `session.createNativeQuery()`:

```java
@Override
public Uni<LedgerEntry> save(final LedgerEntry entry, final String tenancyId) {
    entry.tenancyId = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    return repo.getSession().flatMap(session ->
        session.createNativeQuery(
            "MERGE INTO ledger_subject_sequence AS t " +
            "USING (SELECT CAST(?1 AS UUID) AS sid) AS s ON t.subject_id = s.sid " +
            "WHEN MATCHED THEN UPDATE SET next_seq = t.next_seq + 1 " +
            "WHEN NOT MATCHED THEN INSERT (subject_id, next_seq) VALUES (s.sid, 2)")
            .setParameter(1, entry.subjectId).executeUpdate()
        .flatMap(i -> session.createNativeQuery(
            "SELECT next_seq - 1 FROM ledger_subject_sequence WHERE subject_id = ?1",
            Integer.class)
            .setParameter(1, entry.subjectId).getSingleResultOrNull())
        .map(seq -> { entry.sequenceNumber = seq != null ? seq : 1; return entry; })
        .flatMap(e -> session.persist(e).replaceWith(e)));
}
```

### `LedgerWriteService.record()` — remove sequence computation

Delete: the `findLatestBySubjectId` call and the `sequenceNumber` local variable (current lines 161–165). Delete: `entry.sequenceNumber = sequenceNumber` (current line 182). The `save()` call at the end sets the sequence. No other changes.

### `ReactiveLedgerWriteService.record()` — remove sequence from reactive chain

Remove `ledger.findLatestBySubjectId(resolvedSubjectId, tenancyId)` from the reactive flatMap chain and the `sequenceNumber` assignment on the entry. The reactive `save()` now sets it.

### `StubLedgerEntryJpaRepository.save()` — simulate sequence

Add a `HashMap<UUID, Integer> sequenceCounters` field. In `save()`, compute `entry.sequenceNumber = sequenceCounters.merge(entry.subjectId, 1, Integer::sum)` before appending to `entries`. The `findLatestBySubjectId()` stub method remains (interface compliance) but is no longer called by `LedgerWriteService`.

### Test infrastructure

Add `runtime/src/test/resources/import-qhorus-test.sql`:
```sql
CREATE TABLE IF NOT EXISTS ledger_subject_sequence (
    subject_id UUID PRIMARY KEY,
    next_seq   BIGINT NOT NULL
);
```

Add to test `application.properties`:
```properties
quarkus.hibernate-orm.qhorus.sql-load-script=import-qhorus-test.sql
```

This table is not a JPA entity — Hibernate `drop-and-create` cannot create it. The SQL init script runs after schema generation and is the correct hook for this case. Reactive tests with PostgreSQL get it via Flyway migrations in `classpath:db/ledger/migration` (already configured; migrations run in the `%reactive-pg` profile).

---

## #255 — Delete `LedgerEntryJpaRepository`, activate library class

### `application.properties` (main)

Add:
```properties
casehub.ledger.datasource=qhorus
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

`casehub.ledger.datasource=qhorus` routes `LedgerEntityManagerProducer` to produce `@LedgerPersistenceUnit EntityManager` from the qhorus PU. Both `JpaLedgerEntryRepository` and `JpaLedgerMerkleFrontierRepository` then operate against the qhorus schema. `casehub-ledger` has a Jandex index (`META-INF/jandex.idx`) — no `quarkus.index-dependency` config needed.

### `application.properties` (test)

Add the same `casehub.ledger.datasource=qhorus`. The `selected-alternatives` entries carry into tests from main `application.properties` — no duplication needed.

`ledger_merkle_frontier` is a `@Entity` in `io.casehub.ledger.runtime` (which is in `quarkus.hibernate-orm.qhorus.packages`) — `drop-and-create` creates it. `ledger_subject_sequence` is created by the SQL init script from #256.

### Delete `LedgerEntryJpaRepository.java`

The class becomes unreachable after the `selected-alternatives` entry activates `JpaLedgerEntryRepository`. Delete it.

### Rename `StubLedgerEntryJpaRepository` → `StubLedgerEntryRepository`

Update all references in `LedgerWriteServiceTest` and `LedgerWritePropagationTest`.

### Behavioral note — actorId tokenisation

`JpaLedgerEntryRepository.save()` calls `actorIdentityProvider.tokenise(entry.actorId)`. This is the ledger's privacy/pseudonymisation layer, separate from qhorus's `InstanceActorIdProvider.resolve()`. With default config (`casehub.ledger.identity.tokenisation.enabled` defaults to false), `tokenise()` is a no-op. Tests are unaffected. In production with tokenisation on, the resolved persona ID is correctly pseudonymised.

---

## #262 — Batch `findByMessageIds()` for `getChannelTimeline()`

### `MessageLedgerEntryRepository.findByMessageIds()`

Add one new method:

```java
public List<MessageLedgerEntry> findByMessageIds(Collection<Long> messageIds) {
    if (messageIds.isEmpty()) {
        return List.of();
    }
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e WHERE e.messageId IN :ids",
            MessageLedgerEntry.class)
        .setParameter("ids", messageIds)
        .getResultList();
}
```

### `QhorusMcpTools.getChannelTimeline()` — eliminate N+1

Replace the per-message `findByMessageId()` loop with:

```java
// Batch-fetch all EVENT ledger entries in one query
List<Long> eventIds = messages.stream()
    .filter(m -> m.messageType == MessageType.EVENT)
    .map(m -> m.id)
    .toList();
Map<Long, MessageLedgerEntry> ledgerByMessageId = eventIds.isEmpty()
    ? Map.of()
    : ledgerRepo.findByMessageIds(eventIds).stream()
        .collect(Collectors.toMap(e -> e.messageId, e -> e));

return messages.stream()
    .map(m -> entityMapper.toTimelineEntry(m,
        m.messageType == MessageType.EVENT ? ledgerByMessageId.get(m.id) : null))
    .toList();
```

### `ReactiveQhorusMcpTools.getChannelTimeline()` — same change

The reactive version has the same N+1 pattern. Add a `findByMessageIds()` equivalent using `@Blocking` and the same batch query, collect into a map, and map the result set.

### Test coverage

`ChannelTimelineTest` already asserts on `tool_name`, `duration_ms`, `token_count` for EVENT entries — it implicitly verifies the batch fetch path. No new tests needed; existing tests prove correctness.

---

## File change summary

| File | Action |
|------|--------|
| `runtime/src/main/java/.../ledger/LedgerEntryJpaRepository.java` | Modify: add MERGE sequence to `save()` → then **Delete** in #255 |
| `runtime/src/main/java/.../ledger/ReactiveLedgerEntryJpaRepository.java` | Modify: add reactive MERGE sequence to `save()` |
| `runtime/src/main/java/.../ledger/LedgerWriteService.java` | Modify: remove findLatestBySubjectId + sequenceNumber |
| `runtime/src/main/java/.../ledger/ReactiveLedgerWriteService.java` | Modify: remove findLatestBySubjectId from reactive chain |
| `runtime/src/main/java/.../ledger/MessageLedgerEntryRepository.java` | Modify: add `findByMessageIds()` |
| `runtime/src/main/java/.../mcp/QhorusMcpTools.java` | Modify: batch fetch in `getChannelTimeline()` |
| `runtime/src/main/java/.../mcp/ReactiveQhorusMcpTools.java` | Modify: batch fetch in `getChannelTimeline()` |
| `runtime/src/main/resources/application.properties` | Modify: add `casehub.ledger.datasource=qhorus` + `selected-alternatives` |
| `runtime/src/test/resources/application.properties` | Modify: add `casehub.ledger.datasource=qhorus` + SQL load script |
| `runtime/src/test/resources/import-qhorus-test.sql` | **Create**: `ledger_subject_sequence` DDL |
| `runtime/src/test/java/.../ledger/StubLedgerEntryJpaRepository.java` | Modify: add sequence counter to `save()` → **Rename** to `StubLedgerEntryRepository` in #255 |
| `runtime/src/test/java/.../ledger/LedgerWriteServiceTest.java` | Modify: update import after rename |
| `runtime/src/test/java/.../ledger/LedgerWritePropagationTest.java` | Modify: update import after rename |

---

## Test strategy

- `LedgerWriteServiceTest` — existing unit tests for sequence numbering verify the stub's new counter logic. Existing attestation, causal chain, and telemetry tests are unchanged.
- `ChannelTimelineTest` — existing telemetry assertions implicitly verify the batch fetch path.
- `@QuarkusTest` integration suite — full run verifies H2 schema (including `ledger_subject_sequence` via SQL init) and `JpaLedgerEntryRepository` integration.
- Reactive tests — all `@Disabled`; the reactive MERGE SQL change is verified by the non-reactive path for now; reactive coverage added when PostgreSQL DevServices tests are re-enabled.

---

## Constraints and non-goals

- Does not add Flyway migrations — qhorus Flyway already includes `classpath:db/ledger/migration` which creates `ledger_subject_sequence` in PostgreSQL. H2 tests get it via the SQL init script.
- Does not add `casehub-ledger-memory` dependency — not needed.
- Does not change `MessageLedgerEntryRepository` query semantics — only adds one new method.
- Does not change `ReactiveLedgerEntryJpaRepository` beyond `save()` — all other methods unchanged.
- Protocol PP-20260607-d83ba5 compliance: `LedgerEntryJpaRepository` (qhorus-owned, deleted in #255) correctly uses `FROM LedgerEntry`; `JpaLedgerEntryRepository` (library) does the same. The protocol remains valid for any future implementations.

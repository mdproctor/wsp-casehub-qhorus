# Design: Ledger Sequence + Repository Cleanup (#256, #255, #262)

**Branch:** `issue-256-ledger-sequence-allocator`  
**Covers:** #256 (sequence to save()), #255 (delete LedgerEntryJpaRepository), #262 (batch findByMessageIds)  
**Date:** 2026-06-09

---

## Problem Statement

`LedgerWriteService.record()` and `ReactiveLedgerWriteService.record()` both manually compute sequence numbers via `findLatestBySubjectId()` before persisting a `MessageLedgerEntry`. This was a necessary intermediate state (#253) because `JpaLedgerEntryRepository.save()` (the library class) calls `LedgerSequenceAllocator` in `save()`, which would conflict with qhorus pre-setting the field. The fix is a two-step migration: move sequence to `save()` (#256), then delete the qhorus-owned JPA implementation (#255).

Additionally, `getChannelTimeline()` (blocking path) issues one `findByMessageId()` query per EVENT message in the result window — up to 200 individual queries for a full page (#262). The reactive path has the opposite problem: it performs no ledger lookup at all for EVENTs, resulting in null telemetry fields.

---

## Approach

Use the real JPA repositories in both production and tests (not `InMemoryLedgerEntryRepository`). This is required because qhorus has a split persistence model: `LedgerWriteService` writes `MessageLedgerEntry` via `LedgerEntryRepository.save()`, while `MessageLedgerEntryRepository.findByMessageId()` queries the same rows via JPQL. If `LedgerEntryRepository` routes to an in-memory store and `MessageLedgerEntryRepository` queries H2, timeline tests will silently lose EVENT telemetry. `casehub-ledger-memory` / `InMemoryLedgerEntryRepository` in tests is rejected.

The only non-entity table that `drop-and-create` misses is `ledger_subject_sequence` (used by `LedgerSequenceAllocator`). This is solved with a SQL init script for the qhorus PU in H2 tests.

---

## #256 — Sequence assignment moves to `save()`

### `LedgerEntryJpaRepository.save()` (qhorus-owned, deleted in #255)

Add sequence assignment using the same MERGE SQL and flush discipline as `LedgerSequenceAllocator`. Uses the existing `@PersistenceUnit("qhorus") EntityManager`. The `em.flush()` between MERGE and SELECT is required: JPA AUTO flush mode does not flush before native queries, and without it the SELECT may read stale data. This exactly replicates `LedgerSequenceAllocator.nextSequenceNumber()`:

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
    em.flush();  // required: AUTO flush mode doesn't flush before native queries
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

Add reactive sequence via `session.createNativeQuery()`. Include `session.flush()` between MERGE and SELECT to exactly replicate the allocator contract (even if Hibernate Reactive's sequential pipeline makes it less strictly necessary):

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
        .flatMap(i -> session.flush())
        .flatMap(v -> session.createNativeQuery(
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

Add a `HashMap<UUID, Integer> sequenceCounters` field. In `save()`:
```java
entry.sequenceNumber = sequenceCounters.merge(entry.subjectId, 1, Integer::sum);
```
Assign before appending to `entries`. The `findLatestBySubjectId()` stub method remains (interface compliance) but is no longer called by `LedgerWriteService`.

### `LedgerEntryJpaRepositoryTest` — clean up misleading `makePlain()`

Remove `e.sequenceNumber = seq` from `makePlain()`. After #256, `LedgerEntryJpaRepository.save()` overwrites `sequenceNumber` via MERGE (first call on any subjectId → 1, second → 2), so the explicit assignments are ignored. By coincidence the values match, but test code that "happens to work" is a correctness hazard. The assertions remain valid: MERGE deterministically produces the same values. The test is deleted in #255; remove the misleading assignments now.

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

`ledger_subject_sequence` is not a JPA entity — Hibernate `drop-and-create` cannot create it. The SQL init script runs after schema generation. In reactive/PostgreSQL tests, Flyway migrations in `classpath:db/ledger/migration` already create it.

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

`casehub.ledger.datasource=qhorus` routes `LedgerEntityManagerProducer` to produce `@LedgerPersistenceUnit EntityManager` from the qhorus PU. Both `JpaLedgerEntryRepository` (writes + sequence + hash chain + Merkle frontier) and `JpaLedgerMerkleFrontierRepository` (frontier read/replace) then operate against the qhorus schema. `casehub-ledger` has a Jandex index (`META-INF/jandex.idx`) — no `quarkus.index-dependency` config needed.

### `application.properties` (test)

Add `casehub.ledger.datasource=qhorus`. The `selected-alternatives` entries carry into tests from main `application.properties`.

`ledger_merkle_frontier` is a `@Entity` in `io.casehub.ledger.runtime` (which is in `quarkus.hibernate-orm.qhorus.packages`) — `drop-and-create` creates it. `ledger_subject_sequence` is created by the SQL init script from #256.

### Delete `LedgerEntryJpaRepository.java`

The class becomes unreachable after the `selected-alternatives` entry activates `JpaLedgerEntryRepository`. Delete it.

### Delete `LedgerEntryJpaRepositoryTest.java`

The test's purpose was proving the #253 fix — that `LedgerEntryJpaRepository` uses `FROM LedgerEntry` rather than `FROM MessageLedgerEntry`. After #255, `LedgerEntryJpaRepository` is gone. The library's `JpaLedgerEntryRepository` is not qhorus's responsibility to test. Additionally, the test passes `null` tenancyId to `save()` and query methods. `JpaLedgerEntryRepository.save()` sets `entry.tenancyId = tenancyId` directly (unlike the qhorus class which defaulted null → `DEFAULT_TENANT_ID`); all library query methods filter `AND e.tenancyId = :tenancyId` with null → `tenancyId = NULL` in SQL → never matches → all three tests fail. Delete the test; do not rewrite it.

### Rename `StubLedgerEntryJpaRepository` → `StubLedgerEntryRepository`

Update all references in `LedgerWriteServiceTest` and `LedgerWritePropagationTest`.

### Behavioral changes from activating `JpaLedgerEntryRepository`

All of these activate based on config, which may differ between test and production deployments:

1. **actorId tokenisation** — `actorIdentityProvider.tokenise(entry.actorId)` runs. With default config (`casehub.ledger.identity.tokenisation.enabled` defaults to false), this is a no-op. With tokenisation enabled, the resolved persona ID is pseudonymised. This is correct — qhorus's `InstanceActorIdProvider.resolve()` maps instance→persona; the ledger's `ActorIdentityProvider` then pseudonymises for privacy. Additive, not conflicting.

2. **Merkle hash chain** — when `casehub.ledger.hash-chain.enabled=true` (set to true in test `application.properties`), every `save()` call: computes `entry.digest = LedgerMerkleTree.leafHash(entry)`, calls `frontierRepo.findBySubjectId()` (1 query) + `frontierRepo.replace()` (delete + insert), and fires `LedgerMerklePublisher.publish()` (CDI event). This adds approximately 2 DB queries and 1 CDI event per ledger write. This is intentional and correct — qhorus is a normative accountability layer and tamper evidence is architectural, not optional. `casehub.ledger.hash-chain.enabled=false` is NOT set in the main `application.properties`; disabling the Merkle chain would make `LedgerVerificationService.verify()` meaningless.

3. **decisionContext sanitisation** — `decisionContextSanitiser.sanitise()` runs when `casehub.ledger.decision-context.enabled=true`. Test `application.properties` has it false — no-op in tests.

### Transaction semantic change in `writeAttestation()`

`JpaLedgerEntryRepository.saveAttestation()` carries `@Transactional` (`REQUIRED`). When called from `LedgerWriteService.writeAttestation()` (inside the REQUIRES_NEW tx from `record()`), the CDI proxy joins the enclosing transaction. If `saveAttestation()` throws `IllegalArgumentException` (entry not found for that tenancyId), the CDI interceptor marks the REQUIRES_NEW tx rollback-only before `writeAttestation()`'s try-catch fires. The catch prevents the exception from propagating, but the tx is permanently rollback-only — the subsequent `ledger.save(entry, tenancyId)` persists to the L1 cache but the commit rolls back, silently losing the ledger entry.

**Assessment of risk:** The current `LedgerEntryJpaRepository.saveAttestation()` (qhorus-owned) has no `@Transactional`, so exceptions degrade gracefully. After #255, they poison the tx.

**Why the normal flow is safe:** `writeAttestation()` is only called when `findEntryById(resolvedCausedByEntryId, tenancyId)` returns present — the entry was found with the same tenancyId in the same REQUIRES_NEW tx. `saveAttestation()`'s internal re-fetch uses the same id and tenancyId in the same tx against committed data — it will find the entry. The `IllegalArgumentException` branch requires tenancyId to match on `findEntryById` but mismatch on `saveAttestation()`'s re-fetch, which is not possible under correct operation.

**Long-term mitigation:** Move attestation writes to a dedicated `@Transactional(REQUIRES_NEW)` CDI bean, so attestation failures roll back their own independent transaction without poisoning the entry-write transaction. Track as a follow-up issue.

---

## #262 — Batch `findByMessageIds()` for `getChannelTimeline()` and fix reactive telemetry

### Diagnosis

**Blocking path (`QhorusMcpTools`):** N+1 query — one `findByMessageId()` per EVENT message in the page (up to 200 per call).

**Reactive path (`ReactiveQhorusMcpTools`):** `blockingGetChannelTimeline()` at line 1424 calls `this::toTimelineEntry` which resolves to `QhorusMcpToolsBase.toTimelineEntry(Message m)` → `entityMapper.toTimelineEntry(m)` (single-arg, passes null for ledger entry). The reactive path performs **no** ledger lookup — EVENT entries show null `tool_name`, `duration_ms`, `token_count`. This is a missing-telemetry bug, not N+1.

### `MessageLedgerEntryRepository.findByMessageIds()`

Add one new method:

```java
public List<MessageLedgerEntry> findByMessageIds(final Collection<Long> messageIds) {
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

Replace the per-message `findByMessageId()` loop with a batch fetch:

```java
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

### `ReactiveQhorusMcpTools.blockingGetChannelTimeline()` — add missing ledger lookup

The reactive fix mirrors the blocking one exactly. `ReactiveQhorusMcpTools` already injects `MessageLedgerEntryRepository ledgerRepo` (line 127) — no new injection needed. Replace `messages.stream().map(this::toTimelineEntry).toList()` with the same batch-fetch pattern:

```java
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

### Test coverage

`ChannelTimelineTest` already asserts on `tool_name`, `duration_ms`, `token_count` for EVENT entries — it implicitly verifies the batch fetch path. No new tests needed; existing tests prove correctness.

---

## File change summary

| File | Issue | Action |
|------|-------|--------|
| `runtime/src/main/java/.../ledger/LedgerEntryJpaRepository.java` | #256 | Modify: add MERGE + flush + SELECT sequence to `save()` |
| `runtime/src/main/java/.../ledger/LedgerEntryJpaRepository.java` | #255 | **Delete** |
| `runtime/src/main/java/.../ledger/ReactiveLedgerEntryJpaRepository.java` | #256 | Modify: add reactive MERGE + flush + SELECT sequence to `save()` |
| `runtime/src/main/java/.../ledger/LedgerWriteService.java` | #256 | Modify: remove `findLatestBySubjectId` + `sequenceNumber` |
| `runtime/src/main/java/.../ledger/ReactiveLedgerWriteService.java` | #256 | Modify: remove `findLatestBySubjectId` from reactive chain |
| `runtime/src/main/java/.../ledger/MessageLedgerEntryRepository.java` | #262 | Modify: add `findByMessageIds()` |
| `runtime/src/main/java/.../mcp/QhorusMcpTools.java` | #262 | Modify: batch fetch in `getChannelTimeline()` |
| `runtime/src/main/java/.../mcp/ReactiveQhorusMcpTools.java` | #262 | Modify: batch fetch in `blockingGetChannelTimeline()` |
| `runtime/src/main/resources/application.properties` | #255 | Modify: add `casehub.ledger.datasource=qhorus` + `selected-alternatives` |
| `runtime/src/test/resources/application.properties` | #256 | Modify: add `casehub.ledger.datasource=qhorus` + SQL load script |
| `runtime/src/test/resources/import-qhorus-test.sql` | #256 | **Create**: `ledger_subject_sequence` DDL |
| `runtime/src/test/java/.../ledger/StubLedgerEntryJpaRepository.java` | #256 | Modify: add `sequenceCounters` HashMap + sequence assignment in `save()` |
| `runtime/src/test/java/.../ledger/LedgerEntryJpaRepositoryTest.java` | #256 | Modify: remove `e.sequenceNumber = seq` from `makePlain()` |
| `runtime/src/test/java/.../ledger/LedgerEntryJpaRepositoryTest.java` | #255 | **Delete** |
| `runtime/src/test/java/.../ledger/StubLedgerEntryJpaRepository.java` | #255 | **Rename** → `StubLedgerEntryRepository` |
| `runtime/src/test/java/.../ledger/LedgerWriteServiceTest.java` | #255 | Modify: update import after rename |
| `runtime/src/test/java/.../ledger/LedgerWritePropagationTest.java` | #255 | Modify: update import after rename |

---

## Test strategy

- `LedgerWriteServiceTest` — existing sequence tests (`record_firstEntry_sequenceNumberIsOne`, `record_threeEntries_sequenceNumbersIncrement`) verify the stub's new counter logic. Existing attestation, causal chain, and telemetry tests unchanged.
- `LedgerEntryJpaRepositoryTest` — cleaned up in #256 (remove misleading `sequenceNumber` assignments), deleted in #255.
- `ChannelTimelineTest` — existing telemetry assertions implicitly verify both the batch fetch path (blocking) and the newly-added ledger lookup path (reactive).
- `@QuarkusTest` integration suite — full run verifies H2 schema (including `ledger_subject_sequence` via SQL init) and `JpaLedgerEntryRepository` integration, including Merkle chain writes.
- Reactive tests — `ReactiveMessageServiceTest` is enabled and runs against PostgreSQL DevServices; the reactive MERGE sequence change is exercised there. Other reactive tests remain `@Disabled`.

---

## Constraints and non-goals

- Does not add Flyway migrations — qhorus Flyway already includes `classpath:db/ledger/migration` which creates `ledger_subject_sequence` in PostgreSQL. H2 tests get it via the SQL init script.
- Does not add `casehub-ledger-memory` dependency — not needed.
- Does not disable the Merkle hash chain — tamper evidence is architectural for qhorus's normative role.
- Does not add an `AttestationWriter` REQUIRES_NEW wrapper — tracked as a follow-up; normal flow is safe.
- Does not change `MessageLedgerEntryRepository` query semantics — only adds one new method.
- Protocol PP-20260607-d83ba5 compliance: `LedgerEntryJpaRepository` (qhorus-owned, deleted in #255) and the library's `JpaLedgerEntryRepository` both correctly use `FROM LedgerEntry`. The protocol remains valid for any future implementations.

# Design: Ledger Sequence + Repository Cleanup (#256, #255, #262)

**Branch:** `issue-256-ledger-sequence-allocator`  
**Covers:** #256 (sequence to save()), #255 (delete LedgerEntryJpaRepository), #262 (batch findByMessageIds)  
**Date:** 2026-06-09

---

## Problem Statement

### Correctness: TOCTOU race in sequence assignment

`LedgerWriteService.record()` and `ReactiveLedgerWriteService.record()` both compute the next sequence number via `findLatestBySubjectId()` before persisting. This has a TOCTOU race:

```
T1: SELECT MAX(sequenceNumber) WHERE subjectId = X  → 3  →  will persist seq=4
T2: SELECT MAX(sequenceNumber) WHERE subjectId = X  → 3  →  will persist seq=4
Both persist with sequenceNumber=4. Duplicate sequences. IDX_LEDGER_ENTRY_SUBJECT_SEQ fires.
```

The MERGE approach (`LedgerSequenceAllocator`) eliminates this: MERGE acquires an exclusive row lock on `ledger_subject_sequence` for the transaction duration. T2 blocks until T1 commits, guaranteeing T2 sees the incremented value.

### API conflict (secondary)

`JpaLedgerEntryRepository.save()` (the library class) calls `LedgerSequenceAllocator.nextSequenceNumber()` internally. The current qhorus code pre-sets `entry.sequenceNumber` before calling `save()` — the library would overwrite it. Removing the pre-set is a prerequisite for switching to the library class.

### Missing telemetry in reactive timeline (separate bug)

`ReactiveQhorusMcpTools.blockingGetChannelTimeline()` calls `this::toTimelineEntry` → `QhorusMcpToolsBase.toTimelineEntry(Message m)` → `entityMapper.toTimelineEntry(m)` (single-arg, passes null for ledger entry). EVENT messages in the reactive timeline have null `tool_name`, `duration_ms`, `token_count`. The blocking path (#262) has an N+1 version of this lookup; both are fixed with the same batch method.

---

## Approach

Use the real JPA repositories in both production and tests (not `InMemoryLedgerEntryRepository`). Reason: qhorus has a split persistence model — `LedgerWriteService` writes `MessageLedgerEntry` via `LedgerEntryRepository.save()`, while `MessageLedgerEntryRepository.findByMessageId()` queries the same rows via JPQL on the qhorus PU. Routing `LedgerEntryRepository` to an in-memory store breaks the timeline lookup in tests. `casehub-ledger-memory` is not added to this project.

The only non-entity table that `drop-and-create` misses is `ledger_subject_sequence` (used by `LedgerSequenceAllocator`, not a JPA entity). Solved with a SQL init script for the qhorus PU in H2 tests.

---

## #256 — Sequence assignment moves to `save()`; reactive Merkle chain added

### `LedgerEntryJpaRepository.save()` (qhorus-owned, deleted in #255)

Add sequence assignment using the exact same MERGE SQL and flush discipline as `LedgerSequenceAllocator.nextSequenceNumber()`. The `em.flush()` between MERGE and SELECT is required: JPA AUTO flush mode does not flush before native queries; without it the SELECT may read the pre-MERGE state from the L1 cache.

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
    em.flush(); // required: AUTO flush mode doesn't flush before native queries
    Number nextSeq = (Number) em.createNativeQuery(
        "SELECT next_seq - 1 FROM ledger_subject_sequence WHERE subject_id = ?1")
        .setParameter(1, entry.subjectId)
        .getSingleResult();
    entry.sequenceNumber = nextSeq.intValue();
    em.persist(entry);
    return entry;
}
```

### `ReactiveLedgerEntryJpaRepository.save()` — MERGE sequence + Merkle chain

The reactive `save()` gains two things: (1) the MERGE-based sequence allocation, and (2) the Merkle hash chain. The reactive path has never had Merkle evidence. The spec's principle — "tamper evidence is architectural for a normative system" — applies equally to both stacks. The implementation is feasible: `LedgerMerkleTree.leafHash()` is pure Java, `LedgerMerklePublisher.publish()` is a plain `@ApplicationScoped` bean using `HttpClient.sendAsync()` (safe to call from any context, no-op when `casehub.ledger.merkle.publish.url` is absent), and the frontier update uses the reactive session directly.

Add two injections to `ReactiveLedgerEntryJpaRepository`:
```java
@Inject LedgerConfig ledgerConfig;
@Inject LedgerMerklePublisher merklePublisher;
```

Revised `save()`:
```java
@Override
public Uni<LedgerEntry> save(final LedgerEntry entry, final String tenancyId) {
    entry.tenancyId = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    return repo.getSession().flatMap(session ->
        // Step 1: atomic sequence allocation (MERGE + flush + SELECT)
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
        .flatMap(seq -> {
            entry.sequenceNumber = seq != null ? seq : 1;
            // Step 2: tokenise actorId (matching JpaLedgerEntryRepository.save() order)
            // so both stacks produce identical canonical bytes for LedgerMerkleTree.leafHash().
            // actorIdentityProvider is already injected in this class (pre-existing field).
            if (entry.actorId != null) {
                entry.actorId = actorIdentityProvider.tokenise(entry.actorId);
            }
            // Step 3: compute digest (pure Java — must be after sequence AND tokenisation)
            if (ledgerConfig.hashChain().enabled()) {
                entry.digest = LedgerMerkleTree.leafHash(entry);
            }
            // Step 4: persist entry
            return session.persist(entry).replaceWith(entry);
        })
        .flatMap(e -> {
            // Step 5: Merkle frontier update (conditional)
            if (!ledgerConfig.hashChain().enabled()) {
                return Uni.createFrom().item(e);
            }
            return session.createQuery(
                    "SELECT f FROM LedgerMerkleFrontier f WHERE f.subjectId = :sid ORDER BY f.level ASC",
                    LedgerMerkleFrontier.class)
                .setParameter("sid", e.subjectId)
                .getResultList()
                .flatMap(currentFrontier -> {
                    final List<LedgerMerkleFrontier> newFrontier =
                        LedgerMerkleTree.append(e.digest, currentFrontier, e.subjectId);
                    final Set<Integer> newLevels = newFrontier.stream()
                        .map(n -> n.level).collect(java.util.stream.Collectors.toSet());
                    // Mirror JpaLedgerMerkleFrontierRepository.replace() semantics.
                    // createMutationQuery() is required for JPQL DELETE — createQuery(String)
                    // without a result type is @Deprecated in Hibernate Reactive 3.2.5.Final.
                    Uni<Integer> deleteOld = newLevels.isEmpty()
                        ? Uni.createFrom().item(0)
                        : session.createMutationQuery(
                            "DELETE FROM LedgerMerkleFrontier f WHERE f.subjectId = :sid AND f.level NOT IN :levels")
                          .setParameter("sid", e.subjectId)
                          .setParameter("levels", newLevels)
                          .executeUpdate();
                    return deleteOld.flatMap(del -> {
                        // Per-node: delete exact level, then persist new node (sequential chain)
                        Uni<Void> chain = Uni.createFrom().voidItem();
                        for (final LedgerMerkleFrontier node : newFrontier) {
                            chain = chain
                                .flatMap(v -> session.createNamedQuery(
                                    "LedgerMerkleFrontier.deleteBySubjectAndLevel")
                                    .setParameter("subjectId", e.subjectId)
                                    .setParameter("level", node.level)
                                    .executeUpdate().replaceWithVoid())
                                .flatMap(v -> session.persist(node));
                        }
                        return chain;
                    }).invoke(() ->
                        merklePublisher.publish(e.subjectId, e.sequenceNumber,
                            LedgerMerkleTree.treeRoot(newFrontier)))
                    .replaceWith(e);
                });
        }));
}
```

### `LedgerWriteService.record()` — remove sequence computation

Delete lines 164–165 (the `findLatestBySubjectId` call and `sequenceNumber` local variable — confirmed from source: 161–163 is the `tenancyId` declaration which remains, used on lines 170, 199, 202, 207). Delete `entry.sequenceNumber = sequenceNumber` (line 182). Also retitle or remove the section comment at lines 157–160 ("Sequence number") since sequence assignment moves to `save()` — `tenancyId` stays but is no longer a sequence-specific concern. The `save()` call assigns the sequence.

### `ReactiveLedgerWriteService.record()` — remove sequence from reactive chain

Remove `ledger.findLatestBySubjectId(resolvedSubjectId, tenancyId)` from the flatMap chain. Remove `entry.sequenceNumber = sequenceNumber` assignment.

### `StubLedgerEntryJpaRepository.save()` — simulate sequence

Add `private final HashMap<UUID, Integer> sequenceCounters = new HashMap<>()`. In `save()`:
```java
entry.sequenceNumber = sequenceCounters.merge(entry.subjectId, 1, Integer::sum);
```
Assign before appending to `entries`. The `findLatestBySubjectId()` method stays (interface compliance) but is no longer called by `LedgerWriteService`.

### `LedgerEntryJpaRepositoryTest` — remove misleading sequence assignments

Remove `e.sequenceNumber = seq` from `makePlain()`. After #256, `LedgerEntryJpaRepository.save()` overwrites `sequenceNumber` via MERGE (first save → 1, second → 2 for the same subjectId). The assignments are silently ignored; coincidental value match is a correctness hazard. The assertions remain valid since MERGE produces the same deterministic values. The test is deleted in #255; clean up the misleading code now.

### Test infrastructure

Create `runtime/src/test/resources/import-qhorus-test.sql`:
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

**Known behavior:** `ledger_subject_sequence` rows are not reset by `drop-and-create`. Because H2 is configured with `DB_CLOSE_DELAY=-1`, the in-memory database persists for the JVM lifetime; rows survive Quarkus context restarts triggered by `@TestProfile`. `CREATE TABLE IF NOT EXISTS` prevents startup errors on restart, but stale rows from prior contexts remain. Tests must use fresh random UUIDs per run as subjectIds to avoid cross-context sequence pollution. All current qhorus tests do this (channels have random IDs); this must remain an invariant for any new ledger-write tests.

For reactive/PostgreSQL tests, Flyway migrations in `classpath:db/ledger/migration` create the table; `%reactive-pg.quarkus.flyway.qhorus.clean-at-start=true` resets it between runs.

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

`casehub.ledger.datasource=qhorus` routes `LedgerEntityManagerProducer` to produce `@LedgerPersistenceUnit EntityManager` from the qhorus PU. `casehub-ledger` has a Jandex index — no `quarkus.index-dependency` config needed.

### `application.properties` (test)

Add `casehub.ledger.datasource=qhorus` at the base level (non-profile). Delete the now-redundant `%reactive-pg.casehub.ledger.datasource=qhorus` line (line 80) — it sets the same value as the base property; keeping it is misleading.

### Delete `LedgerEntryJpaRepository.java`

Unreachable after `selected-alternatives` activates `JpaLedgerEntryRepository`.

### Delete `LedgerEntryJpaRepositoryTest.java`

Purpose of the test was proving the #253 fix — that the qhorus-owned `LedgerEntryJpaRepository` uses `FROM LedgerEntry` rather than `FROM MessageLedgerEntry`. That class no longer exists. The library's `JpaLedgerEntryRepository` is not qhorus's responsibility to test.

Additionally: the test passes `null` as tenancyId to `save()` and query methods. `JpaLedgerEntryRepository.save()` sets `entry.tenancyId = tenancyId` directly (unlike the qhorus class which defaulted null → `DEFAULT_TENANT_ID`). All library query methods filter `AND e.tenancyId = :tenancyId`; with null this generates `tenancyId = NULL` in SQL — never matches — all tests fail. Do not rewrite; delete.

### Rename `StubLedgerEntryJpaRepository` → `StubLedgerEntryRepository`

Update imports in `LedgerWriteServiceTest` and `LedgerWritePropagationTest`. Name reflects the interface being stubbed, not the deleted implementation.

### Behavioral changes from activating `JpaLedgerEntryRepository`

All activate based on config; no change to qhorus defaults required.

1. **actorId tokenisation** — `actorIdentityProvider.tokenise(entry.actorId)` runs. Defaults to false (`casehub.ledger.identity.tokenisation.enabled=false`), so no-op in tests. With tokenisation on, the resolved persona ID is pseudonymised. This is additive and correct — qhorus's `InstanceActorIdProvider` maps instance→persona; the ledger's `ActorIdentityProvider` pseudonymises for privacy. **Note:** the reactive path (`ReactiveLedgerEntryJpaRepository.save()`, revised in #256) now also calls `actorIdentityProvider.tokenise()` before computing `entry.digest`, ensuring both stacks produce identical canonical bytes for `LedgerMerkleTree.leafHash()` when tokenisation is enabled. This is a correctness requirement — without it, the same message written via different stacks would produce different digest values, making `LedgerVerificationService.verify()` inconsistent.

2. **Merkle hash chain** — when `casehub.ledger.hash-chain.enabled=true` (set in test `application.properties`), every `save()` call: (a) computes `entry.digest = LedgerMerkleTree.leafHash(entry)`, (b) calls `updateMerkleFrontier()` via `JpaLedgerMerkleFrontierRepository` (~2 DB queries), (c) fires `LedgerMerklePublisher.publish()` (async HTTP, no-op when url not configured). This is expected and correct — qhorus is a normative accountability layer. `casehub.ledger.hash-chain.enabled=false` is NOT added to `application.properties`; disabling tamper evidence would break `LedgerVerificationService.verify()`. Note: `LedgerMerklePublisher.publish()` checks `casehub.ledger.merkle.publish.url` — tests don't set this, so it is always a no-op in tests.

3. **decisionContext sanitisation** — `decisionContextSanitiser.sanitise()` runs when `casehub.ledger.decision-context.enabled=true`. Test `application.properties` has it false — no-op in tests.

4. **Benign: entity listeners** — `LedgerTraceListener` and `LedgerIdentityEnforcementListener` already fire on every `MessageLedgerEntry` persist today (they are JPA entity listeners; any `em.persist()` on a `LedgerEntry` subtype triggers them, regardless of which repository calls it). `JpaLedgerEntryRepository.save()` does not add new entity listener activation.

### Transaction semantic change in `writeAttestation()`

`JpaLedgerEntryRepository.saveAttestation()` carries `@Transactional` (`REQUIRED`). Called from within `LedgerWriteService.writeAttestation()` (inside the REQUIRES_NEW tx from `record()`), the CDI proxy joins the enclosing transaction. If `saveAttestation()` throws `IllegalArgumentException` (entry not found for that tenancyId), the CDI interceptor marks the REQUIRES_NEW tx rollback-only before `writeAttestation()`'s try-catch fires. The catch prevents propagation, but the tx is permanently rollback-only — `ledger.save(entry, tenancyId)` runs but the commit rolls back, silently losing the ledger entry.

**Why the normal flow is safe:** `writeAttestation()` is called only when `findEntryById(resolvedCausedByEntryId, tenancyId)` returns present. `saveAttestation()`'s internal re-fetch uses the same id and tenancyId in the same REQUIRES_NEW tx against committed data — it will find the entry. The `IllegalArgumentException` requires tenancyId to match on `findEntryById` but mismatch on `saveAttestation()`'s re-fetch; not possible under correct operation.

**Long-term fix:** move attestation writes to a dedicated `@Transactional(REQUIRES_NEW)` CDI bean so attestation failures roll back their own independent transaction. Track as follow-up.

---

## #262 — Batch `findByMessageIds()` + fix reactive telemetry

### Diagnosis

- **Blocking path (`QhorusMcpTools`):** N+1 — one `findByMessageId()` per EVENT in the page (up to 200).
- **Reactive path (`ReactiveQhorusMcpTools`):** zero ledger lookups — `blockingGetChannelTimeline()` calls `this::toTimelineEntry` → `QhorusMcpToolsBase.toTimelineEntry(Message m)` → single-arg `entityMapper.toTimelineEntry(m)` which passes null for the ledger entry. EVENT telemetry is missing, not slow.

### `MessageLedgerEntryRepository.findByMessageIds()`

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

### `ReactiveQhorusMcpTools.blockingGetChannelTimeline()` — add missing lookup

`ReactiveQhorusMcpTools` already injects `MessageLedgerEntryRepository ledgerRepo` (line 127). The fix replaces `messages.stream().map(this::toTimelineEntry).toList()` with the same batch-fetch pattern as the blocking tool. No new injection needed.

---

## File change summary

| File | Issue | Action |
|------|-------|--------|
| `runtime/src/main/java/.../ledger/LedgerEntryJpaRepository.java` | #256 | Modify: add MERGE + flush + SELECT sequence to `save()` |
| `runtime/src/main/java/.../ledger/LedgerEntryJpaRepository.java` | #255 | **Delete** |
| `runtime/src/main/java/.../ledger/ReactiveLedgerEntryJpaRepository.java` | #256 | Modify: add MERGE + flush + SELECT sequence + full Merkle chain to `save()`; inject `LedgerConfig` + `LedgerMerklePublisher` |
| `runtime/src/main/java/.../ledger/LedgerWriteService.java` | #256 | Modify: remove `findLatestBySubjectId` + `sequenceNumber` |
| `runtime/src/main/java/.../ledger/ReactiveLedgerWriteService.java` | #256 | Modify: remove `findLatestBySubjectId` from reactive chain |
| `runtime/src/main/java/.../ledger/MessageLedgerEntryRepository.java` | #262 | Modify: add `findByMessageIds()` |
| `runtime/src/main/java/.../mcp/QhorusMcpTools.java` | #262 | Modify: batch fetch in `getChannelTimeline()` |
| `runtime/src/main/java/.../mcp/ReactiveQhorusMcpTools.java` | #262 | Modify: batch fetch in `blockingGetChannelTimeline()` |
| `runtime/src/main/resources/application.properties` | #255 | Modify: add `casehub.ledger.datasource=qhorus` + `selected-alternatives` |
| `runtime/src/test/resources/application.properties` | #256/#255 | Modify: add `casehub.ledger.datasource=qhorus` + SQL load script; delete redundant `%reactive-pg.casehub.ledger.datasource=qhorus` |
| `runtime/src/test/resources/import-qhorus-test.sql` | #256 | **Create**: `ledger_subject_sequence` DDL |
| `runtime/src/test/java/.../ledger/StubLedgerEntryJpaRepository.java` | #256 | Modify: add `sequenceCounters` HashMap + sequence assignment in `save()` |
| `runtime/src/test/java/.../ledger/LedgerEntryJpaRepositoryTest.java` | #256 | Modify: remove `e.sequenceNumber = seq` from `makePlain()` |
| `runtime/src/test/java/.../ledger/LedgerEntryJpaRepositoryTest.java` | #255 | **Delete** |
| `runtime/src/test/java/.../ledger/StubLedgerEntryJpaRepository.java` | #255 | **Rename** → `StubLedgerEntryRepository` |
| `runtime/src/test/java/.../ledger/LedgerWriteServiceTest.java` | #255 | Modify: update import after rename |
| `runtime/src/test/java/.../ledger/LedgerWritePropagationTest.java` | #255 | Modify: update import after rename |
| `runtime/src/test/java/.../ledger/ReactiveChannelTimelineTest.java` | #262 | **Create**: `@Disabled` reactive timeline test asserting EVENT telemetry fields |

---

## Test strategy

- `LedgerWriteServiceTest` — existing sequence tests (`record_firstEntry_sequenceNumberIsOne`, `record_threeEntries_sequenceNumbersIncrement`) verify the stub's new counter logic. All attestation, causal chain, and telemetry tests unchanged.
- `LedgerEntryJpaRepositoryTest` — `makePlain()` cleaned up in #256, test deleted in #255.
- `ChannelTimelineTest` — existing `timeline_events_haveToolName`, `timeline_events_haveDurationMsAndTokenCount` etc. verify the **blocking** batch fetch path. These tests inject `QhorusMcpTools` (the blocking tool exclusively) and do not exercise `ReactiveQhorusMcpTools.blockingGetChannelTimeline()`.
- `@QuarkusTest` integration suite — full run verifies H2 schema (including `ledger_subject_sequence` via SQL init) and `JpaLedgerEntryRepository` integration with Merkle chain writes.
- `ReactiveMessageServiceTest` — currently `@Disabled` (requires PostgreSQL DevServices); exercises `ReactiveMessageService.dispatch()` → `ReactiveLedgerWriteService.record()` → `ReactiveLedgerEntryJpaRepository.save()` (MERGE sequence + Merkle chain). Does NOT call `getChannelTimeline()`.
- **Gap:** the reactive timeline fix (`blockingGetChannelTimeline()` batch fetch) has no direct test. `blockingGetChannelTimeline()` is gated by `@IfBuildProperty(casehub.qhorus.reactive.enabled=true)`, which requires the reactive build profile (PostgreSQL DevServices). A `ReactiveChannelTimelineTest` asserting EVENT `tool_name`/`duration_ms`/`token_count` fields is added as a new `@Disabled` test class, matching the pattern of `ReactiveMessageServiceTest`. It will run when DevServices tests are re-enabled for CI.

---

## Constraints and non-goals

- Does not add Flyway migrations — qhorus Flyway already includes `classpath:db/ledger/migration` which creates `ledger_subject_sequence` in PostgreSQL.
- Does not add `casehub-ledger-memory` dependency.
- Does not disable the Merkle hash chain in either blocking or reactive paths.
- Does not add an `AttestationWriter` REQUIRES_NEW wrapper — tracked as follow-up.
- Protocol PP-20260607-d83ba5 compliance: both `LedgerEntryJpaRepository` (deleted in #255) and the library's `JpaLedgerEntryRepository` correctly use `FROM LedgerEntry`.

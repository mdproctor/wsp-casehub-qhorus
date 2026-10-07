# Distributed Mesh Phase 1: Runtime Safety Net — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #475 — epic: distributed qhorus mesh — standalone service with clustering
**Issue group:** #475

**Goal:** Add DB-level safety locks to three qhorus runtime subsystems so they are correct under concurrent multi-node writes, without changing single-node behavior.

**Architecture:** Three surgical changes to existing code. Each adds a DB-level lock as a safety net for a read-modify-write pattern currently protected only by JVM-local `synchronized`. In single-node mode, the existing `synchronized` prevents contention, so the DB locks are uncontended and add negligible overhead. In multi-node mode (future Phase 2+), the DB locks prevent data corruption during edge cases like ownership transfer.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, PostgreSQL (H2 for tests)

## Global Constraints

- Java 21 source, Java 26 JVM
- All tests must pass on H2 (CI) — `SELECT FOR UPDATE` degrades to row-level lock on H2, which is sufficient for correctness
- No behavioral change in single-node mode — existing tests must pass unchanged
- Use `ide-tooling` for all structural code edits
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` is the verification command

---

## Batch 1: Merkle Frontier Locking

### Task 1: Add pessimistic lock to Merkle frontier read in QhorusLedgerEntryRepository

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/QhorusLedgerEntryRepository.java:88-128`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/QhorusLedgerMerkleFrontierRepository.java:17-20`
- Test: `runtime/src/test/java/io/casehub/qhorus/ledger/MerkleFrontierLockingTest.java` (create)

**Interfaces:**
- Consumes: `LedgerMerkleFrontierRepository.findBySubjectId(UUID, String)` — existing
- Produces: `QhorusLedgerMerkleFrontierRepository.findBySubjectIdForUpdate(UUID, String)` — new method that issues `SELECT ... FOR UPDATE`

- [ ] **Step 1: Write the failing test**

Create a test that verifies the frontier is locked during the save operation. This is a CDI-free unit test that checks the new method exists and returns the frontier.

```java
package io.casehub.qhorus.ledger;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.ledger.QhorusLedgerEntryRepository;
import io.casehub.qhorus.runtime.ledger.QhorusLedgerMerkleFrontierRepository;
import io.quarkus.hibernate.orm.PersistenceUnit;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
@TestTransaction
class MerkleFrontierLockingTest {

    @Inject QhorusLedgerMerkleFrontierRepository frontierRepo;
    @Inject @PersistenceUnit("qhorus") EntityManager em;

    @Test
    void findBySubjectIdForUpdate_returnsNullWhenNoFrontier() {
        var result = frontierRepo.findBySubjectIdForUpdate(UUID.randomUUID(), "default");
        assertThat(result).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MerkleFrontierLockingTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — `findBySubjectIdForUpdate` method does not exist

- [ ] **Step 3: Add findBySubjectIdForUpdate to QhorusLedgerMerkleFrontierRepository**

The method uses `entityManager.createQuery` with `LockModeType.PESSIMISTIC_WRITE` to issue `SELECT ... FOR UPDATE`.

Use `ide_insert_member` to add to `QhorusLedgerMerkleFrontierRepository`:

```java
public io.casehub.ledger.api.model.LedgerMerkleFrontier findBySubjectIdForUpdate(java.util.UUID subjectId, String tenancyId) {
    var results = em.createQuery(
            "SELECT f FROM LedgerMerkleFrontier f WHERE f.subjectId = ?1 AND f.tenancyId = ?2",
            io.casehub.ledger.jpa.LedgerMerkleFrontier.class)
        .setParameter(1, subjectId)
        .setParameter(2, tenancyId)
        .setLockMode(jakarta.persistence.LockModeType.PESSIMISTIC_WRITE)
        .getResultList();
    return results.isEmpty() ? null : results.get(0);
}
```

The class needs an `@Inject @LedgerPersistenceUnit EntityManager em` field — add it via `ide_insert_member`.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MerkleFrontierLockingTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 5: Wire findBySubjectIdForUpdate into QhorusLedgerEntryRepository.save()**

In `QhorusLedgerEntryRepository.save()` at line ~122, replace the call to `frontierRepo.findBySubjectId(entry.subjectId, tenancyId)` with `frontierRepo.findBySubjectIdForUpdate(entry.subjectId, tenancyId)`.

Use `ide_replace_text_in_file` to change the call site.

The `synchronized` keyword stays — it protects the single-JVM fast path. The `FOR UPDATE` is the multi-node safety net.

- [ ] **Step 6: Run full runtime tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: 2069+ tests pass, 0 failures

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/QhorusLedgerMerkleFrontierRepository.java \
       runtime/src/main/java/io/casehub/qhorus/runtime/ledger/QhorusLedgerEntryRepository.java \
       runtime/src/test/java/io/casehub/qhorus/ledger/MerkleFrontierLockingTest.java
git commit -m "feat(#475): add SELECT FOR UPDATE to Merkle frontier read in ledger save

Prevents concurrent frontier corruption when multiple nodes write to
the same channel. The synchronized keyword stays for the single-JVM
fast path; FOR UPDATE is the multi-node safety net.

Refs #475"
```

---

## Batch 2: LAST_WRITE Optimistic Locking

### Task 2: Add @Version to MessageEntity and handle OptimisticLockException in LAST_WRITE path

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageEntity.java:89-90`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java:371-432`
- Test: `runtime/src/test/java/io/casehub/qhorus/message/LastWriteOptimisticLockTest.java` (create)

**Interfaces:**
- Consumes: `MessageEntity.version` field (existing, `int version = 0`)
- Produces: `MessageEntity.version` annotated with `@Version` (JPA-managed optimistic lock); `MessageService.dispatch()` LAST_WRITE path retries on `OptimisticLockException`

- [ ] **Step 1: Write the failing test**

Create a test verifying that LAST_WRITE dispatch succeeds and the version is managed by JPA. This validates the `@Version` annotation works.

```java
package io.casehub.qhorus.message;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.runtime.message.MessageEntity;
import io.casehub.qhorus.testing.QhorusTestHelper;
import io.quarkus.hibernate.orm.PersistenceUnit;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
@TestTransaction
class LastWriteOptimisticLockTest {

    @Inject QhorusTestHelper helper;
    @Inject @PersistenceUnit("qhorus") EntityManager em;

    @Test
    void lastWrite_secondDispatch_incrementsVersion() {
        var ch = helper.createChannel("lw-version-" + UUID.randomUUID().toString().substring(0, 8),
                ChannelSemantic.LAST_WRITE);
        helper.registerInstance("agent-lw", null);

        helper.sendMessage(ch.name(), "agent-lw", "status", "first");
        helper.sendMessage(ch.name(), "agent-lw", "status", "second");

        var entity = em.createQuery(
                "SELECT e FROM Message e WHERE e.channelId = ?1 AND e.sender = ?2",
                MessageEntity.class)
            .setParameter(1, ch.id())
            .setParameter(2, "agent-lw")
            .getSingleResult();

        assertThat(entity.content).isEqualTo("second");
        assertThat(entity.version).isGreaterThanOrEqualTo(1);
    }
}
```

- [ ] **Step 2: Run test to verify it passes (baseline)**

This test should actually pass already with the manual version increment. Run it to establish the baseline:

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LastWriteOptimisticLockTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS (manual `version + 1` already works)

- [ ] **Step 3: Add @Version annotation to MessageEntity.version**

In `MessageEntity.java:89`, add `@jakarta.persistence.Version` annotation to the `version` field.

Use `ide_replace_text_in_file`:
- Search: `@Column(name = "version", nullable = false)\n    public int version = 0;`
- Replace: `@jakarta.persistence.Version\n    @Column(name = "version", nullable = false)\n    public int version = 0;`

**Important:** With `@Version`, Hibernate automatically manages the version increment and adds a WHERE clause to UPDATE statements: `UPDATE message SET ... WHERE id = ? AND version = ?`. If the version doesn't match (concurrent update), Hibernate throws `OptimisticLockException`.

- [ ] **Step 4: Remove manual version increment from MessageService LAST_WRITE path**

In `MessageService.java:387`, the code currently does `.version(last.version() + 1)`. With `@Version`, Hibernate handles this automatically. Remove the manual increment — change `.version(last.version() + 1)` to remove the `.version(...)` call entirely (let Hibernate manage it).

Use `ide_replace_text_in_file` in the MessageService file.

- [ ] **Step 5: Add OptimisticLockException retry to LAST_WRITE path**

Wrap the LAST_WRITE put in a retry loop (max 3 attempts). On `OptimisticLockException`, re-read the last message and retry the overwrite.

The retry needs to be in `MessageService.dispatch()` around the `messageStore.put(updated)` call in the LAST_WRITE block (~line 389). Wrap the LAST_WRITE block (lines 371-432) in a retry structure:

```java
int lastWriteRetries = 0;
while (true) {
    try {
        // existing LAST_WRITE logic: find last, build updated, put
        break;
    } catch (jakarta.persistence.OptimisticLockException e) {
        if (++lastWriteRetries >= 3) throw e;
        // re-read and retry
    }
}
```

- [ ] **Step 6: Run test to verify it passes with @Version**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LastWriteOptimisticLockTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 7: Run full runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: 2069+ tests pass, 0 failures

**Watch for:** Other tests that manually set `version` on `MessageEntity` or assert exact version values. The `@Version` annotation changes who manages the field — tests that construct `MessageEntity` directly may need adjustment.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageEntity.java \
       runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java \
       runtime/src/test/java/io/casehub/qhorus/message/LastWriteOptimisticLockTest.java
git commit -m "feat(#475): add @Version optimistic locking to LAST_WRITE channel path

Hibernate now manages MessageEntity.version with automatic WHERE
clause on UPDATE. LAST_WRITE dispatch retries up to 3 times on
OptimisticLockException (concurrent overwrite from another node).

Refs #475"
```

---

## Batch 3: Commitment Pessimistic Locking

### Task 3: Add findByCorrelationIdForUpdate to CommitmentStore and wire into CommitmentService

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CommitmentReader.java:16`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCommitmentStore.java:51-60`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCommitmentStore.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java:130-183`
- Test: `runtime/src/test/java/io/casehub/qhorus/message/CommitmentLockingTest.java` (create)

**Interfaces:**
- Consumes: `CommitmentReader.findByCorrelationId(String)` — existing
- Produces: `CommitmentReader.findByCorrelationIdForUpdate(String)` — new method with `SELECT FOR UPDATE`; `CommitmentService.fulfill/decline/fail` use it for state transitions

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.qhorus.message;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.store.CommitmentStore;
import io.casehub.qhorus.testing.QhorusTestHelper;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
@TestTransaction
class CommitmentLockingTest {

    @Inject CommitmentStore commitmentStore;
    @Inject QhorusTestHelper helper;

    @Test
    void findByCorrelationIdForUpdate_returnsCommitment() {
        var ch = helper.createChannel("commit-lock-" + UUID.randomUUID().toString().substring(0, 8));
        helper.registerInstance("agent-cl", null);
        var result = helper.sendMessage(ch.name(), "agent-cl", "command", "do something");
        String corrId = result.correlationId();

        var commitment = commitmentStore.findByCorrelationIdForUpdate(corrId);

        assertThat(commitment).isPresent();
        assertThat(commitment.get().correlationId()).isEqualTo(corrId);
    }

    @Test
    void findByCorrelationIdForUpdate_emptyWhenNotFound() {
        var commitment = commitmentStore.findByCorrelationIdForUpdate("nonexistent-" + UUID.randomUUID());
        assertThat(commitment).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CommitmentLockingTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — `findByCorrelationIdForUpdate` does not exist

- [ ] **Step 3: Add findByCorrelationIdForUpdate to CommitmentReader interface**

Use `ide_insert_member` to add to `CommitmentReader`:

```java
java.util.Optional<Commitment> findByCorrelationIdForUpdate(String correlationId);
```

- [ ] **Step 4: Implement in JpaCommitmentStore**

Use `ide_insert_member` to add to `JpaCommitmentStore`:

```java
@Override
public java.util.Optional<Commitment> findByCorrelationIdForUpdate(String correlationId) {
    String tid = currentPrincipal.tenancyId();
    var results = em.createQuery(
            "SELECT e FROM Commitment e WHERE e.correlationId = ?1 AND e.tenancyId = ?2",
            io.casehub.qhorus.runtime.message.CommitmentEntity.class)
        .setParameter(1, correlationId)
        .setParameter(2, tid)
        .setLockMode(jakarta.persistence.LockModeType.PESSIMISTIC_WRITE)
        .getResultList();
    return results.isEmpty() ? java.util.Optional.empty() : java.util.Optional.of(results.get(0).toDomain());
}
```

- [ ] **Step 5: Implement in InMemoryCommitmentStore**

Use `ide_insert_member` to add to `InMemoryCommitmentStore`. In-memory has no real locking — delegate to `findByCorrelationId`:

```java
@Override
public java.util.Optional<io.casehub.qhorus.api.message.Commitment> findByCorrelationIdForUpdate(String correlationId) {
    return findByCorrelationId(correlationId);
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CommitmentLockingTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 7: Wire findByCorrelationIdForUpdate into CommitmentService**

In `CommitmentService.java`, change `fulfill()` (~line 130), `decline()` (~line 155), and `fail()` (~line 183) to use `store.findByCorrelationIdForUpdate(correlationId)` instead of `store.findByCorrelationId(correlationId)`.

Use `ide_replace_text_in_file` for each method — search for `store.findByCorrelationId(` and replace with `store.findByCorrelationIdForUpdate(` in the three state-transition methods only. Do NOT change read-only callers (e.g., `CommitmentService.findByCorrelationId()` public method that delegates to the store for queries).

- [ ] **Step 8: Run full runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: 2069+ tests pass, 0 failures

- [ ] **Step 9: Run full project tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS, all modules pass

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/store/CommitmentReader.java \
       runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCommitmentStore.java \
       persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCommitmentStore.java \
       runtime-core/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java \
       runtime/src/test/java/io/casehub/qhorus/message/CommitmentLockingTest.java
git commit -m "feat(#475): add SELECT FOR UPDATE to commitment state transitions

CommitmentService.fulfill/decline/fail now use findByCorrelationIdForUpdate
which issues SELECT FOR UPDATE to prevent double-fulfillment under
concurrent multi-node writes.

Refs #475"
```

---

## References

- [2026-10-06-distributed-mesh-design.md] — design spec this plan implements (Section 9.1)
- [QhorusLedgerEntryRepository.java:88-128] — synchronized save() with Merkle frontier read-modify-write
- [MessageService.java:371-432] — LAST_WRITE channel handling
- [MessageEntity.java:89-90] — version field without @Version
- [CommitmentService.java:130-183] — fulfill/decline/fail without locking
- [JpaCommitmentStore.java:51-60] — findByCorrelationId without FOR UPDATE
- [GitHub #475] — distributed mesh epic

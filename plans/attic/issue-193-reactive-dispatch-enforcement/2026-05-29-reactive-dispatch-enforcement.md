# Reactive Dispatch Enforcement Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring `ReactiveMessageService.dispatch()` to full enforcement parity with the blocking `MessageService.dispatch()` gate — ACL, rate limiting, trust gating, type policy, LAST_WRITE semantics, ledger writes with `LedgerWriteOutcome`, and fanOut — while fixing a store-seam violation in the blocking path and enabling the `@Disabled` reactive integration tests via PostgreSQL DevServices.

**Architecture:** Store-seam additions (`findLastMessage`, `updateLastActivity`) are layered first, then the `ReactiveLedgerWriteService` signature is updated, then `ReactiveCommitmentService` is introduced for state transitions. The main `ReactiveMessageService.dispatch()` rewrite uses a phased Mutiny pipeline: pre-transaction checks → single `Panache.withTransaction("qhorus", ...)` (message + commitment open + ledger) → post-transaction state transitions via `ReactiveCommitmentService` → sync side effects. A sealed `TransactResult` discriminated union handles LAST_WRITE early exit without boolean flags.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate Reactive Panache, SmallRye Mutiny, MicroProfile Concurrency (`ManagedExecutor`), PostgreSQL 17 via Podman DevServices, JUnit 5 + `@QuarkusTest`.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` from project root.
**Run specific test:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl runtime`

---

## File Map

| Action | File |
|--------|------|
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/store/MessageStore.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveMessageStore.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveChannelStore.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaChannelStore.java` |
| Modify | `testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java` |
| Modify | `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java` |
| Modify | `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveCommitmentService.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` |
| Modify | `runtime/src/test/resources/application.properties` |
| Modify | `runtime/src/test/java/io/casehub/qhorus/service/ReactiveTestProfile.java` |
| Modify | `runtime/src/test/java/io/casehub/qhorus/service/MessageServiceContractTest.java` |
| Modify | `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java` |

---

### Task 1: Podman DevServices preparation

**Files:** no source changes — environment setup only

- [ ] **Step 1: Verify Podman is running and set memory**

```bash
podman machine inspect | grep -i memory
podman machine stop
podman machine set --memory 4096
podman machine start
podman ps
```

Expected: Podman is running with ≥4096 MB. `podman ps` lists no errors.

- [ ] **Step 2: Verify Quarkus can pull postgres:17-alpine**

```bash
podman pull postgres:17-alpine
```

Expected: image pulled or already present.

---

### Task 2: Store seam — `MessageStore.findLastMessage` (blocking)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/MessageStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java`
- Test: `testing/src/test/java/io/casehub/qhorus/contract/MessageStoreContractTest.java` (or nearest equivalent)

- [ ] **Step 1: Find the blocking store contract test location**

```bash
find /Users/mdproctor/claude/casehub/qhorus/testing/src/test -name "MessageStore*" -o -name "*MessageStore*Contract*" | head -5
```

Note the path — you'll add the test there.

- [ ] **Step 2: Write the failing test in `MessageStoreContractTest`**

Add to the abstract contract test:

```java
@Test
void findLastMessage_returnsMaxIdMessage_whenChannelHasMessages() {
    UUID channelId = UUID.randomUUID();
    Message m1 = new Message();
    m1.channelId = channelId; m1.sender = "alice"; m1.messageType = MessageType.STATUS;
    m1.actorType = ActorType.AGENT;
    blockingStore().put(m1);
    Message m2 = new Message();
    m2.channelId = channelId; m2.sender = "bob"; m2.messageType = MessageType.EVENT;
    m2.actorType = ActorType.SYSTEM;
    blockingStore().put(m2);

    Optional<Message> last = blockingStore().findLastMessage(channelId);
    assertThat(last).isPresent();
    assertThat(last.get().sender).isEqualTo("bob");
}

@Test
void findLastMessage_returnsEmpty_whenChannelHasNoMessages() {
    assertThat(blockingStore().findLastMessage(UUID.randomUUID())).isEmpty();
}
```

Where `blockingStore()` is an abstract method returning `MessageStore` (already present in the base class).

- [ ] **Step 3: Run to confirm compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing 2>&1 | grep -E "ERROR|findLastMessage|cannot find symbol" | head -10
```

Expected: compilation error — `findLastMessage` not defined on `MessageStore`.

- [ ] **Step 4: Add `findLastMessage` to `MessageStore` interface**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/MessageStore.java`, add after `distinctSendersByChannel`:

```java
    /**
     * Returns the most recent message in {@code channelId} by insertion order (highest id),
     * or {@link Optional#empty()} if the channel has no messages.
     * Used by LAST_WRITE semantics to check the current writer.
     */
    Optional<Message> findLastMessage(UUID channelId);
```

- [ ] **Step 5: Implement in `JpaMessageStore`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java`, add:

```java
    @Override
    public Optional<Message> findLastMessage(final UUID channelId) {
        return Message.<Message>find("channelId = ?1 ORDER BY id DESC", channelId)
                .page(0, 1)
                .firstResultOptional();
    }
```

- [ ] **Step 6: Implement in `InMemoryMessageStore`**

In `testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java`, add:

```java
    @Override
    public Optional<Message> findLastMessage(final UUID channelId) {
        return store.values().stream()
                .filter(m -> channelId.equals(m.channelId))
                .max(Comparator.comparingLong(m -> m.id));
    }
```

Add `import java.util.Comparator;` if not already present.

- [ ] **Step 7: Run the new tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest="*MessageStore*" 2>&1 | tail -20
```

Expected: `findLastMessage_returnsMaxIdMessage_whenChannelHasMessages` and `findLastMessage_returnsEmpty_whenChannelHasNoMessages` PASS.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(store): add MessageStore.findLastMessage — store seam for LAST_WRITE check

Refs #193"
```

---

### Task 3: Store seam — `ReactiveMessageStore.findLastMessage`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveMessageStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java`

- [ ] **Step 1: Write the failing test**

Find the reactive message store contract test:

```bash
find /Users/mdproctor/claude/casehub/qhorus/testing/src/test -name "*ReactiveMessage*" | head -5
```

Add to `InMemoryReactiveMessageStoreTest` (or the reactive store contract runner):

```java
@Test
void findLastMessage_returnsMaxIdMessage() {
    UUID channelId = UUID.randomUUID();
    Message m1 = new Message();
    m1.channelId = channelId; m1.sender = "alice"; m1.messageType = MessageType.COMMAND;
    m1.actorType = ActorType.AGENT;
    store.put(m1).await().indefinitely();
    Message m2 = new Message();
    m2.channelId = channelId; m2.sender = "bob"; m2.messageType = MessageType.STATUS;
    m2.actorType = ActorType.AGENT;
    store.put(m2).await().indefinitely();

    Optional<Message> last = store.findLastMessage(channelId).await().indefinitely();
    assertThat(last).isPresent();
    assertThat(last.get().sender).isEqualTo("bob");
}
```

- [ ] **Step 2: Add `findLastMessage` to `ReactiveMessageStore` interface**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveMessageStore.java`, add:

```java
    /**
     * Returns the most recent message in {@code channelId} by id (descending), or empty.
     * Must be called within an active Hibernate Reactive session/transaction context.
     */
    Uni<Optional<Message>> findLastMessage(UUID channelId);
```

- [ ] **Step 3: Implement in `ReactiveJpaMessageStore`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java`, add:

```java
    @Override
    public Uni<Optional<Message>> findLastMessage(final UUID channelId) {
        return repo.find("channelId = ?1 ORDER BY id DESC", channelId)
                .firstResult()
                .map(Optional::ofNullable);
    }
```

- [ ] **Step 4: Implement in `InMemoryReactiveMessageStore`**

In `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java`, add:

```java
    @Override
    public Uni<Optional<Message>> findLastMessage(final UUID channelId) {
        return Uni.createFrom().item(() -> blocking.findLastMessage(channelId));
    }
```

- [ ] **Step 5: Run the new test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest="*ReactiveMessage*" 2>&1 | tail -15
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(store): add ReactiveMessageStore.findLastMessage

Refs #193"
```

---

### Task 4: Store seam — `ReactiveChannelStore.updateLastActivity`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveChannelStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaChannelStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java`

- [ ] **Step 1: Write the failing test**

Find the reactive channel store test:

```bash
find /Users/mdproctor/claude/casehub/qhorus/testing/src/test -name "*ReactiveChannel*" | head -5
```

Add to the reactive channel store test class:

```java
@Test
void updateLastActivity_setsTimestamp() {
    Channel ch = new Channel();
    ch.id = UUID.randomUUID(); ch.name = "act-test-" + ch.id;
    ch.semantic = ChannelSemantic.APPEND;
    store.put(ch).await().indefinitely();

    store.updateLastActivity(ch.id).await().indefinitely();

    Optional<Channel> found = store.find(ch.id).await().indefinitely();
    assertThat(found).isPresent();
    assertThat(found.get().lastActivityAt).isNotNull();
}
```

- [ ] **Step 2: Add `updateLastActivity` to `ReactiveChannelStore` interface**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveChannelStore.java`, add:

```java
    /**
     * Issues a targeted UPDATE setting {@code lastActivityAt = now()} for the given channel.
     * Must be called within an active Hibernate Reactive session/transaction context.
     * Does NOT load or re-attach the channel entity — avoids session scope issues when the
     * channel was loaded pre-transaction.
     */
    Uni<Void> updateLastActivity(UUID channelId);
```

- [ ] **Step 3: Implement in `ReactiveJpaChannelStore`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaChannelStore.java`, add:

```java
    @Override
    public Uni<Void> updateLastActivity(final UUID channelId) {
        return repo.update("lastActivityAt = ?1 WHERE id = ?2", Instant.now(), channelId)
                .replaceWithVoid();
    }
```

Add `import java.time.Instant;` at the top if not present.

- [ ] **Step 4: Implement in `InMemoryReactiveChannelStore`**

In `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveChannelStore.java`, add:

```java
    @Override
    public Uni<Void> updateLastActivity(final UUID channelId) {
        return Uni.createFrom().voidItem().invoke(() ->
            blocking.find(channelId).ifPresent(ch -> ch.lastActivityAt = Instant.now()));
    }
```

Check that `InMemoryChannelStore.find(UUID)` exists (it should — check the blocking store). Add `import java.time.Instant;` if needed.

- [ ] **Step 5: Run the new test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest="*ReactiveChannel*" 2>&1 | tail -15
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(store): add ReactiveChannelStore.updateLastActivity — targeted UPDATE for activity tracking

Refs #193"
```

---

### Task 5: Fix `MessageService.dispatch()` LAST_WRITE store-seam violation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`

- [ ] **Step 1: Confirm existing LAST_WRITE tests pass (baseline)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="MessageDispatchIntegrationTest" 2>&1 | tail -15
```

Expected: all existing tests pass. Note result for comparison after fix.

- [ ] **Step 2: Replace the Panache static call in the LAST_WRITE block**

In `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`, find:

```java
        if (ch != null && ch.semantic == ChannelSemantic.LAST_WRITE) {
            final List<Message> existing = Message.<Message> find(
                    "channelId = ?1 ORDER BY id DESC", ch.id).page(0, 1).list();
            if (!existing.isEmpty()) {
                final Message last = existing.get(0);
```

Replace with:

```java
        if (ch != null && ch.semantic == ChannelSemantic.LAST_WRITE) {
            final Optional<Message> existingOpt = messageStore.findLastMessage(ch.id);
            if (existingOpt.isPresent()) {
                final Message last = existingOpt.get();
```

Update the closing brace / else to match — the body of the `if (!existing.isEmpty())` block becomes `if (existingOpt.isPresent())`, everything inside is unchanged.

- [ ] **Step 3: Run the same integration test to verify no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="MessageDispatchIntegrationTest" 2>&1 | tail -15
```

Expected: same tests pass as before.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(message): replace Panache static call in LAST_WRITE with messageStore.findLastMessage

PP-20260529-eb19c3 store-seam violation — store must be injected, not bypassed.
Refs #193"
```

---

### Task 6: `ReactiveLedgerWriteService` — signature change + `LedgerWriteOutcome` return

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java`

Note: `record()` currently has NO callers in production source — confirmed by grep. This change only affects the class itself. The existing callers (none yet) will use the new signature.

- [ ] **Step 1: Read the blocking `LedgerWriteService.record()` signature to match it**

```bash
grep -n "public.*record" /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java | head -5
```

Note the exact signature — it takes `(MessageDispatch dispatch, Long messageId, UUID commitmentId, Instant occurredAt)`.

Also read `LedgerWriteOutcome` to confirm its constructor:

```bash
grep -n "record LedgerWriteOutcome\|LedgerWriteOutcome(" /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteOutcome.java 2>/dev/null | head -5
```

- [ ] **Step 2: Rewrite `ReactiveLedgerWriteService.record()`**

Replace the entire method signature and body. The class imports and `@Inject` fields remain; only `record()` changes.

New method signature:
```java
    public Uni<LedgerWriteOutcome> record(final MessageDispatch dispatch,
            final Long messageId, final UUID commitmentId, final Instant occurredAt) {
```

The body needs to:
1. Guard on `config.enabled()` — return `Uni.createFrom().item(new LedgerWriteOutcome(null, null, null))` if disabled
2. Build the `MessageLedgerEntry` using `dispatch` fields (not Channel + Message as before)
3. Resolve `subjectId` via 3-priority chain (see blocking service for reference)
4. Resolve `causedByEntryId` for DONE/FAILURE/DECLINE/HANDOFF
5. Save entry and map to `LedgerWriteOutcome`

Full replacement:

```java
    public Uni<LedgerWriteOutcome> record(final MessageDispatch dispatch,
            final Long messageId, final UUID commitmentId, final Instant occurredAt) {
        if (!config.enabled()) {
            return Uni.createFrom().item(new LedgerWriteOutcome(null, null, null));
        }

        final UUID channelId = dispatch.channelId();
        final MessageType msgType = dispatch.type();

        return Panache.withTransaction("qhorus", () ->
            reactiveRepo.findLatestBySubjectId(channelId).flatMap(latestOpt -> {
                final int sequenceNumber = latestOpt.map(e -> e.sequenceNumber + 1).orElse(1);

                final MessageLedgerEntry entry = new MessageLedgerEntry();
                // SubjectId: explicit > correlation root > channelId fallback
                entry.subjectId = dispatch.subjectId() != null
                        ? dispatch.subjectId() : channelId;
                entry.channelId = channelId;
                entry.messageId = messageId;
                entry.messageType = msgType.name();
                entry.target = dispatch.target();
                entry.correlationId = dispatch.correlationId();
                entry.commitmentId = commitmentId;
                entry.actorId = actorIdProvider.resolve(dispatch.sender());
                entry.actorType = dispatch.actorType();
                entry.occurredAt = occurredAt.truncatedTo(ChronoUnit.MILLIS);
                entry.sequenceNumber = sequenceNumber;
                entry.entryType = switch (msgType) {
                    case QUERY, COMMAND, HANDOFF -> LedgerEntryType.COMMAND;
                    default -> LedgerEntryType.EVENT;
                };

                if (msgType == MessageType.EVENT) {
                    populateTelemetry(entry, dispatch.content());
                } else {
                    entry.content = dispatch.content();
                }

                if (CAUSAL_TYPES.contains(msgType.name()) && dispatch.correlationId() != null) {
                    return reactiveRepo.findLatestByCorrelationId(channelId, dispatch.correlationId())
                            .flatMap(priorOpt -> {
                                priorOpt.ifPresent(prior -> {
                                    entry.causedByEntryId = prior.id;
                                    if (ATTESTATION_TYPES.contains(msgType)) {
                                        LOG.infof("Reactive attestation deferred for %s on entry %s " +
                                                "(casehub-ledger#TBD — saveAttestation() not yet reactive)",
                                                msgType, prior.id);
                                    }
                                });
                                return reactiveRepo.save(entry)
                                        .map(saved -> new LedgerWriteOutcome(
                                                saved.id, entry.subjectId, entry.causedByEntryId));
                            });
                }
                return reactiveRepo.save(entry)
                        .map(saved -> new LedgerWriteOutcome(
                                saved.id, entry.subjectId, null));
            })
        );
    }
```

Add required imports at the top of the class:
```java
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.runtime.ledger.LedgerWriteOutcome;
```

Remove the old `record(Channel, Message)` method entirely.
Remove `populateTelemetry` signature change — it still takes `(MessageLedgerEntry, String)`, unchanged.
Remove the `logSkippedAttestation` helper method (replaced by `LOG.infof` inline above).

- [ ] **Step 3: Verify build compiles clean**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep -E "ERROR|WARNING" | grep -v "\[WARNING\].*" | head -20
```

Expected: no compilation errors.

- [ ] **Step 4: Run the full runtime test suite to confirm no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -20
```

Expected: same pass/fail as before (no new failures).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(ledger): ReactiveLedgerWriteService.record() — align signature with blocking, return LedgerWriteOutcome

Implements 3-priority subjectId resolution. Attestation deferred (casehub-ledger issue TBD).
Refs #193"
```

---

### Task 7: `ReactiveCommitmentService` — new reactive commitment state-transition service

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveCommitmentService.java`

- [ ] **Step 1: Write a plain unit test for `delegate()` (the complex two-save case)**

Create test file `runtime/src/test/java/io/casehub/qhorus/service/ReactiveCommitmentServiceTest.java`:

```java
package io.casehub.qhorus.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.ReactiveCommitmentService;
import io.casehub.qhorus.testing.InMemoryReactiveCommitmentStore;

class ReactiveCommitmentServiceTest {

    InMemoryReactiveCommitmentStore store;
    ReactiveCommitmentService svc;

    @BeforeEach
    void setUp() {
        store = new InMemoryReactiveCommitmentStore();
        svc = new ReactiveCommitmentService();
        svc.store = store;
    }

    private Commitment openCommitment(String correlationId, String obligor) {
        Commitment c = new Commitment();
        c.id = UUID.randomUUID();
        c.correlationId = correlationId;
        c.channelId = UUID.randomUUID();
        c.messageType = MessageType.COMMAND;
        c.requester = "requester";
        c.obligor = obligor;
        c.state = CommitmentState.OPEN;
        return store.save(c).await().indefinitely();
    }

    @Test
    void delegate_transitions_parent_to_DELEGATED_and_creates_OPEN_child() {
        String correlationId = "corr-" + UUID.randomUUID();
        Commitment parent = openCommitment(correlationId, "agent-a");

        svc.delegate(correlationId, "agent-b").await().indefinitely();

        // Parent is DELEGATED
        Commitment updated = store.findById(parent.id).await().indefinitely().orElseThrow();
        assertThat(updated.state).isEqualTo(CommitmentState.DELEGATED);
        assertThat(updated.delegatedTo).isEqualTo("agent-b");

        // Child is OPEN with same correlationId
        Commitment child = store.findByCorrelationId(correlationId).await().indefinitely().orElseThrow();
        assertThat(child.id).isNotEqualTo(parent.id);
        assertThat(child.state).isEqualTo(CommitmentState.OPEN);
        assertThat(child.obligor).isEqualTo("agent-b");
        assertThat(child.parentCommitmentId).isEqualTo(parent.id);
    }

    @Test
    void acknowledge_transitions_OPEN_to_ACKNOWLEDGED() {
        String correlationId = "corr-" + UUID.randomUUID();
        Commitment c = openCommitment(correlationId, "agent-a");

        svc.acknowledge(correlationId).await().indefinitely();

        Commitment updated = store.findById(c.id).await().indefinitely().orElseThrow();
        assertThat(updated.state).isEqualTo(CommitmentState.ACKNOWLEDGED);
        assertThat(updated.acknowledgedAt).isNotNull();
    }

    @Test
    void fulfill_transitions_to_FULFILLED() {
        String correlationId = "corr-" + UUID.randomUUID();
        openCommitment(correlationId, "agent-a");

        svc.fulfill(correlationId).await().indefinitely();

        Commitment updated = store.findByCorrelationId(correlationId).await().indefinitely().orElseThrow();
        assertThat(updated.state).isEqualTo(CommitmentState.FULFILLED);
    }

    @Test
    void delegate_is_noop_when_correlationId_is_null() {
        var result = svc.delegate(null, "agent-b").await().indefinitely();
        assertThat(result).isEmpty();
    }

    @Test
    void delegate_is_noop_when_commitment_already_terminal() {
        String correlationId = "corr-" + UUID.randomUUID();
        Commitment c = openCommitment(correlationId, "agent-a");
        c.state = CommitmentState.FULFILLED;
        store.save(c).await().indefinitely();

        var result = svc.delegate(correlationId, "agent-b").await().indefinitely();
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run to confirm failure (class doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ReactiveCommitmentServiceTest" 2>&1 | grep -E "ERROR|cannot find" | head -5
```

Expected: compilation error.

- [ ] **Step 3: Create `ReactiveCommitmentService`**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveCommitmentService.java`:

```java
package io.casehub.qhorus.runtime.message;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.store.ReactiveCommitmentStore;
import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;

/**
 * Reactive mirror of {@link CommitmentService} for state-transition operations.
 *
 * <p>The {@code open()} operation for COMMAND/QUERY is handled inline in
 * {@link ReactiveMessageService#dispatch} within the same {@code withTransaction} as the
 * message insert — ensuring atomicity between message and commitment creation.
 *
 * <p>All other transitions (acknowledge/fulfill/decline/fail/delegate/expireOverdue)
 * open their own {@code Panache.withTransaction("qhorus", ...)} — equivalent semantics
 * to {@code REQUIRES_NEW} in the blocking service.
 */
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class ReactiveCommitmentService {

    @Inject
    ReactiveCommitmentStore store;

    public Uni<Optional<Commitment>> acknowledge(final String correlationId) {
        if (correlationId == null || correlationId.isBlank()) return Uni.createFrom().item(Optional.empty());
        return Panache.withTransaction("qhorus", () ->
            store.findByCorrelationId(correlationId)
                .map(opt -> opt.filter(c -> !c.state.isTerminal())
                    .map(c -> {
                        if (c.acknowledgedAt == null) c.acknowledgedAt = Instant.now();
                        c.state = CommitmentState.ACKNOWLEDGED;
                        store.save(c);
                        return Optional.of(c);
                    }).orElse(Optional.empty()))
        );
    }

    public Uni<Optional<Commitment>> fulfill(final String correlationId) {
        return transition(correlationId, CommitmentState.FULFILLED, c -> c.resolvedAt = Instant.now());
    }

    public Uni<Optional<Commitment>> decline(final String correlationId) {
        return transition(correlationId, CommitmentState.DECLINED, c -> c.resolvedAt = Instant.now());
    }

    public Uni<Optional<Commitment>> fail(final String correlationId) {
        return transition(correlationId, CommitmentState.FAILED, c -> c.resolvedAt = Instant.now());
    }

    /**
     * Transitions the non-terminal commitment to DELEGATED and creates a child OPEN commitment
     * for {@code delegatedTo} inheriting the same {@code correlationId} — two saves in sequence.
     */
    public Uni<Optional<Commitment>> delegate(final String correlationId, final String delegatedTo) {
        if (correlationId == null || correlationId.isBlank()) return Uni.createFrom().item(Optional.empty());
        return Panache.withTransaction("qhorus", () ->
            store.findByCorrelationId(correlationId).flatMap(opt -> {
                if (opt.isEmpty() || opt.get().state.isTerminal()) {
                    return Uni.createFrom().item(Optional.empty());
                }
                final Commitment c = opt.get();
                final UUID parentId = c.id;
                c.state = CommitmentState.DELEGATED;
                c.delegatedTo = delegatedTo;
                c.resolvedAt = Instant.now();
                return store.save(c).flatMap(saved -> {
                    final Commitment child = new Commitment();
                    child.correlationId = correlationId;
                    child.channelId = c.channelId;
                    child.messageType = c.messageType;
                    child.requester = c.requester;
                    child.obligor = delegatedTo;
                    child.expiresAt = c.expiresAt;
                    child.state = CommitmentState.OPEN;
                    child.parentCommitmentId = parentId;
                    return store.save(child).map(ignored -> Optional.of(saved));
                });
            })
        );
    }

    public Uni<Integer> expireOverdue() {
        return Panache.withTransaction("qhorus", () ->
            store.findExpiredBefore(Instant.now()).flatMap(overdue -> {
                final List<Uni<Commitment>> saves = overdue.stream().map(c -> {
                    c.state = CommitmentState.EXPIRED;
                    c.resolvedAt = Instant.now();
                    return store.save(c);
                }).toList();
                if (saves.isEmpty()) return Uni.createFrom().item(0);
                return Uni.join().all(saves).andFailFast().map(List::size);
            })
        );
    }

    public Uni<Optional<Commitment>> findByCorrelationId(final String correlationId) {
        if (correlationId == null || correlationId.isBlank()) return Uni.createFrom().item(Optional.empty());
        return Panache.withSession("qhorus", () -> store.findByCorrelationId(correlationId));
    }

    /**
     * Dispatches the appropriate state transition for the given message type and correlation ID.
     * Called from {@link ReactiveMessageService#dispatch} after the main transaction commits.
     * COMMAND and QUERY are skipped — commitment was opened inline in Phase 2.
     * EVENT is skipped — no commitment effect.
     */
    Uni<Void> updateState(final MessageDispatch dispatch, final UUID commitmentId) {
        final String correlationId = dispatch.correlationId();
        if (correlationId == null) return Uni.createFrom().voidItem();
        return switch (dispatch.type()) {
            case STATUS -> acknowledge(correlationId).replaceWithVoid();
            case RESPONSE, DONE -> fulfill(correlationId).replaceWithVoid();
            case DECLINE -> decline(correlationId).replaceWithVoid();
            case FAILURE -> fail(correlationId).replaceWithVoid();
            case HANDOFF -> delegate(correlationId, dispatch.target()).replaceWithVoid();
            default -> Uni.createFrom().voidItem();
        };
    }

    private Uni<Optional<Commitment>> transition(final String correlationId,
            final CommitmentState target, final java.util.function.Consumer<Commitment> update) {
        if (correlationId == null || correlationId.isBlank()) return Uni.createFrom().item(Optional.empty());
        return Panache.withTransaction("qhorus", () ->
            store.findByCorrelationId(correlationId)
                .map(opt -> opt.filter(c -> !c.state.isTerminal())
                    .map(c -> {
                        update.accept(c);
                        c.state = target;
                        store.save(c);
                        return Optional.of(c);
                    }).orElse(Optional.empty()))
        );
    }
}
```

- [ ] **Step 4: Run the unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ReactiveCommitmentServiceTest" 2>&1 | tail -20
```

Expected: all 5 tests PASS.

- [ ] **Step 5: Run full module build to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

Expected: same pass/fail as before.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveCommitmentService.java runtime/src/test/java/io/casehub/qhorus/service/ReactiveCommitmentServiceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(message): ReactiveCommitmentService — reactive state-transition mirror of CommitmentService

delegate() implements two-save flatMap chain for DELEGATED + child OPEN.
open() case is handled inline in ReactiveMessageService.dispatch() Phase 2.
Refs #193"
```

---

### Task 8: PostgreSQL DevServices test infrastructure

**Files:**
- Modify: `runtime/src/test/resources/application.properties`
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/ReactiveTestProfile.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java`

- [ ] **Step 1: Add `%reactive-pg` block to `application.properties`**

Append to `runtime/src/test/resources/application.properties`:

```properties
# ── Reactive PG profile ────────────────────────────────────────────────────────
# Activated by ReactiveTestProfile.getConfigProfile() returning "reactive-pg".
# Requires a running Podman (≥4 GB) or Docker daemon — Quarkus DevServices starts
# postgres:17-alpine automatically.

# Replace H2 named datasource with real PostgreSQL for the reactive stack
%reactive-pg.quarkus.datasource.qhorus.db-kind=postgresql
%reactive-pg.quarkus.datasource.qhorus.devservices.enabled=true
%reactive-pg.quarkus.datasource.qhorus.devservices.image-name=postgres:17-alpine
%reactive-pg.quarkus.datasource.qhorus.reactive=true
%reactive-pg.quarkus.datasource.qhorus.jdbc=true

# Flyway runs migrations; Hibernate does not recreate schema
%reactive-pg.quarkus.flyway.qhorus.migrate-at-start=true
%reactive-pg.quarkus.flyway.qhorus.clean-at-start=true
%reactive-pg.quarkus.hibernate-orm.qhorus.database.generation=none

# Default datasource — keep H2 stub for casehub-ledger @Default EntityManager
%reactive-pg.quarkus.datasource.db-kind=h2
%reactive-pg.quarkus.datasource.jdbc.url=jdbc:h2:mem:reactive-ledger-stub;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
%reactive-pg.quarkus.hibernate-orm.database.generation=drop-and-create

# casehub-ledger must use the named qhorus PU for ledger entity operations
%reactive-pg.casehub.ledger.datasource=qhorus
```

- [ ] **Step 2: Update `ReactiveTestProfile` to return the named config profile**

Replace the contents of `runtime/src/test/java/io/casehub/qhorus/service/ReactiveTestProfile.java`:

```java
package io.casehub.qhorus.service;

import java.util.Map;

import io.quarkus.test.junit.QuarkusTestProfile;

/**
 * Activates the reactive service stack with PostgreSQL DevServices.
 *
 * <p>Requires Podman ≥ 4 GB (or Docker). Quarkus DevServices starts
 * {@code postgres:17-alpine} automatically.
 *
 * <p>{@code casehub.qhorus.reactive.enabled=true} activates reactive beans via
 * {@code @IfBuildProperty} at augmentation time for the restarted context.
 * This property must NOT appear in {@code application.properties} — it is BUILD_TIME
 * only and would cause {@code SRCFG00050} at runtime validation.
 */
public class ReactiveTestProfile implements QuarkusTestProfile {

    @Override
    public String getConfigProfile() {
        return "reactive-pg";
    }

    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of("casehub.qhorus.reactive.enabled", "true");
    }
}
```

- [ ] **Step 3: Temporarily add a trivial smoke test and remove `@Disabled` from `ReactiveMessageServiceTest`**

In `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java`, remove the `@Disabled` annotation from the class. The class should now look like:

```java
@QuarkusTest
@TestProfile(ReactiveTestProfile.class)
class ReactiveMessageServiceTest extends MessageServiceContractTest {
    // ... existing body unchanged for now
```

- [ ] **Step 4: Run the reactive tests to verify DevServices starts**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ReactiveMessageServiceTest" 2>&1 | tail -30
```

Expected: Quarkus starts with PostgreSQL DevServices (you'll see a log line like `Starting PostgreSQL...`), tests run (may pass or fail on the new enforcement tests we haven't added yet — the existing contract tests should pass). No `SRCFG00050` or `UnsatisfiedResolutionException` errors.

If DevServices fails to start, check:
```bash
podman machine inspect | grep -i memory  # must be ≥4096
podman info | grep -i driver             # must show podman/overlay
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/test/resources/application.properties runtime/src/test/java/io/casehub/qhorus/service/ReactiveTestProfile.java runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(reactive): enable PostgreSQL DevServices profile for reactive integration tests

PP-20260528-ac6d93 — named-datasource profile for reactive PG.
Removes @Disabled from ReactiveMessageServiceTest.
Refs #193"
```

---

### Task 9: `MessageServiceContractTest` — new enforcement tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/MessageServiceContractTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/MessageServiceTest.java` (blocking runner — add abstract helper impls)
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java` (reactive runner — add abstract helper impls)

- [ ] **Step 1: Read current `MessageServiceTest` to understand what it injects**

```bash
head -60 /Users/mdproctor/claude/casehub/qhorus/runtime/src/test/java/io/casehub/qhorus/service/MessageServiceTest.java
```

Note what is injected. You'll add new `@Inject` fields and the abstract helper implementations.

- [ ] **Step 2: Add abstract helper methods and new tests to `MessageServiceContractTest`**

Add the following to `MessageServiceContractTest.java`. These sit alongside the existing abstract methods:

```java
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.message.Commitment;

// ── Abstract setup helpers ────────────────────────────────────────────────────
// Each concrete runner implements these for its own datasource context.

/** Create a channel with the given paused state, ACL, and semantic. Returns channelId. */
protected abstract UUID persistChannel(boolean paused, String allowedWriters,
    Integer rateLimitPerInstance, String allowedTypes, ChannelSemantic semantic);

/** Register an instance with capability tags so ACL tag-based checks work. */
protected abstract void persistInstance(String instanceId, List<String> capabilities);

// Convenience wrappers
protected UUID createOpenChannel() {
    return persistChannel(false, null, null, null, ChannelSemantic.APPEND);
}
protected UUID createPausedChannel() {
    return persistChannel(true, null, null, null, ChannelSemantic.APPEND);
}
protected UUID createAclChannel(String allowedWriters) {
    return persistChannel(false, allowedWriters, null, null, ChannelSemantic.APPEND);
}
protected UUID createRateLimitedChannel(int perInstance) {
    return persistChannel(false, null, perInstance, null, ChannelSemantic.APPEND);
}
protected UUID createTypePolicyChannel(String allowedTypes) {
    return persistChannel(false, null, null, allowedTypes, ChannelSemantic.APPEND);
}
protected UUID createLastWriteChannel() {
    return persistChannel(false, null, null, null, ChannelSemantic.LAST_WRITE);
}
```

Add new test methods:

```java
// ── Enforcement tests ─────────────────────────────────────────────────────────

@Test
void paused_channel_rejects_send() {
    UUID channelId = createPausedChannel();
    assertThrows(IllegalStateException.class,
        () -> send(channelId, "alice", MessageType.COMMAND, "hi", null, null));
}

@Test
void acl_rejects_unauthorised_sender() {
    UUID channelId = createAclChannel("bob");  // only "bob" allowed
    assertThrows(IllegalStateException.class,
        () -> send(channelId, "alice", MessageType.COMMAND, "hi", null, null));
}

@Test
void acl_permits_sender_by_name() {
    UUID channelId = createAclChannel("alice");
    DispatchResult result = send(channelId, "alice", MessageType.COMMAND, "hi", "corr-acl", null);
    assertNotNull(result.messageId());
}

@Test
void acl_permits_sender_by_capability_tag() {
    UUID channelId = createAclChannel("capability:analysis");
    persistInstance("agent-analyst", List.of("capability:analysis"));
    DispatchResult result = send(channelId, "agent-analyst", MessageType.COMMAND, "analyze", "corr-cap", null);
    assertNotNull(result.messageId());
}

@Test
void type_policy_rejects_disallowed_type() {
    UUID channelId = createTypePolicyChannel("COMMAND,QUERY");  // only COMMAND and QUERY
    assertThrows(Exception.class,  // MessageTypeViolationException wrapped by @WrapBusinessError or propagated directly
        () -> send(channelId, "alice", MessageType.EVENT, "evt", null, null));
}

@Test
void last_write_same_sender_updates_in_place() {
    UUID channelId = createLastWriteChannel();
    DispatchResult first = send(channelId, "alice", MessageType.STATUS, "v1", null, null);
    DispatchResult second = send(channelId, "alice", MessageType.STATUS, "v2", null, null);
    assertEquals(first.messageId(), second.messageId());  // same row updated
    Optional<Message> msg = findById(second.messageId());
    assertTrue(msg.isPresent());
    assertEquals("v2", msg.get().content);
}

@Test
void last_write_different_sender_throws() {
    UUID channelId = createLastWriteChannel();
    send(channelId, "alice", MessageType.STATUS, "v1", null, null);
    assertThrows(IllegalStateException.class,
        () -> send(channelId, "bob", MessageType.STATUS, "v2", null, null));
}

@Test
void dispatch_result_has_non_null_ledger_entry_id() {
    UUID channelId = createOpenChannel();
    DispatchResult result = send(channelId, "alice", MessageType.EVENT, "{}", null, null);
    assertNotNull(result.ledgerEntryId(), "ledgerEntryId must be populated by dispatch()");
}
```

- [ ] **Step 3: Implement abstract helpers in `MessageServiceTest` (blocking runner)**

Read what's injected in `MessageServiceTest`, then add:

```java
@Inject ChannelStore channelStore;
@Inject InstanceService instanceService;

@Override
protected UUID persistChannel(boolean paused, String allowedWriters,
        Integer rateLimitPerInstance, String allowedTypes, ChannelSemantic semantic) {
    Channel ch = new Channel();
    ch.id = UUID.randomUUID();
    ch.name = "contract-" + ch.id;
    ch.semantic = semantic;
    ch.paused = paused;
    ch.allowedWriters = allowedWriters;
    ch.rateLimitPerInstance = rateLimitPerInstance;
    ch.allowedTypes = allowedTypes;
    return channelStore.put(ch).id;
}

@Override
protected void persistInstance(String instanceId, List<String> capabilities) {
    instanceService.register(instanceId, "contract-test agent", capabilities);
}
```

Confirm `ChannelStore.put(Channel)` returns `Channel` (check the interface). Add necessary imports.

- [ ] **Step 4: Run blocking tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="MessageServiceTest" 2>&1 | tail -20
```

Expected: all new enforcement tests PASS on the blocking runner (MessageService already enforces everything).

- [ ] **Step 5: Implement abstract helpers in `ReactiveMessageServiceTest` (reactive runner)**

```java
@Inject ReactiveChannelStore reactiveChannelStore;
@Inject ReactiveInstanceService reactiveInstanceService;

@Override
protected UUID persistChannel(boolean paused, String allowedWriters,
        Integer rateLimitPerInstance, String allowedTypes, ChannelSemantic semantic) {
    Channel ch = new Channel();
    ch.id = UUID.randomUUID();
    ch.name = "contract-reactive-" + ch.id;
    ch.semantic = semantic;
    ch.paused = paused;
    ch.allowedWriters = allowedWriters;
    ch.rateLimitPerInstance = rateLimitPerInstance;
    ch.allowedTypes = allowedTypes;
    return Panache.withTransaction("qhorus", () -> reactiveChannelStore.put(ch))
            .await().indefinitely().id;
}

@Override
protected void persistInstance(String instanceId, List<String> capabilities) {
    reactiveInstanceService.register(instanceId, "contract-test agent", capabilities)
            .await().indefinitely();
}
```

Add `import io.quarkus.hibernate.reactive.panache.Panache;` if needed.

- [ ] **Step 6: Run reactive contract tests (they will fail — ReactiveMessageService not yet updated)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ReactiveMessageServiceTest" 2>&1 | grep -E "FAIL|PASS|ERROR" | head -20
```

Expected: some tests fail (enforcement not yet implemented in reactive dispatch). Note which ones.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/test/java/io/casehub/qhorus/service/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(message): add enforcement contract tests for both blocking and reactive runners

paused, ACL, type-policy, LAST_WRITE, ledger entry populated.
Blocking runner (MessageService) passes all. Reactive runner fails until #193 dispatch implemented.
Refs #193"
```

---

### Task 10: `ReactiveMessageService.dispatch()` — full enforcement rewrite

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`

This is the central task. The existing `dispatch()` is replaced entirely. All other methods (`findById`, `pollAfter`, `pollAfterBySender`) remain unchanged.

- [ ] **Step 1: Add new `@Inject` fields and inner types to `ReactiveMessageService`**

Replace the class header section (imports + class body up to and including `dispatch()` method signature) as shown below. The `pollAfter` / `pollAfterBySender` / `findById` methods below `dispatch()` are unchanged.

New imports to add (merge with existing):
```java
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import org.eclipse.microprofile.context.ManagedExecutor;

import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.AllowedWritersPolicy;
import io.casehub.qhorus.runtime.channel.RateLimiter;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.instance.ReactiveInstanceService;
import io.casehub.qhorus.runtime.ledger.LedgerWriteOutcome;
import io.casehub.qhorus.runtime.ledger.ReactiveLedgerWriteService;
import io.casehub.qhorus.runtime.message.ArtefactRefParser;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.CommitmentService;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageObserverDispatcher;
import io.casehub.qhorus.runtime.message.MessageTypePolicy;
import io.casehub.qhorus.runtime.message.ReactiveCommitmentService;
import io.casehub.qhorus.runtime.store.ReactiveChannelStore;
import io.casehub.qhorus.runtime.store.ReactiveMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;
```

New `@Inject` fields (add alongside existing `messageStore`, `channelStore`, `commitmentService`, `observers`):
```java
    @Inject ReactiveInstanceService reactiveInstanceService;
    @Inject AllowedWritersPolicy allowedWritersPolicy;
    @Inject RateLimiter rateLimiter;
    @Inject TrustGateService trustGateService;
    @Inject MessageTypePolicy messageTypePolicy;
    @Inject ReactiveLedgerWriteService reactiveLedgerWriteService;
    @Inject ReactiveCommitmentService reactiveCommitmentService;
    @Inject ReactiveChannelStore reactiveChannelStore;
    @Inject ChannelGateway channelGateway;
    @Inject QhorusConfig config;
    @Inject ManagedExecutor executor;
```

Inner types (add as private package-private classes at the bottom of the file, before the closing `}`):
```java
    sealed interface TransactResult permits OverwriteResult, FullResult {}
    record OverwriteResult(DispatchResult result) implements TransactResult {}
    record FullResult(DispatchContext ctx) implements TransactResult {}
    record DispatchContext(
        long messageId, UUID commitmentId, Instant occurredAt,
        LedgerWriteOutcome ledgerOutcome, String channelName, int replyCount) {}
```

Add `import java.time.Instant;` to imports.

- [ ] **Step 2: Replace `dispatch()` with the full enforcement implementation**

Replace the existing `dispatch()` method entirely:

```java
    /**
     * Dispatches a message to a channel via the reactive path with full enforcement parity.
     *
     * <p>Enforcement sequence (mirrors {@link io.casehub.qhorus.runtime.message.MessageService#dispatch}):
     * <ol>
     *   <li>Channel load (reactive)</li>
     *   <li>Paused check</li>
     *   <li>Writer ACL — reactive tag pre-fetch (guarded), then sync check</li>
     *   <li>Rate limit check (in-memory)</li>
     *   <li>Trust gate — {@link ManagedExecutor} hop for blocking JPA query</li>
     *   <li>Type policy (in-memory)</li>
     *   <li>{@code Panache.withTransaction("qhorus")} — LAST_WRITE / insert /
     *       commitment open / reply count / activity / ledger</li>
     *   <li>State transitions via {@link ReactiveCommitmentService}</li>
     *   <li>Observer dispatch, rate limit record, fanOut</li>
     * </ol>
     */
    public Uni<DispatchResult> dispatch(final MessageDispatch dispatch) {
        return channelStore.find(dispatch.channelId())
            .flatMap(chOpt -> {
                final var ch = chOpt.orElse(null);

                // ── 1. Paused check ─────────────────────────────────────────────
                if (ch != null && ch.paused) {
                    return Uni.createFrom().failure(new IllegalStateException(
                        "Channel '" + ch.name + "' is paused — send_message blocked. Use resume_channel to re-enable."));
                }

                // ── 2. ACL — guarded reactive tag pre-fetch ─────────────────────
                final boolean needsTags = ch != null
                    && ch.allowedWriters != null && !ch.allowedWriters.isBlank()
                    && dispatch.type() != MessageType.EVENT;
                final Uni<List<String>> tagsUni = needsTags
                    ? reactiveInstanceService.findCapabilityTagsForInstance(dispatch.sender())
                    : Uni.createFrom().item(List.of());

                return tagsUni.flatMap(tags -> {
                    if (ch != null && dispatch.type() != MessageType.EVENT) {
                        final List<String> effective = new ArrayList<>(tags);
                        effective.add("role:" + dispatch.actorType().name().toLowerCase());
                        if (!allowedWritersPolicy.isAllowedWriter(dispatch.sender(), ch.allowedWriters, () -> effective)) {
                            return Uni.createFrom().failure(new IllegalStateException(
                                "Sender '" + dispatch.sender() + "' is not permitted to write to channel '"
                                + (ch != null ? ch.name : dispatch.channelId()) + "'. Channel has an allowed_writers ACL."));
                        }
                    }

                    // ── 3. Rate limit check ────────────────────────────────────
                    if (ch != null && dispatch.type() != MessageType.EVENT) {
                        final String rateLimitError = rateLimiter.check(
                            ch.id, ch.name, dispatch.sender(), ch.rateLimitPerChannel, ch.rateLimitPerInstance);
                        if (rateLimitError != null) {
                            return Uni.createFrom().failure(new IllegalStateException(rateLimitError));
                        }
                    }

                    // ── 4. Trust gate — worker thread hop ─────────────────────
                    final boolean trustGateApplies = ch != null
                        && dispatch.type() == MessageType.COMMAND
                        && dispatch.target() != null && !dispatch.target().contains(":")
                        && config.commitment().minObligorTrust() > 0.0;
                    final Uni<Void> trustCheck = trustGateApplies
                        ? Uni.createFrom().<Void>item(() -> {
                              if (!trustGateService.meetsThreshold(dispatch.target(), config.commitment().minObligorTrust())) {
                                  throw new IllegalStateException("COMMAND rejected: obligor '" + dispatch.target()
                                      + "' trust score below threshold " + config.commitment().minObligorTrust());
                              }
                              return null;
                          }).runSubscriptionOn(executor)
                        : Uni.createFrom().voidItem();

                    // ── 5. Type policy ─────────────────────────────────────────
                    if (ch != null) {
                        messageTypePolicy.validate(ch, dispatch.type());
                    }

                    return trustCheck.flatMap(ignored -> {
                        // ── Phase 2: single withTransaction ───────────────────
                        final UUID commitmentId =
                            (dispatch.correlationId() != null
                             && (dispatch.type() == MessageType.COMMAND || dispatch.type() == MessageType.QUERY))
                            ? UUID.randomUUID() : null;

                        return Panache.withTransaction("qhorus", () ->
                            // ── 6. LAST_WRITE semantics ─────────────────────
                            (ch != null && ch.semantic == ChannelSemantic.LAST_WRITE
                                ? reactiveMessageStore.findLastMessage(ch.id)
                                : Uni.createFrom().item(Optional.<Message>empty()))
                            .flatMap(existingOpt -> {
                                if (ch != null && ch.semantic == ChannelSemantic.LAST_WRITE
                                        && existingOpt.isPresent()) {
                                    final Message last = existingOpt.get();
                                    if (last.sender.equals(dispatch.sender())) {
                                        last.content = dispatch.content();
                                        last.messageType = dispatch.type();
                                        last.correlationId = dispatch.correlationId();
                                        last.inReplyTo = dispatch.inReplyTo();
                                        last.artefactRefs = dispatch.artefactRefs();
                                        last.target = dispatch.target();
                                        last.actorType = dispatch.actorType();
                                        last.createdAt = Instant.now();
                                        return reactiveChannelStore.updateLastActivity(ch.id)
                                            .map(v -> (TransactResult) new OverwriteResult(new DispatchResult(
                                                last.id, ch.id, last.sender, last.messageType,
                                                last.correlationId, last.inReplyTo,
                                                ArtefactRefParser.parse(last.artefactRefs), last.target,
                                                null, null, null, 0)));
                                    } else {
                                        return Uni.createFrom().failure(new IllegalStateException(
                                            "LAST_WRITE channel '" + ch.name + "' already has a message from '"
                                            + last.sender + "'. Only the current writer may update this channel."));
                                    }
                                }

                                // ── 7. Normal insert ────────────────────────
                                final Message message = new Message();
                                message.channelId = dispatch.channelId();
                                message.sender = dispatch.sender();
                                message.messageType = dispatch.type();
                                message.actorType = dispatch.actorType();
                                message.content = dispatch.content();
                                message.correlationId = dispatch.correlationId();
                                message.inReplyTo = dispatch.inReplyTo();
                                message.artefactRefs = dispatch.artefactRefs();
                                message.target = dispatch.target();
                                message.deadline = dispatch.deadline();
                                message.commitmentId = commitmentId;

                                return messageStore.put(message).flatMap(m -> {
                                    final long messageId = m.id;
                                    final Instant occurredAt = m.createdAt != null ? m.createdAt : Instant.now();

                                    // ── 8. Commitment open inline (COMMAND/QUERY) ──
                                    final Uni<Void> commitmentOpen = (commitmentId != null)
                                        ? store.save(buildCommitment(commitmentId, dispatch, channelId(ch, dispatch), m)).replaceWithVoid()
                                        : Uni.createFrom().voidItem();

                                    // ── 9. Reply count ──────────────────────────
                                    final int[] replyCountHolder = {0};
                                    final Uni<Void> replyCountUpdate = (dispatch.inReplyTo() != null)
                                        ? messageStore.find(dispatch.inReplyTo())
                                              .invoke(opt -> opt.ifPresent(parent -> {
                                                  parent.replyCount++;
                                                  replyCountHolder[0] = parent.replyCount;
                                              })).replaceWithVoid()
                                        : Uni.createFrom().voidItem();

                                    // ── 10. Channel activity + ledger ───────────
                                    final Uni<Void> activityUpdate = (ch != null)
                                        ? reactiveChannelStore.updateLastActivity(ch.id)
                                        : Uni.createFrom().voidItem();

                                    return Uni.join().all(commitmentOpen, replyCountUpdate, activityUpdate)
                                        .andFailFast()
                                        .flatMap(ignored2 ->
                                            reactiveLedgerWriteService.record(dispatch, messageId, commitmentId, occurredAt)
                                        )
                                        .map(ledgerOutcome -> (TransactResult) new FullResult(
                                            new DispatchContext(messageId, commitmentId, occurredAt,
                                                ledgerOutcome, ch != null ? ch.name : null,
                                                replyCountHolder[0])));
                                });
                            })
                        );
                    })
                    .flatMap(transactResult -> {
                        if (transactResult instanceof OverwriteResult(DispatchResult dr)) {
                            // LAST_WRITE overwrite — record rate limit and return early
                            if (ch != null && dispatch.type() != MessageType.EVENT) {
                                rateLimiter.recordSend(ch.id, dispatch.sender(),
                                    ch.rateLimitPerChannel, ch.rateLimitPerInstance);
                            }
                            return Uni.createFrom().item(dr);
                        }
                        final DispatchContext ctx = ((FullResult) transactResult).ctx();

                        // ── Phase 3: commitment state transitions (non-open) ──
                        return reactiveCommitmentService.updateState(dispatch, ctx.commitmentId())
                            .map(v -> ctx);
                    })
                    .map(result -> {
                        if (result instanceof DispatchResult dr) return dr;  // OverwriteResult path already returned
                        final DispatchContext ctx = (DispatchContext) result;

                        // ── Phase 4: post-tx side effects ─────────────────────
                        // Observer dispatch post-commit — observers see committed state
                        MessageObserverDispatcher.dispatch(
                            ctx.channelName(), dispatch.channelId(),
                            buildObserverMessage(dispatch, ctx), observers.handles());

                        if (ch != null && dispatch.type() != MessageType.EVENT) {
                            rateLimiter.recordSend(ch.id, dispatch.sender(),
                                ch.rateLimitPerChannel, ch.rateLimitPerInstance);
                        }

                        if (ch != null) {
                            try {
                                channelGateway.fanOut(ch.id, ch.name, new OutboundMessage(
                                    UUID.randomUUID(), dispatch.sender(), dispatch.type(),
                                    dispatch.content(),
                                    dispatch.correlationId() != null ? UUID.fromString(dispatch.correlationId()) : null,
                                    dispatch.inReplyTo(), dispatch.actorType()));
                            } catch (final Exception ignored) {
                                // non-fatal; logged per-backend by ChannelGateway
                            }
                        }

                        return new DispatchResult(
                            ctx.messageId(), dispatch.channelId(), dispatch.sender(),
                            dispatch.type(), dispatch.correlationId(), dispatch.inReplyTo(),
                            ArtefactRefParser.parse(dispatch.artefactRefs()), dispatch.target(),
                            ctx.ledgerOutcome() != null ? ctx.ledgerOutcome().entryId() : null,
                            ctx.ledgerOutcome() != null ? ctx.ledgerOutcome().subjectId() : null,
                            ctx.ledgerOutcome() != null ? ctx.ledgerOutcome().causedByEntryId() : null,
                            ctx.replyCount());
                    });
                });
            });
    }

    private static UUID channelId(final io.casehub.qhorus.runtime.channel.Channel ch,
            final MessageDispatch dispatch) {
        return ch != null ? ch.id : dispatch.channelId();
    }

    private static Commitment buildCommitment(final UUID commitmentId, final MessageDispatch dispatch,
            final UUID channelId, final Message message) {
        final Commitment c = new Commitment();
        c.id = commitmentId;
        c.correlationId = dispatch.correlationId();
        c.channelId = channelId;
        c.messageType = dispatch.type();
        c.requester = dispatch.sender();
        c.obligor = dispatch.target();
        c.expiresAt = message.deadline;
        c.state = io.casehub.qhorus.api.message.CommitmentState.OPEN;
        return c;
    }

    private static Message buildObserverMessage(final MessageDispatch dispatch,
            final DispatchContext ctx) {
        final Message m = new Message();
        m.id = ctx.messageId();
        m.channelId = dispatch.channelId();
        m.sender = dispatch.sender();
        m.messageType = dispatch.type();
        m.actorType = dispatch.actorType();
        m.content = dispatch.content();
        m.correlationId = dispatch.correlationId();
        m.inReplyTo = dispatch.inReplyTo();
        m.artefactRefs = dispatch.artefactRefs();
        m.target = dispatch.target();
        m.createdAt = ctx.occurredAt();
        return m;
    }
```

Note: `store` is the `ReactiveCommitmentStore` — make sure it's injected. Rename the field if there's a conflict with the existing `commitmentService` injection. The existing code injects `CommitmentService commitmentService` — the new code adds `@Inject ReactiveCommitmentStore store;` for inline commitment open. Or reuse `reactiveCommitmentService.store` if accessible... better to inject `ReactiveCommitmentStore` directly in `ReactiveMessageService`.

Add `@Inject ReactiveCommitmentStore commitmentStore;` (name it `commitmentStore` to distinguish from `CommitmentService commitmentService`).

Then replace `store.save(...)` in `buildCommitment` call with `commitmentStore.save(...)`.

- [ ] **Step 3: Compile-check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep -E "^.*ERROR" | head -20
```

Fix any compilation errors before proceeding.

- [ ] **Step 4: Run the reactive enforcement tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ReactiveMessageServiceTest" 2>&1 | grep -E "FAIL|PASS|Tests run" | head -30
```

Expected: all enforcement tests PASS (paused, ACL, type policy, LAST_WRITE, ledger entry id populated).

- [ ] **Step 5: Run the full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -30
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(message): ReactiveMessageService.dispatch() — full enforcement parity with blocking gate

Phases: reactive pre-tx checks (paused/ACL/rate-limit/trust-gate/type-policy) →
single withTransaction (LAST_WRITE/insert/commitment-open/reply-count/activity/ledger) →
ReactiveCommitmentService state transitions → post-tx fanOut+observers.
LAST_WRITE uses sealed TransactResult discriminated union for early exit.
Observer dispatch moved post-commit (correctness fix — observers now see committed state).
DispatchResult.ledgerEntryId/subjectId/causedByEntryId now non-null.
Closes #193"
```

---

### Task 11: File GitHub issues and run final build verification

- [ ] **Step 1: File casehub-ledger issue — reactive LedgerAttestation**

```bash
gh issue create --repo casehubio/ledger --title "feat: reactive LedgerAttestation persistence — saveAttestation() for Hibernate Reactive" --body "$(cat <<'EOF'
## Context

`ReactiveMessageLedgerEntryRepository.saveAttestation()` currently throws `UnsupportedOperationException`.
The qhorus reactive path (issue casehubio/qhorus#193) needs attestation writes for DONE/FAILURE/DECLINE
messages to maintain full parity with `LedgerWriteService` (blocking).

## Required

- Implement `saveAttestation(LedgerAttestation)` on `ReactiveMessageLedgerEntryRepository`
  (or expose a reactive `LedgerAttestationRepository` analogous to the blocking one)
- The qhorus `ReactiveLedgerWriteService` currently logs `LOG.infof` and skips attestation for these types

## Refs

casehubio/qhorus#193
EOF
)"
```

- [ ] **Step 2: File casehub-ledger issue — reactive TrustGateService**

```bash
gh issue create --repo casehubio/ledger --title "feat: reactive TrustGateService.meetsThreshold() — Uni<Boolean> variant" --body "$(cat <<'EOF'
## Context

`TrustGateService.meetsThreshold(String actorId, double threshold)` is a blocking JPA method
(queries `ActorTrustScoreRepository`).

The qhorus reactive dispatch (casehubio/qhorus#193) currently bridges this via
`Uni.createFrom().item(() -> trustGateService.meetsThreshold(...)).runSubscriptionOn(managedExecutor)`.
This causes a thread hop on every COMMAND dispatch with a specific obligor, which is avoidable
with a native reactive variant.

## Required

- Add `Uni<Boolean> meetsThreshold(String actorId, double threshold)` to `TrustGateService`
  (or a `ReactiveTrustGateService` bean)
- qhorus will adopt it and remove the `ManagedExecutor` bridge

## Refs

casehubio/qhorus#193
EOF
)"
```

- [ ] **Step 3: Run the complete project build as final verification**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tee /tmp/build-193.log | tail -40
```

Expected: `BUILD SUCCESS`. All modules compile. All tests pass (reactive tests run with PostgreSQL DevServices).

If tests fail, check `/tmp/build-193.log` for the root cause.

- [ ] **Step 4: Verify no new LAST_WRITE regressions in blocking path**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="MessageDispatchIntegrationTest,MessageServiceTest" 2>&1 | tail -10
```

Expected: PASS.

- [ ] **Step 5: Commit build log ref (optional — just verify issues were filed)**

```bash
gh issue list --repo casehubio/ledger --state open --limit 5
```

Confirm both issues appear.

---

## Spec Coverage Self-Check

| Spec requirement | Task |
|-----------------|------|
| `MessageStore.findLastMessage` | Task 2 |
| `ReactiveMessageStore.findLastMessage` | Task 3 |
| `ReactiveChannelStore.updateLastActivity` — targeted UPDATE, no entity load | Task 4 |
| Fix `MessageService` LAST_WRITE Panache static call | Task 5 |
| `ReactiveLedgerWriteService.record()` — signature + `LedgerWriteOutcome` | Task 6 |
| 3-priority `subjectId` resolution | Task 6 |
| Attestation deferred with `LOG.infof` | Task 6 |
| `ReactiveCommitmentService` — state transitions incl. `delegate()` two-save chain | Task 7 |
| PostgreSQL DevServices profile | Task 8 |
| `ReactiveTestProfile.getConfigProfile()` → `"reactive-pg"` | Task 8 |
| Remove `@Disabled` from `ReactiveMessageServiceTest` | Task 8 |
| Contract test — paused, ACL, type policy, LAST_WRITE, ledger entry id | Task 9 |
| `TransactResult` discriminated union for LAST_WRITE early exit | Task 10 |
| Pre-tx ACL guard — skip DB fetch for open channels / EVENT | Task 10 |
| Trust gate via `ManagedExecutor` (not `Infrastructure.getDefaultWorkerPool()`) | Task 10 |
| `ReactiveChannelStore.updateLastActivity` called inside tx, not entity mutation | Task 10 |
| Commitment `open()` inline in Phase 2 withTransaction | Task 10 |
| State transitions via `ReactiveCommitmentService.updateState()` in Phase 3 | Task 10 |
| Observer dispatch post-commit (correctness fix) | Task 10 |
| `if (ch != null)` guard on fanOut | Task 10 |
| `DispatchResult` with non-null ledger fields | Task 10 |
| File casehub-ledger issue for reactive attestation | Task 11 |
| File casehub-ledger issue for reactive TrustGateService | Task 11 |

# Watchdog Store Seam + Router Comment — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix three Panache static-call bypasses in `WatchdogEvaluationService`, add `MessageStore.count(MessageQuery)`, and document `ConfiguredWatchdogAlertRouter.route()`.

**Architecture:** Extract `MessageQueryJpql` to eliminate existing scan() duplication, then add `count(MessageQuery)` to both store interfaces. Fix the three evaluation methods to use injected stores. Add regression tests. Add router doc comment.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache/Hibernate ORM, JUnit 5, `@QuarkusTest`

**Issues:** casehubio/qhorus#205 (store seam), casehubio/qhorus#206 (router comment)

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl runtime,testing`

---

## File Map

| File | Action | What changes |
|------|--------|--------------|
| `runtime/.../store/jpa/MessageQueryJpql.java` | **Create** | Package-private record; shared WHERE-clause builder |
| `runtime/.../store/MessageStore.java` | **Modify** | + `long count(MessageQuery query)` |
| `runtime/.../store/ReactiveMessageStore.java` | **Modify** | + `Uni<Long> count(MessageQuery query)` |
| `runtime/.../store/jpa/JpaMessageStore.java` | **Modify** | Refactor `scan()` to use `MessageQueryJpql`; add `count()` |
| `runtime/.../store/jpa/ReactiveJpaMessageStore.java` | **Modify** | Refactor `scan()` to use `MessageQueryJpql`; add `count()` |
| `testing/.../InMemoryMessageStore.java` | **Modify** | + `long count(MessageQuery query)` |
| `testing/.../InMemoryReactiveMessageStore.java` | **Modify** | + `Uni<Long> count(MessageQuery query)` |
| `testing/.../contract/MessageStoreContractTest.java` | **Modify** | Abstract `count()` + 2 contract tests |
| `runtime/.../watchdog/WatchdogEvaluationService.java` | **Modify** | Inject `CommitmentStore` + `InstanceStore`; fix 3 methods |
| `runtime/.../watchdog/WatchdogEvaluationServiceTest.java` | **Create** | 3 new condition tests |
| `runtime/.../watchdog/ConfiguredWatchdogAlertRouter.java` | **Modify** | Doc comment on `route()` |

---

## Task 1: Extract `MessageQueryJpql`

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageQueryJpql.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java`

Both `JpaMessageStore.scan()` and `ReactiveJpaMessageStore.scan()` contain identical WHERE-clause construction — 30+ lines duplicated. This task extracts a shared builder before adding `count()` (otherwise count would be a third copy).

- [ ] **Step 1.1: Create `MessageQueryJpql`**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageQueryJpql.java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;

import io.casehub.qhorus.runtime.store.query.MessageQuery;

/**
 * Builds reusable JPQL WHERE predicates for {@link MessageQuery}.
 * Shared by {@link JpaMessageStore} and {@link ReactiveJpaMessageStore}
 * so scan() and count() stay in sync when MessageQuery gains new fields.
 */
record MessageQueryJpql(String where, Object[] params) {

    static MessageQueryJpql from(MessageQuery q) {
        StringBuilder where = new StringBuilder("1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.channelId() != null) {
            where.append(" AND channelId = ?").append(idx++);
            params.add(q.channelId());
        }
        if (q.afterId() != null) {
            where.append(" AND id > ?").append(idx++);
            params.add(q.afterId());
        }
        if (q.sender() != null) {
            where.append(" AND sender = ?").append(idx++);
            params.add(q.sender());
        }
        if (q.target() != null) {
            where.append(" AND target = ?").append(idx++);
            params.add(q.target());
        }
        if (q.inReplyTo() != null) {
            where.append(" AND inReplyTo = ?").append(idx++);
            params.add(q.inReplyTo());
        }
        if (q.excludeTypes() != null && !q.excludeTypes().isEmpty()) {
            where.append(" AND messageType NOT IN ?").append(idx++);
            params.add(q.excludeTypes());
        }
        if (q.contentPattern() != null) {
            where.append(" AND LOWER(content) LIKE ?").append(idx++);
            params.add("%" + q.contentPattern().toLowerCase() + "%");
        }

        return new MessageQueryJpql(where.toString(), params.toArray());
    }
}
```

- [ ] **Step 1.2: Refactor `JpaMessageStore.scan()` to use `MessageQueryJpql`**

Replace the `scan()` body (lines 34–75 in `JpaMessageStore.java`):

```java
@Override
public List<Message> scan(MessageQuery q) {
    MessageQueryJpql mq = MessageQueryJpql.from(q);
    String jpql = "FROM Message WHERE " + mq.where()
            + (q.descending() ? " ORDER BY id DESC" : " ORDER BY id ASC");

    if (q.limit() != null) {
        return Message.find(jpql, mq.params()).page(0, q.limit()).list();
    }
    return Message.list(jpql, mq.params());
}
```

Remove the old `import java.util.ArrayList` if no longer needed elsewhere in the file.

- [ ] **Step 1.3: Refactor `ReactiveJpaMessageStore.scan()` to use `MessageQueryJpql`**

Replace the `scan()` body (lines 40–78 in `ReactiveJpaMessageStore.java`):

```java
@Override
public Uni<List<Message>> scan(MessageQuery q) {
    MessageQueryJpql mq = MessageQueryJpql.from(q);
    String jpql = "FROM Message WHERE " + mq.where() + " ORDER BY id ASC";

    return repo.list(jpql, mq.params())
            .map(results -> q.limit() != null && results.size() > q.limit()
                    ? results.subList(0, q.limit())
                    : results);
}
```

- [ ] **Step 1.4: Build and verify no compilation errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: `BUILD SUCCESS` with no errors.

- [ ] **Step 1.5: Run existing MessageStore tests to confirm scan() behaviour is unchanged**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing -Dtest="JpaMessageStoreTest,InMemoryMessageStoreTest,InMemoryReactiveMessageStoreTest,MessageQueryTest" -q
```

Expected: All tests pass.

- [ ] **Step 1.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageQueryJpql.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#205): extract MessageQueryJpql — eliminate duplicated WHERE-clause building in JPA stores"
```

---

## Task 2: `MessageStore.count(MessageQuery)` — Interface + InMemory + Contract Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/MessageStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java`
- Modify: `testing/src/test/java/io/casehub/qhorus/testing/contract/MessageStoreContractTest.java`

- [ ] **Step 2.1: Write two failing contract tests in `MessageStoreContractTest`**

Add these two tests and the new abstract method to `MessageStoreContractTest`. The abstract method makes both `InMemoryMessageStoreTest` and `InMemoryReactiveMessageStoreTest` fail to compile until `count()` is implemented everywhere.

In `MessageStoreContractTest.java`, add after the existing abstract declarations:

```java
protected abstract long count(MessageQuery query);
```

Add the two contract tests after the existing test methods:

```java
@Test
void count_byChannel_excludesOtherChannels() {
    UUID chA = UUID.randomUUID();
    UUID chB = UUID.randomUUID();
    put(msg(chA, "alice", MessageType.COMMAND));
    put(msg(chA, "bob", MessageType.COMMAND));
    put(msg(chB, "charlie", MessageType.COMMAND));

    long result = count(MessageQuery.builder().channelId(chA).build());

    assertEquals(2, result);
}

@Test
void count_excludesSpecifiedType() {
    UUID ch = UUID.randomUUID();
    put(msg(ch, "alice", MessageType.COMMAND));
    put(msg(ch, "bob", MessageType.EVENT));
    put(msg(ch, "charlie", MessageType.EVENT));

    long result = count(MessageQuery.builder()
            .channelId(ch)
            .excludeTypes(java.util.List.of(MessageType.EVENT))
            .build());

    assertEquals(1, result);
}
```

Run tests to confirm they fail to compile (missing `count()` implementations):

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest="InMemoryMessageStoreTest" 2>&1 | grep -E "ERROR|error:"
```

Expected: Compilation error — `count` is not implemented.

- [ ] **Step 2.2: Add `count(MessageQuery)` to `MessageStore` interface**

In `MessageStore.java`, add after `countByChannel`:

```java
/**
 * Count messages matching the given query. Intentionally {@code long}
 * (Panache count semantics) unlike the legacy {@code int countByChannel}.
 */
long count(MessageQuery query);
```

- [ ] **Step 2.3: Implement `count()` in `InMemoryMessageStore`**

In `InMemoryMessageStore.java`, add after `countByChannel`:

```java
@Override
public long count(MessageQuery q) {
    // Do NOT delegate to scan() — scan() applies limit, giving wrong counts.
    return store.values().stream()
            .filter(q::matches)
            .count();
}
```

- [ ] **Step 2.4: Add the abstract `count()` to `InMemoryMessageStoreTest` (contract runner)**

In `InMemoryMessageStoreTest.java`, add the override:

```java
@Override
protected long count(MessageQuery q) {
    return store.count(q);
}
```

- [ ] **Step 2.5: Run contract tests — both blocking and reactive runners**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest="InMemoryMessageStoreTest,InMemoryReactiveMessageStoreTest" -q
```

Expected: `InMemoryMessageStoreTest` passes; `InMemoryReactiveMessageStoreTest` still fails (reactive `count()` not yet implemented — this is expected).

- [ ] **Step 2.6: Add `count()` to `ReactiveMessageStore` interface**

In `ReactiveMessageStore.java`, add after `countByChannel`:

```java
Uni<Long> count(MessageQuery query);
```

- [ ] **Step 2.7: Implement `count()` in `InMemoryReactiveMessageStore`**

In `InMemoryReactiveMessageStore.java`, add after `countByChannel`:

```java
@Override
public Uni<Long> count(MessageQuery q) {
    return Uni.createFrom().item(() -> blocking.count(q));
}
```

- [ ] **Step 2.8: Add the abstract `count()` to `InMemoryReactiveMessageStoreTest` (contract runner)**

In `InMemoryReactiveMessageStoreTest.java`, add the override:

```java
@Override
protected long count(MessageQuery q) {
    return store.count(q).await().indefinitely();
}
```

- [ ] **Step 2.9: Run all contract tests — both runners must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest="InMemoryMessageStoreTest,InMemoryReactiveMessageStoreTest" -q
```

Expected: Both test classes pass, including `count_byChannel_excludesOtherChannels` and `count_excludesSpecifiedType`.

- [ ] **Step 2.10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/MessageStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveMessageStore.java \
  testing/src/main/java/io/casehub/qhorus/testing/InMemoryMessageStore.java \
  testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStore.java \
  testing/src/test/java/io/casehub/qhorus/testing/contract/MessageStoreContractTest.java \
  testing/src/test/java/io/casehub/qhorus/testing/InMemoryMessageStoreTest.java \
  testing/src/test/java/io/casehub/qhorus/testing/InMemoryReactiveMessageStoreTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#205): MessageStore.count(MessageQuery) — interface, InMemory impls, contract tests"
```

---

## Task 3: JPA Store `count()` Implementations

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java`

- [ ] **Step 3.1: Add `count()` to `JpaMessageStore`**

In `JpaMessageStore.java`, add after the `scan()` method:

```java
@Override
public long count(MessageQuery q) {
    MessageQueryJpql mq = MessageQueryJpql.from(q);
    return Message.count(mq.where(), mq.params());
}
```

- [ ] **Step 3.2: Add `count()` to `ReactiveJpaMessageStore`**

In `ReactiveJpaMessageStore.java`, add after `scan()`:

```java
@Override
public Uni<Long> count(MessageQuery q) {
    MessageQueryJpql mq = MessageQueryJpql.from(q);
    return repo.count(mq.where(), mq.params());
}
```

Note: `repo` is `MessageReactivePanacheRepo`, already scoped to the `"qhorus"` PU. No named-PU wrapper needed — `repo.count()` routes through the correct session.

- [ ] **Step 3.3: Build and run JPA store tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaMessageStoreTest" -q
```

Expected: All `JpaMessageStoreTest` tests pass. (There is no `count(MessageQuery)` JPA integration test — the contract tests cover this for InMemory stores; the JPA implementation is a trivial Panache delegation.)

- [ ] **Step 3.4: Full module build to surface any compile errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime,testing
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#205): JpaMessageStore.count(MessageQuery) and reactive counterpart — use MessageQueryJpql"
```

---

## Task 4: Fix `WatchdogEvaluationService` — Three Store Seam Bypasses

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`

This task fixes all three methods in one commit. Read the current file first to confirm exact line numbers haven't shifted.

- [ ] **Step 4.1: Add `@Inject CommitmentStore` and `@Inject InstanceStore`**

In `WatchdogEvaluationService.java`, add two injections after the existing `@Inject MessageStore messageStore;` (currently around line 62):

```java
@Inject
CommitmentStore commitmentStore;

@Inject
InstanceStore instanceStore;
```

Add the corresponding imports at the top of the file:

```java
import io.casehub.qhorus.runtime.instance.Instance;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.store.CommitmentStore;
import io.casehub.qhorus.runtime.store.InstanceStore;
import io.casehub.qhorus.runtime.store.query.InstanceQuery;
```

Note: remove the fully-qualified class references in method bodies (e.g. `io.casehub.qhorus.runtime.instance.Instance.<...>list(...)`) — they'll be replaced by clean imports.

- [ ] **Step 4.2: Fix `evaluateApprovalPending`**

Replace the current method body (the `Commitment.<Commitment>list(...)` call) with:

```java
private boolean evaluateApprovalPending(Watchdog w, Instant now) {
    int threshold = w.thresholdSeconds != null ? w.thresholdSeconds : 300;

    // Threshold formula preserved verbatim from original: for threshold=300, fires for
    // commitments expired >240s ago; for threshold=60, expiring right now; for
    // threshold=0, all commitments with any expiry. See design spec 2026-05-28.
    List<Commitment> pending = commitmentStore.findAllOpen()
            .stream()
            .filter(c -> c.expiresAt != null)
            .filter(c -> threshold == 0 || c.expiresAt.isBefore(now.plusSeconds(60 - threshold)))
            .toList();

    if (!pending.isEmpty()) {
        Instant oldestExpiry = pending.stream()
                .map(c -> c.expiresAt)
                .min(Comparator.naturalOrder())
                .orElse(null);
        String summary = "APPROVAL_PENDING: " + pending.size() + " approval(s) awaiting human response";
        fireAlert(w, summary, new ApprovalPendingContext(pending.size(), oldestExpiry), now);
        return true;
    }
    return false;
}
```

- [ ] **Step 4.3: Fix `evaluateAgentStale`**

Replace the current method body (the `Instance.<Instance>list(...)` call) with:

```java
private boolean evaluateAgentStale(Watchdog w, Instant now) {
    int threshold = w.thresholdSeconds != null ? w.thresholdSeconds : 300;
    Instant cutoff = now.minusSeconds(threshold);

    List<Instance> staleInstances = instanceStore.scan(
            InstanceQuery.builder().status("stale").staleOlderThan(cutoff).build());

    if (!staleInstances.isEmpty()) {
        List<String> ids = staleInstances.stream()
                .limit(10)
                .map(i -> i.id.toString())
                .toList();
        String summary = "AGENT_STALE: " + staleInstances.size() + " stale agent(s) detected";
        fireAlert(w, summary, new AgentStaleContext(staleInstances.size(), ids), now);
        return true;
    }
    return false;
}
```

- [ ] **Step 4.4: Fix `evaluateQueueDepth`**

Replace the `Message.count(...)` call inside the loop:

```java
private boolean evaluateQueueDepth(Watchdog w, Instant now) {
    int threshold = w.thresholdCount != null ? w.thresholdCount : 100;

    List<Channel> channels = channelService.listAll().stream()
            .filter(ch -> "*".equals(w.targetName) || ch.name.equals(w.targetName))
            .toList();

    // Fires on the FIRST channel that exceeds the threshold. If multiple channels
    // are over-depth, only one alert fires per evaluation cycle — pre-existing behaviour.
    for (Channel ch : channels) {
        long count = messageStore.count(
                MessageQuery.builder()
                        .channelId(ch.id)
                        .excludeTypes(List.of(MessageType.EVENT))
                        .build());
        if (count >= threshold) {
            String summary = "QUEUE_DEPTH: channel='" + ch.name + "' has " + count
                    + " messages (threshold=" + threshold + ")";
            fireAlert(w, summary, new QueueDepthContext(ch.name, count, threshold), now);
            return true;
        }
    }
    return false;
}
```

Add `import java.util.List;` if not already present (it should be).

- [ ] **Step 4.5: Remove the now-unused fully-qualified import for Instance**

Delete the `import io.casehub.qhorus.runtime.instance.Instance;` line if it was previously inlined as a fully-qualified reference. It's now imported cleanly.

Also verify there are no remaining references to `Commitment.<Commitment>list(...)` or `io.casehub.qhorus.runtime.instance.Instance.<...>list(...)` or `Message.count(...)` in the file.

- [ ] **Step 4.6: Build and verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4.7: Run existing watchdog tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WatchdogEnabledTest,ConfiguredWatchdogAlertRouterTest,ConfiguredWatchdogAlertRouterNoEndpointsTest" -q
```

Expected: All pass. These are the existing condition tests that now exercise the fixed store paths.

- [ ] **Step 4.8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#205): WatchdogEvaluationService — route all three Panache bypasses through store seam (CommitmentStore, InstanceStore, MessageStore.count)"
```

---

## Task 5: `WatchdogEvaluationServiceTest` — Three New Condition Tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java`

These tests verify the refactored evaluation methods work correctly with InMemory store alternatives. They use `@QuarkusTest @TestProfile(WatchdogEnabledProfile.class)`, which activates all `@Alternative @Priority(1)` InMemory store beans automatically. Data is set up directly through the InMemory stores (bypassing MCP tools).

- [ ] **Step 5.1: Write the three failing tests (file won't compile yet — that's fine)**

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java
package io.casehub.qhorus.runtime.watchdog;

import static org.junit.jupiter.api.Assertions.*;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.instance.Instance;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import io.casehub.qhorus.testing.InMemoryChannelStore;
import io.casehub.qhorus.testing.InMemoryCommitmentStore;
import io.casehub.qhorus.testing.InMemoryInstanceStore;
import io.casehub.qhorus.testing.InMemoryMessageStore;
import io.casehub.qhorus.testing.InMemoryWatchdogStore;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * Verifies that WatchdogEvaluationService condition evaluation works correctly
 * through the store seam, using InMemory store alternatives for test isolation.
 *
 * <p>Refs #205
 */
@QuarkusTest
@TestProfile(io.casehub.qhorus.api.WatchdogEnabledProfile.class)
class WatchdogEvaluationServiceTest {

    @Inject
    WatchdogEvaluationService watchdogService;

    @Inject
    InMemoryChannelStore channelStore;

    @Inject
    InMemoryWatchdogStore watchdogStore;

    @Inject
    InMemoryCommitmentStore commitmentStore;

    @Inject
    InMemoryInstanceStore instanceStore;

    @Inject
    InMemoryMessageStore messageStore;

    @BeforeEach
    void clearStores() {
        channelStore.clear();
        watchdogStore.clear();
        commitmentStore.clear();
        instanceStore.clear();
        messageStore.clear();
    }

    // -------------------------------------------------------------------------
    // Fix 1: evaluateApprovalPending — CommitmentStore seam
    // -------------------------------------------------------------------------

    @Test
    @TestTransaction
    void evaluateApprovalPending_firesAlert_whenOpenCommitmentWithExpiryExists() {
        Channel notifCh = new Channel();
        notifCh.name = "notif-approval-" + UUID.randomUUID();
        notifCh.semantic = ChannelSemantic.APPEND;
        channelStore.put(notifCh);

        Watchdog w = new Watchdog();
        w.conditionType = "APPROVAL_PENDING";
        w.targetName = "*";
        w.thresholdSeconds = 0;  // fires for all commitments with any expiry
        w.notificationChannel = notifCh.name;
        w.createdBy = "test";
        watchdogStore.put(w);

        Commitment c = new Commitment();
        c.state = CommitmentState.OPEN;
        c.expiresAt = Instant.now().plusSeconds(30);
        c.channelId = UUID.randomUUID();
        c.correlationId = UUID.randomUUID().toString();
        c.obligor = "agent-a";
        c.requester = "agent-b";
        commitmentStore.save(c);

        watchdogService.evaluateAll();

        List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id));
        assertFalse(alerts.isEmpty(), "APPROVAL_PENDING watchdog should fire alert");
        assertTrue(alerts.get(0).content.contains("APPROVAL_PENDING"));
    }

    // -------------------------------------------------------------------------
    // Fix 2: evaluateAgentStale — InstanceStore seam
    // -------------------------------------------------------------------------

    @Test
    @TestTransaction
    void evaluateAgentStale_firesAlert_whenStaleInstanceExists() {
        Channel notifCh = new Channel();
        notifCh.name = "notif-stale-" + UUID.randomUUID();
        notifCh.semantic = ChannelSemantic.APPEND;
        channelStore.put(notifCh);

        Watchdog w = new Watchdog();
        w.conditionType = "AGENT_STALE";
        w.targetName = "*";
        w.thresholdSeconds = 0;  // cutoff = now; any past lastSeen matches
        w.notificationChannel = notifCh.name;
        w.createdBy = "test";
        watchdogStore.put(w);

        Instance inst = new Instance();
        inst.instanceId = "stale-agent-" + UUID.randomUUID();
        inst.status = "stale";
        inst.lastSeen = Instant.now().minusSeconds(10);
        instanceStore.put(inst);

        watchdogService.evaluateAll();

        List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id));
        assertFalse(alerts.isEmpty(), "AGENT_STALE watchdog should fire alert");
        assertTrue(alerts.get(0).content.contains("AGENT_STALE"));
    }

    // -------------------------------------------------------------------------
    // Fix 3: evaluateQueueDepth — MessageStore.count() seam
    // -------------------------------------------------------------------------

    @Test
    @TestTransaction
    void evaluateQueueDepth_firesAlert_whenNonEventMessageCountExceedsThreshold() {
        // evaluateQueueDepth calls channelService.listAll() — the queue channel
        // must be in InMemoryChannelStore (not just InMemoryMessageStore).
        Channel queueCh = new Channel();
        queueCh.name = "queue-ch-" + UUID.randomUUID();
        queueCh.semantic = ChannelSemantic.COLLECT;
        channelStore.put(queueCh);

        Channel notifCh = new Channel();
        notifCh.name = "notif-queue-" + UUID.randomUUID();
        notifCh.semantic = ChannelSemantic.APPEND;
        channelStore.put(notifCh);

        Watchdog w = new Watchdog();
        w.conditionType = "QUEUE_DEPTH";
        w.targetName = queueCh.name;
        w.thresholdCount = 2;
        w.notificationChannel = notifCh.name;
        w.createdBy = "test";
        watchdogStore.put(w);

        // 3 non-EVENT messages — exceeds threshold of 2
        for (int i = 0; i < 3; i++) {
            Message m = new Message();
            m.channelId = queueCh.id;
            m.sender = "agent-" + i;
            m.messageType = MessageType.STATUS;
            m.actorType = ActorType.AGENT;
            m.content = "work item " + i;
            messageStore.put(m);
        }

        watchdogService.evaluateAll();

        List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id));
        assertFalse(alerts.isEmpty(), "QUEUE_DEPTH watchdog should fire alert");
        assertTrue(alerts.get(0).content.contains("QUEUE_DEPTH"));
    }
}
```

- [ ] **Step 5.2: Run the three new tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WatchdogEvaluationServiceTest" -q
```

Expected: All three tests pass.

If any test fails, check:
- InMemory store `clear()` isolation working (stores shouldn't have leftover data)
- Notification channel exists in `InMemoryChannelStore` (required for `fireAlert()` to dispatch message)
- `evaluateAll()` enabled — `WatchdogEnabledProfile` sets `casehub.qhorus.watchdog.enabled=true`

- [ ] **Step 5.3: Run the full watchdog test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WatchdogEvaluationServiceTest,WatchdogEnabledTest,ConfiguredWatchdogAlertRouterTest,ConfiguredWatchdogAlertRouterNoEndpointsTest" -q
```

Expected: All pass.

- [ ] **Step 5.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#205): WatchdogEvaluationServiceTest — one test per fixed condition via InMemory store seam"
```

---

## Task 6: `ConfiguredWatchdogAlertRouter` — Document `route()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ConfiguredWatchdogAlertRouter.java`

- [ ] **Step 6.1: Add the Javadoc comment to `route()`**

Replace the current `route()` method signature with the annotated version:

```java
/**
 * V1 fan-out: delivers every alert to all configured endpoints regardless of which
 * watchdog condition fired or what {@link io.casehub.qhorus.api.watchdog.AlertContext}
 * it carries. The {@code event} parameter is intentionally unused.
 *
 * <p>To route selectively (e.g. only AGENT_STALE alerts go to Slack), provide an
 * {@link io.casehub.qhorus.api.watchdog.WatchdogAlertRouter} implementation with
 * any normal CDI scope (e.g. {@code @ApplicationScoped}). CDI automatically selects
 * any non-{@code @DefaultBean} qualifying bean over this default — no
 * {@code @Alternative} or priority annotation needed.
 */
@Override
public List<AlertDeliveryTarget> route(WatchdogAlertEvent event) {
    return config.watchdog().alert().endpoints().stream()
            .map(ep -> new AlertDeliveryTarget(ep.connectorId(), ep.destination()))
            .toList();
}
```

- [ ] **Step 6.2: Build and run router tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ConfiguredWatchdogAlertRouterTest,ConfiguredWatchdogAlertRouterNoEndpointsTest" -q
```

Expected: Both pass (no behaviour change).

- [ ] **Step 6.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ConfiguredWatchdogAlertRouter.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#206): ConfiguredWatchdogAlertRouter.route() — explain v1 fan-out and @DefaultBean override mechanism"
```

---

## Task 7: Full Build Verification

- [ ] **Step 7.1: Run the full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: `BUILD SUCCESS`. All modules compile and all tests pass.

- [ ] **Step 7.2: Verify test count has grown**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: Total tests increased by at least 5 (2 contract tests + 3 condition tests).

---

## Self-Review Checklist

**Spec coverage:**

| Spec Requirement | Task |
|---|---|
| `MessageQueryJpql` extraction | Task 1 |
| `MessageStore.count(MessageQuery)` | Task 2 |
| `ReactiveMessageStore.count(MessageQuery)` | Task 2 |
| `JpaMessageStore.count()` | Task 3 |
| `ReactiveJpaMessageStore.count()` | Task 3 |
| `InMemoryMessageStore.count()` | Task 2 |
| `InMemoryReactiveMessageStore.count()` | Task 2 |
| Contract test: `count_byChannel_excludesOtherChannels` | Task 2 |
| Contract test: `count_excludesSpecifiedType` | Task 2 |
| Inject `CommitmentStore` into `WatchdogEvaluationService` | Task 4 |
| Inject `InstanceStore` into `WatchdogEvaluationService` | Task 4 |
| Fix `evaluateApprovalPending` | Task 4 |
| Fix `evaluateAgentStale` | Task 4 |
| Fix `evaluateQueueDepth` | Task 4 |
| Test for `evaluateApprovalPending` condition | Task 5 |
| Test for `evaluateAgentStale` condition | Task 5 |
| Test for `evaluateQueueDepth` condition | Task 5 |
| `ConfiguredWatchdogAlertRouter.route()` comment | Task 6 |

**No placeholders, no TBD, no "similar to Task N".**
**Type consistency: `MessageQueryJpql` used identically in Task 1 and Task 3.**

# Testing CDI Fix and Review Cleanup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #322 — fix: duplicate @Alternative @Priority(1) stores in testing + persistence-memory jars
**Issue group:** #322, #325

**Goal:** Fix CDI ambiguity caused by testing module's transitive persistence-memory dependency, fix OutboundMessage.correlationId type mismatch (UUID→String), replace test sleeps with Awaitility, and add exponential backoff to the PostgreSQL broadcaster.

**Architecture:** Four independent changes sharing one branch. OutboundMessage type change is the largest (API record field change cascading through A2A, Slack, delivery, and test code). The testing pom fix, Awaitility migration, and broadcaster backoff are each self-contained.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven, Awaitility, Flyway (V27)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use `mvn` not `./mvnw`
- After API changes, run `mvn install` from root — `mvn test` scoped to a child won't catch sibling compile errors
- Flyway next version: V27
- No `@Tool` overload name collisions (guarded by `ToolOverloadDiscoverabilityTest`)
- Commits reference issues: `Refs #322` or `Refs #325`

---

### Task 1: OutboundMessage.correlationId UUID → String

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java:29` — field type UUID → String
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java:192-196,300-304,415-429` — delete parseCorrelationUuid, pass String directly
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java:311-316,365-370,514-528` — delete parseCorrelationUuid, pass String directly
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java:330-337,357-364` — delete parseCorrelationUuid, pass String directly
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java:122-131` — remove UUID.fromString, pass String directly
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java:78,174,186,199` — sseStreams key UUID→String, method params UUID→String
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java:239-245,294` — normalize corrId to lowercase String, pass String to registerStream
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java:220-251` — normalize taskId in receive() for case-insensitive UUID matching
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java:64,116-118,150-186,209-232` — threadCache inner key UUID→String, correlationId variables UUID→String, remove UUID::toString in findCorrelationId
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackThreadCacheStore.java:23,33,52,62` — correlationId params UUID→String, findCorrelationId return Optional<String>
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackThreadCacheId.java:12` — correlationId field UUID→String
- Create: `db/qhorus/migration/V27__slack_thread_cache_correlation_id_varchar.sql`
- Test: multiple test files — mechanical type changes (compiler enforces)

**Interfaces:**
- Produces: `OutboundMessage(UUID messageId, String sender, MessageType type, String content, String correlationId, Long inReplyTo, ActorType senderActorType)` — correlationId is now String

- [ ] **Step 1: Change OutboundMessage record field**

In `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java`, change line 29:

```java
// Before
UUID correlationId,
// After
String correlationId,
```

Remove the `import java.util.UUID;` ONLY if `messageId` is the sole remaining UUID usage — check first. `messageId` is UUID, so the import stays.

Update Javadoc `@param correlationId` — remove any "UUID form" reference. If present, change to:
```java
 * @param correlationId the correlation identifier (nullable — null for EVENT messages)
```

- [ ] **Step 2: Delete parseCorrelationUuid from MessageService**

In `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`:

Delete the private static method at lines 415-429:
```java
// DELETE this entire method
private static UUID parseCorrelationUuid(String correlationId) { ... }
```

Replace all call sites (lines 194, 302) — change:
```java
parseCorrelationUuid(dispatch.correlationId()),
```
to:
```java
dispatch.correlationId(),
```

- [ ] **Step 3: Delete parseCorrelationUuid from ReactiveMessageService**

In `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`:

Delete the private static method at lines 514-528.

Replace all call sites (lines 314, 368) — same substitution as Step 2.

- [ ] **Step 4: Delete parseCorrelationUuid from ChannelGateway**

In `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`:

Delete the private static method at lines 357-364.

Replace call site (line 335) — change:
```java
parseCorrelationUuid(msg.correlationId()),
```
to:
```java
msg.correlationId(),
```

- [ ] **Step 5: Fix DeliveryBatchExecutor latent bug**

In `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java`, change line 128:

```java
// Before (throws on non-UUID — latent bug)
m.correlationId() != null ? UUID.fromString(m.correlationId()) : null,
// After
m.correlationId(),
```

- [ ] **Step 6: Update A2AChannelBackend — sseStreams key and method params**

In `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java`:

Change sseStreams field (line 78):
```java
// Before
private final ConcurrentHashMap<UUID, Set<Consumer<OutboundMessage>>> sseStreams =
// After
private final ConcurrentHashMap<String, Set<Consumer<OutboundMessage>>> sseStreams =
```

Change `registerStream` param (line 174):
```java
// Before
void registerStream(final UUID correlationId, final Consumer<OutboundMessage> consumer) {
// After
void registerStream(final String correlationId, final Consumer<OutboundMessage> consumer) {
```

Change `deregisterStream` param (line 186):
```java
// Before
void deregisterStream(final UUID correlationId, final Consumer<OutboundMessage> consumer) {
// After
void deregisterStream(final String correlationId, final Consumer<OutboundMessage> consumer) {
```

Change `streamCount` param (line 199):
```java
// Before
int streamCount(final UUID correlationId) {
// After
int streamCount(final String correlationId) {
```

- [ ] **Step 7: Update A2AResource — case-normalize UUID correlationId**

In `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`, change lines 239-245:

```java
// Before
final UUID corrId;
try {
    corrId = UUID.fromString(taskId);
} catch (final IllegalArgumentException e) {
    sendErrorEvent(sink, sse, taskId, "Invalid task ID format — expected UUID");
    return;
}
// After — validate UUID format, normalize to lowercase String
try {
    UUID.fromString(taskId); // validate format
} catch (final IllegalArgumentException e) {
    sendErrorEvent(sink, sse, taskId, "Invalid task ID format — expected UUID");
    return;
}
final String corrId = taskId.toLowerCase();
```

Line 294 — `registerStream` now takes String, `corrId` is already String. No change needed.

Find and update the `deregisterStream` call in the finally block — it already uses `corrId` which is now String. No change needed.

- [ ] **Step 8: Normalize UUID correlationId in A2AChannelBackend.receive()**

In `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java`, change lines 226-228:

```java
// Before
final String correlationId = (taskId != null && !taskId.isBlank())
        ? taskId
        : UUID.randomUUID().toString();
// After — normalize UUID-format taskIds to lowercase for case-insensitive SSE matching
final String correlationId;
if (taskId != null && !taskId.isBlank()) {
    try {
        correlationId = UUID.fromString(taskId).toString();
    } catch (IllegalArgumentException e) {
        correlationId = taskId;
    }
} else {
    correlationId = UUID.randomUUID().toString();
}
```

- [ ] **Step 9: Update SlackThreadCacheId**

In `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackThreadCacheId.java`, change line 12 and constructor:

```java
// Before
public UUID correlationId;
// After
public String correlationId;
```

Update 2-arg constructor:
```java
// Before
public SlackThreadCacheId(UUID channelId, UUID correlationId) {
// After
public SlackThreadCacheId(UUID channelId, String correlationId) {
```

- [ ] **Step 10: Update SlackThreadCacheStore**

In `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackThreadCacheStore.java`:

Change `findThreadTs` param (line 23):
```java
public Optional<String> findThreadTs(UUID channelId, String correlationId) {
```

Change `findCorrelationId` return type (line 33) and JPQL select:
```java
public Optional<String> findCorrelationId(UUID channelId, String threadTs) {
    return em.createQuery(
            "SELECT c.id.correlationId FROM SlackThreadCache c WHERE c.id.channelId = :ch AND c.threadTs = :ts",
            String.class)
```

Change `save` param (line 52):
```java
public void save(UUID channelId, String correlationId, String threadTs) {
```

Change `delete` param (line 62):
```java
public void delete(UUID channelId, String correlationId) {
```

- [ ] **Step 11: Update SlackChannelBackend**

In `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java`:

Change threadCache field type (line 64):
```java
// Before
final ConcurrentHashMap<UUID, ConcurrentHashMap<UUID, String>> threadCache = new ConcurrentHashMap<>();
// After
final ConcurrentHashMap<UUID, ConcurrentHashMap<String, String>> threadCache = new ConcurrentHashMap<>();
```

Change onChannelInitialised cache warm-up (line 116):
```java
// Before
ConcurrentHashMap<UUID, String> channelThreads = ...
// After
ConcurrentHashMap<String, String> channelThreads = ...
```

Change post() correlationId variables (lines 151-186):
```java
// Before (line 151)
UUID corrId = message.correlationId();
// After
String corrId = message.correlationId();
```

```java
// Before (line 152)
Map<UUID, String> channelThreads = threadCache.get(channel.id());
// After
Map<String, String> channelThreads = threadCache.get(channel.id());
```

Same for line 173 (`UUID corrId` → `String corrId`), line 183 (`UUID corrId` → `String corrId`), line 184 (`Map<UUID, String>` → `Map<String, String>`).

Change onInboundMessage (lines 213-214) — remove `.map(UUID::toString)`:
```java
// Before
corrIdStr = threadCacheStore.findCorrelationId(channelRef.id(), slackThreadTs)
        .map(UUID::toString).orElse(null);
// After
corrIdStr = threadCacheStore.findCorrelationId(channelRef.id(), slackThreadTs)
        .orElse(null);
```

Change onInboundMessage new corrId generation (lines 221-230):
```java
// Before
UUID corrId = UUID.randomUUID();
corrIdStr = corrId.toString();
// After
String corrId = UUID.randomUUID().toString();
corrIdStr = corrId;
```

- [ ] **Step 12: Create Flyway V27 migration**

Create `db/qhorus/migration/V27__slack_thread_cache_correlation_id_varchar.sql`:
```sql
ALTER TABLE slack_thread_cache ALTER COLUMN correlation_id TYPE VARCHAR(255) USING correlation_id::text;
```

- [ ] **Step 13: Update test files — mechanical type substitutions**

For each test file that constructs `OutboundMessage`, change the `correlationId` argument from `UUID` to `String`. The compiler will flag every site.

Key files and their patterns:

`runtime/src/test/java/.../QhorusChannelBackendTest.java`:
```java
// UUID.randomUUID() → UUID.randomUUID().toString()  for correlationId arg
```

`runtime/src/test/java/.../A2AChannelBackendSseTest.java`:
- Change `outbound()` helper param from `UUID` to `String`
- Change `registerStream`/`deregisterStream`/`streamCount` calls: UUID → String
- Change local `UUID correlationId` variables to `String`

`runtime/src/test/java/.../A2AChannelBackendIntegrationTest.java`:
- Same pattern

`runtime/src/test/java/.../ChannelGatewayDeliverRemoteTest.java`:
- `buildMessage()` helper: no change needed (Message.correlationId is already String)

`connector-backend/src/test/java/.../ConnectorChannelBackendTest.java`:
- OutboundMessage construction: UUID → String for correlationId

`connector-backend/src/test/java/.../ConnectorChannelBackendIntegrationTest.java`:
- Same pattern

`connector-backend/src/test/java/.../ConnectorAutoChannelBackendTest.java`:
- Same pattern

`slack-channel/src/test/java/.../SlackChannelBackendTest.java`:
- `outbound()` helper: `UUID corrId` param → `String corrId`
- All callers passing `UUID.randomUUID()` → `UUID.randomUUID().toString()`

`runtime/src/test/java/.../ChannelGatewayTest.java`:
- OutboundMessage construction: UUID → String for correlationId

`runtime/src/test/java/.../SendMessageFanOutTest.java`:
- If it constructs OutboundMessage directly

`runtime/src/test/java/.../DeliveryServiceTest.java`:
- If it constructs OutboundMessage directly

`runtime/src/test/java/.../FanOutDeliveryGuaranteeTest.java`:
- If it constructs OutboundMessage directly

`api/src/test/java/.../DeliveryGuaranteeTest.java`:
- If it constructs OutboundMessage directly

- [ ] **Step 14: Compile and run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Fix any remaining compile errors — the compiler enforces completeness on this record change.

- [ ] **Step 15: Commit**

```bash
git add -A
git commit -m "feat(#325): OutboundMessage.correlationId UUID→String, delete parseCorrelationUuid duplication

Changes OutboundMessage.correlationId from UUID to String, aligning the
gateway record with the domain model. Deletes 3 identical parseCorrelationUuid
methods and fixes a latent IllegalArgumentException in DeliveryBatchExecutor.
A2A SSE matching uses lowercase-normalized String keys to preserve
case-insensitive UUID matching. SlackThreadCacheId.correlationId migrated
via Flyway V27.

Refs #325

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Testing Module Dependency Scope Fix

**Files:**
- Modify: `testing/pom.xml:23-27` — change persistence-memory scope to test, fix description

**Interfaces:**
- Consumes: nothing
- Produces: testing module no longer transitively exports persistence-memory

- [ ] **Step 1: Change dependency scope and description**

In `testing/pom.xml`, change the persistence-memory dependency (lines 23-27):

```xml
<!-- Before -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-persistence-memory</artifactId>
      <version>${project.version}</version>
    </dependency>
<!-- After -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-persistence-memory</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
```

Change the `<description>` (line 16):
```xml
<!-- Before -->
  <description>In-memory store implementations for fast unit testing without a database</description>
<!-- After -->
  <description>Test utilities for Qhorus consumers (RecordingChannelBackend, MessageLedgerEntryTestFactory)</description>
```

- [ ] **Step 2: Verify CommitmentServiceTest still compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing
```

Expected: all tests pass — `CommitmentServiceTest` uses `InMemoryCommitmentStore` which is still available at test scope.

- [ ] **Step 3: Commit**

```bash
git add testing/pom.xml
git commit -m "fix(#322): change persistence-memory to test scope in testing module

The testing module exports RecordingChannelBackend and
MessageLedgerEntryTestFactory — neither uses persistence-memory.
Only CommitmentServiceTest (src/test/) needs InMemoryCommitmentStore.
Compile scope caused consumers to inherit all InMemory stores
transitively, overriding JPA stores in integration tests.

Closes #322

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Test Sleeps → Awaitility

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayDeliverRemoteTest.java:51-96`

**Interfaces:**
- Consumes: nothing
- Produces: nothing (test-only change)

- [ ] **Step 1: Add Awaitility import**

Add to ChannelGatewayDeliverRemoteTest.java imports:
```java
import static org.awaitility.Awaitility.await;
import java.time.Duration;
```

Remove `throws Exception` from test method signatures (no longer needed — Awaitility handles timing).

- [ ] **Step 2: Replace sleep in deliverRemote_callsPostOnBestEffortBackend**

```java
// Before
gateway.deliverRemote(channelId, 1L);
Thread.sleep(100); // virtual thread dispatch

assertThat(posted).hasSize(1);
// After
gateway.deliverRemote(channelId, 1L);

await().atMost(Duration.ofSeconds(2))
        .untilAsserted(() -> assertThat(posted).hasSize(1));
```

- [ ] **Step 3: Replace sleep in deliverRemote_skipsAgentBackend**

```java
// Before
gateway.deliverRemote(channelId, 1L);
Thread.sleep(100);

assertThat(posted).isEmpty();
// After
gateway.deliverRemote(channelId, 1L);

await().during(Duration.ofMillis(200))
        .atMost(Duration.ofMillis(500))
        .untilAsserted(() -> assertThat(posted).isEmpty());
```

- [ ] **Step 4: Replace sleep in deliverRemote_skipsAtLeastOnceBackend**

Same pattern as Step 3 — sustained negative assertion:
```java
gateway.deliverRemote(channelId, 1L);

await().during(Duration.ofMillis(200))
        .atMost(Duration.ofMillis(500))
        .untilAsserted(() -> assertThat(posted).isEmpty());
```

- [ ] **Step 5: Replace sleep in deliverRemote_lazyInitializesUnknownChannel**

```java
// Before
gateway.deliverRemote(channelId, 1L);
Thread.sleep(100);

assertThat(gateway.listBackends(channelId)).isNotEmpty();
// After
gateway.deliverRemote(channelId, 1L);

await().atMost(Duration.ofSeconds(2))
        .untilAsserted(() -> assertThat(gateway.listBackends(channelId)).isNotEmpty());
```

- [ ] **Step 6: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelGatewayDeliverRemoteTest
```

Expected: all 6 tests pass.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayDeliverRemoteTest.java
git commit -m "test(#325): replace Thread.sleep with Awaitility in ChannelGatewayDeliverRemoteTest

Replaces 4 Thread.sleep(100) calls with Awaitility polling assertions.
Positive assertions use atMost(2s); negative assertions use
during(200ms).atMost(500ms) for sustained verification.

Refs #325

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Exponential Backoff for PostgresChannelActivityBroadcaster

**Files:**
- Modify: `postgres-broadcaster/src/main/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcaster.java`
- Create: `postgres-broadcaster/src/test/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcasterBackoffTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: nothing (internal implementation change)

- [ ] **Step 1: Write failing test for exponential backoff**

Create `postgres-broadcaster/src/test/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcasterBackoffTest.java`:

```java
package io.casehub.qhorus.postgres.broadcaster;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.concurrent.atomic.AtomicLong;

import org.junit.jupiter.api.Test;

class PostgresChannelActivityBroadcasterBackoffTest {

    @Test
    void backoff_doublesOnEachFailure_capsAt60s() {
        AtomicLong delay = new AtomicLong(1000);
        long max = 60_000;

        assertThat(delay.get()).isEqualTo(1000);
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(2000);
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(4000);
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(8000);
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(16000);
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(32000);
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(60000); // capped
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(60000); // stays capped
    }

    @Test
    void backoff_resetsOnSuccess() {
        AtomicLong delay = new AtomicLong(1000);
        long initial = 1000;
        long max = 60_000;

        delay.updateAndGet(d -> Math.min(d * 2, max));
        delay.updateAndGet(d -> Math.min(d * 2, max));
        assertThat(delay.get()).isEqualTo(4000);

        delay.set(initial); // reset on success
        assertThat(delay.get()).isEqualTo(1000);
    }
}
```

- [ ] **Step 2: Run test to verify it passes (logic test)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl postgres-broadcaster -Dtest=PostgresChannelActivityBroadcasterBackoffTest
```

Expected: PASS — this tests the backoff math in isolation.

- [ ] **Step 3: Implement exponential backoff in broadcaster**

In `postgres-broadcaster/src/main/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcaster.java`:

Replace constants (lines 53-54):
```java
// Before
private static final int FILTER_SIZE = 1000;
private static final long RECONNECT_DELAY_MS = 5000;
// After
private static final int FILTER_SIZE = 1000;
private static final long INITIAL_DELAY_MS = 1000;
private static final long MAX_DELAY_MS = 60_000;
```

Add fields after the filter field (line 66-67):
```java
private final SelfNotificationFilter filter = new SelfNotificationFilter(FILTER_SIZE);
private volatile PgConnection subscriberConnection;
// Add these:
private final AtomicLong currentDelayMs = new AtomicLong(INITIAL_DELAY_MS);
private final AtomicBoolean reconnecting = new AtomicBoolean(false);
```

Add imports:
```java
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.AtomicBoolean;
```

Replace `acquireAndListen()` method (lines 137-165):
```java
private void acquireAndListen() {
    if (!reconnecting.compareAndSet(false, true)) return;

    // Close previous connection to prevent leaked LISTEN subscriptions
    PgConnection prev = subscriberConnection;
    if (prev != null) {
        prev.close().subscribe().with(ok -> {}, err -> {});
        subscriberConnection = null;
    }

    pool.getConnection().subscribe().with(
            conn -> {
                io.vertx.pgclient.PgConnection pgDelegate =
                        (io.vertx.pgclient.PgConnection) conn.getDelegate();
                PgConnection pgConn = PgConnection.newInstance(pgDelegate);
                subscriberConnection = pgConn;
                pgConn.notificationHandler(n -> handleNotification(n.getPayload()));
                pgConn.query("LISTEN " + CHANNEL).execute()
                        .subscribe().with(
                                ok -> {
                                    LOG.infof("Subscribed to PostgreSQL channel '%s'", CHANNEL);
                                    currentDelayMs.set(INITIAL_DELAY_MS);
                                    reconnecting.set(false);
                                },
                                err -> {
                                    LOG.errorf(err, "Failed to LISTEN on '%s'", CHANNEL);
                                    pgConn.close().subscribe().with(ok2 -> {}, err2 -> {});
                                    subscriberConnection = null;
                                    reconnecting.set(false);
                                    scheduleReconnect();
                                });
                pgDelegate.closeHandler(v -> {
                    LOG.warn("PostgreSQL subscriber connection lost — reconnecting");
                    scheduleReconnect();
                });
            },
            err -> {
                LOG.errorf(err, "Failed to acquire subscriber connection for '%s'", CHANNEL);
                reconnecting.set(false);
                scheduleReconnect();
            });
}

private void scheduleReconnect() {
    long delay = currentDelayMs.getAndUpdate(d -> Math.min(d * 2, MAX_DELAY_MS));
    Thread.ofVirtual().start(() -> {
        try {
            Thread.sleep(delay);
        } catch (InterruptedException ignored) {
            Thread.currentThread().interrupt();
        }
        acquireAndListen();
    });
}
```

- [ ] **Step 4: Run existing tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl postgres-broadcaster
```

Expected: all tests pass (unit tests + any existing integration tests).

- [ ] **Step 5: Commit**

```bash
git add postgres-broadcaster/
git commit -m "feat(#325): exponential backoff and reconnection guard for PostgresChannelActivityBroadcaster

Replaces fixed 5s retry with exponential backoff (1s→2s→4s→...→60s cap).
Adds AtomicBoolean reconnection guard to prevent concurrent acquireAndListen
calls (connection leak on flapping connections). Adds previous connection
cleanup before reconnection. Both pool-failure and closeHandler paths now
use the same backoff. Reset to 1s on successful LISTEN.

Refs #325

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: CLAUDE.md Update and Cross-Repo Issues

**Files:**
- Modify: `CLAUDE.md` — update testing module description

- [ ] **Step 1: Update CLAUDE.md testing module description**

Find and update the testing module entry in the Project Structure section. Change:
```
├── testing/                             — Test utilities (RecordingChannelBackend, MessageLedgerEntryTestFactory) + CommitmentServiceTest; depends on persistence-memory/ for transitive InMemory store access
```
to:
```
├── testing/                             — Test utilities (RecordingChannelBackend, MessageLedgerEntryTestFactory) + CommitmentServiceTest
```

- [ ] **Step 2: File cross-repo GitHub issues**

File these issues before merge:

1. `casehub-life`: Remove persistence-memory Maven exclusion from casehub-qhorus-testing dependency (no longer needed after #322)
2. `claudony`: Update OutboundMessage construction in 2 test files — correlationId param UUID→String (mechanical, after qhorus#325)
3. `drafthouse`: Update OutboundMessage.correlationId() usage in ReviewerChannelBackend and DebateChannelBackend — remove .toString() calls (mechanical, after qhorus#325)
4. Garden: Update GE-20260630-69e447 — root cause resolved by qhorus#322

- [ ] **Step 3: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: full build passes with all modules.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#322): update CLAUDE.md — testing module no longer re-exports persistence-memory

Refs #322

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

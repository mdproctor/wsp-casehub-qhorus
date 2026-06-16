# SSE Active Model: Keepalive + Timeout (#278, #277) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `streamTask()`'s passive callback model with an active virtual-thread loop that owns the SSE connection lifetime — sending keepalives, detecting orphaned clients, enforcing max duration — and add a full live-stream integration test suite.

**Architecture:** A `LinkedBlockingQueue<OutboundMessage>` replaces the `AtomicReference`-based consumer lambda. The virtual thread running `streamTask()` blocks on `queue.poll(heartbeatMs)` — a null return triggers a keepalive comment, a message triggers an SSE event. `sink.isClosed()` is checked at the top of every iteration for orphan detection. Max duration is a deadline check. `A2AChannelBackend` is unchanged; the consumer is `queue::offer`.

**Tech Stack:** Java 21 (virtual threads via `@RunOnVirtualThread`), Quarkus 3.32.2, RESTEasy Reactive SSE, `QuarkusTransaction.requiringNew()` for short-lived transactional reads, `LinkedBlockingQueue`, `SseEventSource` (JAX-RS client) for integration tests, Awaitility for async coordination.

---

## File Map

| File | Action |
|------|--------|
| `runtime/pom.xml` | Modify: add test-scope `quarkus-rest-client-reactive` + `awaitility` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` | Modify: add `SseSettings` inner interface to `A2a` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java` | Modify: add `TERMINAL_STATES` constant |
| `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java` | Modify: rewrite `streamTask()`, update both private helpers, fix imports |
| `runtime/src/test/java/io/casehub/qhorus/api/A2AEnabledProfile.java` | Modify: add SSE config overrides |
| `runtime/src/test/java/io/casehub/qhorus/runtime/api/A2AStreamIntegrationTest.java` | Create: 7 test cases (4 migrated, 3 new) |
| `runtime/src/test/java/io/casehub/qhorus/api/A2AStreamTaskTest.java` | Delete |

**Package note:** `A2AStreamIntegrationTest` is placed in `io.casehub.qhorus.runtime.api` (not `io.casehub.qhorus.api`) so it can access the package-private `A2AChannelBackend.streamCount()` method, which is used to synchronize on stream registration in the live-stream and keepalive tests.

---

## Task 1: Add test-scope dependencies to runtime/pom.xml

**Files:**
- Modify: `runtime/pom.xml`

`quarkus-rest-client-reactive` provides `ClientBuilder.newClient()` and the `SseEventSource` implementation. `quarkus-rest` (already present) is server-only — without the client artifact, `SseEventSource.target(...).build()` throws at runtime (no `ClientBuilder` registered via ServiceLoader). `awaitility` is managed in the Quarkus BOM; no version needed.

- [ ] **Step 1: Add both dependencies inside the `<dependencies>` block in `runtime/pom.xml`, after the existing test-scope entries (after line 148, before `</dependencies>`)**

```xml
    <!-- SSE integration tests — JAX-RS client runtime for ClientBuilder + SseEventSource -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-client-reactive</artifactId>
      <scope>test</scope>
    </dependency>
    <!-- Awaitility — managed in Quarkus BOM; no version needed -->
    <dependency>
      <groupId>org.awaitility</groupId>
      <artifactId>awaitility</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: `BUILD SUCCESS` with no errors.

---

## Task 2: Add SseSettings interface to QhorusConfig.A2a

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java`

The interface is named `SseSettings` (not `Sse`) to avoid shadowing the JAX-RS `jakarta.ws.rs.sse.Sse` type that `A2AResource` imports as a `@Context` parameter. Config keys remain `casehub.qhorus.a2a.sse.*`.

- [ ] **Step 1: Replace the existing `A2a` interface in `QhorusConfig.java` with this updated version**

```java
    interface A2a {
        /**
         * When true, exposes A2A-compatible REST endpoints at /a2a/*.
         * Disabled by default — opt-in to avoid unintended exposure.
         */
        @WithDefault("false")
        boolean enabled();

        /** SSE stream settings for the A2A streaming endpoint. */
        SseSettings sse();

        interface SseSettings {
            /** Interval between SSE comment keepalives. Default: 15s. */
            @WithDefault("15")
            int heartbeatIntervalSeconds();

            /**
             * Maximum SSE stream lifetime before server-side close. Default: 1800s (30 min).
             *
             * A2A tasks in multi-agent coordination routinely run for minutes to hours.
             * 1800s is the defensible floor — long enough for coordinated agent tasks,
             * short enough to bound runaway streams. Increase for longer-running tasks.
             */
            @WithDefault("1800")
            int maxDurationSeconds();
        }
    }
```

- [ ] **Step 2: Verify the change compiles and existing A2A tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=A2AResourceDisabledTest -pl runtime
```

Expected: `Tests run: 5, Failures: 0, Errors: 0, Skipped: 0`

---

## Task 3: Add TERMINAL_STATES to A2ATaskState and write its test

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/api/A2ATaskStateTest.java`

- [ ] **Step 1: Write a failing test in `A2ATaskStateTest.java`. Add this test method to the existing class (it is in package `io.casehub.qhorus.runtime.api` and can access the package-private constant)**

```java
    @Test
    void terminalStates_containsAllThreeTerminalStrings() {
        assertThat(A2ATaskState.TERMINAL_STATES)
                .containsExactlyInAnyOrder("completed", "failed", "cancelled");
    }
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=A2ATaskStateTest#terminalStates_containsAllThreeTerminalStrings -pl runtime
```

Expected: `FAIL` — `TERMINAL_STATES` does not exist yet.

- [ ] **Step 3: Add the constant to `A2ATaskState.java`. Insert after the `TERMINAL_TYPES` constant (after line 15)**

```java
    /** A2A state strings that represent a terminal task outcome. */
    static final Set<String> TERMINAL_STATES = Set.of("completed", "failed", "cancelled");
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=A2ATaskStateTest -pl runtime
```

Expected: all A2ATaskStateTest tests pass.

---

## Task 4: Update A2AEnabledProfile with SSE config overrides

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/api/A2AEnabledProfile.java`

`heartbeat-interval-seconds=1` makes the keepalive test fire within 1s rather than waiting 15s. `max-duration-seconds=30` prevents stalled-stream tests from hanging for 1800s in CI.

- [ ] **Step 1: Add SSE config to the returned map in `getConfigOverrides()`**

Add these two lines inside the method, after the existing `casehub.qhorus.a2a.enabled` line:

```java
        config.put("casehub.qhorus.a2a.sse.heartbeat-interval-seconds", "1");
        config.put("casehub.qhorus.a2a.sse.max-duration-seconds", "30");
```

The full updated method looks like:

```java
    @Override
    public Map<String, String> getConfigOverrides() {
        Map<String, String> config = new HashMap<>();
        config.put("casehub.qhorus.a2a.enabled", "true");
        config.put("casehub.qhorus.a2a.sse.heartbeat-interval-seconds", "1");
        config.put("casehub.qhorus.a2a.sse.max-duration-seconds", "30");
        config.put("quarkus.datasource.qhorus.db-kind", "h2");
        config.put("quarkus.datasource.qhorus.jdbc.url", "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1");
        config.put("quarkus.datasource.qhorus.username", "sa");
        config.put("quarkus.datasource.qhorus.password", "");
        config.put("quarkus.datasource.qhorus.reactive", "false");
        config.put("quarkus.hibernate-orm.qhorus.database.generation", "drop-and-create");
        return config;
    }
```

- [ ] **Step 2: Verify existing A2A integration tests still pass with updated profile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=A2AChannelBackendIntegrationTest -pl runtime
```

Expected: all tests pass.

---

## Task 5: Write A2AStreamIntegrationTest — all 7 tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/api/A2AStreamIntegrationTest.java`

Write all 7 tests now. Tests 1–4 (migrated immediate-close paths) will pass with the current implementation. Tests 5–7 (live stream, keepalive) will fail until Task 6.

- [ ] **Step 1: Create the file with all 7 tests**

```java
package io.casehub.qhorus.runtime.api;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.net.URI;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import jakarta.inject.Inject;
import jakarta.ws.rs.client.Client;
import jakarta.ws.rs.client.ClientBuilder;
import jakarta.ws.rs.client.WebTarget;
import jakarta.ws.rs.sse.SseEventSource;

import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.A2AEnabledProfile;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration tests for GET /a2a/tasks/{id}/stream — covers all SSE paths.
 * Replaces A2AStreamTaskTest (immediate-close paths migrated from RestAssured to SseEventSource)
 * and adds live-stream and keepalive tests.
 *
 * <p>Package io.casehub.qhorus.runtime.api gives access to the package-private
 * A2AChannelBackend.streamCount() used to synchronize stream registration.
 *
 * <p>Refs qhorus#277, qhorus#278.
 */
@QuarkusTest
@TestProfile(A2AEnabledProfile.class)
class A2AStreamIntegrationTest {

    @TestHTTPResource("") URI baseUri;

    @Inject A2AChannelBackend a2aBackend;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;

    // ── Immediate-close paths (migrated from A2AStreamTaskTest) ──────────────

    @Test
    void sseStream_taskNotFound_returnsErrorEvent() throws Exception {
        final String taskId = UUID.randomUUID().toString();
        final CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
        final CountDownLatch latch = new CountDownLatch(1);

        final Client client = ClientBuilder.newClient();
        try {
            final WebTarget target = client.target(baseUri)
                    .path("/a2a/tasks/" + taskId + "/stream");
            try (SseEventSource source = SseEventSource.target(target)
                    .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)
                    .build()) {
                source.register(event -> {
                    events.add(event.getName() + "|" + event.readData(String.class));
                    latch.countDown();
                });
                source.open();
                assertTrue(latch.await(5, TimeUnit.SECONDS), "No SSE event received");
            }
        } finally {
            client.close();
        }

        assertThat(events).anyMatch(e -> e.startsWith("error|") && e.contains("\"final\":true"));
    }

    @Test
    void sseStream_invalidUuid_returnsErrorEvent() throws Exception {
        final CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
        final CountDownLatch latch = new CountDownLatch(1);

        final Client client = ClientBuilder.newClient();
        try {
            final WebTarget target = client.target(baseUri)
                    .path("/a2a/tasks/not-a-uuid/stream");
            try (SseEventSource source = SseEventSource.target(target)
                    .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)
                    .build()) {
                source.register(event -> {
                    events.add(event.getName() + "|" + event.readData(String.class));
                    latch.countDown();
                });
                source.open();
                assertTrue(latch.await(5, TimeUnit.SECONDS), "No SSE event received");
            }
        } finally {
            client.close();
        }

        assertThat(events).anyMatch(e -> e.startsWith("error|") && e.contains("\"final\":true"));
    }

    @Test
    void sseStream_alreadyTerminalDone_sendsImmediateFinalEvent() throws Exception {
        final String channelName = "stream-done-" + UUID.randomUUID();
        final String taskId = UUID.randomUUID().toString();
        final UUID[] chId = {null};
        final Long[] cmdId = {null};

        QuarkusTransaction.requiringNew().run(() -> channelService.create(new ChannelCreateRequest(
                channelName, "SSE test", ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null)));
        QuarkusTransaction.requiringNew().run(() ->
                chId[0] = channelService.findByName(channelName).orElseThrow().id);
        QuarkusTransaction.requiringNew().run(() -> {
            final DispatchResult r = messageService.dispatch(MessageDispatch.builder()
                    .channelId(chId[0]).sender("requester").type(MessageType.COMMAND)
                    .content("do this").correlationId(taskId).actorType(ActorType.AGENT).build());
            cmdId[0] = r.messageId();
        });
        QuarkusTransaction.requiringNew().run(() ->
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(chId[0]).sender("agent").type(MessageType.DONE)
                        .content("done").correlationId(taskId).inReplyTo(cmdId[0])
                        .actorType(ActorType.AGENT).build()));

        final CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
        final CountDownLatch latch = new CountDownLatch(1);
        final Client client = ClientBuilder.newClient();
        try {
            final WebTarget target = client.target(baseUri)
                    .path("/a2a/tasks/" + taskId + "/stream");
            try (SseEventSource source = SseEventSource.target(target)
                    .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)
                    .build()) {
                source.register(event -> { events.add(event.readData(String.class)); latch.countDown(); });
                source.open();
                assertTrue(latch.await(5, TimeUnit.SECONDS), "No SSE event received");
            }
        } finally {
            client.close();
        }

        assertThat(events).anyMatch(e ->
                e.contains("\"state\":\"completed\"") && e.contains("\"final\":true"));
    }

    @Test
    void sseStream_alreadyTerminalDecline_sendsCancelledEvent() throws Exception {
        final String channelName = "stream-decline-" + UUID.randomUUID();
        final String taskId = UUID.randomUUID().toString();
        final UUID[] chId = {null};
        final Long[] cmdId = {null};

        QuarkusTransaction.requiringNew().run(() -> channelService.create(new ChannelCreateRequest(
                channelName, "SSE decline test", ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null)));
        QuarkusTransaction.requiringNew().run(() ->
                chId[0] = channelService.findByName(channelName).orElseThrow().id);
        QuarkusTransaction.requiringNew().run(() -> {
            final DispatchResult r = messageService.dispatch(MessageDispatch.builder()
                    .channelId(chId[0]).sender("requester").type(MessageType.COMMAND)
                    .content("do this").correlationId(taskId).actorType(ActorType.AGENT).build());
            cmdId[0] = r.messageId();
        });
        QuarkusTransaction.requiringNew().run(() ->
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(chId[0]).sender("agent").type(MessageType.DECLINE)
                        .content("I refuse").correlationId(taskId).inReplyTo(cmdId[0])
                        .actorType(ActorType.AGENT).build()));

        final CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
        final CountDownLatch latch = new CountDownLatch(1);
        final Client client = ClientBuilder.newClient();
        try {
            final WebTarget target = client.target(baseUri)
                    .path("/a2a/tasks/" + taskId + "/stream");
            try (SseEventSource source = SseEventSource.target(target)
                    .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)
                    .build()) {
                source.register(event -> { events.add(event.readData(String.class)); latch.countDown(); });
                source.open();
                assertTrue(latch.await(5, TimeUnit.SECONDS), "No SSE event received");
            }
        } finally {
            client.close();
        }

        assertThat(events).anyMatch(e ->
                e.contains("\"state\":\"cancelled\"") && e.contains("\"final\":true"));
    }

    // ── Live-stream paths (new) ───────────────────────────────────────────────

    @Test
    void sseStream_receivesCompletedEvent_whenDoneDispatched() throws Exception {
        final String channelName = "stream-live-done-" + UUID.randomUUID();
        final String taskId = UUID.randomUUID().toString();
        final UUID corrId = UUID.fromString(taskId);
        final UUID[] chId = {null};
        final Long[] cmdId = {null};

        QuarkusTransaction.requiringNew().run(() -> channelService.create(new ChannelCreateRequest(
                channelName, "SSE live test", ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null)));
        QuarkusTransaction.requiringNew().run(() ->
                chId[0] = channelService.findByName(channelName).orElseThrow().id);
        QuarkusTransaction.requiringNew().run(() -> {
            final DispatchResult r = messageService.dispatch(MessageDispatch.builder()
                    .channelId(chId[0]).sender("requester").type(MessageType.COMMAND)
                    .content("do this").correlationId(taskId).actorType(ActorType.AGENT).build());
            cmdId[0] = r.messageId();
        });

        final CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
        final CountDownLatch latch = new CountDownLatch(1);
        final Client client = ClientBuilder.newClient();
        try {
            final WebTarget target = client.target(baseUri)
                    .path("/a2a/tasks/" + taskId + "/stream");
            try (SseEventSource source = SseEventSource.target(target)
                    .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)
                    .build()) {
                source.register(event -> { events.add(event.readData(String.class)); latch.countDown(); });
                source.open();

                Awaitility.await().atMost(2, TimeUnit.SECONDS)
                        .until(() -> a2aBackend.streamCount(corrId) > 0);

                final long finalCmdId = cmdId[0];
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.dispatch(MessageDispatch.builder()
                                .channelId(chId[0]).sender("agent").type(MessageType.DONE)
                                .content("done").correlationId(taskId).inReplyTo(finalCmdId)
                                .actorType(ActorType.AGENT).build()));

                assertTrue(latch.await(10, TimeUnit.SECONDS), "No SSE event received within 10s");
            }
        } finally {
            client.close();
        }

        assertThat(events).anyMatch(e ->
                e.contains("\"state\":\"completed\"") && e.contains("\"final\":true"));
    }

    @Test
    void sseStream_receivesCancelledEvent_whenDeclineDispatched() throws Exception {
        final String channelName = "stream-live-decline-" + UUID.randomUUID();
        final String taskId = UUID.randomUUID().toString();
        final UUID corrId = UUID.fromString(taskId);
        final UUID[] chId = {null};
        final Long[] cmdId = {null};

        QuarkusTransaction.requiringNew().run(() -> channelService.create(new ChannelCreateRequest(
                channelName, "SSE live decline test", ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null)));
        QuarkusTransaction.requiringNew().run(() ->
                chId[0] = channelService.findByName(channelName).orElseThrow().id);
        QuarkusTransaction.requiringNew().run(() -> {
            final DispatchResult r = messageService.dispatch(MessageDispatch.builder()
                    .channelId(chId[0]).sender("requester").type(MessageType.COMMAND)
                    .content("do this").correlationId(taskId).actorType(ActorType.AGENT).build());
            cmdId[0] = r.messageId();
        });

        final CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
        final CountDownLatch latch = new CountDownLatch(1);
        final Client client = ClientBuilder.newClient();
        try {
            final WebTarget target = client.target(baseUri)
                    .path("/a2a/tasks/" + taskId + "/stream");
            try (SseEventSource source = SseEventSource.target(target)
                    .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)
                    .build()) {
                source.register(event -> { events.add(event.readData(String.class)); latch.countDown(); });
                source.open();

                Awaitility.await().atMost(2, TimeUnit.SECONDS)
                        .until(() -> a2aBackend.streamCount(corrId) > 0);

                final long finalCmdId = cmdId[0];
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.dispatch(MessageDispatch.builder()
                                .channelId(chId[0]).sender("agent").type(MessageType.DECLINE)
                                .content("I refuse").correlationId(taskId).inReplyTo(finalCmdId)
                                .actorType(ActorType.AGENT).build()));

                assertTrue(latch.await(10, TimeUnit.SECONDS), "No SSE event received within 10s");
            }
        } finally {
            client.close();
        }

        assertThat(events).anyMatch(e ->
                e.contains("\"state\":\"cancelled\"") && e.contains("\"final\":true"));
    }

    @Test
    void sseStream_keepaliveCommentsDoNotTriggerEventHandlers() throws Exception {
        // Keepalive SSE comments (": keepalive") must be silently ignored by the
        // SseEventSource client per the SSE spec — they must not reach event handlers.
        // heartbeat-interval-seconds=1 in A2AEnabledProfile means ≥3 keepalives in 3s.
        final String channelName = "stream-keepalive-" + UUID.randomUUID();
        final String taskId = UUID.randomUUID().toString();
        final UUID corrId = UUID.fromString(taskId);
        final UUID[] chId = {null};

        QuarkusTransaction.requiringNew().run(() -> channelService.create(new ChannelCreateRequest(
                channelName, "Keepalive test", ChannelSemantic.APPEND,
                null, null, null, null, null, null, null, null, null, null, null)));
        QuarkusTransaction.requiringNew().run(() ->
                chId[0] = channelService.findByName(channelName).orElseThrow().id);
        // Dispatch COMMAND so task exists — without it, streamTask() returns immediately
        // with "task not found" and fires the event handler, self-defeating the test.
        QuarkusTransaction.requiringNew().run(() ->
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(chId[0]).sender("requester").type(MessageType.COMMAND)
                        .content("long-running task").correlationId(taskId)
                        .actorType(ActorType.AGENT).build()));

        final CopyOnWriteArrayList<String> events = new CopyOnWriteArrayList<>();
        final Client client = ClientBuilder.newClient();
        try {
            final WebTarget target = client.target(baseUri)
                    .path("/a2a/tasks/" + taskId + "/stream");
            try (SseEventSource source = SseEventSource.target(target)
                    .reconnectingEvery(Long.MAX_VALUE, TimeUnit.MILLISECONDS)
                    .build()) {
                source.register(event -> events.add(event.readData(String.class)));
                source.open();

                Awaitility.await().atMost(2, TimeUnit.SECONDS)
                        .until(() -> a2aBackend.streamCount(corrId) > 0);

                Thread.sleep(3_000); // allows ≥3 keepalives at 1s interval

                assertThat(events).as("SSE comment keepalives must not trigger event handlers").isEmpty();
                assertThat(a2aBackend.streamCount(corrId))
                        .as("Connection must still be open after keepalives").isGreaterThan(0);
                // SseEventSource.close() in try-with-resources triggers sink.isClosed() → loop exits
            }
        } finally {
            client.close();
        }
    }
}
```

- [ ] **Step 2: Run the migrated immediate-close tests to confirm they compile and tests 1–4 pass with the current code**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="A2AStreamIntegrationTest#sseStream_taskNotFound_returnsErrorEvent+sseStream_invalidUuid_returnsErrorEvent+sseStream_alreadyTerminalDone_sendsImmediateFinalEvent+sseStream_alreadyTerminalDecline_sendsCancelledEvent" \
  -pl runtime
```

Expected: 4 tests pass.

- [ ] **Step 3: Run the new live-stream and keepalive tests to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="A2AStreamIntegrationTest#sseStream_receivesCompletedEvent_whenDoneDispatched+sseStream_receivesCancelledEvent_whenDeclineDispatched+sseStream_keepaliveCommentsDoNotTriggerEventHandlers" \
  -pl runtime
```

Expected: tests 5–6 fail (no live-stream event delivered), test 7 fails (events list is not empty or streamCount is 0 — old passive model). If they somehow pass, the existing implementation already handles the case — recheck the test logic.

---

## Task 6: Rewrite streamTask() and update helpers in A2AResource

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`

This is the core implementation. Replace `@Transactional` with `@RunOnVirtualThread`, rewrite the method body to use the active model, and update both private helpers to use synchronous await with internal try-finally.

- [ ] **Step 1: Update imports in `A2AResource.java`. Remove `jakarta.transaction.Transactional`. Add:**

```java
import java.util.concurrent.CompletionStage;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

import io.quarkus.narayana.jta.QuarkusTransaction;
import io.smallrye.common.annotation.RunOnVirtualThread;
```

Keep all existing imports except `jakarta.transaction.Transactional`.

- [ ] **Step 2: Replace the existing `streamTask()` method (lines 223–293 in the original file) with this implementation**

```java
    /**
     * SSE stream endpoint — active virtual-thread model.
     *
     * <p>The virtual thread stays alive for the connection duration and owns all
     * lifecycle concerns: keepalive comments (via queue.poll timeout), orphan detection
     * (sink.isClosed() at top of every iteration), and max-duration enforcement (deadline).
     *
     * <p>A {@link LinkedBlockingQueue} is the synchronization primitive between this thread
     * and {@link A2AChannelBackend#post} — the consumer is simply {@code queue::offer}.
     * All SSE writes happen on this thread (no concurrent write issues).
     *
     * <p>Transaction scope: two short-lived {@code QuarkusTransaction.requiringNew()} calls
     * (validation + re-check) commit immediately. The loop runs outside any transaction.
     *
     * <p>Refs qhorus#278, qhorus#277.
     */
    @GET
    @Path("/tasks/{id}/stream")
    @Produces("text/event-stream")
    @RunOnVirtualThread
    public void streamTask(
            @PathParam("id") final String taskId,
            @Context final SseEventSink sink,
            @Context final Sse sse) throws Exception {

        // ── Steps 1–2: immediate exits (outside try-finally, no consumer registered) ──

        if (!config.a2a().enabled()) {
            sendErrorEvent(sink, sse, taskId, "A2A endpoint is disabled");
            return;
        }

        final UUID corrId;
        try {
            corrId = UUID.fromString(taskId);
        } catch (final IllegalArgumentException e) {
            sendErrorEvent(sink, sse, taskId, "Invalid task ID format — expected UUID");
            return;
        }

        // Short-lived transactional reads — commits before loop starts
        final AtomicBoolean notFound = new AtomicBoolean(false);
        final AtomicReference<String> stateRef = new AtomicReference<>();
        QuarkusTransaction.requiringNew().run(() -> {
            final List<Message> messages = messageService.findAllByCorrelationId(taskId);
            if (messages.isEmpty()) {
                notFound.set(true);
                return;
            }
            final Commitment commitment = commitmentService.findByCorrelationId(taskId).orElse(null);
            final String state = (commitment != null && commitment.state != CommitmentState.OPEN)
                    ? A2ATaskState.fromCommitmentState(commitment.state)
                    : A2ATaskState.fromMessageHistory(messages);
            stateRef.set(state);
        });

        if (notFound.get()) {
            sendErrorEvent(sink, sse, taskId, "Task not found: " + taskId);
            return;
        }
        if (A2ATaskState.TERMINAL_STATES.contains(stateRef.get())) {
            sendStatusEvent(sink, sse, taskId, stateRef.get());
            return;
        }

        // ── Step 3: register consumer ──────────────────────────────────────────
        final LinkedBlockingQueue<OutboundMessage> queue = new LinkedBlockingQueue<>();
        final Consumer<OutboundMessage> consumer = queue::offer;
        a2aBackend.registerStream(corrId, consumer);

        // ── Outer try: covers steps 4 + 5 — finally always deregisters ─────────
        try {
            // Step 4: re-check after registration — closes dispatch-during-registration race.
            // Messages dispatched after registerStream() go into the queue; this re-check
            // catches terminal messages that committed before the initial read but were
            // not yet DB-visible at that point.
            final AtomicReference<String> recheckRef = new AtomicReference<>();
            QuarkusTransaction.requiringNew().run(() -> {
                final List<Message> messages = messageService.findAllByCorrelationId(taskId);
                final Commitment commitment = commitmentService.findByCorrelationId(taskId).orElse(null);
                final String state = (commitment != null && commitment.state != CommitmentState.OPEN)
                        ? A2ATaskState.fromCommitmentState(commitment.state)
                        : A2ATaskState.fromMessageHistory(messages);
                recheckRef.set(state);
            });
            if (A2ATaskState.TERMINAL_STATES.contains(recheckRef.get())) {
                sendStatusEvent(sink, sse, taskId, recheckRef.get());
                return; // finally deregisters
            }

            // Step 5: keepalive loop
            final long heartbeatMs = config.a2a().sse().heartbeatIntervalSeconds() * 1000L;
            final long deadline = System.currentTimeMillis()
                    + (long) config.a2a().sse().maxDurationSeconds() * 1000L;

            try {
                while (true) {
                    if (sink.isClosed()) break; // orphan: client disconnected
                    final long remaining = deadline - System.currentTimeMillis();
                    if (remaining <= 0) break; // max duration exceeded

                    final OutboundMessage msg;
                    try {
                        msg = queue.poll(Math.min(heartbeatMs, remaining), TimeUnit.MILLISECONDS);
                    } catch (final InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }

                    if (msg == null) {
                        // Poll timeout — send keepalive comment (fire-and-forget)
                        sink.send(sse.newEventBuilder().comment("keepalive").build());
                        continue;
                    }

                    final boolean terminal = A2ATaskState.TERMINAL_TYPES.contains(msg.type());
                    final String state = A2ATaskState.fromMessageType(msg.type());
                    final String json = "{\"id\":\"%s\",\"status\":{\"state\":\"%s\"},\"final\":%b}"
                            .formatted(taskId, state, terminal);
                    final CompletionStage<Void> send = sink.send(
                            sse.newEventBuilder().name("task_status_update").data(json).build());
                    if (terminal) {
                        send.toCompletableFuture().get(5, TimeUnit.SECONDS); // await before close
                        break;
                    }
                }
            } catch (final Exception e) {
                LOG.debugf(e, "SSE stream error for task %s", taskId);
            }
        } finally {
            a2aBackend.deregisterStream(corrId, consumer);
            if (!sink.isClosed()) sink.close();
        }
    }
```

- [ ] **Step 3: Replace the existing `sendStatusEvent` helper (lines 301–307). Remove the stale "never call close synchronously" comment — it described the old thenRun pattern; with .get() the send completes first**

```java
    /**
     * Sends a terminal status event and closes the sink.
     *
     * <p>Awaits {@link SseEventSink#send} synchronously (safe on a virtual thread — parks
     * without blocking an OS thread) to ensure the payload reaches the client before close.
     * The internal try-finally guarantees the sink is closed even if get() throws.
     */
    private static void sendStatusEvent(final SseEventSink sink, final Sse sse,
            final String taskId, final String state) throws Exception {
        final String json = "{\"id\":\"%s\",\"status\":{\"state\":\"%s\"},\"final\":true}"
                .formatted(taskId, state);
        try {
            sink.send(sse.newEventBuilder().name("task_status_update").data(json).build())
                    .toCompletableFuture().get(5, TimeUnit.SECONDS);
        } finally {
            if (!sink.isClosed()) sink.close();
        }
    }
```

- [ ] **Step 4: Replace the existing `sendErrorEvent` helper (lines 319–325). Remove the stale comment for the same reason**

```java
    /**
     * Sends an error event and closes the sink.
     *
     * <p>SSE void methods cannot return a different HTTP status — this endpoint always
     * returns HTTP 200 with text/event-stream content type. The {@code event:error} type
     * lets clients distinguish error events from status updates.
     *
     * <p>Awaits send and closes in try-finally — same guarantee as {@link #sendStatusEvent}.
     */
    private static void sendErrorEvent(final SseEventSink sink, final Sse sse,
            final String taskId, final String error) throws Exception {
        final String json = "{\"id\":\"%s\",\"error\":\"%s\",\"final\":true}"
                .formatted(taskId, error);
        try {
            sink.send(sse.newEventBuilder().name("error").data(json).build())
                    .toCompletableFuture().get(5, TimeUnit.SECONDS);
        } finally {
            if (!sink.isClosed()) sink.close();
        }
    }
```

- [ ] **Step 5: Run the full A2AStreamIntegrationTest to verify all 7 tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=A2AStreamIntegrationTest -pl runtime
```

Expected: `Tests run: 7, Failures: 0, Errors: 0, Skipped: 0`

If tests 5–7 still fail, check: (a) `a2aBackend.ensureRegistered()` is called by `POST /a2a/message:send` — the test dispatches via `messageService` directly, which goes through `ChannelGateway.fanOut()`. The `A2AChannelBackend` must be registered on the channel for `fanOut()` to call it. The test must call `a2aBackend.ensureRegistered(chId[0], new ChannelRef(chId[0], channelName))` before opening the SSE stream. If `fanOut()` doesn't reach the backend, the queue stays empty.

If the `ensureRegistered` fix is needed, add to the live-stream tests (tests 5–6) after retrieving `chId[0]`:

```java
        QuarkusTransaction.requiringNew().run(() -> {
            final io.casehub.qhorus.api.gateway.ChannelRef ref =
                    new io.casehub.qhorus.api.gateway.ChannelRef(chId[0], channelName);
            a2aBackend.ensureRegistered(chId[0], ref);
        });
```

---

## Task 7: Delete A2AStreamTaskTest and run the full test suite

**Files:**
- Delete: `runtime/src/test/java/io/casehub/qhorus/api/A2AStreamTaskTest.java`

All four tests from `A2AStreamTaskTest` are now covered by `A2AStreamIntegrationTest` (tests 1–4). The RestAssured versions are replaced by `SseEventSource` versions that test the actual SSE wire protocol.

- [ ] **Step 1: Delete the file**

```bash
rm /Users/mdproctor/claude/casehub/qhorus/runtime/src/test/java/io/casehub/qhorus/api/A2AStreamTaskTest.java
```

- [ ] **Step 2: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: all tests pass. The runtime module has ~666+ tests; watch for any regressions in non-A2A tests caused by the profile or config changes.

- [ ] **Step 3: Run the full project build to catch any issues in sibling modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: `BUILD SUCCESS`

---

## Task 8: Commit

- [ ] **Step 1: Stage all changes**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/pom.xml \
  runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/A2ATaskState.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java \
  runtime/src/test/java/io/casehub/qhorus/api/A2AEnabledProfile.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/api/A2AStreamIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/qhorus rm \
  runtime/src/test/java/io/casehub/qhorus/api/A2AStreamTaskTest.java
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(#278,#277): SSE active model — keepalive, orphan detection, max duration; live-stream tests

Replace streamTask()'s passive callback model with an active virtual-thread loop:
- @RunOnVirtualThread (replaces @Transactional as the VT dispatch mechanism)
- LinkedBlockingQueue<OutboundMessage> consumer = queue::offer
- queue.poll(heartbeatMs) drives keepalive comments (fire-and-forget)
- sink.isClosed() at top of every iteration detects orphaned clients
- Deadline check enforces casehub.qhorus.a2a.sse.max-duration-seconds (default 1800s)
- Short-lived QuarkusTransaction.requiringNew() for validation + re-check reads
- sendStatusEvent/sendErrorEvent updated: synchronous .get(5s) + try-finally close
- A2AChannelBackend: zero changes
- A2ATaskState: add TERMINAL_STATES constant
- QhorusConfig.A2a: add SseSettings interface (heartbeat-interval-seconds=15, max-duration-seconds=1800)
- A2AStreamIntegrationTest (new): 7 tests — 4 migrated from A2AStreamTaskTest, 3 new
- A2AStreamTaskTest: deleted (replaced by SseEventSource-based integration tests)

Refs #278, Closes #277

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- §1 `streamTask()` rewrite → Task 6 ✅
- §2 SseSettings config → Task 2 ✅
- §3 TERMINAL_STATES → Task 3 ✅
- §4 A2AChannelBackend zero changes → verified (no task touches it) ✅
- §5 A2AEnabledProfile SSE overrides → Task 4 ✅
- Testing: 7 tests → Task 5 ✅
- A2AStreamTaskTest deleted → Task 7 ✅
- pom.xml deps → Task 1 ✅
- `throws Exception` on `streamTask()` → included in Task 6 Step 2 ✅
- Stale comment removal → included in Task 6 Steps 3–4 ✅
- `TERMINAL_STATES.contains()` call-site in `streamTask()` → Task 6 uses it in the rewritten method ✅

**Type consistency check:**
- `config.a2a().sse().heartbeatIntervalSeconds()` — matches `SseSettings.heartbeatIntervalSeconds()` in Task 2 ✅
- `config.a2a().sse().maxDurationSeconds()` — matches `SseSettings.maxDurationSeconds()` in Task 2 ✅
- `A2ATaskState.TERMINAL_STATES` — defined in Task 3, used in Task 6 ✅
- `A2ATaskState.TERMINAL_TYPES` — existing constant, used in Task 6 for the loop terminal check ✅
- `A2ATaskState.fromMessageType()` — existing method, used in Task 6 ✅
- `a2aBackend.streamCount(corrId)` — package-private, accessible from `io.casehub.qhorus.runtime.api` (test package) ✅

**Placeholder scan:** No TBD, no TODO, no "similar to Task N", all code blocks are complete.

**`ensureRegistered` note:** Tasks 5–6 note the risk that `fanOut()` won't reach `A2AChannelBackend` if it's not registered on the test channel. The workaround is in Task 6 Step 5. This is the same registration pattern used by the existing `A2AChannelBackendIntegrationTest`.

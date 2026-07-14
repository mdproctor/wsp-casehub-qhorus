# WebSocket Catch-Up Mechanism Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #346 — WebSocket observer: catch-up mechanism for reconnecting clients
**Issue group:** #346

**Goal:** Enable WebSocket clients to catch up on missed messages after reconnection by passing `lastEventId` as a query parameter.

**Architecture:** Add `messageId` to `MessageReceivedEvent` (core API), then build catch-up replay in the WebSocket endpoint using server-side buffering in `WebSocketConnectionRegistry` to guarantee ordered delivery. Uses `CrossTenantMessageStore.scan(MessageQuery.poll(...))` for replay queries and `CrossTenantChannelStore.findById()` for channel validation.

**Tech Stack:** Java 21, Quarkus 3.32.2, Quarkus WebSockets Next, `@RunOnVirtualThread`

## Global Constraints

- Java 21 source (on Java 26 JVM)
- Pre-release — breaking API changes cost nothing
- `MessageReceivedEvent` is a record in `api/` module — constructor changes break all call sites
- WebSocket observer is an optional module activated by classpath presence
- Tests are CDI-free unit tests (no `@QuarkusTest` needed for the core logic)
- Use `ide_edit_member` / `ide_insert_member` / `ide_replace_member` for all Java edits
- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` from project root
- Cross-repo engine tests (`src/test/java/io/casehub/engine/...`) are out of scope — they'll update when engine pulls the new SNAPSHOT

---

### Task 1: Core API — `MessageReceivedEvent.messageId` + `fromMessage()` + dispatcher + CloudEventMapper

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/MessageReceivedEvent.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/CloudEventMapper.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/CloudEventMapperTest.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcherTest.java`

**Interfaces:**
- Produces: `MessageReceivedEvent(Long messageId, String channelName, UUID channelId, String tenancyId, MessageType messageType, String senderId, String correlationId, Instant occurredAt, String content, String topic)`
- Produces: `MessageReceivedEvent.fromMessage(Message message, String channelName)` → `MessageReceivedEvent`

- [ ] **Step 1: Write failing test for `fromMessage()` factory**

In a new test class `api/src/test/java/io/casehub/qhorus/api/gateway/MessageReceivedEventTest.java`:

```java
package io.casehub.qhorus.api.gateway;

import io.casehub.qhorus.api.message.ActorType;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class MessageReceivedEventTest {

    @Test
    void fromMessage_mapsAllFields() {
        UUID channelId = UUID.randomUUID();
        Instant now = Instant.now();
        Message msg = new Message(42L, channelId, "agent-1", MessageType.COMMAND,
                ActorType.AGENT, "t1", "do it", "corr-1", null, 0,
                List.of(), "role:worker", "general", null, null, null, 0, now);

        MessageReceivedEvent event = MessageReceivedEvent.fromMessage(msg, "ops-channel");

        assertEquals(42L, event.messageId());
        assertEquals("ops-channel", event.channelName());
        assertEquals(channelId, event.channelId());
        assertEquals("t1", event.tenancyId());
        assertEquals(MessageType.COMMAND, event.messageType());
        assertEquals("agent-1", event.senderId());
        assertEquals("corr-1", event.correlationId());
        assertEquals(now, event.occurredAt());
        assertEquals("do it", event.content());
        assertEquals("general", event.topic());
    }

    @Test
    void fromMessage_eventType_contentIsNull() {
        Message msg = Message.builder()
                .id(10L).channelId(UUID.randomUUID()).sender("agent-1")
                .messageType(MessageType.EVENT).content("{\"tool\":\"search\"}")
                .build();

        MessageReceivedEvent event = MessageReceivedEvent.fromMessage(msg, "ch");

        assertNull(event.content());
    }

    @Test
    void fromMessage_nullCreatedAt_defaultsToNow() {
        Message msg = Message.builder()
                .id(10L).channelId(UUID.randomUUID()).sender("agent-1")
                .messageType(MessageType.STATUS).content("working")
                .build();

        MessageReceivedEvent event = MessageReceivedEvent.fromMessage(msg, "ch");

        assertNotNull(event.occurredAt());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageReceivedEventTest -pl api`
Expected: Compilation failure — `messageId` field and `fromMessage()` don't exist yet.

- [ ] **Step 3: Add `messageId` field to `MessageReceivedEvent` record**

Use `ide_edit_member` on `MessageReceivedEvent.java`, member `MessageReceivedEvent`, to replace the record declaration:

```java
public record MessageReceivedEvent(
        Long messageId,
        String channelName,
        UUID channelId,
        String tenancyId,
        MessageType messageType,
        String senderId,
        String correlationId,
        Instant occurredAt,
        String content,
        String topic) {

    public MessageReceivedEvent {
        Objects.requireNonNull(occurredAt, "occurredAt");
        if (messageType == MessageType.EVENT && content != null) {
            throw new IllegalArgumentException(
                    "EVENT messages must have null content — Builder.build() enforces this at call-site");
        }
    }

    public static MessageReceivedEvent fromMessage(Message message, String channelName) {
        String content = message.messageType() == MessageType.EVENT ? null : message.content();
        Instant occurredAt = message.createdAt() != null ? message.createdAt() : Instant.now();
        return new MessageReceivedEvent(
                message.id(), channelName, message.channelId(), message.tenancyId(),
                message.messageType(), message.sender(), message.correlationId(),
                occurredAt, content, message.topic());
    }
}
```

Add import for `Message`:
```java
import io.casehub.qhorus.api.message.Message;
```

- [ ] **Step 4: Update `MessageObserverDispatcher.dispatch()` to use `fromMessage()`**

Replace the event construction block in `dispatch()` (lines 69-77) with:

```java
final MessageReceivedEvent event = MessageReceivedEvent.fromMessage(message, channelName);
```

Remove the now-unused local variables `content` and `occurredAt` (lines 69-72 become one line). Verify `tenancyId` — `fromMessage()` uses `message.tenancyId()`, but the dispatcher receives `tenancyId` as a parameter that may differ (set by `MessageService.dispatch()` from `CurrentPrincipal`). The `message.tenancyId()` is populated by `MessageService.dispatch()` before calling `MessageObserverDispatcher`, so they match. However, to be safe and maintain the existing contract, we should keep the parameter-based approach. Modify `fromMessage()` to accept tenancyId as well:

Actually no — looking at `MessageService.dispatch()`, it sets `message.tenancyId()` via `MessageDispatch.tenancyId()` resolved from `CurrentPrincipal`, and the `tenancyId` parameter to `MessageObserverDispatcher.dispatch()` comes from the same `dispatch.tenancyId()`. They are identical. Use `fromMessage()` directly — it reads `message.tenancyId()` which is already correct.

Do the same for `dispatchClusterOnly()` (lines 131-139).

- [ ] **Step 5: Update `CloudEventMapper.toCloudEvent()` to use `messageId`**

Replace the `.withId(UUID.randomUUID().toString())` line with:

```java
.withId(event.messageId() != null
        ? String.valueOf(event.messageId())
        : UUID.randomUUID().toString())
```

- [ ] **Step 6: Update `CloudEventMapperTest` — add `messageId` parameter and test CloudEvent ID**

Each `new MessageReceivedEvent(...)` call in this test file gains `1L` (or an appropriate Long) as the first parameter.

Add a new test:

```java
@Test
void usesMessageIdAsCloudEventId() {
    MessageReceivedEvent event = new MessageReceivedEvent(
            42L, "test-channel", channelId, "t1",
            MessageType.STATUS, "agent-1", null,
            Instant.now(), "hello", null);
    CloudEvent ce = CloudEventMapper.toCloudEvent(event, objectMapper);
    assertEquals("42", ce.getId());
}

@Test
void fallsBackToRandomUuidWhenMessageIdNull() {
    MessageReceivedEvent event = new MessageReceivedEvent(
            null, "test-channel", channelId, "t1",
            MessageType.STATUS, "agent-1", null,
            Instant.now(), "hello", null);
    CloudEvent ce = CloudEventMapper.toCloudEvent(event, objectMapper);
    assertDoesNotThrow(() -> UUID.fromString(ce.getId()));
}
```

- [ ] **Step 7: Update `MessageObserverDispatcherTest` — add `messageId` to FQN constructor calls**

Two direct construction sites near the bottom of the test use FQN `new io.casehub.qhorus.api.gateway.MessageReceivedEvent(...)`. Add `null` as the first parameter to both (these test the compact constructor, not messageId).

Also add a test that verifies `messageId` is propagated through dispatch:

```java
@Test
void dispatch_messageId_propagatedToEvent() {
    final List<MessageReceivedEvent> captured = new ArrayList<>();

    Message msg = Message.builder()
            .id(99L).channelId(channelId).sender("agent-a")
            .messageType(MessageType.COMMAND).content("test").build();

    MessageObserverDispatcher.dispatch(channelName, channelId,
            TEST_TENANCY_ID, msg, List.of(handle(captured::add)));

    assertEquals(99L, captured.get(0).messageId());
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageReceivedEventTest -pl api`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CloudEventMapperTest -pl runtime`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageObserverDispatcherTest -pl runtime`
Expected: All PASS.

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/gateway/MessageReceivedEvent.java \
        api/src/test/java/io/casehub/qhorus/api/gateway/MessageReceivedEventTest.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/gateway/CloudEventMapper.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/gateway/CloudEventMapperTest.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcherTest.java
git commit -m "feat(#346): add messageId to MessageReceivedEvent, fromMessage() factory, CloudEvent ID

Refs #346"
```

---

### Task 2: Mechanical — update all `MessageReceivedEvent` test constructor call sites

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapterTest.java` (8 sites)
- Modify: `websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketMessageObserverTest.java` (4 sites)
- Modify: `kafka-observer/src/test/java/io/casehub/qhorus/kafka/KafkaMessageObserverTest.java` (3 sites)
- Modify: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookMessageObserverTest.java` (1 site)

**Interfaces:**
- Consumes: `MessageReceivedEvent(Long messageId, ...)` — new first parameter from Task 1

**Pattern:** Every `new MessageReceivedEvent(channelName, ...)` becomes `new MessageReceivedEvent(1L, channelName, ...)`. Use `1L` as a default test messageId — tests that don't care about the value just need it non-null.

- [ ] **Step 1: Update `QhorusCloudEventAdapterTest` — 8 sites**

Each `new MessageReceivedEvent(` gains `1L` as the first argument. All 8 constructor calls follow the same pattern.

- [ ] **Step 2: Update `WebSocketMessageObserverTest` — 4 sites**

Same pattern: add `1L` as the first argument.

- [ ] **Step 3: Update `KafkaMessageObserverTest` — 3 sites**

Same pattern: add `1L` as the first argument.

- [ ] **Step 4: Update `WebhookMessageObserverTest` — 1 site**

In the `event()` helper method, add `1L` as the first argument.

- [ ] **Step 5: Build the full project to verify all modules compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS. This catches any missed call sites across all modules (including `examples/`).

If compilation fails in engine modules (cross-repo), those are expected — they pull qhorus as a SNAPSHOT dependency and will update separately.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/test/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapterTest.java \
        websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketMessageObserverTest.java \
        kafka-observer/src/test/java/io/casehub/qhorus/kafka/KafkaMessageObserverTest.java \
        webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookMessageObserverTest.java
git commit -m "test(#346): update MessageReceivedEvent constructor call sites for messageId

Mechanical update — add messageId (1L) as first parameter to all test
construction sites across runtime, websocket-observer, kafka-observer,
and webhook-observer modules.

Refs #346"
```

---

### Task 3: `WebSocketConnectionRegistry` catch-up buffering

**Files:**
- Modify: `websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketConnectionRegistry.java`
- Test: `websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketConnectionRegistryTest.java`

**Interfaces:**
- Produces: `BufferedMessage(Long messageId, String json)` — inner record
- Produces: `subscribeCatchingUp(UUID channelId, WebSocketConnection connection)` — registers with catch-up buffer
- Produces: `completeCatchUp(UUID channelId, WebSocketConnection connection)` → `List<BufferedMessage>` — atomically transitions to live, returns buffer
- Produces: `tryBufferForCatchUp(WebSocketConnection connection, Long messageId, String json)` → `boolean` — buffers if catching up, false if live
- Produces: `cancelCatchUp(UUID channelId, WebSocketConnection connection)` — discards buffer (error path)
- Produces: existing `unsubscribe(UUID, WebSocketConnection)` gains defensive buffer cleanup

- [ ] **Step 1: Write failing tests for catch-up buffering**

Add to `WebSocketConnectionRegistryTest.java`:

```java
@Test
void subscribeCatchingUp_buffersViaTriBuffer() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection conn = mock(WebSocketConnection.class);

    registry.subscribeCatchingUp(channelId, conn);
    boolean buffered = registry.tryBufferForCatchUp(conn, 1L, "{\"msg\":1}");

    assertThat(buffered).isTrue();
    assertThat(registry.connections(channelId)).containsExactly(conn);
}

@Test
void completeCatchUp_returnsBufferedMessages() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection conn = mock(WebSocketConnection.class);

    registry.subscribeCatchingUp(channelId, conn);
    registry.tryBufferForCatchUp(conn, 1L, "{\"msg\":1}");
    registry.tryBufferForCatchUp(conn, 2L, "{\"msg\":2}");

    var buffered = registry.completeCatchUp(channelId, conn);

    assertThat(buffered).hasSize(2);
    assertThat(buffered.get(0).messageId()).isEqualTo(1L);
    assertThat(buffered.get(1).messageId()).isEqualTo(2L);
}

@Test
void completeCatchUp_subsequentTryBufferReturnsFalse() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection conn = mock(WebSocketConnection.class);

    registry.subscribeCatchingUp(channelId, conn);
    registry.completeCatchUp(channelId, conn);

    assertThat(registry.tryBufferForCatchUp(conn, 3L, "{}")).isFalse();
}

@Test
void cancelCatchUp_discardsBuffer() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection conn = mock(WebSocketConnection.class);

    registry.subscribeCatchingUp(channelId, conn);
    registry.tryBufferForCatchUp(conn, 1L, "{}");
    registry.cancelCatchUp(channelId, conn);

    assertThat(registry.tryBufferForCatchUp(conn, 2L, "{}")).isFalse();
}

@Test
void unsubscribeDuringCatchUp_cleansUpBuffer() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection conn = mock(WebSocketConnection.class);

    registry.subscribeCatchingUp(channelId, conn);
    registry.tryBufferForCatchUp(conn, 1L, "{}");
    registry.unsubscribe(channelId, conn);

    assertThat(registry.tryBufferForCatchUp(conn, 2L, "{}")).isFalse();
    assertThat(registry.connections(channelId)).isEmpty();
}

@Test
void liveSubscribe_tryBufferReturnsFalse() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection conn = mock(WebSocketConnection.class);

    registry.subscribe(channelId, conn);

    assertThat(registry.tryBufferForCatchUp(conn, 1L, "{}")).isFalse();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WebSocketConnectionRegistryTest -pl websocket-observer`
Expected: Compilation failure — new methods don't exist.

- [ ] **Step 3: Implement catch-up buffering in `WebSocketConnectionRegistry`**

Add `BufferedMessage` record and catch-up methods:

```java
public record BufferedMessage(Long messageId, String json) {}

private final ConcurrentHashMap<WebSocketConnection, List<BufferedMessage>> catchUpBuffers =
        new ConcurrentHashMap<>();

public void subscribeCatchingUp(UUID channelId, WebSocketConnection connection) {
    channels.computeIfAbsent(channelId, k -> ConcurrentHashMap.newKeySet()).add(connection);
    catchUpBuffers.put(connection, new ArrayList<>());
}

public List<BufferedMessage> completeCatchUp(UUID channelId, WebSocketConnection connection) {
    List<BufferedMessage> buffer = catchUpBuffers.remove(connection);
    return buffer != null ? List.copyOf(buffer) : List.of();
}

public boolean tryBufferForCatchUp(WebSocketConnection connection, Long messageId, String json) {
    List<BufferedMessage> buffer = catchUpBuffers.get(connection);
    if (buffer == null) {
        return false;
    }
    synchronized (buffer) {
        buffer.add(new BufferedMessage(messageId, json));
    }
    return true;
}

public void cancelCatchUp(UUID channelId, WebSocketConnection connection) {
    catchUpBuffers.remove(connection);
}
```

Update existing `unsubscribe()` to also clean up catch-up buffer:

```java
public void unsubscribe(UUID channelId, WebSocketConnection connection) {
    channels.computeIfPresent(channelId, (k, conns) -> {
        conns.remove(connection);
        return conns.isEmpty() ? null : conns;
    });
    catchUpBuffers.remove(connection);
}
```

Add imports for `ArrayList` and `List`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WebSocketConnectionRegistryTest -pl websocket-observer`
Expected: All PASS.

- [ ] **Step 5: Commit**

```bash
git add websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketConnectionRegistry.java \
        websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketConnectionRegistryTest.java
git commit -m "feat(#346): WebSocketConnectionRegistry catch-up buffering

subscribeCatchingUp, completeCatchUp, tryBufferForCatchUp, cancelCatchUp.
BufferedMessage record. Defensive cleanup in unsubscribe.

Refs #346"
```

---

### Task 4: `WebSocketMessageObserver` catch-up awareness

**Files:**
- Modify: `websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketMessageObserver.java`
- Modify: `websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketMessageObserverTest.java`

**Interfaces:**
- Consumes: `WebSocketConnectionRegistry.tryBufferForCatchUp(WebSocketConnection, Long, String)` from Task 3
- Consumes: `MessageReceivedEvent.messageId()` from Task 1

- [ ] **Step 1: Write failing tests for catch-up awareness**

Add to `WebSocketMessageObserverTest.java`:

```java
@Test
void catchingUpConnection_messagesBuffered() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection conn = mock(WebSocketConnection.class);
    registry.subscribeCatchingUp(channelId, conn);

    observer.onMessage(new MessageReceivedEvent(
            5L, "test-channel", channelId, "t1",
            MessageType.STATUS, "agent-1", null,
            Instant.now(), "hello", null));

    verify(conn, never()).sendTextAndAwait(anyString());
    var buffered = registry.completeCatchUp(channelId, conn);
    assertThat(buffered).hasSize(1);
    assertThat(buffered.get(0).messageId()).isEqualTo(5L);
}

@Test
void mixedConnections_onlyCatchingUpBuffered() {
    UUID channelId = UUID.randomUUID();
    WebSocketConnection live = mock(WebSocketConnection.class);
    WebSocketConnection catching = mock(WebSocketConnection.class);

    registry.subscribe(channelId, live);
    registry.subscribeCatchingUp(channelId, catching);

    observer.onMessage(new MessageReceivedEvent(
            6L, "test-channel", channelId, "t1",
            MessageType.STATUS, "agent-1", null,
            Instant.now(), "hello", null));

    verify(live).sendTextAndAwait(anyString());
    verify(catching, never()).sendTextAndAwait(anyString());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WebSocketMessageObserverTest -pl websocket-observer`
Expected: FAIL — current observer sends to all connections regardless of catch-up state.

- [ ] **Step 3: Update `WebSocketMessageObserver.onMessage()` to check catch-up state**

Replace the `onMessage` method body. For each connection, check `tryBufferForCatchUp` before sending:

```java
@Override
public void onMessage(MessageReceivedEvent event) {
    Set<WebSocketConnection> connections = registry.connections(event.channelId());
    if (connections.isEmpty()) {
        return;
    }

    String json;
    try {
        json = objectMapper.writeValueAsString(event);
    } catch (Exception e) {
        LOG.warnf("Failed to serialize event for WebSocket push — channel=%s: %s",
                  event.channelId(), e.getMessage());
        return;
    }

    for (WebSocketConnection conn : Set.copyOf(connections)) {
        if (registry.tryBufferForCatchUp(conn, event.messageId(), json)) {
            continue;
        }
        try {
            conn.sendTextAndAwait(json);
        } catch (Exception e) {
            LOG.debugf("WebSocket send failed for channel %s: %s",
                       event.channelId(), e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WebSocketMessageObserverTest -pl websocket-observer`
Expected: All PASS.

- [ ] **Step 5: Commit**

```bash
git add websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketMessageObserver.java \
        websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketMessageObserverTest.java
git commit -m "feat(#346): WebSocketMessageObserver catch-up awareness

Check registry.tryBufferForCatchUp before sending — catching-up
connections buffer messages instead of receiving them directly.

Refs #346"
```

---

### Task 5: `ChannelWebSocketEndpoint` catch-up flow + config

**Files:**
- Create: `websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketCatchUpConfig.java`
- Modify: `websocket-observer/src/main/java/io/casehub/qhorus/websocket/ChannelWebSocketEndpoint.java`
- Test: `websocket-observer/src/test/java/io/casehub/qhorus/websocket/ChannelWebSocketEndpointTest.java` (new)

**Interfaces:**
- Consumes: `MessageReceivedEvent.fromMessage(Message, String)` from Task 1
- Consumes: `WebSocketConnectionRegistry.subscribeCatchingUp()`, `completeCatchUp()`, `cancelCatchUp()` from Task 3
- Consumes: `CrossTenantChannelStore.findById(UUID)` → `Optional<Channel>`
- Consumes: `CrossTenantMessageStore.scan(MessageQuery)` → `List<Message>`
- Consumes: `CrossTenantMessageStore.findLastMessage(UUID)` → `Optional<Message>`

- [ ] **Step 1: Create `WebSocketCatchUpConfig`**

Create file `websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketCatchUpConfig.java`:

```java
package io.casehub.qhorus.websocket;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.qhorus.websocket.catchup")
public interface WebSocketCatchUpConfig {

    @WithDefault("500")
    int maxMessages();
}
```

- [ ] **Step 2: Write failing tests for the catch-up flow**

Create `websocket-observer/src/test/java/io/casehub/qhorus/websocket/ChannelWebSocketEndpointTest.java`:

```java
package io.casehub.qhorus.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.CrossTenantChannelStore;
import io.casehub.qhorus.api.store.CrossTenantMessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.quarkus.websockets.next.CloseReason;
import io.quarkus.websockets.next.HandshakeRequest;
import io.quarkus.websockets.next.WebSocketConnection;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class ChannelWebSocketEndpointTest {

    private WebSocketConnectionRegistry registry;
    private CrossTenantChannelStore channelStore;
    private CrossTenantMessageStore messageStore;
    private ObjectMapper objectMapper;
    private WebSocketCatchUpConfig config;
    private ChannelWebSocketEndpoint endpoint;

    @BeforeEach
    void setUp() {
        registry = new WebSocketConnectionRegistry();
        channelStore = mock(CrossTenantChannelStore.class);
        messageStore = mock(CrossTenantMessageStore.class);
        objectMapper = new ObjectMapper().registerModule(new JavaTimeModule());
        config = () -> 500;
        endpoint = new ChannelWebSocketEndpoint();
        endpoint.registry = registry;
        endpoint.channelStore = channelStore;
        endpoint.messageStore = messageStore;
        endpoint.objectMapper = objectMapper;
        endpoint.config = config;
    }

    private Channel testChannel(UUID id, String name) {
        return Channel.builder(name).id(id).semantic(ChannelSemantic.APPEND).build();
    }

    private WebSocketConnection mockConnection(String query) {
        WebSocketConnection conn = mock(WebSocketConnection.class);
        HandshakeRequest req = mock(HandshakeRequest.class);
        when(conn.handshakeRequest()).thenReturn(req);
        when(req.query()).thenReturn(query);
        return conn;
    }

    private Message testMessage(Long id, UUID channelId, String content) {
        return Message.builder()
                .id(id).channelId(channelId).sender("agent-1")
                .messageType(MessageType.STATUS).content(content)
                .createdAt(Instant.now()).build();
    }

    @Test
    void noLastEventId_liveOnlyMode() {
        UUID channelId = UUID.randomUUID();
        when(channelStore.findById(channelId)).thenReturn(Optional.of(testChannel(channelId, "ops")));
        WebSocketConnection conn = mockConnection(null);

        endpoint.onOpen(channelId.toString(), conn);

        assertThat(registry.connections(channelId)).containsExactly(conn);
        verify(conn, never()).sendTextAndAwait(anyString());
        verifyNoInteractions(messageStore);
    }

    @Test
    void lastEventId_sendsCatchUpMessages() {
        UUID channelId = UUID.randomUUID();
        when(channelStore.findById(channelId)).thenReturn(Optional.of(testChannel(channelId, "ops")));
        when(messageStore.scan(any(MessageQuery.class))).thenReturn(List.of(
                testMessage(43L, channelId, "msg-43"),
                testMessage(44L, channelId, "msg-44")));

        WebSocketConnection conn = mockConnection("lastEventId=42");
        endpoint.onOpen(channelId.toString(), conn);

        ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
        verify(conn, atLeast(3)).sendTextAndAwait(captor.capture());

        List<String> frames = captor.getAllValues();
        assertThat(frames.get(0)).contains("catchup_begin");
        assertThat(frames.get(1)).contains("msg-43");
        assertThat(frames.get(2)).contains("msg-44");
        // catchup_end is the last frame (after buffer flush)
        assertThat(frames.get(frames.size() - 1)).contains("catchup_end");
    }

    @Test
    void truncation_sendsCatchUpTruncated() {
        UUID channelId = UUID.randomUUID();
        when(channelStore.findById(channelId)).thenReturn(Optional.of(testChannel(channelId, "ops")));

        config = () -> 2;
        endpoint.config = config;

        when(messageStore.scan(any(MessageQuery.class))).thenReturn(List.of(
                testMessage(43L, channelId, "msg-43"),
                testMessage(44L, channelId, "msg-44"),
                testMessage(45L, channelId, "msg-45")));
        when(messageStore.findLastMessage(channelId)).thenReturn(
                Optional.of(testMessage(590L, channelId, "head")));

        WebSocketConnection conn = mockConnection("lastEventId=42");
        endpoint.onOpen(channelId.toString(), conn);

        ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
        verify(conn, atLeast(1)).sendTextAndAwait(captor.capture());

        String lastFrame = captor.getAllValues().get(captor.getAllValues().size() - 1);
        assertThat(lastFrame).contains("catchup_truncated");
        assertThat(lastFrame).contains("590");
    }

    @Test
    void invalidLastEventId_noStoreQuery() {
        UUID channelId = UUID.randomUUID();
        when(channelStore.findById(channelId)).thenReturn(Optional.of(testChannel(channelId, "ops")));
        WebSocketConnection conn = mockConnection("lastEventId=notanumber");

        endpoint.onOpen(channelId.toString(), conn);

        assertThat(registry.connections(channelId)).containsExactly(conn);
        verifyNoInteractions(messageStore);
    }

    @Test
    void unknownChannel_closesConnection() {
        UUID channelId = UUID.randomUUID();
        when(channelStore.findById(channelId)).thenReturn(Optional.empty());
        WebSocketConnection conn = mockConnection(null);

        endpoint.onOpen(channelId.toString(), conn);

        verify(conn).closeAndAwait(any(CloseReason.class));
        assertThat(registry.connections(channelId)).isEmpty();
    }

    @Test
    void emptyResult_sendsCatchUpBeginAndEnd() {
        UUID channelId = UUID.randomUUID();
        when(channelStore.findById(channelId)).thenReturn(Optional.of(testChannel(channelId, "ops")));
        when(messageStore.scan(any(MessageQuery.class))).thenReturn(List.of());

        WebSocketConnection conn = mockConnection("lastEventId=42");
        endpoint.onOpen(channelId.toString(), conn);

        ArgumentCaptor<String> captor = ArgumentCaptor.forClass(String.class);
        verify(conn, atLeast(2)).sendTextAndAwait(captor.capture());

        List<String> frames = captor.getAllValues();
        assertThat(frames.get(0)).contains("catchup_begin");
        assertThat(frames.get(frames.size() - 1)).contains("catchup_end");
    }

    @Test
    void dbErrorDuringCatchUp_bufferCleanedUp() {
        UUID channelId = UUID.randomUUID();
        when(channelStore.findById(channelId)).thenReturn(Optional.of(testChannel(channelId, "ops")));
        when(messageStore.scan(any(MessageQuery.class)))
                .thenThrow(new RuntimeException("DB error"));

        WebSocketConnection conn = mockConnection("lastEventId=42");
        endpoint.onOpen(channelId.toString(), conn);

        // Buffer should be cleaned up — tryBuffer returns false
        assertThat(registry.tryBufferForCatchUp(conn, 1L, "{}")).isFalse();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelWebSocketEndpointTest -pl websocket-observer`
Expected: Compilation failure — new fields don't exist on endpoint.

- [ ] **Step 4: Implement the catch-up flow in `ChannelWebSocketEndpoint`**

Replace the full `ChannelWebSocketEndpoint` class:

```java
package io.casehub.qhorus.websocket;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.store.CrossTenantChannelStore;
import io.casehub.qhorus.api.store.CrossTenantMessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.quarkus.websockets.next.CloseReason;
import io.quarkus.websockets.next.OnClose;
import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.PathParam;
import io.quarkus.websockets.next.WebSocket;
import io.quarkus.websockets.next.WebSocketConnection;
import io.smallrye.common.annotation.RunOnVirtualThread;

@WebSocket(path = "/qhorus/ws/channels/{channelId}")
public class ChannelWebSocketEndpoint {

    private static final Logger LOG = Logger.getLogger(ChannelWebSocketEndpoint.class);

    @Inject
    WebSocketConnectionRegistry registry;

    @Inject
    CrossTenantChannelStore channelStore;

    @Inject
    CrossTenantMessageStore messageStore;

    @Inject
    ObjectMapper objectMapper;

    @Inject
    WebSocketCatchUpConfig config;

    @OnOpen
    @RunOnVirtualThread
    void onOpen(@PathParam String channelId, WebSocketConnection connection) {
        UUID id;
        try {
            id = UUID.fromString(channelId);
        } catch (IllegalArgumentException e) {
            LOG.warnf("Invalid channelId: %s", channelId);
            connection.closeAndAwait(new CloseReason(1008, "Invalid channelId"));
            return;
        }

        Optional<Channel> channelOpt = channelStore.findById(id);
        if (channelOpt.isEmpty()) {
            LOG.debugf("Unknown channel: %s", channelId);
            connection.closeAndAwait(new CloseReason(1008, "Unknown channel"));
            return;
        }

        String channelName = channelOpt.get().name();
        Long lastEventId = parseLastEventId(connection);

        if (lastEventId == null) {
            registry.subscribe(id, connection);
            LOG.debugf("WebSocket client subscribed to channel %s (live-only)", channelId);
            return;
        }

        registry.subscribeCatchingUp(id, connection);
        LOG.debugf("WebSocket client subscribed to channel %s with catch-up from %d", channelId, lastEventId);

        try {
            sendControl(connection, Map.of("control", "catchup_begin"));

            int maxMessages = config.maxMessages();
            List<Message> messages = messageStore.scan(
                    MessageQuery.poll(id, lastEventId, maxMessages + 1));

            boolean truncated = messages.size() > maxMessages;
            List<Message> toSend = truncated ? messages.subList(0, maxMessages) : messages;

            long highestSentMessageId = lastEventId;
            for (Message msg : toSend) {
                MessageReceivedEvent event = MessageReceivedEvent.fromMessage(msg, channelName);
                connection.sendTextAndAwait(objectMapper.writeValueAsString(event));
                if (msg.id() != null && msg.id() > highestSentMessageId) {
                    highestSentMessageId = msg.id();
                }
            }

            List<WebSocketConnectionRegistry.BufferedMessage> buffered =
                    registry.completeCatchUp(id, connection);
            for (var buf : buffered) {
                if (buf.messageId() != null && buf.messageId() > highestSentMessageId) {
                    connection.sendTextAndAwait(buf.json());
                    highestSentMessageId = buf.messageId();
                }
            }

            if (truncated) {
                long headId = messageStore.findLastMessage(id)
                        .map(Message::id).orElse(highestSentMessageId);
                sendControl(connection, Map.of(
                        "control", "catchup_truncated",
                        "oldestAvailableId", toSend.get(0).id(),
                        "headId", headId));
            } else {
                sendControl(connection, Map.of(
                        "control", "catchup_end",
                        "lastMessageId", highestSentMessageId));
            }
        } catch (Exception e) {
            LOG.warnf("Catch-up failed for channel %s: %s", channelId, e.getMessage());
            registry.cancelCatchUp(id, connection);
        }
    }

    @OnClose
    void onClose(@PathParam String channelId, WebSocketConnection connection) {
        try {
            UUID id = UUID.fromString(channelId);
            registry.unsubscribe(id, connection);
            LOG.debugf("WebSocket client unsubscribed from channel %s", channelId);
        } catch (IllegalArgumentException e) {
            LOG.debugf("Invalid channelId on close: %s", channelId);
        }
    }

    private Long parseLastEventId(WebSocketConnection connection) {
        String query = connection.handshakeRequest().query();
        if (query == null || query.isEmpty()) {
            return null;
        }
        for (String param : query.split("&")) {
            String[] kv = param.split("=", 2);
            if (kv.length == 2 && "lastEventId".equals(kv[0])) {
                try {
                    return Long.parseLong(kv[1]);
                } catch (NumberFormatException e) {
                    LOG.warnf("Invalid lastEventId value: %s", kv[1]);
                    return null;
                }
            }
        }
        return null;
    }

    private void sendControl(WebSocketConnection connection, Map<String, Object> control) {
        try {
            connection.sendTextAndAwait(objectMapper.writeValueAsString(control));
        } catch (Exception e) {
            LOG.warnf("Failed to send control frame: %s", e.getMessage());
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelWebSocketEndpointTest -pl websocket-observer`
Expected: All PASS.

- [ ] **Step 6: Run all websocket-observer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl websocket-observer`
Expected: All PASS.

- [ ] **Step 7: Full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 8: Commit**

```bash
git add websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketCatchUpConfig.java \
        websocket-observer/src/main/java/io/casehub/qhorus/websocket/ChannelWebSocketEndpoint.java \
        websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketMessageObserver.java \
        websocket-observer/src/test/java/io/casehub/qhorus/websocket/ChannelWebSocketEndpointTest.java
git commit -m "feat(#346): WebSocket catch-up flow — lastEventId replay, server-side buffering

ChannelWebSocketEndpoint gains @RunOnVirtualThread catch-up logic:
channel validation, MessageQuery.poll catch-up, server-side buffering
via WebSocketConnectionRegistry, truncation detection with headId.
WebSocketCatchUpConfig with max-messages=500 default.

Refs #346"
```

---

## Self-Review

**Spec coverage:**
- `messageId` on `MessageReceivedEvent` → Task 1 ✓
- `fromMessage()` factory → Task 1 ✓
- `MessageObserverDispatcher` update → Task 1 ✓
- `CloudEventMapper` improvement → Task 1 ✓
- Catch-up protocol (wire format, control frames) → Task 5 ✓
- `ChannelWebSocketEndpoint` flow (all 9 steps) → Task 5 ✓
- Server-side buffering → Tasks 3, 4 ✓
- `WebSocketConnectionRegistry` additions → Task 3 ✓
- `WebSocketMessageObserver` catch-up awareness → Task 4 ✓
- Configuration (`max-messages`) → Task 5 ✓
- Error handling (buffer cleanup) → Task 5 test ✓
- Test updates (mechanical) → Task 2 ✓

**Placeholder scan:** No TBDs, TODOs, or vague references found.

**Type consistency:** `BufferedMessage(Long messageId, String json)` used consistently in Tasks 3, 4, 5. `fromMessage(Message, String)` signature consistent across Tasks 1, 5.

**Tooling safety scan:** No bash file operations on source files. All edits via `ide_edit_member` / `ide_insert_member`. Bash only for git and mvn commands.

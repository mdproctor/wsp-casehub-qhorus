# #219 Connector Channel Backend — Completion Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor `QhorusEntityMapper` from Option A (self-querying) to Option B (caller-supplied binding), add `findAll()` batch support, fix the integration test's `@ObservesAsync` flakiness, register native image SQL resources, and update platform docs.

**Architecture:** The pre-written code (committed in two commits on this branch) already provides `ConnectorChannelBackend`, `ChannelConnectorBinding`, `ChannelBindingStore`, `ChannelCreateRequest`, V14 migration, and all unit tests. `ChannelDetail` already has the `ConnectorBinding` nested record and 14-arg constructor. The mapper currently uses Option A (injects `ChannelBindingStore`, queries per-channel). This plan refactors to Option B: `QhorusMcpToolsBase` holds the store and supplies bindings to the pure mapper. `list_channels` uses a pre-loaded `findAll()` map; all other tool call sites use the single-item lookup in the base class. The dashboard follows the same pattern with explicit worker-pool dispatch.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache (named `qhorus` datasource), Mutiny, Mockito, AssertJ, JUnit 5. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn …`

---

## File Map

| File | Action | Why |
|------|--------|-----|
| `runtime/…/store/ChannelBindingStore.java` | Modify | Add `findAll()` to interface |
| `runtime/…/store/jpa/JpaChannelBindingStore.java` | Modify | Implement `findAll()` |
| `testing/…/InMemoryChannelBindingStore.java` | Modify | Implement `findAll()` |
| `testing/…/contract/ChannelBindingStoreContractTest.java` | Modify | Add 4 `findAll` contract tests |
| `runtime/…/QhorusEntityMapper.java` | Modify | Option B: remove store, add `Optional<ChannelConnectorBinding>` param |
| `runtime/…/mcp/QhorusMcpToolsBase.java` | Modify | Add `ChannelBindingStore` injection + two `toChannelDetail` overloads |
| `runtime/…/mcp/QhorusMcpTools.java` | Modify | `list_channels`: pre-load `findAll()` map, pass to batch overload |
| `runtime/…/mcp/ReactiveQhorusMcpTools.java` | Modify | `list_channels`: wrap `findAll()` in worker-pool Uni, pass to batch overload |
| `runtime/…/dashboard/QhorusDashboardService.java` | Modify | Add store injection, remove private `toChannelDetail`, fix `listChannels()` |
| `runtime/…/dashboard/QhorusDashboardServiceTest.java` | Modify | Stub `findAll()`, fix mapper constructor, wire `service.bindingStore` |
| `connector-backend/…/ConnectorChannelBackendIntegrationTest.java` | Modify | Remove `InboundConnectorService`; call `backend.onInboundMessage()` directly; remove sleeps |
| `deployment/…/QhorusProcessor.java` | Modify | Add `NativeImageResourcePatternsBuildItem` with two SQL globs |
| `casehub-parent/docs/PLATFORM.md` | Modify | Cross-Repo Dependency Map row for `connector-backend` |
| `casehub-parent/docs/protocols/casehub/cross-foundation-bridge-module-placement.md` | Modify | Add runtime-coupling exception paragraph |

---

## Task 1: Add `findAll()` to `ChannelBindingStore` (TDD)

**Files:**
- Modify: `testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelBindingStoreContractTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/ChannelBindingStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelBindingStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java`

- [ ] **Step 1: Write 4 failing contract tests**

Add these four tests to `ChannelBindingStoreContractTest`. Add `import java.util.Map;` if not already present.

```java
@Test
void findAll_emptyStore_returnsEmptyMap() {
    assertTrue(store().findAll().isEmpty());
}

@Test
void findAll_afterPut_containsBinding() {
    UUID channelId = UUID.randomUUID();
    store().put(binding(channelId, "sms", "key-fa1", "twilio", "+1"));
    Map<UUID, ChannelConnectorBinding> result = store().findAll();
    assertTrue(result.containsKey(channelId));
    assertEquals("+1", result.get(channelId).outboundDestination);
}

@Test
void findAll_afterDelete_excludesDeletedBinding() {
    UUID channelId = UUID.randomUUID();
    store().put(binding(channelId, "sms", "key-fa2", "twilio", "+2"));
    store().delete(channelId);
    assertFalse(store().findAll().containsKey(channelId));
}

@Test
void findAll_returnsSnapshotNotLiveView() {
    UUID channelId = UUID.randomUUID();
    store().put(binding(channelId, "sms", "key-fa3", "twilio", "+3"));
    Map<UUID, ChannelConnectorBinding> snapshot = store().findAll();
    store().delete(channelId);
    assertTrue(snapshot.containsKey(channelId)); // snapshot unchanged after delete
}
```

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=InMemoryChannelBindingStoreTest 2>&1 | grep -E "ERROR|findAll|cannot find"
```
Expected: compile error — `findAll()` is not defined in `ChannelBindingStore`.

- [ ] **Step 3: Add `findAll()` to the interface**

In `ChannelBindingStore.java`, add import and method:

```java
import java.util.Map;
// …existing imports…

public interface ChannelBindingStore {

    Optional<ChannelConnectorBinding> findByChannelId(UUID channelId);
    Optional<ChannelConnectorBinding> findByKey(String inboundConnectorId, String externalKey);
    void put(ChannelConnectorBinding binding);
    void delete(UUID channelId);

    /** Returns a snapshot of all bindings keyed by channelId. Callers must not mutate the map. */
    Map<UUID, ChannelConnectorBinding> findAll();
}
```

- [ ] **Step 4: Implement in `JpaChannelBindingStore`**

Add imports and override:

```java
import java.util.Map;
import java.util.stream.Collectors;
// …existing imports…

@Override
public Map<UUID, ChannelConnectorBinding> findAll() {
    return ChannelConnectorBinding.<ChannelConnectorBinding>listAll().stream()
            .collect(Collectors.toMap(b -> b.channelId, b -> b));
}
```

- [ ] **Step 5: Implement in `InMemoryChannelBindingStore`**

```java
@Override
public Map<UUID, ChannelConnectorBinding> findAll() {
    return Map.copyOf(byChannelId);
}
```

`Map` is already imported. `byChannelId` is the `ConcurrentHashMap<UUID, ChannelConnectorBinding>` field.

- [ ] **Step 6: Run contract tests — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing
```
Expected: all tests pass including the 4 new `findAll` cases.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/ChannelBindingStore.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelBindingStore.java \
  testing/src/main/java/io/casehub/qhorus/testing/InMemoryChannelBindingStore.java \
  testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelBindingStoreContractTest.java

git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(#219): add ChannelBindingStore.findAll() — batch binding lookup for list_channels

Refs #219
EOF
)"
```

---

## Task 2: Refactor `QhorusEntityMapper` + Update Base Class + Dashboard (Atomic)

These three files form a single compile unit — changing the mapper breaks the base class and dashboard simultaneously. Apply all changes before building.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java`

- [ ] **Step 1: Rewrite `QhorusEntityMapper` — Option B**

Replace the entire class body. Remove `ChannelBindingStore` field and constructor parameter. Add `Optional<ChannelConnectorBinding>` parameter to `toChannelDetail`.

```java
package io.casehub.qhorus.runtime;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Optional;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.qhorus.api.channel.ChannelDetail;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.message.Message;

@ApplicationScoped
public class QhorusEntityMapper {

    @Inject
    ObjectMapper mapper;

    QhorusEntityMapper() {}

    public QhorusEntityMapper(ObjectMapper mapper) {
        this.mapper = mapper;
    }

    public ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                         Optional<ChannelConnectorBinding> binding) {
        ChannelDetail.ConnectorBinding detailBinding = binding
                .map(b -> new ChannelDetail.ConnectorBinding(
                        b.inboundConnectorId, b.externalKey,
                        b.outboundConnectorId, b.outboundDestination))
                .orElse(null);
        return new ChannelDetail(
                ch.id, ch.name, ch.description,
                ch.semantic != null ? ch.semantic.name() : null,
                ch.barrierContributors, messageCount,
                ch.lastActivityAt != null ? ch.lastActivityAt.toString() : null,
                ch.paused, ch.allowedWriters, ch.adminInstances,
                ch.rateLimitPerChannel, ch.rateLimitPerInstance, ch.allowedTypes,
                detailBinding);
    }

    public Map<String, Object> toTimelineEntry(Message m) {
        Map<String, Object> entry = new LinkedHashMap<>();
        entry.put("id", m.id);
        if (m.messageType == MessageType.EVENT) {
            entry.put("type", "EVENT");
            entry.put("created_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("occurred_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("agent_id", m.sender);
            entry.put("message_type", null);
            String toolName = null;
            Long durationMs = null;
            Long tokenCount = null;
            if (m.content != null) {
                try {
                    JsonNode node = mapper.readTree(m.content);
                    JsonNode tn = node.get("tool_name");
                    if (tn != null && tn.isTextual()) toolName = tn.asText();
                    JsonNode dm = node.get("duration_ms");
                    if (dm != null && dm.isNumber()) durationMs = dm.asLong();
                    JsonNode tc = node.get("token_count");
                    if (tc != null && tc.isNumber()) tokenCount = tc.asLong();
                } catch (Exception ignored) {
                }
            }
            entry.put("tool_name", toolName);
            entry.put("duration_ms", durationMs);
            entry.put("token_count", tokenCount);
        } else {
            entry.put("type", "MESSAGE");
            entry.put("created_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("sender", m.sender);
            entry.put("message_type", m.messageType != null ? m.messageType.name().toLowerCase() : null);
            entry.put("content", m.content);
            entry.put("correlation_id", m.correlationId);
            entry.put("tool_name", null);
        }
        return entry;
    }
}
```

- [ ] **Step 2: Update `QhorusMcpToolsBase` — add `ChannelBindingStore` + two `toChannelDetail` overloads**

Add these imports at the top of `QhorusMcpToolsBase.java`:
```java
import java.util.Optional;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;
```

Add this field after the existing `@Inject QhorusEntityMapper entityMapper`:
```java
@Inject
ChannelBindingStore bindingStore;
```

Replace the existing `protected ChannelDetail toChannelDetail(Channel ch, long messageCount)` method with two overloads:

```java
/** Single-item path — looks up binding by channel ID. Used by all tool call sites except list_channels. */
protected ChannelDetail toChannelDetail(Channel ch, long messageCount) {
    return entityMapper.toChannelDetail(ch, messageCount,
            bindingStore.findByChannelId(ch.id));
}

/** Batch path — caller pre-loads all bindings; used by list_channels to avoid N+1 queries. */
protected ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                        Map<UUID, ChannelConnectorBinding> allBindings) {
    return entityMapper.toChannelDetail(ch, messageCount,
            Optional.ofNullable(allBindings.get(ch.id)));
}
```

- [ ] **Step 3: Update `QhorusDashboardService` — remove private mapping, add store, fix `listChannels()`**

Add these imports to `QhorusDashboardService.java`:
```java
import java.util.Optional;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;
import io.smallrye.mutiny.infrastructure.Infrastructure;
```

Add this field (the `entityMapper` injection is already present):
```java
@Inject io.casehub.qhorus.runtime.store.ChannelBindingStore bindingStore;
```

Replace `listChannels()` with the worker-pool–safe version:

```java
public Uni<List<ChannelDetail>> listChannels() {
    return Uni.createFrom().item(bindingStore::findAll)
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
            .flatMap(bindings -> channelService.listAll().flatMap(channels -> {
                if (channels.isEmpty()) return Uni.createFrom().item(List.of());
                List<Uni<ChannelDetail>> unis = channels.stream()
                        .map(ch -> messageStore.countByChannel(ch.id)
                                .map(count -> entityMapper.toChannelDetail(ch, count,
                                        Optional.ofNullable(bindings.get(ch.id)))))
                        .toList();
                return Uni.join().all(unis).andFailFast();
            }));
}
```

Delete the private `toChannelDetail(Channel ch, int count)` method entirely (roughly lines 145–159 in the original file).

- [ ] **Step 4: Fix `QhorusDashboardServiceTest` `setUp()`**

The `setUp()` method currently constructs `new QhorusEntityMapper(new ObjectMapper(), bindingStore)`. After the refactor:
1. Stub `bindingStore.findAll()` so `listChannels()` doesn't NPE
2. Remove `bindingStore` from the mapper constructor call
3. Wire `service.bindingStore`

Replace the `setUp()` body (lines 51–62) with:

```java
@BeforeEach
void setUp() {
    service = new QhorusDashboardService();
    service.channelService = channelService;
    service.instanceService = instanceService;
    service.messageService = messageService;
    service.messageStore = messageStore;
    io.casehub.qhorus.runtime.store.ChannelBindingStore bindingStore =
            mock(io.casehub.qhorus.runtime.store.ChannelBindingStore.class);
    when(bindingStore.findAll()).thenReturn(Map.of());
    when(bindingStore.findByChannelId(any())).thenReturn(Optional.empty());
    service.bindingStore = bindingStore;
    service.entityMapper = new QhorusEntityMapper(new ObjectMapper());
    reset(channelService, instanceService, messageService, messageStore);
}
```

- [ ] **Step 5: Build and run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests pass. If `QhorusDashboardServiceTest` fails on `listChannels` tests, check that `Infrastructure.getDefaultWorkerPool()` is properly available — it's from `io.smallrye.mutiny:mutiny` which is already on the classpath.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java

git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
refactor(#219): Option B mapper — caller supplies binding, QhorusMcpToolsBase owns lookup

QhorusEntityMapper.toChannelDetail now takes Optional<ChannelConnectorBinding> instead of
querying the store itself. ChannelBindingStore moves to QhorusMcpToolsBase (shared by both
blocking and reactive tool classes). QhorusDashboardService.listChannels loads bindings via
runSubscriptionOn(worker-pool) to avoid blocking the Vert.x event loop.

Refs #219
EOF
)"
```

---

## Task 3: Update `list_channels` for Batch Binding Path

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Update `QhorusMcpTools.listChannels()`**

Add import at top if not present:
```java
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
```

Find the `listChannels()` method (line ~287). The current body:
```java
List<Channel> channels = channelService.listAll();
if (channels.isEmpty()) {
    return List.of();
}
Map<UUID, Long> countByChannel = messageStore.countAllByChannel();
return channels.stream()
        .map(ch -> toChannelDetail(ch, countByChannel.getOrDefault(ch.id, 0L)))
        .toList();
```

Replace with:
```java
List<Channel> channels = channelService.listAll();
if (channels.isEmpty()) {
    return List.of();
}
Map<UUID, Long> countByChannel = messageStore.countAllByChannel();
Map<UUID, ChannelConnectorBinding> allBindings = bindingStore.findAll();
return channels.stream()
        .map(ch -> toChannelDetail(ch, countByChannel.getOrDefault(ch.id, 0L), allBindings))
        .toList();
```

`bindingStore` is inherited from `QhorusMcpToolsBase` — no new injection needed.

- [ ] **Step 2: Update `ReactiveQhorusMcpTools.listChannels()`**

Add imports at top if not present:
```java
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.smallrye.mutiny.infrastructure.Infrastructure;
```

Find the `listChannels()` method (line ~245). The current body:
```java
return channelService.listAll().flatMap(channels -> {
    if (channels.isEmpty()) {
        return Uni.createFrom().item(List.of());
    }
    List<Uni<ChannelDetail>> unis = channels.stream()
            .map(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())))
            .toList();
    return Uni.join().all(unis).andFailFast();
});
```

Replace with:
```java
return Uni.createFrom().item(bindingStore::findAll)
        .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
        .flatMap(allBindings -> channelService.listAll().flatMap(channels -> {
            if (channels.isEmpty()) {
                return Uni.createFrom().item(List.of());
            }
            List<Uni<ChannelDetail>> unis = channels.stream()
                    .map(ch -> messageStore.countByChannel(ch.id)
                            .map(count -> toChannelDetail(ch, count.longValue(), allBindings)))
                    .toList();
            return Uni.join().all(unis).andFailFast();
        }));
```

`bindingStore` is inherited from `QhorusMcpToolsBase`. `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` is required because `bindingStore.findAll()` is a blocking JPA call and `list_channels` MCP tools return `Uni<T>` without a `@Blocking` annotation.

- [ ] **Step 3: Build and run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java

git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(#219): list_channels pre-loads binding map — eliminates N+1 findByChannelId per channel

Refs #219
EOF
)"
```

---

## Task 4: Fix Integration Test — Remove `@ObservesAsync` Flakiness

**File:** `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendIntegrationTest.java`

GE-20260513-b15933 documents that `@ObservesAsync` events are silently not delivered in `@QuarkusTest`. `ConnectorChannelBackend.onChannelInitialised` is `@Observes` (sync — safe), but `onInboundMessage` is `@ObservesAsync`. Tests must call `backend.onInboundMessage(msg)` directly through the CDI proxy instead of firing through `InboundConnectorService.receive()`. The CDI async wiring gap is tracked as #221.

- [ ] **Step 1: Rewrite the integration test**

Replace the entire file content:

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;
import io.casehub.connectors.InboundMessage;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.testing.InMemoryChannelBindingStore;
import io.casehub.qhorus.testing.InMemoryChannelStore;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class ConnectorChannelBackendIntegrationTest {

    @Inject ConnectorChannelBackend backend;
    @Inject ChannelService channelService;
    @Inject ChannelGateway gateway;
    @Inject InMemoryChannelStore channelStore;
    @Inject InMemoryChannelBindingStore channelBindingStore;

    @InjectMock MessageService messageService;
    @InjectMock ConnectorService connectorService;

    private UUID channelId;

    @BeforeEach
    void setUp() {
        channelStore.clear();
        channelBindingStore.clear();

        Channel ch = channelService.create(new ChannelCreateRequest(
                "sms-alice", "Alice's SMS conversation", ChannelSemantic.APPEND,
                null, null, null, null, null, null,
                "twilio-sms-inbound", "+15551110000", "twilio-sms", "+15551110000"));
        channelId = ch.id;
        // initChannel fires @Observes ChannelInitialisedEvent synchronously —
        // ConnectorChannelBackend.onChannelInitialised populates cache before setUp returns.
        gateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name));
    }

    @AfterEach
    void tearDown() {
        gateway.closeChannel(channelId, new ChannelRef(channelId, "sms-alice"));
    }

    @Test
    void inboundMessage_routesToMessageService() {
        InboundMessage msg = new InboundMessage("twilio-sms-inbound", "+15551110000",
                "+14155552671", "I need help", Instant.now(), Map.of());

        // Call through CDI proxy — synchronous; no async waiting required.
        backend.onInboundMessage(msg);

        verify(messageService).dispatch(argThat(d ->
                d.channelId().equals(channelId)
                && "human:+15551110000".equals(d.sender())
                && "I need help".equals(d.content())));
    }

    @Test
    void unknownSender_noChannelBound_discardCounterIncremented() {
        double before = backend.discardedCount("twilio-sms-inbound");

        InboundMessage msg = new InboundMessage("twilio-sms-inbound", "+99999",
                "+14155552671", "hello", Instant.now(), Map.of());
        backend.onInboundMessage(msg);

        verify(messageService, never()).dispatch(any());
        assertThat(backend.discardedCount("twilio-sms-inbound")).isGreaterThan(before);
    }

    @Test
    void fanOut_sendsViaConnectorService() {
        // Cache already populated by @BeforeEach → gateway.initChannel() → @Observes (sync).
        OutboundMessage outbound = new OutboundMessage(UUID.randomUUID(), "agent",
                MessageType.RESPONSE, "We can help", null, null, ActorType.AGENT);
        gateway.fanOut(channelId, "sms-alice", outbound);

        // timeout required — ChannelGateway.fanOut() dispatches backend.post() on a virtual thread.
        ArgumentCaptor<ConnectorMessage> captor = ArgumentCaptor.forClass(ConnectorMessage.class);
        verify(connectorService, timeout(1000).atLeastOnce()).send(eq("twilio-sms"), captor.capture());
        assertThat(captor.getValue().destination()).isEqualTo("+15551110000");
        assertThat(captor.getValue().body()).isEqualTo("We can help");
    }

    @Test
    void duplicateBinding_throws() {
        assertThatThrownBy(() ->
            channelService.create(new ChannelCreateRequest(
                "sms-bob", "Bob's channel", ChannelSemantic.APPEND,
                null, null, null, null, null, null,
                "twilio-sms-inbound", "+15551110000",   // same key as alice — should conflict
                "twilio-sms", "+15551110000"))
        ).isInstanceOf(IllegalStateException.class)
         .hasMessageContaining("Connector binding already exists");
    }
}
```

- [ ] **Step 2: Run connector-backend tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connector-backend
```
Expected: all 4 integration tests + 3 unit test classes pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackendIntegrationTest.java

git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
fix(#219): call backend.onInboundMessage() directly in integration test — eliminates @ObservesAsync flakiness

GE-20260513-b15933: @ObservesAsync not reliably delivered in @QuarkusTest.
Call the CDI proxy directly for synchronous, deterministic test execution.
fanOut still uses timeout(1000) — ChannelGateway dispatches post() on a virtual thread.
CDI async wiring gap tracked as #221.

Refs #219
EOF
)"
```

---

## Task 5: Add `NativeImageResourcePatternsBuildItem` to `QhorusProcessor`

**File:** `deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java`

Per PP-20260528-flyway-ext-reg. `LedgerProcessor` in `casehub-ledger-deployment` does **not** register `db/ledger/migration/*.sql` for native image — Qhorus must do it here or ledger migrations are absent in native builds.

- [ ] **Step 1: Update `QhorusProcessor`**

Add import:
```java
import io.quarkus.deployment.builditem.nativeimage.NativeImageResourcePatternsBuildItem;
```

Add build step inside the class (after the existing `feature()` method):

```java
@BuildStep
NativeImageResourcePatternsBuildItem registerMigrationResources() {
    return NativeImageResourcePatternsBuildItem.builder()
            .includeGlob("db/qhorus/migration/*.sql")
            .includeGlob("db/ledger/migration/*.sql")
            .build();
}
```

- [ ] **Step 2: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,runtime,deployment -DskipTests
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java

git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
fix(#219): register Flyway SQL files for native image in QhorusProcessor

LedgerProcessor does not self-register db/ledger/migration/*.sql. Both globs required
to include qhorus domain migrations and ledger base migrations in native builds.
PP-20260528-flyway-ext-reg.

Refs #219
EOF
)"
```

---

## Task 6: Update PLATFORM.md and Protocol

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/protocols/casehub/cross-foundation-bridge-module-placement.md`

- [ ] **Step 1: Add row to PLATFORM.md Cross-Repo Dependency Map**

Locate the existing row:
```
| `casehub-connectors-core` | `casehub-qhorus` | `connectors` | optional — `WatchdogAlertEvent → ConnectorService.send()` bridge; activates by classpath presence |
```

Add the new row immediately after it:
```
| `casehub-connectors-core` | `casehub-qhorus` | `connector-backend` | optional — `InboundMessage` CDI events → `ConnectorChannelBackend` → Qhorus channel routing; activates by classpath presence |
```

- [ ] **Step 2: Update bridge module protocol with runtime-coupling exception**

Open `cross-foundation-bridge-module-placement.md`. At the end of the existing rule paragraph (after the sentence ending "…without their own restart recovery logic."), add a new paragraph:

```markdown

**Exception — runtime coupling:** If the bridge requires the consuming repo's runtime beans (not just its `api` module), placing it in the event-source repo creates a circular dependency. In that case the bridge lives in the **consumer's repo**. `casehub-qhorus/connector-backend` is the canonical example: it depends on `ChannelGateway`, `ChannelService`, and `ChannelBindingStore` from qhorus runtime; moving it to `casehub-connectors` would require qhorus runtime as a dep of connectors, which already depends on connectors.
```

- [ ] **Step 3: Commit to casehub-parent**

```bash
git -C /Users/mdproctor/claude/casehub/parent add \
  docs/PLATFORM.md \
  docs/protocols/casehub/cross-foundation-bridge-module-placement.md

git -C /Users/mdproctor/claude/casehub/parent commit -m "$(cat <<'EOF'
docs(qhorus): register casehub-qhorus connector-backend in cross-repo dep map

Also documents the runtime-coupling exception to PP-20260528-6b1d80 (bridge module placement):
when the bridge depends on the consuming repo's runtime, it lives in the consumer's repo.

Refs casehubio/qhorus#219
EOF
)"
```

---

## Task 7: Full Build and Test Verification

- [ ] **Step 1: Run the complete build from the project root**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
Expected: BUILD SUCCESS across all modules — api, connectors, runtime, connector-backend, deployment, testing, examples.

If any module fails, fix the issue before continuing. Common failure causes:
- Import not added (check the specific class)
- `ChannelConnectorBinding` not imported in `QhorusMcpToolsBase` or `ReactiveQhorusMcpTools`
- Missing `Map` import in `ChannelBindingStore` interface

- [ ] **Step 2: Push the branch**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus push -u origin issue-219-connector-channel-backend
```

- [ ] **Step 3: Confirm all tasks complete**

After the push, verify the branch has the expected commits:
```bash
git -C /Users/mdproctor/claude/casehub/qhorus log --oneline origin/main..HEAD
```
Expected: 7 commits (2 pre-existing + 5 from this plan).

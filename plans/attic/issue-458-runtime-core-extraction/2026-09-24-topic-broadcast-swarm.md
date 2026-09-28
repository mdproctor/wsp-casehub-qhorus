# Topic-Based Broadcast for Swarm Coordination — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #442 — feat: Topic-based broadcast for swarm coordination — hive mind enabler
**Issue group:** #442

**Goal:** Add `ChannelSemantic.BROADCAST` with capability-driven auto-membership so agents can fan-out coordination hints to all peers with a given capability, using the platform notification system as a delivery backend.

**Architecture:** New `BROADCAST` channel semantic restricts to STATUS + EVENT message types (no commitments). `BroadcastMembershipManager` observes CDI events fired from `InstanceService` and auto-creates `broadcast:<capability>` channels, joining/leaving agents as capabilities change. `NotificationChannelBackend` in `notification-bridge/` delivers broadcast messages through the platform notification system.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI async events, existing channel/membership infrastructure

## Global Constraints

- Channel slugs: `broadcast:<capability>` convention — colon is valid in slugs
- BROADCAST channels: `allowedTypes = {STATUS, EVENT}` — hard-enforced by MessageTypePolicy
- No new database tables or columns — BROADCAST is a new ChannelSemantic enum value only
- `notification-bridge/` gains new classes but no new Maven dependencies (casehub-qhorus-api + casehub-platform-api already present)
- CDI events are `@ObservesAsync` — non-blocking, independent transaction context

---

## Batch 1: BROADCAST semantic + auto-membership

### Task 1: ChannelSemantic.BROADCAST + CDI events

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelSemantic.java:3`
- Create: `api/src/main/java/io/casehub/qhorus/api/instance/InstanceRegisteredEvent.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/instance/InstanceDeregisteredEvent.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java:34-55` (register method)
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java:86-89` (deregister method)
- Test: `runtime/src/test/java/io/casehub/qhorus/instance/InstanceServiceEventTest.java`

**Interfaces:**
- Produces: `ChannelSemantic.BROADCAST` enum value
- Produces: `InstanceRegisteredEvent(String instanceId, List<String> previousCapabilities, List<String> currentCapabilities)` CDI event
- Produces: `InstanceDeregisteredEvent(String instanceId, List<String> capabilities)` CDI event

- [ ] **Step 1: Add BROADCAST to ChannelSemantic**

Use `ide_edit_member` to add the new enum constant:

```java
/** Fan-out to all members. Fire-and-forget coordination signals. STATUS + EVENT only. */
BROADCAST
```

Add after `LAST_WRITE` in `ChannelSemantic.java`.

- [ ] **Step 2: Create InstanceRegisteredEvent record**

Create file `api/src/main/java/io/casehub/qhorus/api/instance/InstanceRegisteredEvent.java`:

```java
package io.casehub.qhorus.api.instance;

import java.util.List;

public record InstanceRegisteredEvent(
        String instanceId,
        List<String> previousCapabilities,
        List<String> currentCapabilities) {

    public InstanceRegisteredEvent {
        previousCapabilities = previousCapabilities != null ? List.copyOf(previousCapabilities) : List.of();
        currentCapabilities = currentCapabilities != null ? List.copyOf(currentCapabilities) : List.of();
    }
}
```

- [ ] **Step 3: Create InstanceDeregisteredEvent record**

Create file `api/src/main/java/io/casehub/qhorus/api/instance/InstanceDeregisteredEvent.java`:

```java
package io.casehub.qhorus.api.instance;

import java.util.List;

public record InstanceDeregisteredEvent(
        String instanceId,
        List<String> capabilities) {

    public InstanceDeregisteredEvent {
        capabilities = capabilities != null ? List.copyOf(capabilities) : List.of();
    }
}
```

- [ ] **Step 4: Write failing test for CDI events from InstanceService**

Create `runtime/src/test/java/io/casehub/qhorus/instance/InstanceServiceEventTest.java`:

```java
package io.casehub.qhorus.instance;

import io.casehub.qhorus.api.instance.InstanceDeregisteredEvent;
import io.casehub.qhorus.api.instance.InstanceRegisteredEvent;
import io.casehub.qhorus.persistence.memory.InMemoryInstanceStore;
import io.casehub.qhorus.runtime.instance.InstanceService;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class InstanceServiceEventTest {

    private InstanceService service;
    private Event<InstanceRegisteredEvent> registeredEvent;
    private Event<InstanceDeregisteredEvent> deregisteredEvent;
    private InMemoryInstanceStore store;

    @SuppressWarnings("unchecked")
    @BeforeEach
    void setUp() {
        store = new InMemoryInstanceStore();
        registeredEvent = mock(Event.class);
        deregisteredEvent = mock(Event.class);
        service = new InstanceService(store);
        service.registeredEvent = registeredEvent;
        service.deregisteredEvent = deregisteredEvent;
    }

    @Test
    void register_firesEventWithCapabilities() {
        service.register("agent-1", "Test agent", List.of("analyzer", "monitor"));

        ArgumentCaptor<InstanceRegisteredEvent> captor = ArgumentCaptor.forClass(InstanceRegisteredEvent.class);
        verify(registeredEvent).fireAsync(captor.capture());

        InstanceRegisteredEvent event = captor.getValue();
        assertThat(event.instanceId()).isEqualTo("agent-1");
        assertThat(event.previousCapabilities()).isEmpty();
        assertThat(event.currentCapabilities()).containsExactlyInAnyOrder("analyzer", "monitor");
    }

    @Test
    void register_existingAgent_firesDiffCapabilities() {
        service.register("agent-1", "Test agent", List.of("analyzer"));
        reset(registeredEvent);

        service.register("agent-1", "Updated agent", List.of("analyzer", "monitor"));

        ArgumentCaptor<InstanceRegisteredEvent> captor = ArgumentCaptor.forClass(InstanceRegisteredEvent.class);
        verify(registeredEvent).fireAsync(captor.capture());

        InstanceRegisteredEvent event = captor.getValue();
        assertThat(event.previousCapabilities()).containsExactly("analyzer");
        assertThat(event.currentCapabilities()).containsExactlyInAnyOrder("analyzer", "monitor");
    }

    @Test
    void deregister_firesEventWithCapabilities() {
        service.register("agent-1", "Test agent", List.of("analyzer"));
        reset(registeredEvent);

        service.deregister("agent-1");

        ArgumentCaptor<InstanceDeregisteredEvent> captor = ArgumentCaptor.forClass(InstanceDeregisteredEvent.class);
        verify(deregisteredEvent).fireAsync(captor.capture());

        InstanceDeregisteredEvent event = captor.getValue();
        assertThat(event.instanceId()).isEqualTo("agent-1");
        assertThat(event.capabilities()).containsExactly("analyzer");
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceServiceEventTest -pl runtime -Dno-format`
Expected: FAIL — `registeredEvent` and `deregisteredEvent` fields don't exist on InstanceService

- [ ] **Step 6: Modify InstanceService to fire CDI events**

In `InstanceService.java`, add two `Event` fields (package-private for test injection):

```java
@Inject
Event<InstanceRegisteredEvent> registeredEvent;

@Inject
Event<InstanceDeregisteredEvent> deregisteredEvent;
```

In the 5-arg `register()` method, before the return, capture previous capabilities and fire:

```java
List<String> previousCaps = existing != null
        ? instanceStore.findCapabilities(existing.id())
        : List.of();

// ... existing register logic ...

if (registeredEvent != null) {
    registeredEvent.fireAsync(new InstanceRegisteredEvent(
            instanceId, previousCaps, capabilityTags));
}
```

In `deregister()`, capture capabilities before deleting, then fire:

```java
public void deregister(String instanceId) {
    instanceStore.findByInstanceId(instanceId)
            .ifPresent(inst -> {
                List<String> caps = instanceStore.findCapabilities(inst.id());
                instanceStore.delete(inst.id());
                if (deregisteredEvent != null) {
                    deregisteredEvent.fireAsync(new InstanceDeregisteredEvent(instanceId, caps));
                }
            });
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceServiceEventTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 8: Run full test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format`
Expected: All tests pass

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/main/java/io/casehub/qhorus/api/channel/ChannelSemantic.java api/src/main/java/io/casehub/qhorus/api/instance/ runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java runtime/src/test/java/io/casehub/qhorus/instance/InstanceServiceEventTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#442): add ChannelSemantic.BROADCAST + CDI events from InstanceService

Add BROADCAST to ChannelSemantic enum for fire-and-forget fan-out channels.
Fire InstanceRegisteredEvent (with capability diff) and InstanceDeregisteredEvent
from InstanceService.register() and deregister() for downstream auto-membership.

Refs #442"
```

---

### Task 2: BroadcastMembershipManager — auto-join/leave on capability changes

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/broadcast/BroadcastMembershipManager.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/broadcast/BroadcastMembershipManagerTest.java`

**Interfaces:**
- Consumes: `InstanceRegisteredEvent(String instanceId, List<String> previousCapabilities, List<String> currentCapabilities)` from Task 1
- Consumes: `InstanceDeregisteredEvent(String instanceId, List<String> capabilities)` from Task 1
- Consumes: `ChannelService.findOrCreate(ChannelCreateRequest)` → `FindOrCreateResult`
- Consumes: `ChannelMembershipService.join(UUID channelId, String memberId, MemberRole role, String tenancyId)` → `ChannelMembership`
- Consumes: `ChannelMembershipService.leave(UUID channelId, String memberId)`
- Consumes: `ChannelStore.findByName(String name)` → `Optional<Channel>`
- Produces: broadcast channels named `broadcast:<capability>` with `ChannelSemantic.BROADCAST` and `allowedTypes = {STATUS, EVENT}`

- [ ] **Step 1: Write failing test for BroadcastMembershipManager**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/broadcast/BroadcastMembershipManagerTest.java`:

```java
package io.casehub.qhorus.runtime.broadcast;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelMembership;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.channel.FindOrCreateResult;
import io.casehub.qhorus.api.channel.MemberRole;
import io.casehub.qhorus.api.instance.InstanceDeregisteredEvent;
import io.casehub.qhorus.api.instance.InstanceRegisteredEvent;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelMembershipService;
import io.casehub.qhorus.runtime.channel.ChannelService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class BroadcastMembershipManagerTest {

    private BroadcastMembershipManager manager;
    private ChannelService channelService;
    private ChannelMembershipService membershipService;

    @BeforeEach
    void setUp() {
        channelService = mock(ChannelService.class);
        membershipService = mock(ChannelMembershipService.class);
        manager = new BroadcastMembershipManager(channelService, membershipService);
    }

    @Test
    void onRegistered_createsChannelAndJoins() {
        UUID channelId = UUID.randomUUID();
        Channel channel = stubChannel(channelId, "broadcast:analyzer");
        when(channelService.findOrCreate(any())).thenReturn(new FindOrCreateResult(channel, true));
        when(membershipService.join(any(), any(), any(), any()))
                .thenReturn(stubMembership(channelId, "agent-1"));

        manager.onRegistered(new InstanceRegisteredEvent("agent-1", List.of(), List.of("analyzer")));

        verify(channelService).findOrCreate(argThat(req ->
                req.name().equals("broadcast:analyzer")
                && req.semantic() == ChannelSemantic.BROADCAST
                && req.allowedTypes().equals(Set.of(MessageType.STATUS, MessageType.EVENT))));
        verify(membershipService).join(eq(channelId), eq("agent-1"), eq(MemberRole.PARTICIPANT), any());
    }

    @Test
    void onRegistered_removedCapability_leavesChannel() {
        UUID channelId = UUID.randomUUID();
        Channel channel = stubChannel(channelId, "broadcast:analyzer");
        when(channelService.findByName("broadcast:analyzer")).thenReturn(Optional.of(channel));

        manager.onRegistered(new InstanceRegisteredEvent(
                "agent-1", List.of("analyzer", "monitor"), List.of("monitor")));

        verify(membershipService).leave(channelId, "agent-1");
        verify(channelService, never()).findOrCreate(argThat(req -> req.name().equals("broadcast:analyzer")));
    }

    @Test
    void onRegistered_unchangedCapability_noop() {
        manager.onRegistered(new InstanceRegisteredEvent(
                "agent-1", List.of("analyzer"), List.of("analyzer")));

        verifyNoInteractions(channelService);
        verifyNoInteractions(membershipService);
    }

    @Test
    void onDeregistered_leavesAllBroadcastChannels() {
        UUID channelId = UUID.randomUUID();
        Channel channel = stubChannel(channelId, "broadcast:analyzer");
        when(channelService.findByName("broadcast:analyzer")).thenReturn(Optional.of(channel));

        manager.onDeregistered(new InstanceDeregisteredEvent("agent-1", List.of("analyzer")));

        verify(membershipService).leave(channelId, "agent-1");
    }

    private Channel stubChannel(UUID id, String name) {
        return Channel.builder(name).id(id).semantic(ChannelSemantic.BROADCAST)
                .tenancyId("default").createdAt(Instant.now()).lastActivityAt(Instant.now()).build();
    }

    private ChannelMembership stubMembership(UUID channelId, String memberId) {
        return new ChannelMembership(UUID.randomUUID(), channelId, memberId,
                MemberRole.PARTICIPANT, "default", Instant.now(), null, null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=BroadcastMembershipManagerTest -pl runtime -Dno-format`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement BroadcastMembershipManager**

Create `runtime-core/src/main/java/io/casehub/qhorus/runtime/broadcast/BroadcastMembershipManager.java`:

```java
package io.casehub.qhorus.runtime.broadcast;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.channel.FindOrCreateResult;
import io.casehub.qhorus.api.channel.MemberRole;
import io.casehub.qhorus.api.instance.InstanceDeregisteredEvent;
import io.casehub.qhorus.api.instance.InstanceRegisteredEvent;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelMembershipService;
import io.casehub.qhorus.runtime.channel.ChannelService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.HashSet;
import java.util.List;
import java.util.Optional;
import java.util.Set;

import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;

@ApplicationScoped
public class BroadcastMembershipManager {

    static final String BROADCAST_PREFIX = "broadcast:";
    private static final Logger LOG = Logger.getLogger(BroadcastMembershipManager.class);

    private final ChannelService channelService;
    private final ChannelMembershipService membershipService;

    @Inject
    public BroadcastMembershipManager(ChannelService channelService,
                                       ChannelMembershipService membershipService) {
        this.channelService = channelService;
        this.membershipService = membershipService;
    }

    void onRegistered(@ObservesAsync InstanceRegisteredEvent event) {
        Set<String> added = new HashSet<>(event.currentCapabilities());
        added.removeAll(event.previousCapabilities());

        Set<String> removed = new HashSet<>(event.previousCapabilities());
        removed.removeAll(event.currentCapabilities());

        for (String cap : added) {
            joinBroadcastChannel(event.instanceId(), cap);
        }
        for (String cap : removed) {
            leaveBroadcastChannel(event.instanceId(), cap);
        }
    }

    void onDeregistered(@ObservesAsync InstanceDeregisteredEvent event) {
        for (String cap : event.capabilities()) {
            leaveBroadcastChannel(event.instanceId(), cap);
        }
    }

    private void joinBroadcastChannel(String instanceId, String capability) {
        try {
            ChannelCreateRequest req = ChannelCreateRequest.builder(BROADCAST_PREFIX + capability)
                    .semantic(ChannelSemantic.BROADCAST)
                    .description("Broadcast channel for capability: " + capability)
                    .allowedTypes(Set.of(MessageType.STATUS, MessageType.EVENT))
                    .build();
            FindOrCreateResult result = channelService.findOrCreate(req);
            Channel channel = result.channel();
            membershipService.join(channel.id(), instanceId, MemberRole.PARTICIPANT, DEFAULT_TENANT_ID);
            LOG.debugf("Agent %s joined broadcast:%s (channel %s)", instanceId, capability, channel.id());
        } catch (Exception e) {
            LOG.warnf("Failed to join broadcast:%s for agent %s: %s", capability, instanceId, e.getMessage());
        }
    }

    private void leaveBroadcastChannel(String instanceId, String capability) {
        try {
            Optional<Channel> channel = channelService.findByName(BROADCAST_PREFIX + capability);
            channel.ifPresent(ch -> {
                membershipService.leave(ch.id(), instanceId);
                LOG.debugf("Agent %s left broadcast:%s (channel %s)", instanceId, capability, ch.id());
            });
        } catch (Exception e) {
            LOG.warnf("Failed to leave broadcast:%s for agent %s: %s", capability, instanceId, e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=BroadcastMembershipManagerTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime-core/src/main/java/io/casehub/qhorus/runtime/broadcast/ runtime/src/test/java/io/casehub/qhorus/runtime/broadcast/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#442): BroadcastMembershipManager — auto-join/leave on capability changes

Observes InstanceRegisteredEvent and InstanceDeregisteredEvent. Diffs
capabilities, auto-creates broadcast:<capability> channels via findOrCreate,
joins/leaves agents as members.

Refs #442"
```

---

### Task 3: MessageDispatcher.broadcast() convenience method

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java:3-5`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (add broadcast impl)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/broadcast/BroadcastDispatchTest.java`

**Interfaces:**
- Consumes: `ChannelStore.findByName(String)` → `Optional<Channel>`
- Consumes: `MessageDispatcher.dispatch(MessageDispatch)` → `DispatchResult`
- Produces: `MessageDispatcher.broadcast(String capabilityTag, MessageType type, String content, String sender, String tenancyId)` → `DispatchResult`

- [ ] **Step 1: Write failing test for broadcast dispatch**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/broadcast/BroadcastDispatchTest.java`:

```java
package io.casehub.qhorus.runtime.broadcast;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.ChannelStore;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class BroadcastDispatchTest {

    @Test
    void broadcast_resolvesChannelAndDispatches() {
        ChannelStore channelStore = mock(ChannelStore.class);
        MessageDispatcher dispatcher = mock(MessageDispatcher.class);

        UUID channelId = UUID.randomUUID();
        Channel channel = Channel.builder("broadcast:analyzer").id(channelId)
                .semantic(ChannelSemantic.BROADCAST)
                .tenancyId("default").createdAt(Instant.now()).lastActivityAt(Instant.now()).build();
        when(channelStore.findByName("broadcast:analyzer")).thenReturn(Optional.of(channel));

        DispatchResult expected = new DispatchResult(1L, channelId, "agent-1",
                MessageType.STATUS, null, null, null, null, null, null, null, 0, null);
        when(dispatcher.dispatch(any())).thenReturn(expected);

        DispatchResult result = dispatcher.broadcast("analyzer", MessageType.STATUS,
                "anomaly detected", "agent-1", "default");

        assertThat(result.channelId()).isEqualTo(channelId);
    }

    @Test
    void broadcast_unknownCapability_throws() {
        ChannelStore channelStore = mock(ChannelStore.class);
        MessageDispatcher dispatcher = mock(MessageDispatcher.class);

        when(channelStore.findByName("broadcast:unknown")).thenReturn(Optional.empty());

        assertThatThrownBy(() ->
                dispatcher.broadcast("unknown", MessageType.STATUS, "test", "agent-1", "default"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=BroadcastDispatchTest -pl runtime -Dno-format`
Expected: FAIL — `broadcast` method does not exist on MessageDispatcher

- [ ] **Step 3: Add broadcast() default method to MessageDispatcher**

In `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java`, add a default method:

```java
package io.casehub.qhorus.api.message;

public interface MessageDispatcher {
    DispatchResult dispatch(MessageDispatch dispatch);

    default DispatchResult broadcast(String capabilityTag, MessageType type, String content,
                                      String sender, String tenancyId) {
        throw new UnsupportedOperationException("broadcast not implemented");
    }
}
```

- [ ] **Step 4: Implement broadcast() in MessageService**

In `MessageService`, override the `broadcast()` method:

```java
@Override
public DispatchResult broadcast(String capabilityTag, MessageType type, String content,
                                 String sender, String tenancyId) {
    String channelName = "broadcast:" + capabilityTag;
    Channel channel = channelStore.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException(
                    "No broadcast channel for capability: " + capabilityTag));
    MessageDispatch dispatch = MessageDispatch.builder()
            .channelId(channel.id())
            .sender(sender)
            .type(type)
            .content(content)
            .tenancyId(tenancyId)
            .build();
    return dispatch(dispatch);
}
```

- [ ] **Step 5: Update test to use real implementation path**

Update `BroadcastDispatchTest` to test against the actual `MessageService.broadcast()` using mocked stores rather than mocking `MessageDispatcher` directly. Or keep as an integration test that verifies end-to-end. The default method test is sufficient for interface-level verification — integration coverage comes from the full `@QuarkusTest` suite.

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=BroadcastDispatchTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format`
Expected: All tests pass

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java runtime/src/test/java/io/casehub/qhorus/runtime/broadcast/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#442): MessageDispatcher.broadcast() convenience method

Resolves broadcast:<capability> channel by name and dispatches STATUS/EVENT
through the standard MessageService.dispatch() pipeline.

Refs #442"
```

---

## Batch 2: Notification delivery backend

### Task 4: NotificationChannelBackend + QhorusBroadcastEvent in notification-bridge

**Files:**
- Create: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusBroadcastEvent.java`
- Create: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/NotificationChannelBackend.java`
- Modify: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusSubscriptionBootstrap.java` (add broadcast subscription)
- Test: `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/NotificationChannelBackendTest.java`
- Test: `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/QhorusBroadcastEventTest.java`

**Interfaces:**
- Consumes: `AgentChannelBackend` SPI from `casehub-qhorus-api`
- Consumes: `ChannelInitialisedEvent(UUID channelId, String channelName, boolean recovered)` CDI event
- Consumes: `ChannelStore.find(UUID)` → `Optional<Channel>` (to check semantic is BROADCAST)
- Consumes: `ChannelMembershipStore.findByChannelId(UUID)` → `List<ChannelMembership>` (member resolution)
- Consumes: `DataSourceRegistry.resolveSource(String, String)` → `Optional<DataSource<?>>`
- Produces: `QhorusBroadcastEvent` implementing `SubscribableEvent`
- Produces: `NotificationChannelBackend` implementing `AgentChannelBackend`

- [ ] **Step 1: Create QhorusBroadcastEvent**

Create `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusBroadcastEvent.java`:

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.subscription.SubscribableEvent;

import java.util.Objects;
import java.util.UUID;

public record QhorusBroadcastEvent(
        String tenancyId,
        String recipientId,
        String capabilityTag,
        UUID channelId,
        String channelName,
        String senderId,
        String messageType,
        String content
) implements SubscribableEvent {

    private static final String TYPE_PREFIX = "io.casehub.qhorus.broadcast.";

    public QhorusBroadcastEvent {
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(recipientId, "recipientId");
        Objects.requireNonNull(capabilityTag, "capabilityTag");
    }

    @Override
    public String type() {
        return TYPE_PREFIX + capabilityTag;
    }
}
```

- [ ] **Step 2: Write test for QhorusBroadcastEvent**

Create `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/QhorusBroadcastEventTest.java`:

```java
package io.casehub.qhorus.notification.bridge;

import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class QhorusBroadcastEventTest {

    @Test
    void type_includesCapabilityTag() {
        var event = new QhorusBroadcastEvent("tenant-1", "agent-2", "analyzer",
                UUID.randomUUID(), "broadcast:analyzer", "agent-1", "STATUS", "anomaly detected");
        assertThat(event.type()).isEqualTo("io.casehub.qhorus.broadcast.analyzer");
    }

    @Test
    void tenancyId_returnsValue() {
        var event = new QhorusBroadcastEvent("tenant-1", "agent-2", "analyzer",
                UUID.randomUUID(), "broadcast:analyzer", "agent-1", "STATUS", "content");
        assertThat(event.tenancyId()).isEqualTo("tenant-1");
    }
}
```

- [ ] **Step 3: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusBroadcastEventTest -pl notification-bridge -Dno-format`
Expected: PASS

- [ ] **Step 4: Write failing test for NotificationChannelBackend**

Create `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/NotificationChannelBackendTest.java`:

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.qhorus.api.channel.ChannelMembership;
import io.casehub.qhorus.api.channel.MemberRole;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.ChannelMembershipStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class NotificationChannelBackendTest {

    private NotificationChannelBackend backend;
    private ChannelMembershipStore membershipStore;
    private DataSourceRegistry dataSourceRegistry;
    private DataSource<Object> dataSource;

    @SuppressWarnings("unchecked")
    @BeforeEach
    void setUp() {
        membershipStore = mock(ChannelMembershipStore.class);
        dataSourceRegistry = mock(DataSourceRegistry.class);
        dataSource = mock(DataSource.class);
        when(dataSourceRegistry.resolveSource(any(), any())).thenReturn(Optional.of(dataSource));
        backend = new NotificationChannelBackend(membershipStore, dataSourceRegistry);
    }

    @Test
    void post_firesEventPerMember() {
        UUID channelId = UUID.randomUUID();
        ChannelRef ref = new ChannelRef(channelId, "broadcast:analyzer");

        when(membershipStore.findByChannelId(channelId)).thenReturn(List.of(
                membership(channelId, "agent-2"),
                membership(channelId, "agent-3")));

        OutboundMessage msg = new OutboundMessage(1L, "agent-1", MessageType.STATUS,
                "anomaly detected", null, null, null, null, null, null);

        backend.post(ref, msg);

        ArgumentCaptor<QhorusBroadcastEvent> captor = ArgumentCaptor.forClass(QhorusBroadcastEvent.class);
        verify(dataSource, times(2)).add(captor.capture());

        List<QhorusBroadcastEvent> events = captor.getAllValues();
        assertThat(events).extracting(QhorusBroadcastEvent::recipientId)
                .containsExactlyInAnyOrder("agent-2", "agent-3");
        assertThat(events).allSatisfy(e -> {
            assertThat(e.capabilityTag()).isEqualTo("analyzer");
            assertThat(e.senderId()).isEqualTo("agent-1");
            assertThat(e.content()).isEqualTo("anomaly detected");
        });
    }

    @Test
    void post_skipsSender() {
        UUID channelId = UUID.randomUUID();
        ChannelRef ref = new ChannelRef(channelId, "broadcast:analyzer");

        when(membershipStore.findByChannelId(channelId)).thenReturn(List.of(
                membership(channelId, "agent-1"),
                membership(channelId, "agent-2")));

        OutboundMessage msg = new OutboundMessage(1L, "agent-1", MessageType.STATUS,
                "signal", null, null, null, null, null, null);

        backend.post(ref, msg);

        ArgumentCaptor<QhorusBroadcastEvent> captor = ArgumentCaptor.forClass(QhorusBroadcastEvent.class);
        verify(dataSource, times(1)).add(captor.capture());
        assertThat(captor.getValue().recipientId()).isEqualTo("agent-2");
    }

    @Test
    void backendId_isPlatformNotifications() {
        assertThat(backend.backendId()).isEqualTo("platform-notifications");
    }

    private ChannelMembership membership(UUID channelId, String memberId) {
        return new ChannelMembership(UUID.randomUUID(), channelId, memberId,
                MemberRole.PARTICIPANT, "default", Instant.now(), null, null);
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=NotificationChannelBackendTest -pl notification-bridge -Dno-format`
Expected: FAIL — class does not exist

- [ ] **Step 6: Implement NotificationChannelBackend**

Create `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/NotificationChannelBackend.java`:

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.qhorus.api.channel.ChannelMembership;
import io.casehub.qhorus.api.gateway.AgentChannelBackend;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.DeliveryGuarantee;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.ActorType;
import io.casehub.qhorus.api.store.ChannelMembershipStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Optional;

import static io.casehub.platform.api.identity.TenancyConstants.PLATFORM_TENANT_ID;
import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;

@ApplicationScoped
public class NotificationChannelBackend implements AgentChannelBackend {

    private static final Logger LOG = Logger.getLogger(NotificationChannelBackend.class);
    private static final String BROADCAST_PREFIX = "broadcast:";

    private final ChannelMembershipStore membershipStore;
    private final DataSourceRegistry dataSourceRegistry;

    @Inject
    public NotificationChannelBackend(ChannelMembershipStore membershipStore,
                                       DataSourceRegistry dataSourceRegistry) {
        this.membershipStore = membershipStore;
        this.dataSourceRegistry = dataSourceRegistry;
    }

    @Override
    public String backendId() { return "platform-notifications"; }

    @Override
    public ActorType actorType() { return ActorType.SYSTEM; }

    @Override
    public DeliveryGuarantee deliveryGuarantee() { return DeliveryGuarantee.BEST_EFFORT; }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        String channelName = channel.channelName();
        if (!channelName.startsWith(BROADCAST_PREFIX)) {
            return;
        }
        String capabilityTag = channelName.substring(BROADCAST_PREFIX.length());

        List<ChannelMembership> members = membershipStore.findByChannelId(channel.channelId());

        Optional<DataSource<?>> ds = dataSourceRegistry.resolveSource(
                NOTIFICATION_DATASOURCE_PATH, PLATFORM_TENANT_ID);
        if (ds.isEmpty()) {
            LOG.warnf("Notification DataSource not available — dropping broadcast to %s", channelName);
            return;
        }

        @SuppressWarnings("unchecked")
        DataSource<Object> source = (DataSource<Object>) ds.get();

        for (ChannelMembership member : members) {
            if (member.memberId().equals(message.sender())) {
                continue;
            }
            try {
                source.add(new QhorusBroadcastEvent(
                        PLATFORM_TENANT_ID,
                        member.memberId(),
                        capabilityTag,
                        channel.channelId(),
                        channelName,
                        message.sender(),
                        message.type().name(),
                        message.content()));
            } catch (Exception e) {
                LOG.warnf("Failed to deliver broadcast to %s: %s", member.memberId(), e.getMessage());
            }
        }
    }
}
```

- [ ] **Step 7: Add notification-bridge dependency on ChannelMembershipStore**

The `notification-bridge/pom.xml` already depends on `casehub-qhorus-api` which contains `ChannelMembershipStore`. No new dependency needed.

- [ ] **Step 8: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=NotificationChannelBackendTest -pl notification-bridge -Dno-format`
Expected: PASS

- [ ] **Step 9: Add broadcast subscription to QhorusSubscriptionBootstrap**

In `QhorusSubscriptionBootstrap.onStartup()`, add after the existing obligation subscriptions:

```java
registerBroadcast(existing, NotificationSeverity.INFO);
```

Add the method:

```java
private void registerBroadcast(Set<String> existing, NotificationSeverity severity) {
    String eventType = "io.casehub.qhorus.broadcast.*";
    if (existing.stream().anyMatch(t -> t.startsWith("io.casehub.qhorus.broadcast."))) {
        LOG.debug("Broadcast subscription already exists — skipping");
        return;
    }
    try {
        subscriptionStore.store(new SubscriptionInput(
                OWNER_ID,
                PLATFORM_TENANT_ID,
                "qhorus.broadcast",
                eventType,
                List.of(),
                List.of(new NotificationTarget(TargetType.EVENT_FIELD, "recipientId")),
                false,
                new NotificationTemplate(
                        "Broadcast from {senderId} on {channelName}",
                        "{content}",
                        severity,
                        "qhorus.broadcast",
                        null,
                        "channel",
                        "channelId",
                        "senderId"),
                true,
                SubscriptionScope.SYSTEM));
        LOG.info("Registered default subscription for broadcast events");
    } catch (Exception e) {
        LOG.warnf("Failed to register broadcast subscription: %s", e.getMessage());
    }
}
```

- [ ] **Step 10: Run full notification-bridge test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -Dno-format`
Expected: All tests pass

- [ ] **Step 11: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: BUILD SUCCESS — all modules compile and tests pass

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add notification-bridge/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#442): NotificationChannelBackend — platform notification delivery for broadcast

NotificationChannelBackend in notification-bridge delivers broadcast messages
via the platform notification system. Resolves channel members, skips sender,
fires QhorusBroadcastEvent per recipient into DataSourceRegistry.

QhorusSubscriptionBootstrap registers wildcard broadcast subscription at startup.

Refs #442"
```

---

## References

- [2026-09-24-topic-broadcast-swarm-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/qhorus/api/channel/ChannelSemantic.java] — existing semantic enum
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java] — capability registration
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java:108] — findOrCreateByName
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelMembershipService.java] — join/leave
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java:181] — fanOut
- [notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/NotificationBridgeObserver.java] — existing notification pattern
- [api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java] — dispatch interface
- [docs/protocols/casehub/channel-initialised-event-observer-idempotency.md] — backend registration pattern
- [docs/protocols/casehub/event-content-free-signal-type.md] — STATUS vs EVENT
- [GitHub #442] — focal issue
- [GitHub engine#1104] — parent hive mind epic

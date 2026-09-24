# Topic-Based Broadcast for Swarm Coordination

**Issue:** casehubio/qhorus#442
**Parent epic:** casehubio/engine#1104 (Hive Mind)
**Depends on:** casehubio/engine#1108 (Agent discovery — closed)

## Problem

Agents in a multi-agent system need to broadcast coordination hints to all peers with a given capability — "signal all analyzers," "alert all monitors" — without knowing which agents exist or managing channel enrollment. Today, `RoutingBridge` resolves `role:X` to a single agent (1:1 routing). There is no fan-out primitive for capability-based broadcast.

The platform notification system exists as a parallel delivery system that qhorus knows nothing about. Adding broadcast through notifications alone would deepen this gap. Instead, qhorus should own the broadcast semantic model and use notifications as a delivery transport.

## Design

### New channel semantic: BROADCAST

Add `BROADCAST` to the `ChannelSemantic` enum. BROADCAST channels differ from existing semantics:

| Property | BROADCAST | APPEND (comparison) |
|----------|-----------|---------------------|
| Fan-out | All members | All registered backends |
| Commitments | None — COMMAND/QUERY denied | Full obligation lifecycle |
| Allowed types | STATUS, EVENT only | All 10 types |
| Membership | Capability-driven auto-join | Manual or lazy human join |
| Lifecycle | Persistent, survives empty membership | Persistent |
| Auto-created | Yes, on first capability registration | Yes, via findOrCreate |

BROADCAST channels are created with `allowedTypes = {STATUS, EVENT}` and `deniedTypes = null`. This uses the existing `MessageTypePolicy` hard-enforcement for COMMAND/QUERY — no new enforcement path needed. STATUS carries content-bearing observations; EVENT carries content-free signals. Both are fire-and-forget speech acts that create no commitments.

### Channel naming convention

Broadcast channels use the slug `broadcast:<capability-tag>`. Examples:
- `broadcast:analyzer` — all agents with capability "analyzer"
- `broadcast:monitor` — all agents with capability "monitor"

The colon separator is already valid in channel slugs (used by `system:` senders). `ChannelSlugValidator` allows it.

### Auto-membership via capability registration

When an agent registers capabilities via `InstanceService.register()`, the system:

1. Computes the diff between old and new capability tags
2. For each **added** capability: `findOrCreateByName("broadcast:<tag>")` + `membershipService.join(channelId, instanceId, MemberRole.PARTICIPANT)`
3. For each **removed** capability: `membershipService.leave(channelId, instanceId)`

**Opt-out:** There is no mechanism to have a capability but opt out of its broadcast channel. If this becomes a problem, an interest-based model (separate "interests" list at registration) can be layered on without breaking the auto-join default.

**Hook point:** A new `@ApplicationScoped` bean `BroadcastMembershipManager` in `runtime-core/` (same package as `ChannelService` and `ChannelMembershipService`) observes instance registration. It does NOT live inside `InstanceService` — separation of concerns. Instead:

```
InstanceService.register()
  → fires CDI event: InstanceRegisteredEvent(instanceId, oldCapabilities, newCapabilities)
  → BroadcastMembershipManager @ObservesAsync
    → diffs capabilities
    → findOrCreate broadcast channels
    → join/leave memberships
```

**Channel creation:** Uses `ChannelService.findOrCreateByName()` which handles concurrent creation races via `REQUIRES_NEW` in `ChannelCreateHelper`. The channel is created with:

```java
ChannelCreateRequest.builder("broadcast:" + capabilityTag)
    .semantic(ChannelSemantic.BROADCAST)
    .description("Broadcast channel for capability: " + capabilityTag)
    .allowedTypes(Set.of(MessageType.STATUS, MessageType.EVENT))
    .build();
```

The channel is marked `autoCreated = true`.

**Deregistration:** When an agent deregisters via `InstanceService.deregister()`, fire `InstanceDeregisteredEvent`. `BroadcastMembershipManager` removes the agent from all broadcast channels. The channel itself persists (D4) — it retains its slug, ledger history, and configuration.

### Dispatch to broadcast channels

Sending a broadcast uses the standard dispatch pipeline:

```java
MessageDispatch.builder()
    .channelId(broadcastChannel.id())
    .sender(senderInstanceId)
    .type(MessageType.STATUS)
    .content("Anomaly detected in region X — confidence 0.87")
    .build();
```

This flows through `MessageService.dispatch()` → `MessageTypePolicy` (validates STATUS/EVENT allowed) → `LedgerWriteService.record()` (audit entry) → `ChannelGateway.fanOut()` (deliver to all backends).

No new dispatch path. The existing pipeline handles everything.

### Convenience: broadcast by capability

A convenience method on `MessageDispatcher` (the API-layer service facade) resolves capability → channel:

```java
public interface MessageDispatcher {
    // existing
    MessageResult dispatch(MessageDispatch dispatch);

    // new
    MessageResult broadcast(String capabilityTag, MessageType type, String content,
                            String sender, String tenancyId);
}
```

`broadcast()` resolves the channel via `channelStore.findByName("broadcast:" + capabilityTag)`, constructs a `MessageDispatch`, and delegates to `dispatch()`. Throws `IllegalArgumentException` if no broadcast channel exists for the capability (no agents have registered it yet).

### NotificationChannelBackend in notification-bridge

The existing `notification-bridge/` module gains a `ChannelBackend` implementation that delivers broadcast messages to the platform notification system.

```java
@ApplicationScoped
public class NotificationChannelBackend implements AgentChannelBackend {

    @Override
    public String backendId() { return "platform-notifications"; }

    @Override
    public ActorType actorType() { return ActorType.SYSTEM; }

    @Override
    public DeliveryGuarantee deliveryGuarantee() { return DeliveryGuarantee.BEST_EFFORT; }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        // Resolve channel membership → fire QhorusBroadcastEvent per member
        // into DataSourceRegistry (same pattern as NotificationBridgeObserver)
    }
}
```

**Registration:** The backend self-registers for BROADCAST channels via `ChannelInitialisedEvent` observation (same pattern as `A2AChannelBackend.onChannelRecovery()`). It checks the channel semantic and only registers for BROADCAST channels.

**Event type:** A new `QhorusBroadcastEvent` implements `SubscribableEvent`:

```java
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
    @Override
    public String type() {
        return "io.casehub.qhorus.broadcast." + capabilityTag;
    }
}
```

**Subscription bootstrap:** Subscriptions are created lazily — when `NotificationChannelBackend` first registers for a broadcast channel, it creates a SYSTEM-scope subscription for `io.casehub.qhorus.broadcast.<capabilityTag>` with `TargetType.EVENT_FIELD` targeting the `recipientId` field. On `post()`, the backend creates one `QhorusBroadcastEvent` per channel member with `recipientId` set to the member's actor ID. This follows the existing `QhorusObligationEvent` pattern (one event per target actor, field-based routing).

### What the notification backend does NOT do

The notification backend is one delivery transport among many. Broadcast channels also deliver through:
- **WebSocket observer** (if `websocket-observer/` on classpath) — real-time push to browser
- **Kafka observer** (if `kafka-observer/` on classpath) — event streaming
- **Webhook observer** (if `webhook-observer/` on classpath) — HTTP callbacks
- **MessageObserver** implementations — CDI-local processing

The notification backend adds persistent notification delivery with the platform's delivery tracking. It does not replace the other transports.

## Migration

### Database

New Flyway migration: channel semantic already exists as a column — `BROADCAST` is a new enum value. No schema change needed beyond ensuring the enum value is handled by the JPA converter.

No new tables. No new columns. The implementation uses existing channel, membership, and message infrastructure.

### CDI events

Two new CDI events in `api/`:

```java
public record InstanceRegisteredEvent(
    String instanceId,
    List<String> previousCapabilities,
    List<String> currentCapabilities) {}

public record InstanceDeregisteredEvent(String instanceId) {}
```

Fired from `InstanceService.register()` and `deregister()`. Async to avoid blocking registration.

## Scope

**In scope:**
- `ChannelSemantic.BROADCAST` enum value
- `BroadcastMembershipManager` (auto-join/leave on capability changes)
- `InstanceRegisteredEvent` / `InstanceDeregisteredEvent` CDI events
- `MessageDispatcher.broadcast()` convenience method
- `NotificationChannelBackend` in `notification-bridge/`
- `QhorusBroadcastEvent` subscribable event
- Tests: CDI-free unit tests for BroadcastMembershipManager, integration test for auto-join flow

**Out of scope:**
- P2P ephemeral channels (separate follow-up issue)
- Engine integration with dynamic interest registration (engine#1107)
- Custom broadcast topics beyond capability tags

## References

- `api/src/main/java/io/casehub/qhorus/api/channel/ChannelSemantic.java` — existing semantic enum
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java` — capability registration
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java:181` — fanOut dispatch
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java:108` — findOrCreateByName
- `notification-bridge/` — existing notification bridge module
- `docs/protocols/casehub/event-content-free-signal-type.md` — STATUS vs EVENT semantics
- `docs/protocols/casehub/channel-initialised-event-observer-idempotency.md` — backend registration pattern
- Issue casehubio/engine#1104 — parent hive mind epic
- Issue casehubio/engine#1108 — agent discovery dependency (closed)

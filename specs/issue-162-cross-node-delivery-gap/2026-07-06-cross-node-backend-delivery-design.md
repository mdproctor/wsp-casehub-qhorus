# Cross-Node Backend Delivery — Design Spec

**Issue:** casehubio/qhorus#162
**Date:** 2026-07-06
**Status:** Draft

---

## Problem

When multiple JVM processes embed Qhorus against a shared PostgreSQL database,
`ChannelBackend.post()` only fires on the node that dispatched the message.
Backends registered on other nodes — browsers via SSE, A2A SSE streams, Slack
threads — receive nothing until they poll.

This is the sole remaining gap in the multi-node embedded topology. All reads
(channels, messages, commitments, ledger) are consistent across nodes because
they share the database. All writes (dispatch, enforcement, ledger) are correct
from any node. The gap is push notification: other nodes don't know a new
message exists until they query.

---

## Architectural Foundations

### Shared database is a prerequisite, not an option

Multi-node Qhorus requires a shared PostgreSQL database. This is a logical
consequence of Qhorus's purpose as a governance mesh:

- **Channels** must be visible to all participants regardless of which node
  they connect to.
- **Commitments** (OPEN → FULFILLED/DECLINED/FAILED) span nodes — a COMMAND
  dispatched on Node A must be fulfillable from Node B.
- **The ledger** is a single tamper-evident audit trail. Per-node ledgers
  produce incoherent governance records.
- **Instance discovery** (by capability, by role) must return all registered
  agents, not just those on the local node.

Independent databases per node produce two independent governance systems.
This is architecturally invalid. The design does not attempt to support it.

### MessageObserver dispatch stays local

`MessageObserver` implementations fire on the dispatching node only. This is
correct:

- **LOCAL observers** (clinical PI monitoring, AML scanning, CloudEvent
  adapter) should process each message exactly once. Firing on all nodes
  creates duplicate processing.
- **CLUSTER observers** cross process boundaries via their own transport
  implementation. The `Scope` enum describes the implementation's reach,
  not where it fires.

No changes to `MessageObserver` dispatch, `MessageObserverDispatcher`, or
the `Scope` enum.

### ChannelBackend fan-out is the gap

`ChannelGateway.fanOut()` iterates the in-memory `registry` — a per-node
`ConcurrentHashMap`. Backends registered on other nodes are invisible.

The fix: after a message commits, notify other nodes so they can fire their
local backends from the shared database.

---

## Design

### SPI: `ChannelActivityBroadcaster`

A new gateway-category interface in `casehub-qhorus-api`:

```java
package io.casehub.qhorus.api.gateway;

@FunctionalInterface
public interface ChannelActivityBroadcaster {

    void broadcast(ChannelActivityEvent event);

    record ChannelActivityEvent(
        java.util.UUID channelId,
        String channelName,
        Long messageId
    ) {}
}
```

**Placement:** `api/gateway/` — this is an integration contract (bridges
external transport infrastructure into the runtime). Follows the
api-interface-taxonomy protocol.

**Default:** `NoOpChannelActivityBroadcaster` in `runtime/`, annotated
`@DefaultBean @ApplicationScoped`. Single-node deployments pay zero overhead.

### Wiring into `MessageService.dispatch()`

After the message is persisted and `fanOut()` completes, register a JTA
`afterCompletion` synchronization that calls `broadcaster.broadcast()` on
`STATUS_COMMITTED`. This guarantees the message is visible in the shared
database before any receiving node tries to read it.

```java
// In MessageService.dispatch(), consolidating post-commit signals:
tsr.registerInterposedSynchronization(new Synchronization() {
    @Override public void beforeCompletion() {}
    @Override public void afterCompletion(int status) {
        if (status == STATUS_COMMITTED) {
            if (hasTracked) deliverySignalQueue.signal(channelId);
            broadcaster.broadcast(new ChannelActivityEvent(
                channelId, channelName, messageId));
        }
    }
});
```

The same wiring applies to `ReactiveMessageService.dispatch()` — the
broadcaster fires after the reactive transaction commits.

The LAST_WRITE overwrite path also fires the broadcaster — the existing
separate `afterCompletion` registration for that path is consolidated into
the same pattern.

### Receiving side: `ChannelGateway.deliverRemote()`

A new package-private method on `ChannelGateway`:

```java
void deliverRemote(UUID channelId, Long messageId) {
    Message msg = crossTenantMessageStore.find(messageId).orElse(null);
    if (msg == null) return;
    Channel ch = crossTenantChannelStore.findById(channelId).orElse(null);
    if (ch == null) return; // channel deleted between dispatch and delivery

    ChannelRef ref = new ChannelRef(channelId, ch.name());
    OutboundMessage outbound = new OutboundMessage(
        UUID.randomUUID(), msg.sender(), msg.messageType(), msg.content(),
        msg.correlationId() != null
            ? UUID.fromString(msg.correlationId()) : null,
        msg.inReplyTo(), msg.actorType());

    List<BackendEntry> entries = registry.getOrDefault(channelId, List.of());
    for (BackendEntry entry : List.copyOf(entries)) {
        if (entry.backend() == agentBackend) continue;
        ChannelBackend backend = entry.backend();
        if (backend.deliveryGuarantee() == DeliveryGuarantee.AT_LEAST_ONCE) {
            continue; // pump handles these
        }
        Thread.ofVirtual().start(() -> {
            try { backend.post(ref, outbound); }
            catch (Exception ex) {
                LOG.warnf("Remote delivery: backend %s failed on channel %s: %s",
                    backend.backendId(), channelId, ex.getMessage());
            }
        });
    }
}
```

Mirrors `fanOut()`: same virtual thread dispatch, same error handling, same
agent backend skip. Differences:

1. Reads the message AND channel from the shared DB (fanOut receives them
   from the caller)
2. Skips AT_LEAST_ONCE backends (the delivery pump handles them — the
   broadcaster signals `DeliverySignalQueue` separately)
3. Package-private (called by broadcaster implementations, not by consumers)
4. Takes `(channelId, messageId)` not `(channelId, channelName, outbound)`
   — resolves everything from the shared DB

---

## PostgreSQL Implementation Module

### Module: `postgres-broadcaster/`

**Artifact:** `casehub-qhorus-postgres-broadcaster`

Follows the casehub-work precedent (`casehub-work-postgres-broadcaster`):
`@Alternative @Priority(1)`, activated by classpath presence, zero
configuration.

```
casehub-qhorus/
├── postgres-broadcaster/
│   ├── pom.xml
│   └── src/main/java/io/casehub/qhorus/postgres/broadcaster/
│       └── PostgresChannelActivityBroadcaster.java
```

### Sending side

On `broadcast()`, fires `pg_notify`:

```java
pool.preparedQuery("SELECT pg_notify($1, $2)")
    .execute(Tuple.of(CHANNEL, event.channelId() + ":" + event.messageId()));
```

PostgreSQL channel: `qhorus_channel_activity`.
Payload: `channelId:messageId` — lightweight, well under 8KB limit.

### Receiving side

Holds a persistent `PgConnection` via `@PostConstruct`:

```java
pgConn.notificationHandler(this::handleNotification);
pgConn.query("LISTEN qhorus_channel_activity").execute();
```

On notification:
1. Parse `channelId:messageId` from payload
2. Check self-notification filter (skip if this node dispatched it)
3. Call `channelGateway.deliverRemote(channelId, messageId)` — the gateway
   resolves `channelName` from `ChannelStore` (shared DB) inside `deliverRemote()`
4. Signal `deliverySignalQueue.signal(channelId)` for AT_LEAST_ONCE backends

### Self-notification filtering

PostgreSQL NOTIFY delivers to all listeners, including the sender. The
broadcaster maintains a bounded `Set` (e.g. `LinkedHashSet` capped at 1000)
of recently-dispatched messageIds. On receiving a notification, if the
messageId is in the set, skip it — local delivery already happened via
`fanOut()`.

### CDI activation

```java
@ApplicationScoped
@Alternative
@Priority(1)
public class PostgresChannelActivityBroadcaster
        implements ChannelActivityBroadcaster { ... }
```

Displaces `NoOpChannelActivityBroadcaster @DefaultBean` by classpath
presence. No `quarkus.arc.selected-alternatives` needed.

### Dependencies

- `quarkus-reactive-pg-client` — for `PgPool` and `PgConnection`
- `casehub-qhorus-api` — for `ChannelActivityBroadcaster`
- `casehub-qhorus` (runtime) — for `ChannelGateway` and `DeliverySignalQueue`

### Lossy delivery is acceptable

LISTEN/NOTIFY is lossy — notifications are missed during connection drops.
This is acceptable because:

- **BEST_EFFORT backends** are best-effort by definition
- **AT_LEAST_ONCE backends** have `DeliveryService` reconciliation every
  30s as a backup — the notification is a latency optimisation, not a
  correctness requirement
- The subscriber connection auto-reconnects on failure

---

## Downstream Impact

### Claudony: `FleetMessageRelayObserver` becomes redundant

Once `casehub-qhorus-postgres-broadcaster` is on claudony's classpath:

1. Node A dispatches → commits → `pg_notify`
2. Node B receives → reads message → `deliverRemote()` → `ClaudonyChannelBackend.post()` → `channelEventBus.emit()` → SSE

This is the same end result as the current fleet relay, but owned by qhorus
infrastructure rather than claudony-specific code.

**Migration:** Both mechanisms can coexist (browsers get two ticks —
idempotent). Remove `FleetMessageRelayObserver` in a separate claudony
issue after the broadcaster is stable.

### Documentation updates

- **`docs/messaging-architecture.md`**: Remove "Known Architectural Gap"
  section. Document shared PostgreSQL as a multi-node prerequisite. Document
  the broadcaster SPI and PostgreSQL implementation.
- **`CLAUDE.md`**: Add `postgres-broadcaster/` to project structure.
  Document module test conventions.
- **`PLATFORM.md`**: Add `Cross-node backend delivery` to capability
  ownership table → `casehub-qhorus` (postgres-broadcaster module). Update
  the "Cross-cutting message notification" row.

### Issue triage

| Issue | Effect |
|-------|--------|
| #162 | Closed by this work |
| #163 (Kafka/WebSocket/Webhook CLUSTER observers) | Remains open — independent concern (MessageObserver transports, not backend fan-out). Urgency drops. |
| #165 (SmallRye bridge) | Remains open — independent |
| New claudony issue | Remove `FleetMessageRelayObserver`, add `casehub-qhorus-postgres-broadcaster` dep |

---

## Testing Strategy

### Unit tests (no DB, no CDI)

- **`NoOpChannelActivityBroadcaster`** — verify no-op behaviour
- **Self-notification filter** — verify bounded set skips locally-dispatched
  messageIds and evicts oldest entries when full
- **`ChannelGateway.deliverRemote()`** — verify: reads from store, skips
  agent backend, skips AT_LEAST_ONCE, calls `post()` on BEST_EFFORT
  backends with correct `OutboundMessage` fields. Uses
  `InMemoryMessageStore` + inline backend stubs.

### Integration tests (`@QuarkusTest`, H2)

- **Broadcast fires after commit:** `MessageService.dispatch()` → verify
  `broadcaster.broadcast()` called with correct
  channelId/channelName/messageId. Use `@InjectMock ChannelActivityBroadcaster`.
- **No broadcast on rollback:** Verify broadcaster is NOT called when the
  dispatch transaction rolls back.
- **LAST_WRITE path:** Verify broadcaster fires for LAST_WRITE overwrite
  dispatches.

### PostgreSQL integration tests (DevServices, `postgres-broadcaster/`)

- **Full round-trip:** Dispatch message → verify NOTIFY fires → verify
  `handleNotification()` triggers `deliverRemote()` → verify local backend
  receives `post()`.
- **Self-notification filtering:** Dispatch locally → verify handler skips
  the notification.
- **Concurrent dispatch:** Multiple rapid dispatches → verify all
  notifications arrive and all backends fire.
- **Connection drop recovery:** Document behaviour — missed notifications
  during reconnection are caught by `DeliveryService` reconciliation for
  AT_LEAST_ONCE backends; BEST_EFFORT backends accept the loss.

Pattern follows `casehub-work` `PostgresBroadcasterIT` — Quarkus DevServices
starts `postgres:17-alpine` automatically.

---

## Module Structure After Implementation

```
casehub-qhorus/
├── api/                              — (add ChannelActivityBroadcaster SPI)
│   └── gateway/
│       ├── ChannelActivityBroadcaster.java
│       └── ... (existing)
├── runtime/                          — (add NoOp default, wire broadcast,
│   │                                    add deliverRemote to ChannelGateway)
│   └── gateway/
│       ├── ChannelGateway.java       — (add deliverRemote())
│       ├── NoOpChannelActivityBroadcaster.java
│       └── ... (existing)
├── postgres-broadcaster/             — NEW MODULE
│   ├── pom.xml
│   └── src/
│       ├── main/java/.../postgres/broadcaster/
│       │   └── PostgresChannelActivityBroadcaster.java
│       └── test/java/.../postgres/broadcaster/
│           └── PostgresChannelActivityBroadcasterIT.java
└── ... (existing modules unchanged)
```

---

## Platform Precedent

This design directly follows `casehub-work`'s broadcaster pattern:

| Aspect | casehub-work | casehub-qhorus (this design) |
|--------|-------------|------|
| SPI | `WorkItemEventBroadcaster` | `ChannelActivityBroadcaster` |
| Default | `LocalWorkItemEventBroadcaster` | `NoOpChannelActivityBroadcaster` |
| PostgreSQL impl | `PostgresWorkItemEventBroadcaster` | `PostgresChannelActivityBroadcaster` |
| Module | `postgres-broadcaster/` | `postgres-broadcaster/` |
| Activation | `@Alternative @Priority(1)` | `@Alternative @Priority(1)` |
| PG channel | `casehub_work_events` | `qhorus_channel_activity` |
| Post-commit | `@Observes(AFTER_SUCCESS)` | JTA `afterCompletion` |
| Self-filter | N/A (CDI event dedup) | Bounded messageId set |

The difference in post-commit mechanism (CDI `TransactionPhase` vs JTA
synchronization) reflects the existing wiring in each module — casehub-work
uses CDI lifecycle events, qhorus uses JTA directly. Both achieve the same
guarantee: notification fires only after the message is committed and
visible in the shared database.

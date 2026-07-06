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

`channelName` is for broadcaster-local use (logging, metrics, filtering).
It is not part of any wire protocol — each broadcaster decides what to
transmit. The PostgreSQL implementation sends only `channelId:messageId`;
the receiving side resolves the name from the shared database.

**Placement:** `api/gateway/` — this is an integration contract (bridges
external transport infrastructure into the runtime). Follows the
api-interface-taxonomy protocol.

**Default:** `NoOpChannelActivityBroadcaster` in `runtime/`, annotated
`@DefaultBean @ApplicationScoped`. Single-node deployments pay zero overhead.

### API Additions

**`CrossTenantMessageStore.find(Long id)`** — new method on the existing
interface, with a corresponding JPA implementation. `deliverRemote()` runs
without tenant context (the message could originate from any tenant), so the
cross-tenant store is the correct home. The tenant-scoped `MessageStore.find()`
already exists — the cross-tenant variant mirrors it.

### Wiring into dispatch paths

The broadcaster fires after the message transaction commits in all dispatch
paths. `MessageService` (blocking) and `ReactiveMessageService` (reactive)
use different post-commit mechanisms. Both also add `fanOut()` to the
LAST_WRITE overwrite path, fixing a pre-existing gap where local
BEST_EFFORT backends were not notified of overwrites.

#### Blocking path (`MessageService.dispatch()`)

After `fanOut()` completes, register a JTA `afterCompletion` synchronization
that calls `broadcaster.broadcast()` on `STATUS_COMMITTED`. This guarantees
the message is visible in the shared database before any receiving node
tries to read it.

```java
// Consolidating post-commit signals:
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

This synchronization always registers — even when the broadcaster is
`NoOpChannelActivityBroadcaster`. The overhead of a JTA interposed
synchronization registration is negligible (list append), and gating on
broadcaster type would couple `MessageService` to implementation details.

`MessageObserverDispatcher` registers its own separate JTA synchronization
for observer dispatch (refs #166). Ordering between interposed
synchronizations is undefined in JTA, but irrelevant here — observer
dispatch, broadcaster notification, and delivery signaling are independent
side effects with no ordering dependency.

#### Blocking LAST_WRITE overwrite path

Currently, the LAST_WRITE overwrite branch returns early before `fanOut()`,
so local BEST_EFFORT backends miss content updates. This is a pre-existing
gap: a browser connected to the dispatching node never sees the overwrite
until it polls. Fix: add `fanOut()` and the broadcaster sync to the
overwrite path.

```java
// In the LAST_WRITE overwrite branch, after messageStore.put(updated):
channelGateway.fanOut(ch.id(), ch.name(), new OutboundMessage(...));
tsr.registerInterposedSynchronization(new Synchronization() {
    @Override public void beforeCompletion() {}
    @Override public void afterCompletion(int status) {
        if (status == STATUS_COMMITTED) {
            deliverySignalQueue.signal(channelId);
            broadcaster.broadcast(new ChannelActivityEvent(
                channelId, channelName, saved.id()));
        }
    }
});
```

Observer dispatch (`MessageObserverDispatcher`) is intentionally excluded
from the overwrite path — an overwrite is a content update, not a new
message event.

#### Reactive path (`ReactiveMessageService.dispatch()`)

The reactive path has no JTA `TransactionSynchronizationRegistry`.
Post-commit side effects run in the `.flatMap()` chain after
`Panache.withTransaction()` returns — a Mutiny pipeline stage, not a JTA
synchronization callback. The broadcaster fires here alongside `fanOut()`
and `deliverySignalQueue`:

```java
// In Phase 4 (post-commit), after fanOut and deliverySignalQueue.signal:
broadcaster.broadcast(new ChannelActivityEvent(
    ch.id(), ch.name(), ctx.messageId()));
```

#### Reactive LAST_WRITE overwrite path

Currently, the reactive overwrite path (the `OverwriteResult` branch) skips
Phase 4 entirely — no `fanOut()`, no `deliverySignalQueue.signal()`, no
observers. This is asymmetric with the blocking overwrite path which at
least signals `deliverySignalQueue`. Fix: add `deliverySignalQueue`,
`fanOut()`, and `broadcaster.broadcast()` to the reactive overwrite path:

```java
if (result instanceof OverwriteResult or) {
    if (ch != null && dispatch.type() != MessageType.EVENT) {
        rateLimiter.recordSend(ch.id(), dispatch.sender(),
                ch.rateLimitPerChannel(), ch.rateLimitPerInstance());
    }
    if (ch != null) {
        channelGateway.fanOut(ch.id(), ch.name(), new OutboundMessage(...));
        deliverySignalQueue.signal(ch.id());
        broadcaster.broadcast(new ChannelActivityEvent(
            ch.id(), ch.name(), or.result().messageId()));
    }
    return Uni.createFrom().item(or.result());
}
```

### Receiving side: `ChannelGateway.deliverRemote()`

A new public method on `ChannelGateway`:

```java
public void deliverRemote(UUID channelId, Long messageId) {
    Message msg = crossTenantMessageStore.find(messageId).orElse(null);
    if (msg == null) return;
    Channel ch = crossTenantChannelStore.findById(channelId).orElse(null);
    if (ch == null) return; // channel deleted between dispatch and delivery

    // Lazy channel initialization: if this node has no registry entry,
    // initialize the channel so backends can register via
    // ChannelInitialisedEvent before delivery is attempted.
    if (!registry.containsKey(channelId)) {
        initChannel(channelId, new ChannelRef(channelId, ch.name()));
    }

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

`deliverRemote()` is public because the broadcaster (in
`postgres/broadcaster/` package) and the gateway (in `runtime/gateway/`
package) are in different modules. Access restriction is by convention —
callers are broadcaster implementations only.

Mirrors `fanOut()`: same virtual thread dispatch, same error handling, same
agent backend skip. Differences:

1. Reads the message AND channel from the shared DB (fanOut receives them
   from the caller). **Requires** `CrossTenantMessageStore.find(Long id)` —
   see § API Additions.
2. Skips AT_LEAST_ONCE backends (the delivery pump handles them — the
   broadcaster signals `DeliverySignalQueue` separately)
3. Lazy-initializes the channel if this node's registry has no entry for it
   (handles dynamic channel creation on remote nodes — see below)
4. Takes `(channelId, messageId)` not `(channelId, channelName, outbound)`
   — resolves everything from the shared DB

**Lazy channel initialization** solves the dynamic-creation gap:
`ChannelGateway.onStart()` initializes the registry from
`crossTenantChannelStore.listAll()` at startup, but channels created on
other nodes after startup are invisible. `ChannelInitialisedEvent` is a
local CDI event — it does not propagate across nodes. When `deliverRemote()`
encounters an unknown `channelId`, it calls `initChannel()`, which fires
`ChannelInitialisedEvent` locally, giving backends (e.g.,
`ClaudonyChannelBackend`, which observes this event) a chance to register
before delivery proceeds.

**Known limitation:** lazy initialization adds registry entries but has no
corresponding cleanup for remote channel deletion. `closeChannel()` only
fires locally (called by `ChannelService.delete()`); the broadcaster has no
`channel_deleted` notification. Stale entries accumulate between restarts
for channels created remotely then deleted remotely. This is acceptable
because: channel creation/deletion is a heavyweight governance operation
(not high-frequency), stale entries are cleaned on restart (`onStart()`
re-initializes from the database), and the per-entry cost is small (a
`List<BackendEntry>` with one `agentBackend` entry). Periodic
reconciliation (compare registry keys against
`crossTenantChannelStore.listAll()`) is a future improvement if the
accumulation proves problematic in long-running deployments.

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
pgConn.closedFuture().onComplete(ar -> reconnect());
```

The `closedFuture()` callback detects connection loss (network failure,
PostgreSQL restart, idle timeout). Reconnection sequence: acquire a new
`PgConnection` from the pool, re-register the notification handler, and
re-issue `LISTEN qhorus_channel_activity`. PostgreSQL LISTEN is
session-scoped — a new connection has no active subscriptions. Missing the
re-LISTEN after reconnection produces a silent failure: the connection
looks healthy but no notifications arrive.

On notification:
1. Parse `channelId:messageId` from payload (non-blocking string split)
2. Check self-notification filter (non-blocking set lookup — skip if this
   node dispatched it)
3. Offload blocking work to a virtual thread:

```java
Thread.ofVirtual().start(() -> {
    channelGateway.deliverRemote(channelId, messageId);
    deliverySignalQueue.signal(channelId);
});
```

Steps 1–2 run on the Vert.x event loop (non-blocking). Step 3 offloads to
a virtual thread because `deliverRemote()` performs blocking JPA queries
(`crossTenantMessageStore.find()`, `crossTenantChannelStore.findById()`)
and potentially fires synchronous CDI events via `initChannel()`. Blocking
the event loop would stall all concurrent notification handling. Virtual
threads are the natural fit — cheap, handle blocking I/O correctly, and
already used inside `deliverRemote()` for backend dispatch.

### Self-notification filtering

PostgreSQL NOTIFY delivers to all listeners, including the sender. The
broadcaster maintains a bounded set of recently-dispatched messageIds using
`Collections.synchronizedSet(new LinkedHashSet<>())`, capped at 1000
entries. On receiving a notification, if the messageId is in the set, skip
it — local delivery already happened via `fanOut()`.

**Thread safety:** dispatch threads (application threads calling
`broadcast()`) add entries to the set; the Vert.x event loop (notification
handler) reads and removes. `Collections.synchronizedSet()` provides mutual
exclusion. Contention is low — dispatch throughput is bounded by database
write speed, and the notification handler runs on a single Vert.x event
loop thread.

The filter is an optimization, not a correctness requirement. If a
self-notification slips through (e.g., after entry eviction),
`deliverRemote()` runs unnecessarily — local `fanOut()` already delivered,
so backends receive a redundant `post()` call, which is harmless.

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
- The subscriber connection detects loss via `closedFuture()`, reconnects,
  and re-issues `LISTEN` (see § Receiving side). Notifications during
  the reconnection window are lost — acceptable for the same reason

---

## Downstream Impact

### Claudony: `FleetMessageRelayObserver` becomes redundant

Once `casehub-qhorus-postgres-broadcaster` is on claudony's classpath:

1. Node A dispatches → commits → `pg_notify`
2. Node B receives → reads message → `deliverRemote()` → `ClaudonyChannelBackend.post()` → `channelEventBus.emit()` → SSE

This is the same end result as the current fleet relay, but owned by qhorus
infrastructure rather than claudony-specific code.

**Migration:** Both mechanisms can coexist during transition.
`ClaudonyChannelBackend.post()` emits a channel-name tick to the SSE
event bus — it is a poll trigger, not message content. Browsers receiving
two ticks from different delivery paths make two poll requests; the second
returns nothing new. No client-side dedup is needed for correctness —
duplicate polls are harmless, not duplicate messages. Remove
`FleetMessageRelayObserver` in a separate claudony issue after the
broadcaster is stable.

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
| #162 | Closed with a comment explaining the architectural ruling: shared-DB topology is the only supported multi-node configuration. Independent-database topologies (Options 2/3 from the issue body) are architecturally invalid. If independent-DB topology becomes a future concern, open a separate issue. |
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
  backends with correct `OutboundMessage` fields. Verify lazy channel
  initialization: unknown channelId triggers `initChannel()` before
  delivery. Uses `InMemoryMessageStore` + inline backend stubs.

### Integration tests (`@QuarkusTest`, H2)

- **Broadcast fires after commit:** `MessageService.dispatch()` → verify
  `broadcaster.broadcast()` called with correct
  channelId/channelName/messageId. Use `@InjectMock ChannelActivityBroadcaster`.
- **No broadcast on rollback:** Verify broadcaster is NOT called when the
  dispatch transaction rolls back.
- **LAST_WRITE path:** Verify broadcaster fires AND `fanOut()` is called
  for LAST_WRITE overwrite dispatches (both blocking and reactive paths).
  Verify `deliverySignalQueue` is signaled in both paths.

### PostgreSQL integration tests (DevServices, `postgres-broadcaster/`)

- **Full round-trip:** Dispatch message → verify NOTIFY fires → verify
  `handleNotification()` triggers `deliverRemote()` → verify local backend
  receives `post()`.
- **Self-notification filtering:** Dispatch locally → verify handler skips
  the notification.
- **Concurrent dispatch:** Multiple rapid dispatches → verify all
  notifications arrive and all backends fire.
- **Connection drop and reconnection:** Simulate connection loss → verify
  `closedFuture()` fires → verify reconnection acquires new connection →
  verify `LISTEN` re-issued → verify notifications resume. Missed
  notifications during reconnection are caught by `DeliveryService`
  reconciliation for AT_LEAST_ONCE backends; BEST_EFFORT backends accept
  the loss.

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

The SPI shapes differ intentionally. `WorkItemEventBroadcaster.stream()`
returns `Multi<WorkItemLifecycleEvent>` — a streaming interface where the
broadcaster IS the SSE delivery mechanism. `ChannelActivityBroadcaster.broadcast()`
is fire-and-forget — it notifies, then the receiving node's existing
`ChannelGateway` infrastructure handles delivery. The casehub-work default
(`LocalWorkItemEventBroadcaster`) is active (provides local in-process
streaming); the qhorus default (`NoOpChannelActivityBroadcaster`) is a
no-op. This reflects different integration patterns: casehub-work's
broadcaster replaces the delivery path, while qhorus's broadcaster
augments it. The module structure and activation pattern
(`@Alternative @Priority(1)`, classpath presence) are genuine parallels.

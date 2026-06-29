# Delivery Guarantee for Registered Channel Backends

**Issue:** casehubio/qhorus#132
**Date:** 2026-06-29
**Status:** Approved

---

## Problem

`ChannelGateway.fanOut()` is fire-and-forget. Each backend receives a virtual thread that calls `post()`, catches exceptions, and logs. Three failure modes are unhandled:

1. **Backend `post()` throws** — transient error (network blip, rate limit, API unavailability). Message silently lost for that backend.
2. **JVM restarts between persist and fanOut** — fanOut never runs. Message lost for ALL backends.
3. **Backend not registered at fanOut time** — gap between re-registration and message dispatch.

Messages are persisted before fanOut (in `MessageService.dispatch()`), so no data is lost from the gateway's perspective. But backends that missed delivery have no catch-up mechanism — the gateway doesn't track what each backend has seen.

## Key Insight

The message store IS the durable outbox. Messages are already persisted before delivery. A separate outbox or DLQ table is unnecessary. The missing piece is **per-backend delivery cursors** — the same pattern as Kafka consumer offsets or the existing `MessageQuery.poll(channelId, afterId, limit)` primitive already in the codebase.

## Approach Evaluation

**Mapping to issue #132:** Issue #132 defines three options: A (at-most-once + client catch-up), B (retry with exponential backoff — recommended), and C (dead-letter queue per backend). The spec evaluates these but reframes around the key insight that the message store IS the durable outbox, making both retry-in-fanOut and a separate DLQ unnecessary. The spec's Option A corresponds to the issue's Option B; the spec's Option B is a new variant not in the issue; the spec's Option C (the chosen approach) supersedes all three issue options.

### A — Inline Retry Only (Issue #132 Option B)

Add retry with exponential backoff inside `fanOut()` virtual threads. This is the approach recommended by issue #132.

Handles transient failures within a single JVM lifetime. Does NOT survive JVM restarts. Does NOT handle "backend not registered at fanOut time." After max retries, message is permanently lost. The issue's suggestion to "accumulate messages in-memory... for that backend within a bounded window" adds memory pressure and still loses messages on JVM restart.

**Rejected:** Incomplete — only addresses failure mode 1. The issue's recommendation was made before the key insight that the message store itself is the durable outbox.

### B — Cursor + Dual Delivery (fanOut + Reconciler)

Keep fanOut delivering to ALL backends. Add a cursor per backend, advanced by fanOut on success. Background reconciler fills gaps.

Creates a concurrency problem: fanOut and reconciler can deliver the same message simultaneously, causing duplicates. For Slack, duplicates are visible (same text posted twice to a thread). Requires in-memory delivery tracking to avoid duplicates. Cursor advancement races when messages arrive out of order.

**Rejected:** Correct but unnecessarily complex.

### C — Delivery Pump (Chosen)

fanOut handles BEST_EFFORT backends only (current behavior, zero overhead). AT_LEAST_ONCE backends are served exclusively by an event-driven delivery pump. This supersedes the issue's Option C (DLQ per backend) — the message store IS the durable log, so a separate DLQ table is unnecessary.

Eliminates all concurrency problems: no duplicate delivery, no cursor races, no in-memory tracking. The pump is the sole delivery path for tracked backends.

This is the Kafka consumer pattern: the message store is the log, each tracked backend is a consumer with its own offset, the pump drives consumption.

## Design

### 1. SPI Changes

`DeliveryGuarantee` enum in `casehub-qhorus-api` (`io.casehub.qhorus.api.gateway`):

```java
public enum DeliveryGuarantee {
    BEST_EFFORT,
    AT_LEAST_ONCE
}
```

Default method on `ChannelBackend`:

```java
default DeliveryGuarantee deliveryGuarantee() {
    return DeliveryGuarantee.BEST_EFFORT;
}
```

Existing backends get zero overhead and zero behavioral change. Backends that want tracked delivery override to return `AT_LEAST_ONCE`.

**Backend classification:**

| Backend | Repo | Guarantee | Reasoning |
|---------|------|-----------|-----------|
| QhorusChannelBackend | qhorus | N/A (skipped in fanOut) | Internal — persistence IS the delivery |
| A2AChannelBackend | qhorus | BEST_EFFORT | SSE + CommitmentStore handles own catch-up |
| SlackChannelBackend | qhorus | AT_LEAST_ONCE | Lost Slack message = visible failure |
| ConnectorChannelBackend | qhorus | AT_LEAST_ONCE | Lost delivery = silent failure |
| ClaudonyChannelBackend | claudony | BEST_EFFORT | Has check_messages polling catch-up |
| OpenClawChannelBackend | openclaw | AT_LEAST_ONCE | Lost webhook = missed agent task |
| DebateChannelBackend | drafthouse | BEST_EFFORT | Internal processing — failure is a bug, not transient |
| ReviewerChannelBackend | drafthouse | BEST_EFFORT | Same reasoning as DebateChannelBackend |

### 2. Data Model

`DeliveryCursor` entity in `io.casehub.qhorus.runtime.gateway` on the qhorus datasource:

```java
@Entity
@Table(name = "delivery_cursor",
       uniqueConstraints = @UniqueConstraint(
           name = "uq_delivery_cursor_channel_backend",
           columnNames = {"channel_id", "backend_id"}))
public class DeliveryCursor extends PanacheEntityBase {
    @Id @GeneratedValue
    public UUID id;

    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(name = "backend_id", nullable = false)
    public String backendId;

    @Column(name = "last_delivered_id")
    public Long lastDeliveredId;

    @Column(name = "updated_at")
    public Instant updatedAt;

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;
}
```

Keyed by `(channelId, backendId)`. Channels are tenant-scoped, so no separate `tenancyId` needed. `lastDeliveredId` is the message store Long ID — messages with `id > lastDeliveredId` are pending delivery.

Flyway V25 at `db/qhorus/migration/V25__delivery_cursor.sql`:

```sql
CREATE TABLE delivery_cursor (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id        UUID         NOT NULL,
    backend_id        VARCHAR(255) NOT NULL,
    last_delivered_id BIGINT,
    updated_at        TIMESTAMP,
    created_at        TIMESTAMP    NOT NULL DEFAULT now(),
    CONSTRAINT uq_delivery_cursor_channel_backend UNIQUE (channel_id, backend_id),
    CONSTRAINT fk_delivery_cursor_channel FOREIGN KEY (channel_id) REFERENCES channel(id) ON DELETE CASCADE
);
```

**Cursor lifecycle:**
- Created lazily on first pump cycle for a backend (set to current message head)
- Preserved across deregistration/re-registration (keyed by backendId)
- Deleted automatically on channel deletion via `ON DELETE CASCADE` on the FK constraint — no application-level cleanup needed in `closeChannel()`

### 3. Persistence Seam

`DeliveryCursorStore` in `io.casehub.qhorus.runtime.store`:

```java
public interface DeliveryCursorStore {
    DeliveryCursor save(DeliveryCursor cursor);
    Optional<DeliveryCursor> findByChannelAndBackend(UUID channelId, String backendId);
    List<DeliveryCursor> findByChannel(UUID channelId);
    List<DeliveryCursor> findAll();
    void deleteByChannel(UUID channelId);
}
```

Implementations:
- `JpaDeliveryCursorStore` — `@ApplicationScoped`, standard Panache repo pattern.
- `InMemoryDeliveryCursorStore` — in `casehub-qhorus-testing`, `@Alternative @Priority(1)`.
- `ReactiveDeliveryCursorStore` — `Uni<>` returns, gated by `@IfBuildProperty(casehub.qhorus.reactive.enabled)`. For reactive consumers that need cursor access from reactive code (e.g., health endpoints). Not used by the pump itself (see Known Limitations).

### 4. DeliveryService (the Pump)

`DeliveryService` in `io.casehub.qhorus.runtime.gateway` — `@ApplicationScoped`. Pump thread started in `@PostConstruct` via `ManagedExecutor.execute()` (integrates with Quarkus lifecycle — tasks cancelled on shutdown). The thread blocks on `signalQueue.poll()` immediately, so no queries run before other beans are ready. Per-backend delivery tasks also use `ManagedExecutor` for CDI context propagation and shutdown integration. Shutdown: `@PreDestroy` sets `volatile running = false` and awaits completion of active deliveries (bounded by `activeDeliveries` set emptying within 30s timeout); pump thread exits on next poll timeout (5s max).

**Dependency graph (no cycles):**
```
MessageService → DeliverySignalQueue ← DeliveryService
  (post-commit)                          ↓
                                    ChannelGateway (for trackedEntries — one-directional)
                                    @CrossTenant CrossTenantMessageStore (for message queries)
                                    @CrossTenant CrossTenantChannelStore (for channel name lookup)

ChannelGateway → fanOut() returns hasTracked → MessageService defers signal post-commit
```

`DeliverySignalQueue` is a thin `@ApplicationScoped` bean owning the `LinkedBlockingDeque<UUID>`. `MessageService.dispatch()` signals the queue after its transaction commits (post-commit synchronization). DeliveryService calls `poll()`/`drainTo()`.

**Event-driven path:**

```
dispatch() → persist message → fanOut()
               |                  ├── BEST_EFFORT backends: post() in virtual thread (unchanged)
               |                  └── AT_LEAST_ONCE backends: skipped (returns hasTracked=true)
               |
               └── TRANSACTION COMMITS
                       ↓
                   post-commit: deliverySignalQueue.signal(channelId)
                       ↓
                   pump thread wakes → processChannel(channelId)
                                     → per-backend managed task
                                     → deliverPending() self-drives until caught up
```

**Post-commit signaling:** `fanOut()` returns `boolean hasTracked`. `dispatch()` uses the existing `TransactionSynchronizationRegistry` (already injected for observer fan-out) to defer the signal to after-commit. This ensures the pump always sees the committed message on wakeup — under PostgreSQL READ COMMITTED, a signal fired within the uncommitted transaction would cause the pump's query to return empty.

Pump thread loop:
```java
void pumpLoop() {
    List<UUID> batch = new ArrayList<>();
    while (running) {
        try {
            UUID first = signalQueue.poll(5, TimeUnit.SECONDS);
            if (first != null) {
                batch.add(first);
                signalQueue.drainTo(batch);
                Set<UUID> unique = new HashSet<>(batch);
                batch.clear();
                for (UUID channelId : unique) {
                    try {
                        processChannel(channelId);
                    } catch (Exception e) {
                        LOG.errorf(e, "Error processing channel %s — pump continues", channelId);
                    }
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
}
```

**Per-backend processing:**

Each (channelId, backendId) pair gets its own managed task. `activeDeliveries` concurrent guard prevents duplicate processing by pump and reconciler. Per-backend threads use `ManagedExecutor` (not bare `Thread.ofVirtual().start()`) for CDI request context propagation and Quarkus shutdown integration:

```java
private final Set<String> activeDeliveries = ConcurrentHashMap.newKeySet();

@Inject ManagedExecutor managedExecutor;

void processChannel(UUID channelId) {
    for (BackendEntry entry : gateway.trackedEntries(channelId)) {
        if (isUnhealthy(entry.backend().backendId())) continue; // circuit breaker — reconciler retries
        String key = channelId + ":" + entry.backend().backendId();
        if (activeDeliveries.add(key)) {
            managedExecutor.execute(() -> {
                try {
                    deliverPending(channelId, entry.backend());
                } finally {
                    activeDeliveries.remove(key);
                }
            });
        }
    }
}
```

**Transactional strategy:**

`deliverPending()` is non-transactional. Each loop iteration calls a `@Transactional` helper (`deliverBatch()`) on a CDI-managed bean. This gives each iteration its own persistence context and transaction, avoiding L1 cache staleness across iterations. `ManagedExecutor` propagates the CDI request context to the task thread, so `@Transactional` and `@Inject` work correctly.

**Self-driving delivery loop:**

```java
void deliverPending(UUID channelId, ChannelBackend backend) {
    while (running) {
        BatchResult result = deliverBatch(channelId, backend);
        if (result == BatchResult.EMPTY || result == BatchResult.FAILED) break;
    }
}

@Transactional
BatchResult deliverBatch(UUID channelId, ChannelBackend backend) {
    // Resolve channel name — one query per batch, needed for ChannelRef
    Channel channel = channelStore.findById(channelId).orElse(null);
    if (channel == null) return BatchResult.FAILED; // channel deleted

    DeliveryCursor cursor = cursorStore.findByChannelAndBackend(channelId, backend.backendId())
        .orElseGet(() -> initializeCursor(channelId, backend.backendId()));
    if (cursor == null) return BatchResult.FAILED; // channel deleted during init

    List<Message> batch = messageStore.scan(
        MessageQuery.poll(channelId, cursor.lastDeliveredId, batchSize));
    if (batch.isEmpty()) return BatchResult.EMPTY;

    ChannelRef ref = new ChannelRef(channelId, channel.name);
    for (Message m : batch) {
        try {
            backend.post(ref, toOutbound(m));
            cursor.lastDeliveredId = m.id;
            cursor.updatedAt = Instant.now();
        } catch (Exception e) {
            recordFailure(backend.backendId());
            if (isUnhealthy(backend.backendId())) {
                LOG.warnf("Backend %s marked unhealthy after %d consecutive failures",
                    backend.backendId(), maxConsecutiveFailures);
            }
            // Advance cursor to last successfully delivered message before returning
            if (!m.id.equals(batch.get(0).id) || !cursor.lastDeliveredId.equals(batch.get(0).id)) {
                cursorStore.save(cursor);
            }
            return BatchResult.FAILED; // stop on failure — preserve ordering
        }
    }
    // Advance cursor once per batch — all messages delivered successfully
    cursorStore.save(cursor);
    resetHealth(backend.backendId());
    return BatchResult.MORE;
}
```

Key properties:
- Sequential delivery — messages in store order (by id ASC)
- Stop on failure — preserves ordering guarantee
- Cursor advances per batch — reduces transaction overhead by batch-size factor while maintaining AT_LEAST_ONCE semantics (on mid-batch failure, successfully delivered messages before the failure point are still recorded; on process crash, at most one batch is re-delivered — acceptable given backends must tolerate duplicates)
- Each `deliverBatch()` call gets a fresh persistence context — no L1 cache staleness
- Self-driving — loops until caught up or failure, no re-signaling needed
- `ManagedExecutor` provides CDI context propagation and shutdown cancellation

**Cursor initialization (start from HEAD):**

New cursors are initialized to the current message HEAD — all existing messages are skipped. This is a deliberate "start from now" policy:
- The pump is being added to a system where backends previously received fire-and-forget delivery. Retroactively delivering all historical messages would flood Slack threads and external webhooks with stale content.
- AT_LEAST_ONCE backends are registered at channel creation time. They don't miss messages unless `post()` fails (which the pump will catch going forward) or the JVM restarts (the pump catches up from HEAD at init, then tracks from that point).
- Cursor initialization only happens once per (channel, backend) pair — after that, the cursor is preserved across deregistration/re-registration cycles.

```java
DeliveryCursor initializeCursor(UUID channelId, String backendId) {
    Long head = messageStore.findLastMessage(channelId).map(m -> m.id).orElse(0L);
    DeliveryCursor cursor = new DeliveryCursor();
    cursor.channelId = channelId;
    cursor.backendId = backendId;
    cursor.lastDeliveredId = head;
    cursor.createdAt = Instant.now();
    cursor.updatedAt = Instant.now();
    try {
        return cursorStore.save(cursor);
    } catch (PersistenceException e) {
        // Channel deleted between check and save — abort delivery for this channel
        LOG.debugf("Channel %s deleted during cursor init for backend %s — aborting", channelId, backendId);
        return null;
    }
}
```

The `PersistenceException` catch handles the race where `initializeCursor()` attempts to create a cursor after the channel row has been deleted. The complementary race — cursor created just before channel deletion — is handled by `ON DELETE CASCADE` on the FK constraint, which atomically removes the new cursor when the channel row is deleted. Together, these two mechanisms cover all timing windows.

**Message → OutboundMessage conversion:**

```java
OutboundMessage toOutbound(Message m) {
    return new OutboundMessage(
        UUID.randomUUID(),
        m.sender, m.messageType, m.content,
        m.correlationId != null ? UUID.fromString(m.correlationId) : null,
        m.inReplyTo,
        ActorTypeResolver.resolve(m.sender));
}
```

Uses `ActorTypeResolver.resolve(sender)` — no Message entity migration needed.

**Message queries via CrossTenantMessageStore:**

The pump operates across tenants (it delivers messages regardless of which tenant created them). The `@CrossTenant` pattern already exists for this — `CrossTenantProducer` produces `@CrossTenant`-qualified stores for background services. `CrossTenantMessageStore.scan(MessageQuery)` and `findLastMessage(UUID)` already provide the exact query primitives the pump needs:

```java
@Inject @CrossTenant CrossTenantMessageStore messageStore;
@Inject @CrossTenant CrossTenantChannelStore channelStore;

// Message queries:
List<Message> batch = messageStore.scan(MessageQuery.poll(channelId, cursor.lastDeliveredId, batchSize));
Optional<Long> head = messageStore.findLastMessage(channelId).map(m -> m.id);

// Channel name resolution (one query per batch for ChannelRef construction):
Channel channel = channelStore.findById(channelId).orElse(null);
```

`MessageQuery.poll(channelId, afterId, limit)` already builds a query with `channelId`, `afterId`, and `limit` — ordered by `id ASC` (the default). No new repository needed. This follows ARC42STORIES §4: "Services inject store interfaces, never Panache entity statics."

**Health tracking (in-memory circuit breaker):**

```java
private final ConcurrentHashMap<String, Integer> consecutiveFailures = new ConcurrentHashMap<>();
private final Set<String> unhealthy = ConcurrentHashMap.newKeySet();
```

Keyed by backendId. Threshold: `casehub.qhorus.delivery.max-consecutive-failures` (default 10). Acts as a circuit breaker: unhealthy backends are skipped by the event-driven pump (`processChannel()` checks `isUnhealthy()` before spawning delivery tasks). The scheduled reconciler retries ALL backends including unhealthy ones — when a retry succeeds, `resetHealth()` clears the unhealthy flag, reopening the circuit. This prevents a permanently-failing backend from consuming pump thread time on every signal while ensuring recovery is automatic.

**Scheduled backup (30s):**

`@Scheduled(every = "${casehub.qhorus.delivery.reconciliation-interval:30s}")` — scans all cursors, joins with gateway registry, calls `processChannel()` for each channel with tracked backends. The `activeDeliveries` guard prevents concurrent processing with the event-driven pump.

**Gateway access:**

Package-private method on ChannelGateway:

```java
List<BackendEntry> trackedEntries(UUID channelId) {
    List<BackendEntry> entries = registry.getOrDefault(channelId, List.of());
    return List.copyOf(entries).stream()
        .filter(e -> e.backend() != agentBackend)
        .filter(e -> e.backend().deliveryGuarantee() == DeliveryGuarantee.AT_LEAST_ONCE)
        .toList();
}
```

`List.copyOf()` before streaming — snapshot prevents ConcurrentModificationException.

### 5. fanOut() Changes

```java
public boolean fanOut(UUID channelId, String channelName, OutboundMessage message) {
    ChannelRef ref = new ChannelRef(channelId, Objects.requireNonNull(channelName, "channelName"));
    List<BackendEntry> entries = registry.getOrDefault(channelId, List.of());
    boolean hasTracked = false;
    for (BackendEntry entry : List.copyOf(entries)) {
        if (entry.backend() == agentBackend) continue;
        ChannelBackend backend = entry.backend();
        if (deliveryEnabled && backend.deliveryGuarantee() == DeliveryGuarantee.AT_LEAST_ONCE) {
            hasTracked = true;
            continue; // pump handles delivery — signaled post-commit by dispatch()
        }
        // BEST_EFFORT backends always get fire-and-forget.
        // AT_LEAST_ONCE backends also get fire-and-forget when delivery is disabled
        // (safe fallback — preserves pre-pump behavior).
        Thread.ofVirtual().start(() -> {
            try {
                backend.post(ref, message);
            } catch (Exception ex) {
                LOG.errorf(ex, "Backend %s failed on fanOut to channel %s",
                        backend.backendId(), channelId);
            }
        });
    }
    return hasTracked; // caller (dispatch) defers signal to post-commit
}
```

Return type changed from `void` to `boolean` — backward compatible (existing callers that ignore the return value still compile). `fanOut()` no longer signals the queue directly; `dispatch()` defers the signal to after the transaction commits via `TransactionSynchronizationRegistry`:

```java
// In MessageService.dispatch(), after fanOut():
boolean hasTracked = channelGateway.fanOut(ch.id, ch.name, outbound);
if (hasTracked) {
    final UUID signalChannelId = ch.id;
    tsr.registerInterposedSynchronization(new Synchronization() {
        @Override public void beforeCompletion() {}
        @Override public void afterCompletion(int status) {
            if (status == STATUS_COMMITTED) {
                deliverySignalQueue.signal(signalChannelId);
            }
        }
    });
}
``` `closeChannel()` requires no changes — cursor cleanup is handled automatically by `ON DELETE CASCADE` on the `delivery_cursor.channel_id` FK constraint when the channel row is deleted.

### 6. Configuration

`@ConfigMapping(prefix = "casehub.qhorus.delivery")`:

| Property | Default | Purpose |
|----------|---------|---------|
| `enabled` | `true` | Master switch — when `false`, pump and reconciler do not start; `fanOut()` delivers to ALL backends fire-and-forget (safe fallback to pre-pump behavior) |
| `batch-size` | `100` | Max messages per backend per pump cycle |
| `max-consecutive-failures` | `10` | Unhealthy threshold |
| `reconciliation-interval` | `30s` | Scheduled backup interval |

### 7. Testing Strategy

**Unit tests (CDI-free):** DeliveryService core logic — cursor init, ordered delivery, failure handling, health circuit breaker, independent backend processing. fanOut changes — BEST_EFFORT direct, AT_LEAST_ONCE skipped+signaled, mixed.

**Integration tests (@QuarkusTest):** End-to-end pump — dispatch to tracked backend, verify delivery via RecordingChannelBackend + cursor advancement. Reconciler catch-up — first delivery fails, reconciler retries. Channel deletion — cursor cleanup. closeChannel race — verify graceful handling when channel is deleted during active delivery.

**RecordingChannelBackend enhancement:** `RecordingChannelBackend` currently implements bare `ChannelBackend` and will return `BEST_EFFORT` from the default `deliveryGuarantee()` method. Add a `deliveryGuarantee` constructor parameter (defaulting to `BEST_EFFORT`) so integration tests can create `new RecordingChannelBackend("test-tracked", ActorType.SYSTEM, DeliveryGuarantee.AT_LEAST_ONCE)` for pump tests.

**Contract tests:** DeliveryCursorStoreContractTest — abstract base with blocking + reactive runners.

### 8. Cross-Repo Propagation

OpenClawChannelBackend: override to AT_LEAST_ONCE (file issue on casehub-openclaw). PLATFORM.md: update Capability Ownership entry.

## Known Limitations

- **Single-node only.** Multi-node concurrent reconciliation may cause duplicate delivery. #162 tracks cross-node delivery as a separate concern. Cursor design is compatible with optimistic locking for future multi-node support.
- **Blocking-only pump.** The delivery pump runs as a blocking background service and has no reactive counterpart. This is intentional: the pump is not on the request hot path, runs on its own managed threads, and performs sequential cursor-based I/O. The reactive dual-stack pattern (§4 in ARC42STORIES) exists for request-path services where the caller may use reactive HTTP. `ReactiveDeliveryCursorStore` exists for reactive consumers that need cursor access from reactive code (e.g., health check endpoints), but the pump itself does not need a reactive variant. The `casehub.qhorus.reactive.enabled` build-time property does not affect the pump — it runs in both blocking and reactive builds.
- **LAST_WRITE channels unsupported** for AT_LEAST_ONCE. Content-change detection is a different problem than cursor-based catch-up. No current AT_LEAST_ONCE backend uses LAST_WRITE.
- **EVENT messages delivered to AT_LEAST_ONCE backends.** The pump delivers all message types including EVENT (null content, telemetry-only), consistent with current `fanOut()` behavior. Backends are responsible for filtering message types they don't handle. Changing the pump to filter EVENTs would create an inconsistency between normal delivery (fanOut sends EVENTs) and catch-up delivery (pump skips EVENTs).
- **Shutdown edge case.** Successful `post()` followed by failed cursor save during JVM shutdown causes duplicate delivery on restart. Inherent to at-least-once semantics; window is milliseconds. Mitigated by `ManagedExecutor` shutdown integration and `@PreDestroy` awaiting active deliveries.
- **AT_LEAST_ONCE backends must tolerate duplicate delivery.** Inherent property of at-least-once guarantees, same as Kafka consumers. Per-batch cursor advancement means at most one batch (default 100 messages) may be re-delivered on mid-batch failure.

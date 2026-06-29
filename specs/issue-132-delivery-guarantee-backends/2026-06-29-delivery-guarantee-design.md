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

### A — Inline Retry Only

Add retry with exponential backoff inside `fanOut()` virtual threads.

Handles transient failures within a single JVM lifetime. Does NOT survive JVM restarts. Does NOT handle "backend not registered at fanOut time." After max retries, message is permanently lost.

**Rejected:** Incomplete — only addresses failure mode 1.

### B — Cursor + Dual Delivery (fanOut + Reconciler)

Keep fanOut delivering to ALL backends. Add a cursor per backend, advanced by fanOut on success. Background reconciler fills gaps.

Creates a concurrency problem: fanOut and reconciler can deliver the same message simultaneously, causing duplicates. For Slack, duplicates are visible (same text posted twice to a thread). Requires in-memory delivery tracking to avoid duplicates. Cursor advancement races when messages arrive out of order.

**Rejected:** Correct but unnecessarily complex.

### C — Delivery Pump (Chosen)

fanOut handles BEST_EFFORT backends only (current behavior, zero overhead). AT_LEAST_ONCE backends are served exclusively by an event-driven delivery pump.

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
    CONSTRAINT fk_delivery_cursor_channel FOREIGN KEY (channel_id) REFERENCES channel(id)
);
```

**Cursor lifecycle:**
- Created lazily on first pump cycle for a backend (set to current message head)
- Preserved across deregistration/re-registration (keyed by backendId)
- Deleted on channel deletion (`closeChannel()`)

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
- `JpaDeliveryCursorStore` — `@ApplicationScoped`, standard Panache repo pattern. `save()` uses `@Transactional(REQUIRES_NEW)` — each cursor advance is independently durable.
- `InMemoryDeliveryCursorStore` — in `casehub-qhorus-testing`, `@Alternative @Priority(1)`.
- `ReactiveDeliveryCursorStore` — `Uni<>` returns, gated by `@IfBuildProperty(casehub.qhorus.reactive.enabled)`.

### 4. DeliveryService (the Pump)

`DeliveryService` in `io.casehub.qhorus.runtime.gateway` — `@ApplicationScoped`. Pump thread started in `@PostConstruct` via `ManagedExecutor.execute()` (integrates with Quarkus lifecycle — tasks cancelled on shutdown). The thread blocks on `signalQueue.poll()` immediately, so no queries run before other beans are ready. Shutdown: `@PreDestroy` sets `volatile running = false`; pump thread exits on next poll timeout (5s max).

**Dependency graph (no cycles):**
```
ChannelGateway → DeliverySignalQueue ← DeliveryService
                                        ↓
                                   ChannelGateway (for trackedEntries — one-directional)
```

`DeliverySignalQueue` is a thin `@ApplicationScoped` bean owning the `LinkedBlockingDeque<UUID>`. ChannelGateway calls `signal()`. DeliveryService calls `poll()`/`drainTo()`.

**Event-driven path:**

```
dispatch() → persist message → fanOut()
                                  ├── BEST_EFFORT backends: post() in virtual thread (unchanged)
                                  └── AT_LEAST_ONCE backends: signal DeliverySignalQueue(channelId)
                                                                  ↓
                                              pump thread wakes → processChannel(channelId)
                                                                → per-backend virtual thread
                                                                → deliverPending() self-drives until caught up
```

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
                    processChannel(channelId);
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

Each (channelId, backendId) pair gets its own virtual thread. `activeDeliveries` concurrent guard prevents duplicate processing by pump and reconciler:

```java
private final Set<String> activeDeliveries = ConcurrentHashMap.newKeySet();

void processChannel(UUID channelId) {
    for (BackendEntry entry : gateway.trackedEntries(channelId)) {
        String key = channelId + ":" + entry.backend().backendId();
        if (activeDeliveries.add(key)) {
            try {
                Thread.ofVirtual().start(() -> {
                    try {
                        deliverPending(channelId, entry.backend());
                    } finally {
                        activeDeliveries.remove(key);
                    }
                });
            } catch (Exception e) {
                activeDeliveries.remove(key);
                LOG.errorf(e, "Failed to start delivery thread for %s", key);
            }
        }
    }
}
```

**Self-driving delivery loop:**

```java
void deliverPending(UUID channelId, ChannelBackend backend) {
    while (running) {
        DeliveryCursor cursor = cursorStore.findByChannelAndBackend(channelId, backend.backendId())
            .orElseGet(() -> initializeCursor(channelId, backend.backendId()));
        List<Message> batch = messageRepo.findAfterCursor(channelId, cursor.lastDeliveredId, batchSize);
        if (batch.isEmpty()) break;
        for (Message m : batch) {
            try {
                backend.post(toChannelRef(channelId), toOutbound(m));
                cursor.lastDeliveredId = m.id;
                cursor.updatedAt = Instant.now();
                cursor = cursorStore.save(cursor); // REQUIRES_NEW — use returned entity
                resetHealth(backend.backendId());
            } catch (Exception e) {
                recordFailure(backend.backendId());
                if (isUnhealthy(backend.backendId())) {
                    LOG.warnf("Backend %s marked unhealthy after %d consecutive failures",
                        backend.backendId(), maxConsecutiveFailures);
                }
                return; // stop on failure — preserve ordering
            }
        }
    }
}
```

Key properties:
- Sequential delivery — messages in store order (by id ASC)
- Stop on failure — preserves ordering guarantee
- Cursor advances per message — partial batch delivery is durable
- Self-driving — loops until caught up or failure, no re-signaling needed
- `cursor = cursorStore.save(cursor)` — uses returned entity after REQUIRES_NEW merge

**Cursor initialization:**

```java
DeliveryCursor initializeCursor(UUID channelId, String backendId) {
    Long head = messageRepo.findLastMessageId(channelId).orElse(0L);
    DeliveryCursor cursor = new DeliveryCursor();
    cursor.channelId = channelId;
    cursor.backendId = backendId;
    cursor.lastDeliveredId = head;
    cursor.createdAt = Instant.now();
    cursor.updatedAt = Instant.now();
    return cursorStore.save(cursor);
}
```

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

**Message queries (tenant-context-free):**

Package-private `DeliveryMessageRepository` with direct JPQL on the qhorus EntityManager, scoped by channelId only (inherently tenant-scoped, no tenant filter needed):

```java
@ApplicationScoped
class DeliveryMessageRepository {
    @Inject @PersistenceUnit("qhorus") EntityManager em;

    List<Message> findAfterCursor(UUID channelId, Long afterId, int limit) {
        return em.createQuery(
            "FROM Message m WHERE m.channelId = :cid AND m.id > :aid ORDER BY m.id ASC",
            Message.class)
            .setParameter("cid", channelId)
            .setParameter("aid", afterId)
            .setMaxResults(limit)
            .getResultList();
    }

    Optional<Long> findLastMessageId(UUID channelId) {
        return em.createQuery(
            "SELECT MAX(m.id) FROM Message m WHERE m.channelId = :cid", Long.class)
            .setParameter("cid", channelId)
            .getResultStream().findFirst();
    }
}
```

**Health tracking (in-memory):**

```java
private final ConcurrentHashMap<String, Integer> consecutiveFailures = new ConcurrentHashMap<>();
private final Set<String> unhealthy = ConcurrentHashMap.newKeySet();
```

Keyed by backendId. Threshold: `casehub.qhorus.delivery.max-consecutive-failures` (default 10). Resets on successful delivery or backend re-registration.

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
public void fanOut(UUID channelId, String channelName, OutboundMessage message) {
    ChannelRef ref = new ChannelRef(channelId, Objects.requireNonNull(channelName, "channelName"));
    List<BackendEntry> entries = registry.getOrDefault(channelId, List.of());
    boolean hasTracked = false;
    for (BackendEntry entry : List.copyOf(entries)) {
        if (entry.backend() == agentBackend) continue;
        ChannelBackend backend = entry.backend();
        if (backend.deliveryGuarantee() == DeliveryGuarantee.AT_LEAST_ONCE) {
            hasTracked = true;
            continue;
        }
        Thread.ofVirtual().start(() -> {
            try {
                backend.post(ref, message);
            } catch (Exception ex) {
                LOG.errorf(ex, "Backend %s failed on fanOut to channel %s",
                        backend.backendId(), channelId);
            }
        });
    }
    if (hasTracked) {
        deliverySignalQueue.signal(channelId);
    }
}
```

Signature unchanged — no caller migration. `closeChannel()` gains one line: `deliveryCursorStore.deleteByChannel(channelId)`.

### 6. Configuration

`@ConfigMapping(prefix = "casehub.qhorus.delivery")`:

| Property | Default | Purpose |
|----------|---------|---------|
| `enabled` | `true` | Master switch |
| `batch-size` | `100` | Max messages per backend per pump cycle |
| `max-consecutive-failures` | `10` | Unhealthy threshold |
| `reconciliation-interval` | `30s` | Scheduled backup interval |

### 7. Testing Strategy

**Unit tests (CDI-free):** DeliveryService core logic — cursor init, ordered delivery, failure handling, health tracking, independent backend processing. fanOut changes — BEST_EFFORT direct, AT_LEAST_ONCE skipped+signaled, mixed.

**Integration tests (@QuarkusTest):** End-to-end pump — dispatch to tracked backend, verify delivery via RecordingChannelBackend + cursor advancement. Reconciler catch-up — first delivery fails, reconciler retries. Channel deletion — cursor cleanup.

**Contract tests:** DeliveryCursorStoreContractTest — abstract base with blocking + reactive runners.

### 8. Cross-Repo Propagation

OpenClawChannelBackend: override to AT_LEAST_ONCE (file issue on casehub-openclaw). PLATFORM.md: update Capability Ownership entry.

## Known Limitations

- **Single-node only.** Multi-node concurrent reconciliation may cause duplicate delivery. #162 tracks cross-node delivery as a separate concern. Cursor design is compatible with optimistic locking for future multi-node support.
- **LAST_WRITE channels unsupported** for AT_LEAST_ONCE. Content-change detection is a different problem than cursor-based catch-up. No current AT_LEAST_ONCE backend uses LAST_WRITE.
- **Shutdown edge case.** Successful `post()` followed by failed cursor save during JVM shutdown causes duplicate delivery on restart. Inherent to at-least-once semantics; window is milliseconds.
- **AT_LEAST_ONCE backends must tolerate duplicate delivery.** Inherent property of at-least-once guarantees, same as Kafka consumers.

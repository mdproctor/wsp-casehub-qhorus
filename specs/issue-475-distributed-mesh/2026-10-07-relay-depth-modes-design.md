# Relay Depth Modes — Design Specification

**Issue:** casehubio/qhorus#475
**Date:** 2026-10-07
**Status:** Draft
**Depends on:** [2026-10-06-distributed-mesh-consolidated.md] (§5 Relay Depth Modes, D24)

## 1. What This Is

A caching layer for the qhorus relay that reduces PostgreSQL read pressure
by serving recent messages from in-memory ring buffers. Two depth modes
configure how much history the cache holds:

| Mode | What's cached | Use case |
|------|---------------|----------|
| Shallow | Recent messages per active channel (LRU, bounded) | Real-time multiplexing, conversation-pace reads |
| Full | Complete message history, mirrored from PostgreSQL | Search, analytics, history queries, read offloading |

Same binary, different config. A relay starts as shallow immediately
(zero startup delay), and in full mode, backfills historical data from
PostgreSQL in the background while serving real-time traffic.

## 2. Architecture

### Cache placement

The cache is a CDI `@Alternative` decorator on `MessageStore` — the
unified read/write persistence interface. All callers (REST, MCP, A2A,
WebSocket, projections) get caching transparently. No caller changes
needed. This follows the same interception pattern as
`WriteRoutingDecorator` on `MessageDispatcher`.

```
Caller → CachingMessageStore (decorator)
           ├── cache hit → return from ring buffer
           ├── cache miss → delegate to JpaMessageStore → populate cache
           └── put() → delegate to JpaMessageStore → populate cache
```

The decorator wraps both read and write paths:
- **Reads:** `scan()`, `find()`, `findRecent()` check cache first
- **Writes:** `put()` delegates to JPA, then adds the returned message
  (with assigned ID) to the ring buffer
- **Pass-through:** `count()`, `countByChannel()`, `countAllByChannel()`,
  `distinctSendersByChannel()`, `delete*()`, `update*()` always delegate
  to JPA unchanged

### Data structure

Per-channel ring buffer stored in a Caffeine cache:

```java
Cache<UUID, ChannelMessageBuffer> channelCache = Caffeine.newBuilder()
        .maximumSize(config.maxChannels())        // default 1000
        .build();
```

Each `ChannelMessageBuffer` wraps a `ConcurrentSkipListMap<Long, Message>`
(keyed by message ID, naturally ordered):

```java
class ChannelMessageBuffer {
    private final ConcurrentSkipListMap<Long, Message> messages;
    private final int maxSize;  // 0 = unbounded (full mode)

    void add(Message msg) {
        messages.put(msg.id(), msg);
        if (maxSize > 0) {
            while (messages.size() > maxSize) {
                messages.pollFirstEntry();  // evict oldest
            }
        }
    }

    List<Message> query(MessageQuery q) {
        Long afterId = q.afterId();
        NavigableMap<Long, Message> range = (afterId != null)
                ? messages.tailMap(afterId, false)
                : messages;
        return range.values().stream()
                .filter(q::matches)
                .limit(q.limit() != null ? q.limit() : 50)
                .toList();
    }

    boolean covers(Long afterId) {
        if (messages.isEmpty()) return false;
        if (afterId == null) return true;  // requesting latest
        return afterId >= messages.firstKey();
    }
}
```

### Cache population — two paths

**Local writes (post-commit):** When `dispatch()` calls
`messageStore.put()`, the decorator delegates to JPA and registers a
JTA `TransactionSynchronizationRegistry.afterCompletion()` callback.
On `STATUS_COMMITTED`, the callback adds the message to the ring
buffer. This ensures the cache never contains uncommitted data — if
the transaction rolls back (ledger write failure, enforcement gate,
commitment conflict), the cache is not polluted. The latency cost is
negligible — the callback fires immediately after commit, before the
dispatch response returns to the caller.

**Remote writes (pg_notify):** When another relay dispatches a message,
`PostgresChannelActivityBroadcaster` delivers the notification.
`ChannelGateway.deliverRemote(channelId, messageId)` calls
`crossTenantMessageStore.find(messageId)` — note: this uses the
`CrossTenantMessageStore` interface, which is a separate interface
hierarchy from `MessageStore`. The cache decorator on `MessageStore`
does NOT intercept this call.

To populate the cache from remote writes, the cache module implements
`MessageObserver` (scope `CLUSTER`). When `deliverRemote()` fires the
CLUSTER-scoped observer dispatch, the cache observer receives the
`MessageReceivedEvent` and adds the message to the ring buffer. This
uses the existing observer infrastructure — no new notification channel.
The observer fires after the transaction commits (via
`TransactionSynchronizationRegistry.afterCompletion`), so the cache
never contains uncommitted data.

### Cache miss behaviour

For `scan(MessageQuery)`:

1. Is the channel in the cache? **No** → fall through to JPA
2. Does the ring buffer cover the requested range?
   (`afterId` is null or >= buffer's earliest message ID) **No** → fall
   through to JPA
3. **Yes** → filter the buffer in-memory using `MessageQuery.matches()`,
   apply `limit`, return results

For `find(Long id)`:
1. Check ring buffer for the message ID → return if found
2. Fall through to JPA → add to ring buffer on load → return

For `findRecent(UUID channelId, int limit)`:
1. If ring buffer has >= `limit` entries → return last `limit` from buffer
2. Otherwise → fall through to JPA

Everything else passes through to JPA unchanged.

## 3. Shallow Mode

**Default mode.** Caches recent messages for active channels.

- Ring buffer per channel, bounded to `max-messages-per-channel` (default 200)
- Channel-level LRU eviction via Caffeine `maximumSize` (default 1000 channels)
- Inactive channels evict first — Caffeine's access-order eviction
- Older messages beyond the buffer roll off the oldest end
- Cache misses (requesting history beyond the buffer) fall through to JPA

**Configuration:**

```properties
casehub.qhorus.cache.enabled=true                  # default when module on classpath
casehub.qhorus.cache.max-channels=1000
casehub.qhorus.cache.max-messages-per-channel=200
```

**Memory footprint estimate:** 1000 channels × 200 messages × ~2KB per
message ≈ 400MB worst case. In practice, most channels have fewer than
200 messages cached.

## 4. Full Mode

**Opt-in mode.** Mirrors the complete message history from PostgreSQL.
The relay serves as an application-level read replica.

### Lifecycle

1. Relay starts → cache module activates in shallow mode (immediately useful)
2. Background sync thread begins: loads channels ordered by most recent activity
3. For each channel: batch-loads messages (`afterId` cursor, ascending, batch size 1000)
4. Real-time messages arrive via pg_notify during sync — no gap
5. When all channels synced → status transitions to READY

### Sync implementation

```java
@ApplicationScoped
class FullSyncService {

    @Inject CacheConfig config;
    @Inject @Named("jpa") MessageStore jpaStore;  // bypass cache decorator
    @Inject ChannelStore channelStore;
    @Inject CachingMessageStore cachingStore;      // to populate buffers

    enum SyncStatus { SYNCING, READY }

    private volatile SyncStatus status = SyncStatus.SYNCING;
    private final Map<UUID, Long> cursors = new ConcurrentHashMap<>();

    @Scheduled(every = "${casehub.qhorus.cache.full-sync-interval:5s}")
    void syncBatch() {
        if (status == SyncStatus.READY) return;

        List<Channel> channels = channelStore.listAll();  // ordered by activity
        boolean allDone = true;

        for (Channel ch : channels) {
            Long cursor = cursors.getOrDefault(ch.id(), 0L);
            MessageQuery q = MessageQuery.poll(ch.id(), cursor, config.fullSyncBatchSize());
            List<Message> batch = jpaStore.scan(q);  // read from DB directly

            if (!batch.isEmpty()) {
                batch.forEach(msg -> cachingStore.addToBuffer(ch.id(), msg));
                cursors.put(ch.id(), batch.getLast().id());
                allDone = false;
                return;  // one batch per tick — don't monopolise the thread
            }
        }

        if (allDone) {
            status = SyncStatus.READY;
            LOG.info("Full sync complete — all channels cached");
        }
    }
}
```

### Differences from shallow mode

| Aspect | Shallow | Full |
|--------|---------|------|
| Ring buffer size | Bounded (200 default) | Unbounded |
| Eviction | LRU per channel | None (all channels retained) |
| Cache miss for old messages | Falls through to JPA | Should not happen after READY |
| Memory footprint | Bounded (~400MB) | Proportional to total history |
| Startup delay | None | None (starts as shallow) |
| Background sync | None | Active until READY |

### Health reporting

```json
{
  "cache": {
    "enabled": true,
    "mode": "full",
    "depth_status": "syncing",
    "channels_cached": 47,
    "channels_total": 120,
    "messages_cached": 15230
  }
}
```

After sync completes: `depth_status` → `"ready"`.

## 5. Module Structure

New Maven module: `casehub-qhorus-cache`

```
cache/
├── pom.xml
└── src/main/java/io/casehub/qhorus/cache/
    ├── CacheConfig.java              — @ConfigMapping(prefix = "casehub.qhorus.cache")
    ├── CachingMessageStore.java      — @Alternative @Priority(1) MessageStore decorator
    ├── ChannelMessageBuffer.java     — Per-channel ring buffer (ConcurrentSkipListMap)
    ├── CachePopulationObserver.java  — MessageObserver (CLUSTER): populates cache from remote writes
    ├── FullSyncService.java          — @Scheduled background sync for full mode
    └── CacheHealthCheck.java         — Health check reporting cache stats and sync status
```

### Dependencies

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-qhorus-api</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-qhorus</artifactId>    <!-- JPA store as delegate -->
</dependency>
<dependency>
  <groupId>com.github.ben-manes.caffeine</groupId>
  <artifactId>caffeine</artifactId>
</dependency>
```

### Configuration

```java
@ConfigMapping(prefix = "casehub.qhorus.cache")
public interface CacheConfig {

    @WithDefault("true")
    boolean enabled();

    @WithDefault("shallow")
    String mode();                    // "shallow" | "full"

    @WithDefault("1000")
    int maxChannels();

    @WithDefault("200")
    int maxMessagesPerChannel();      // ignored in full mode

    @WithDefault("1000")
    int fullSyncBatchSize();

    @WithDefault("5s")
    Duration fullSyncInterval();
}
```

### Activation

The module activates by classpath presence. `CachingMessageStore` uses
`@Alternative @Priority(1)` to displace the JPA `MessageStore` and
`MessageReader` beans. The JPA store is injected as the delegate via
`@Inject @Named("jpa") MessageStore`.

When `cache.enabled=false`, the `@Alternative` bean is suppressed via
`@IfBuildProperty(name = "casehub.qhorus.cache.enabled", stringValue = "true")`.

### Mesh integration

The `mesh/pom.xml` adds `casehub-qhorus-cache` as a dependency. The
`mesh/src/main/resources/application.properties` gains:

```properties
casehub.qhorus.cache.mode=${CASEHUB_QHORUS_CACHE_MODE:shallow}
```

**Configuration interaction with RelayConfig:**
`RelayConfig.depth()` (`casehub.qhorus.relay.depth`) is removed — the
cache module owns depth configuration via `casehub.qhorus.cache.mode`.
The `RelayConfig` interface drops its `depth()` method. This eliminates
the dual-configuration surface: one module, one config prefix, one
place to set the mode.

## 6. Consistency Model

The cache is a **performance optimisation, not a correctness layer**.
Every cache miss falls through to PostgreSQL, which is the single
source of truth.

### Staleness window

- **Local writes:** Zero staleness — inline population
- **Remote writes:** Staleness = pg_notify delivery time (~1-10ms on
  a healthy PostgreSQL). During connection drops, pg_notify is lossy —
  the cache may miss messages until the next explicit read
- **Full mode during sync:** Not-yet-synced channels fall through to
  PostgreSQL transparently

### What the cache does NOT cache

- **Channel metadata** (semantic, allowedTypes, protocols) — always from DB
- **Commitments** — always from DB (state machine consistency)
- **Ledger entries** — always from DB (immutable audit trail)
- **Instance registry** — always from DB (global visibility)
- **Aggregate queries** (count, distinctSenders) — always from DB

The cache accelerates only the message read path — the highest-volume
read operation in the relay.

## 7. Testing Strategy

### Unit tests (CDI-free)

- `ChannelMessageBufferTest` — ring buffer add, eviction, query with
  afterId, limit, filter
- `CachingMessageStoreTest` — cache hit/miss paths, put() population,
  find() population, pass-through for non-cacheable methods
- Mock the delegate `MessageStore`

### Integration tests (@QuarkusTest)

- `CacheIntegrationTest` — end-to-end: dispatch message, read from cache
  (verify no JPA query on hit), verify cache miss falls through
- `FullSyncTest` — background sync populates cache, verify sync status
  transitions, verify fall-through during sync

### Test isolation

- Use unique channel IDs per test (standard qhorus pattern)
- `CachingMessageStore` is `@ApplicationScoped` — cache state persists
  across tests. Tests that need a clean cache call `cache.invalidateAll()`
  in `@BeforeEach`

## 8. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Memory pressure from full mode | Monitoring via health endpoint; ops sizes the JVM heap per relay role |
| Cache incoherence from lost pg_notify | Cache miss falls through to JPA; next explicit read re-populates |
| Ring buffer eviction under burst writes | Buffer size is configurable; conversation-pace throughput rarely exceeds 200 messages in the recent window |
| Full sync overwhelming PostgreSQL | Batch size configurable; sleep interval between batches; one batch per tick |
| ConcurrentSkipListMap contention | Lock-free structure; contention only under concurrent writes to the same channel, which is rare (single writer at Level 4, DB-serialised at Level 3) |

## References

- [decisions.md] D24-D31 — relay depth mode decisions
- [2026-10-06-distributed-mesh-consolidated.md] §5 — depth mode concept
- [PresenceService.java] — Caffeine cache pattern reference
- [PostgresChannelActivityBroadcaster.java] — cross-node notification primitive
- [WriteRoutingDecorator.java] — CDI decorator pattern reference
- [MessageStore.java, MessageReader.java] — store interface contracts
- [MessageQuery.java] — query model with afterId pagination and matches() predicate
- [ChannelGateway.deliverRemote()] — remote message delivery path

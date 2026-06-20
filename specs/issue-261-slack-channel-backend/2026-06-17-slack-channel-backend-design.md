# casehub-qhorus-slack-channel — Design Spec

**Issue:** casehubio/qhorus#261  
**Date:** 2026-06-17  
**Revised:** 2026-06-18 (r5)  
**Status:** In revision

---

## Summary

New optional module `casehub-qhorus-slack-channel` — a Slack bot-backed `HumanParticipatingChannelBackend` for qhorus. Counterpart to `casehub-connectors-slack-bot` (already published). Activates by classpath presence. Uses `SlackBotClient` directly (not `ConnectorService`) to support Slack-native reply threading via `thread_ts`. Credentials follow Tier 1.5 per-binding reference protocol (PP-20260617-per-binding-credential-ref).

---

## Module structure

```
slack-channel/
  artifactId:  casehub-qhorus-slack-channel
  package:     io.casehub.qhorus.slack.channel
  activation:  classpath presence (CDI discovery, no @IfBuildProperty)
  jandex:      jandex-maven-plugin (required for CDI bean scanning)
```

**Files to create (complete listing):**

```
slack-channel/
  pom.xml
  src/main/java/io/casehub/qhorus/slack/channel/
    SlackBotBinding.java               — JPA entity (@Entity, extends PanacheEntityBase)
    SlackBotBindingStore.java          — store SPI interface
    JpaSlackBotBindingStore.java       — Panache-backed store impl (@ApplicationScoped)
    SlackThreadEntry.java              — JPA entity (@Entity, extends PanacheEntityBase)
    SlackThreadCache.java              — CDI service bean owning in-memory + DB (@ApplicationScoped)
    JpaSlackThreadCacheStore.java      — Panache DB delegate used by SlackThreadCache (@ApplicationScoped)
    SlackChannelBackend.java           — HumanParticipatingChannelBackend impl (@ApplicationScoped)
    SlackInboundNormaliser.java        — InboundNormaliser impl (@ApplicationScoped)
    SlackBindingResource.java          — JAX-RS resource (@ApplicationScoped)
    SlackBindingRequest.java           — request record
    SlackBindingView.java              — response record
    SlackThreadCacheCleanupJob.java    — @Scheduled TTL eviction (@ApplicationScoped)
  src/main/resources/
    db/qhorus/migration/           — same classpath path as runtime migrations
      V23__slack_bot_binding.sql   — discovered automatically when JAR is on classpath
      V24__slack_thread_cache.sql

testing/src/main/java/io/casehub/qhorus/testing/
    InMemorySlackBotBindingStore.java  — @Alternative @Priority(1)
    InMemorySlackThreadCacheStore.java — @Alternative @Priority(1) (replaces JpaSlackThreadCacheStore in tests)
```

Added to root `pom.xml` `<modules>` list alongside `connector-backend`.

**Compile dependencies:**
- `casehub-qhorus-api` — `HumanParticipatingChannelBackend`, `ChannelRef`, `InboundHumanMessage`, `OutboundMessage`, `ChannelInitialisedEvent`, `InboundNormaliser`, `NormalisedMessage`
- `casehub-qhorus` — `ChannelGateway`, `ChannelService`, `ChannelBindingStore` (mutual exclusion check)
- `casehub-connectors-core` — `InboundMessage`, `InboundConnectorIds`
- `casehub-connectors-slack-bot` — `SlackBotClient`
- `quarkus-hibernate-orm-panache`, `jakarta.persistence-api`, `jakarta.enterprise.cdi-api`, `eclipse-microprofile-config-api`, `org.jboss.logging` — all `provided`

**Test dependencies:**
- `casehub-qhorus-testing`, `casehub-platform` (MockCurrentPrincipal), `quarkus-junit5`, `quarkus-junit5-mockito`, `quarkus-jdbc-h2`, `assertj`

**`testing/` module** gains `InMemorySlackBotBindingStore` and `InMemorySlackThreadCacheStore`.

---

## Domain model

### `SlackBotBinding` — JPA entity, `qhorus` PU

Follows the `ChannelConnectorBinding` pattern: UUID PK, public fields, no all-args constructor, no `@PrePersist` (channelId is caller-supplied).

```java
@Entity
@Table(name = "slack_bot_binding",
       uniqueConstraints = @UniqueConstraint(
           name = "uq_slack_bot_slack_id",
           columnNames = "slack_channel_id"))
public class SlackBotBinding extends PanacheEntityBase {

    @Id
    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(name = "workspace_id", nullable = false, length = 32)
    public String workspaceId;   // Slack workspace ID (e.g. "T123ABC"); doubles as credential key

    @Column(name = "slack_channel_id", nullable = false, length = 64)
    public String slackChannelId;
}
```

**Unique constraints:** `channel_id` is `@Id` — its uniqueness is enforced by the PK; no additional `UNIQUE(channel_id)` constraint. `slack_channel_id` carries `uq_slack_bot_slack_id` — prevents two Qhorus channels from binding to the same Slack channel, which would cause duplicate inbound routing and double outbound delivery. Cf. `ChannelConnectorBinding.uq_binding_key` (two non-PK columns).

Construction (field assignment — no all-args constructor, matching qhorus entity convention):
```java
var b = new SlackBotBinding();
b.channelId = channelId;
b.workspaceId = request.workspaceId();
b.slackChannelId = request.slackChannelId();
bindingStore.put(b);
```

### `SlackThreadCache` — JPA entity, `qhorus` PU

Composite PK `(channel_id, correlation_id)` — natural key, no synthetic UUID. `SlackThreadCacheId` is `@Embeddable`. The `UNIQUE(channel_id, thread_ts)` constraint ensures at most one active corrId per Slack thread root, making the reverse lookup `(channelId, threadTs) → corrId` deterministic.

```java
@Embeddable
public class SlackThreadCacheId implements Serializable {
    @Column(name = "channel_id", nullable = false) public UUID channelId;
    @Column(name = "correlation_id", nullable = false) public UUID corrId;  // UUID natively, not VARCHAR
    // equals() and hashCode() required — omitted here for brevity
}

@Entity
@Table(name = "slack_thread_cache",
    uniqueConstraints = @UniqueConstraint(
        name = "uq_slack_thread_ts",
        columnNames = {"channel_id", "thread_ts"}))
public class SlackThreadCache extends PanacheEntityBase {

    @EmbeddedId
    public SlackThreadCacheId id;   // (channelId, corrId)

    @Column(name = "thread_ts", nullable = false, length = 64)
    public String threadTs;         // Slack thread root timestamp

    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (createdAt == null) createdAt = Instant.now();
    }
}
```

**`UNIQUE(channel_id, thread_ts)`** enforces the invariant: for any given Slack thread, at most one active corrId exists at a time. This makes the reverse lookup `findCorrIdByThreadTs(channelId, threadTs)` deterministic (one result or none).

---

## Flyway migrations

**Directory:** `slack-channel/src/main/resources/db/qhorus/migration/`  
Same classpath path as runtime migrations. When the slack-channel JAR is on the classpath, Flyway's existing scan of `classpath:db/qhorus/migration` discovers V23/V24 automatically — no new location config entry needed. V23/V24 avoid collision with the runtime's V1–V22 sequence. Version numbers follow `casehub/garden: docs/protocols/casehub/flyway-version-range-allocation.md` Rule 4 (named datasource scoped path) — verify the next available V-number by scanning all branches before writing migrations, as described in that protocol.

### V23__slack_bot_binding.sql

```sql
CREATE TABLE slack_bot_binding (
    channel_id        UUID         NOT NULL,
    workspace_id      VARCHAR(32)  NOT NULL,   -- Slack workspace ID; doubles as credential key
    slack_channel_id  VARCHAR(32)  NOT NULL,   -- Slack channel ID within the workspace
    CONSTRAINT pk_slack_bot_binding
        PRIMARY KEY (channel_id),
    CONSTRAINT fk_slack_binding_channel
        FOREIGN KEY (channel_id) REFERENCES channel(id),
    CONSTRAINT uq_slack_bot_slack_id
        UNIQUE (slack_channel_id)
);
-- Note: no separate UNIQUE(channel_id) — channel_id is @Id; PK already enforces uniqueness.
```

### V24__slack_thread_cache.sql

```sql
CREATE TABLE slack_thread_cache (
    channel_id      UUID        NOT NULL,
    correlation_id  UUID        NOT NULL,   -- UUID natively (not VARCHAR)
    thread_ts       VARCHAR(64) NOT NULL,   -- Slack thread root timestamp
    created_at      TIMESTAMP   NOT NULL,
    CONSTRAINT pk_slack_thread_cache  PRIMARY KEY (channel_id, correlation_id),
    CONSTRAINT uq_slack_thread_ts     UNIQUE (channel_id, thread_ts)
);
CREATE INDEX idx_slack_thread_channel ON slack_thread_cache (channel_id);
```

- **Composite PK `(channel_id, correlation_id)`** — natural key. No synthetic UUID.
- **`UNIQUE(channel_id, thread_ts)`** — ensures at most one active corrId per Slack thread. Makes the reverse lookup `findCorrIdByThreadTs(channelId, threadTs)` deterministic. Attempting to anchor a second corrId to the same thread while the first is active violates this constraint (which is correct — only DONE/FAILURE/DECLINE evict the anchor, freeing the slot).
- **`idx_slack_thread_channel(channel_id)`** — for `deleteAllForChannel()` and bulk `findAllForChannel()` (restart recovery).

---

## Runtime config changes — `runtime/src/main/resources/application.properties`

**Flyway:** No change required. V23/V24 live at `slack-channel/src/main/resources/db/qhorus/migration/` — same classpath path as runtime migrations. When the slack-channel JAR is on the classpath, Flyway's existing scan of `classpath:db/qhorus/migration` discovers V23 and V24 automatically. The existing config must be preserved unchanged:

```properties
# Preserve this unchanged — do NOT replace db/ledger/migration with db/slack-channel/migration
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration
```

**Hibernate packages:** One line addition — append, do not expand the parent package. The protocol `optional-module-jpa-package-registration.md` requires explicit append; expanding `io.casehub.qhorus.runtime` to `io.casehub.qhorus` would scan all future sub-packages unconditionally:

```properties
# Append io.casehub.qhorus.slack — do NOT change io.casehub.qhorus.runtime to io.casehub.qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime,io.casehub.qhorus.slack
```

Safe when module is absent: Hibernate scans the package but finds no classes (no JAR on classpath for that package). No entities registered, nothing breaks.

**Module test `application.properties`** must include the packages append and use `drop-and-create` (no Flyway in test cycle).

---

## Store SPIs

### `SlackBotBindingStore`

```java
Optional<SlackBotBinding> findByChannelId(UUID channelId);
void put(SlackBotBinding binding);
void delete(UUID channelId);
```

**`JpaSlackBotBindingStore`** — `@ApplicationScoped`, named `qhorus` PU. Delegates to Panache static methods on `SlackBotBinding` (which extends `PanacheEntityBase`):

```java
@ApplicationScoped
public class JpaSlackBotBindingStore implements SlackBotBindingStore {

    @Override
    public Optional<SlackBotBinding> findByChannelId(UUID channelId) {
        return SlackBotBinding.findByIdOptional(channelId);  // channelId is the @Id
    }

    @Override
    public void put(SlackBotBinding binding) {
        binding.persistAndFlush();
    }

    @Override
    public void delete(UUID channelId) {
        SlackBotBinding.deleteById(channelId);
    }
}
```

**`InMemorySlackBotBindingStore`** — `@Alternative @Priority(1) @ApplicationScoped` in `testing/`.

### `SlackThreadCacheStore`

```java
Optional<String> findThreadTs(UUID channelId, String correlationId);
Optional<String> findCorrelationId(UUID channelId, String threadTs);  // reverse — uses idx_slack_thread_cache_ts
List<SlackThreadCache> findByChannelId(UUID channelId);               // bulk load on channel init (restart recovery)
void put(UUID channelId, String correlationId, String threadTs);
void delete(UUID channelId, String correlationId);                     // terminal commitment eviction
void deleteAllForChannel(UUID channelId);                              // channel close / admin unbinding
int deleteOlderThan(Instant threshold);                                // TTL cleanup
```

**`JpaSlackThreadCacheStore`** — `@ApplicationScoped`, named `qhorus` PU. Pure DB delegate used by `SlackThreadCache` CDI bean.

**Transaction requirement (critical):** `SlackThreadCache.put()` and `evictOne()` are called from virtual threads (`fanOut()`) and `@ObservesAsync` ManagedExecutor threads — neither has an active JTA transaction. All write methods must be `@Transactional`. REQUIRED propagation starts a new tx when called from transactionless contexts; joins the outer tx when called from `close()` inside `@Transactional delete_channel`.

```java
@ApplicationScoped
public class JpaSlackThreadCacheStore {

    // --- Reads (no @Transactional required — Panache works in read-only context) ---

    public Optional<SlackThreadEntry> findByCorrId(UUID corrId) {
        return SlackThreadEntry.findByIdOptional(corrId);    // PK lookup
    }

    public List<SlackThreadEntry> findAllForChannel(UUID channelId) {
        return SlackThreadEntry.list("channelId = ?1", channelId);  // idx_slack_thread_entry_channel
    }

    public Optional<UUID> findCorrIdByThreadTs(UUID channelId, String threadTs) {
        return SlackThreadCache
            .<SlackThreadCache>find("id.channelId = ?1 AND threadTs = ?2", channelId, threadTs)
            .firstResultOptional()
            .map(e -> e.id.corrId);   // UNIQUE(channel_id, thread_ts) — deterministic
    }

    // --- Writes (all @Transactional) ---

    @Transactional
    public void persist(UUID channelId, UUID corrId, String rootTs) {
        var e = new SlackThreadEntry();
        e.corrId     = corrId;
        e.channelId  = channelId;
        e.rootTs     = rootTs;
        e.createdAt  = Instant.now();
        e.persist();
    }

    @Transactional
    public void delete(UUID corrId) {
        SlackThreadEntry.deleteById(corrId);                 // PK delete
    }

    @Transactional
    public void deleteAllForChannel(UUID channelId) {
        SlackThreadEntry.delete("channelId = ?1", channelId);  // idx_slack_thread_entry_channel
    }

    @Transactional
    public int deleteOlderThan(Instant threshold) {
        return (int) SlackThreadEntry.delete("createdAt < ?1", threshold);
    }
}
```

**`SlackThreadCache`** — `@ApplicationScoped` CDI bean. Owns BOTH the in-memory write-through cache and the DB delegate. `SlackChannelBackend` injects this; callers see a single coherent abstraction.

```java
@ApplicationScoped
public class SlackThreadCache {

    // channelId → (corrId → rootTs) — in-memory write-through cache
    private final ConcurrentHashMap<UUID, ConcurrentHashMap<UUID, String>> cache =
            new ConcurrentHashMap<>();

    @Inject JpaSlackThreadCacheStore store;

    /** Called from onInboundMessage() — in-memory first, DB best-effort. */
    public void put(UUID channelId, UUID corrId, String rootTs) {
        cache.computeIfAbsent(channelId, k -> new ConcurrentHashMap<>())
             .put(corrId, rootTs);                          // in-memory first — never throws
        try {
            store.persist(channelId, corrId, rootTs);       // DB best-effort
        } catch (Exception e) {
            LOG.warnf(e, "Thread anchor DB write failed for channel=%s corrId=%s — " +
                         "in-memory anchor intact; restart recovery disabled for this corrId",
                      channelId, corrId);
        }
    }

    /** Called from post() — memory first, DB fallback for post-restart misses. */
    public Optional<String> get(UUID channelId, UUID corrId) {
        String ts = Optional.ofNullable(cache.get(channelId))
                            .map(m -> m.get(corrId)).orElse(null);
        if (ts != null) return Optional.of(ts);
        return store.findByCorrId(corrId).map(e -> e.rootTs);  // DB fallback
    }

    /** Called from onInboundMessage() — reverse lookup (thread reply path).
     *  UNIQUE(channel_id, thread_ts) ensures at most one result. */
    public Optional<UUID> getCorrIdByThreadTs(UUID channelId, String threadTs) {
        return store.findCorrIdByThreadTs(channelId, threadTs);   // always DB (no reverse in-memory index)
    }

    /** Called from onChannelInitialised() — restart recovery: load DB into memory. */
    public void loadForChannel(UUID channelId) {
        ConcurrentHashMap<UUID, String> channelMap = new ConcurrentHashMap<>();
        store.findAllForChannel(channelId).forEach(e -> channelMap.put(e.corrId, e.rootTs));
        cache.put(channelId, channelMap);
    }

    /** Called from post() terminal eviction — evict one corrId from both layers. */
    public void evictOne(UUID channelId, UUID corrId) {
        Optional.ofNullable(cache.get(channelId)).ifPresent(m -> m.remove(corrId));
        store.delete(corrId);                               // @Transactional inside store
    }

    /** Called from evict() (admin unbinding) — in-memory only; DB rows handled by TTL or close(). */
    public void evictChannel(UUID channelId) {
        cache.remove(channelId);
    }

    /** Called from close() (channel deletion) — evicts memory and deletes all DB rows. */
    public void deleteAllForChannel(UUID channelId) {
        cache.remove(channelId);
        store.deleteAllForChannel(channelId);               // @Transactional inside store
    }

    /** Called from SlackThreadCacheCleanupJob. */
    public int deleteOlderThan(Instant threshold) {
        return store.deleteOlderThan(threshold);            // in-memory not touched — TTL rows are orphaned
    }
}
```

**`InMemorySlackThreadCacheStore`** — `@Alternative @Priority(1) @ApplicationScoped` in `testing/`. Replaces `JpaSlackThreadCacheStore` injection in `SlackThreadCache` during tests. No `@Transactional` needed — pure ConcurrentHashMap.

---

## `SlackChannelBackend`

`@ApplicationScoped`, `BACKEND_ID = "slack-bot"`, implements `HumanParticipatingChannelBackend`. Constructor injection throughout, matching `ConnectorChannelBackend` convention.

### In-memory indexes

```java
// channelId → (workspaceId, slackChannelId, channelName)
private final ConcurrentHashMap<UUID, CacheEntry> channelCache = new ConcurrentHashMap<>();
// slackChannelId → ChannelRef  (inbound routing — O(1) reverse lookup)
private final ConcurrentHashMap<String, ChannelRef> slackIndex = new ConcurrentHashMap<>();

private record CacheEntry(String workspaceId, String slackChannelId, String channelName) {}
```

Thread cache state is owned by the injected `SlackThreadCache` CDI bean — not by `SlackChannelBackend`.

**Token resolution** uses an injected `org.eclipse.microprofile.config.Config` (not the static `ConfigProvider.getConfig()`) so that unit tests can inject a mock `Config` directly via the constructor without anonymous subclass overrides:

```java
// Constructor parameter (alongside gateway, bindingStore, slackBotClient, ...)
private final Config config;

String resolveToken(String workspaceId) {
    return config.getValue("casehub.qhorus.slack-channel.credentials." + workspaceId, String.class);
}
```

In unit tests: `Config config = mock(Config.class); when(config.getValue(...)).thenReturn("xoxb-test");`  
`ConfigProvider.getConfig()` (static access) prevents CDI injection and forces anonymous subclass overrides — the known antipattern. Injected `Config` is the standard CDI approach and matches what Quarkus supports via `@Inject Config config`.

### `@Observes ChannelInitialisedEvent` (sync)

**`@Observes` is required** — `ChannelGateway.initChannel()` uses synchronous `channelInitialisedEvents.fire()`. CDI routes synchronous `fire()` exclusively to `@Observes` observers. `@ObservesAsync` will never execute for this event. Compare: `ConnectorChannelBackend.onChannelInitialised()` also uses `@Observes`.

Registration and **restart recovery** — both happen here:

1. `bindingStore.findByChannelId(channelId)` — if absent, skip (not a Slack-backed channel)
2. Populate `channelCache` and `slackIndex` from binding + event
3. **Restart recovery:** `threadCache.loadForChannel(channelId)` — loads all `SlackThreadEntry` rows for this channel into `SlackThreadCache`'s in-memory map. After a server restart, in-flight commitments retain their thread context; without this load `post()` falls through to the DB fallback but the in-memory cache is cold.
4. `gateway.deregisterBackend(channelId, BACKEND_ID)` — dedup guard
5. `gateway.registerBackend(channelId, this, "human_participating")`

### `close(ChannelRef)` — channel deletion hook

Called only by `ChannelGateway.closeChannel()`. Total cleanup:

```java
CacheEntry entry = channelCache.remove(channel.id());
if (entry != null) slackIndex.remove(entry.slackChannelId());
threadCache.deleteAllForChannel(channel.id());   // clears memory + DB rows
bindingStore.delete(channel.id());
```

### `evict(UUID channelId)` — package-private, admin unbinding

In-memory only. DB thread cache rows preserved — post-unbinding in-flight commitments may still resolve (`post()` finds no `CacheEntry` and returns early; TTL job cleans orphaned rows).

```java
CacheEntry entry = channelCache.remove(channelId);
if (entry != null) slackIndex.remove(entry.slackChannelId());
threadCache.evictChannel(channelId);
```

### `normaliser()` — returns `slackInboundNormaliser` (constructor-injected)

### `actorType()` → `ActorType.HUMAN`

### `post(ChannelRef, OutboundMessage)`

```java
// Guard: fanOut() fires for ALL message types including the telemetry EVENT from receiveHumanMessage()
if (message.type() == MessageType.EVENT) return;   // primary: semantic intent
if (message.content() == null) return;             // safety net: future null-content types

CacheEntry entry = channelCache.get(channel.id());
if (entry == null) {
    LOG.errorf("No Slack binding for channel %s — cannot post", channel.name());
    return;
}

String token = resolveToken(entry.workspaceId());

// OutboundMessage.correlationId is null for COMMAND, QUERY, EVENT;
// non-null for RESPONSE, DONE, FAILURE, DECLINE, STATUS.
String threadTs = null;
if (message.correlationId() != null) {
    // SlackThreadCache.get() checks memory first, falls back to DB on cold start / miss.
    threadTs = threadCache.get(channel.id(), message.correlationId()).orElse(null);
}

PostResult result = slackBotClient.postMessage(token, entry.slackChannelId(),
                                                message.content(), threadTs);
if (!result.ok()) {
    LOG.warnf("Slack post failed on channel %s: %s", channel.name(), result.error());
    meterRegistry.counter("slack_post_failures_total",
                          "channel_id", channel.id().toString()).increment();
    return;
}

// Track thread misses — correlationId was set but no anchor found in memory or DB.
// Common recovery path: after a server restart, onChannelInitialised() reloads anchors
// from DB so this should rarely fire. Fires for edge cases where slackTs was null on
// inbound (anchor not written) or the DB write in onInboundMessage() failed.
// Counter fires BEFORE the recovery anchor below — the miss is counted even when
// the post succeeds and result.ts() creates a new anchor.
if (message.correlationId() != null && threadTs == null) {
    meterRegistry.counter("slack_thread_miss_total",
                          "channel_id", channel.id().toString()).increment();
}

boolean isTerminalType = (message.type() == DONE || message.type() == FAILURE
                          || message.type() == DECLINE);

// Recovery anchor (skip for terminal types — no subsequent posts, avoids pointless persist+delete).
// UX note: recovery creates a NEW top-level bot message; see Known Limitations.
if (message.correlationId() != null && threadTs == null && result.ts() != null && !isTerminalType) {
    threadCache.put(channel.id(), message.correlationId(), result.ts());
}

// Evict on terminal commitment resolution.
// Do NOT use MessageType.isTerminal(): includes HANDOFF (must NOT evict — delegated agent
// continues in same thread) and excludes DECLINE (must evict — commitment rejected).
if (isTerminalType && message.correlationId() != null) {
    threadCache.evictOne(channel.id(), message.correlationId());
}
// RESPONSE: no eviction — human may follow up in the same Slack thread.
// HANDOFF: no eviction — delegated agent continues in the same thread.
```

**Threading model:** `ChannelGateway.fanOut()` uses `Thread.ofVirtual().start(() -> backend.post(...))`. `SlackBotClient.postMessage()` blocks on HTTP and may `Thread.sleep()` on HTTP 429 retry — safe on a virtual thread (unmounted from carrier during blocking). No `@Blocking` needed.

### `@ObservesAsync InboundMessage` → `CompletionStage<Void>`

`CompletionStage<Void>` mirrors `ConnectorChannelBackend` — lets test `.join()` before asserting.

```java
if (!InboundConnectorIds.SLACK_INBOUND.equals(msg.connectorId())) {
    return CompletableFuture.completedFuture(null);
}
ChannelRef channelRef = slackIndex.get(msg.externalChannelRef());
if (channelRef == null) {
    LOG.debugf("No Slack binding for Slack channel %s — discarding", msg.externalChannelRef());
    meterRegistry.counter("slack_inbound_discarded_total",
                          "slack_channel", msg.externalChannelRef()).increment();
    return CompletableFuture.completedFuture(null);
}

String slackThreadTs = msg.metadata().get("slack-thread-ts");
String slackTs       = msg.metadata().get("slack-ts");
String corrId;

UUID corrIdUUID;

if (slackThreadTs != null && !slackThreadTs.equals(slackTs)) {
    // Thread reply — reverse-lookup existing corrId by (channelId, slackThreadTs).
    // UNIQUE(channel_id, thread_ts) ensures this is deterministic: one result or none.
    corrIdUUID = threadCache.getCorrIdByThreadTs(channelRef.id(), slackThreadTs).orElse(null);
    // null = anchor evicted (DONE/FAILURE/DECLINE) or unknown thread → treated as new QUERY below
} else {
    corrIdUUID = null;
}

if (corrIdUUID == null) {
    // New top-level message OR thread reply with no active anchor.
    // Generate fresh corrId and anchor it.
    corrIdUUID = UUID.randomUUID();
    // rootTs: thread replies use slackThreadTs as root (so agent's reply threads in T1);
    //         top-level messages use slackTs (so this message IS the thread root).
    String rootTs = (slackThreadTs != null && !slackThreadTs.equals(slackTs))
            ? slackThreadTs : slackTs;

    // ORDERING INVARIANT: ATTEMPT anchor write before receiveHumanMessage().
    // receiveHumanMessage() triggers fanOut(). If a RESPONSE arrives at post() before
    // the in-memory anchor is set, post() misses and posts to channel root.
    // SlackThreadCache.put() sets in-memory first (never throws), then DB best-effort.
    if (rootTs != null) {
        threadCache.put(channelRef.id(), corrIdUUID, rootTs);
    }
}

// corrId is non-null for thread replies with active anchor (RESPONSE path),
// and non-null for new top-level / no-anchor thread replies (QUERY path).
// The normaliser uses corrId != null + thread-ts present to distinguish RESPONSE from QUERY.
String corrId = corrIdUUID.toString();

// Delivery always proceeds — anchor write outcome does not block message delivery.
// content may be null for media-only Slack messages; QUERY/RESPONSE accept null content.
gateway.receiveHumanMessage(channelRef,
    new InboundHumanMessage(msg.externalSenderId(), msg.content(),
                             msg.receivedAt(), msg.metadata(), corrId, null));
return CompletableFuture.completedFuture(null);
```

---

## `SlackInboundNormaliser`

`@ApplicationScoped`, implements `InboundNormaliser`. Constructor-injected into `SlackChannelBackend`; returned by `normaliser()`.

The `correlationId` is already resolved by `SlackChannelBackend.onInboundMessage()` and passed in `InboundHumanMessage.correlationId()`. The normaliser's job is type inference and pass-through, matching the pattern established by `DefaultInboundNormaliser`.

```java
@ApplicationScoped
public class SlackInboundNormaliser implements InboundNormaliser {

    @Override
    public NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw) {
        String content      = raw.content();
        String slackThreadTs = raw.metadata().get("slack-thread-ts");
        String slackTs      = raw.metadata().get("slack-ts");

        // Type inference — three cases:
        // 1. Slash command → COMMAND
        // 2. Thread reply with active anchor (corrId found by reverse lookup, non-null) → RESPONSE
        // 3. Everything else → QUERY (top-level, or thread reply with evicted/unknown anchor)
        //
        // The UNIQUE(channel_id, thread_ts) constraint ensures the reverse lookup is
        // deterministic. corrId is set to null in onInboundMessage() for the no-anchor case,
        // so the normaliser can distinguish found (RESPONSE) from generated-fresh (QUERY) via corrId.
        //
        // Note: for the RESPONSE case, the corrId may belong to an already-discharged commitment
        // (RESPONSE doesn't evict the anchor). See Known Limitations.
        final MessageType type;
        if (content != null && content.startsWith("/")) {
            type = MessageType.COMMAND;
        } else if (slackThreadTs != null && !slackThreadTs.equals(slackTs)
                   && raw.correlationId() != null) {
            type = MessageType.RESPONSE;   // thread reply with active anchor — corrId was FOUND
        } else {
            type = MessageType.QUERY;      // top-level, or thread reply with no active anchor
        }

        // content may be null for media-only messages (images, files, voice).
        // COMMAND, QUERY, and RESPONSE accept null content at the MessageDispatch level.
        return new NormalisedMessage(
                type,
                content,
                "human:" + raw.externalSenderId(),
                raw.correlationId(),   // pass through — set by SlackChannelBackend
                null,                  // inReplyTo — agent supplies from check_messages ledger ID
                null,
                null);
    }
}
```

**Slash-command detection note:** COMMAND type is inferred from a leading `/`. `SlackInboundConnector.parseMessages()` uses `event.getString("text", "")` — content is always non-null (empty string `""` for media-only Slack messages such as images, files, and voice). The `content != null` guard is therefore defensive for future callers that may pass null, not an active fix for the current Slack path. For empty-string content, `"".startsWith("/")` returns false and type defaults to QUERY. COMMAND requires neither correlationId nor inReplyTo in `MessageDispatch.build()` — the agent receives the COMMAND and responds independently.

---

## `SlackBindingResource`

```java
@Path("/qhorus/slack/bindings")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class SlackBindingResource { ... }
```

**Auth:** None — consistent with all other qhorus REST resources. Network isolation is the security boundary.

### Request/response types

```java
/** PUT request body */
public record SlackBindingRequest(String workspaceId, String slackChannelId) {}

/** GET response body — token never returned */
public record SlackBindingView(UUID channelId, String workspaceId, String slackChannelId) {
    static SlackBindingView from(UUID channelId, SlackBotBinding b) {
        return new SlackBindingView(channelId, b.workspaceId, b.slackChannelId);
    }
}
```

### PUT `/{channelId}`

1. `channelService.findById(channelId)` — 404 if channel absent
2. `channelBindingStore.findByChannelId(channelId)` — 409 if generic `ChannelConnectorBinding` exists
3. Validate credential — check both missing key AND blank value:
   ```java
   String credKey = "casehub.qhorus.slack-channel.credentials." + workspaceId;
   String token;
   try {
       token = ConfigProvider.getConfig().getValue(credKey, String.class);
   } catch (NoSuchElementException e) {
       return Response.status(400).entity("Credential key not found: " + credKey).build();
   }
   if (token.isBlank()) {
       return Response.status(400).entity("Credential " + credKey + " is configured but empty").build();
   }
   ```
   `Config.getValue()` returns `""` without throwing when the property is set to an empty string (e.g. `T123ABC=`). Without the blank check, an empty token is accepted at bind time; `post()` later calls Slack with `Authorization: Bearer ` and receives `{"ok":false,"error":"not_authed"}` — silent delivery failure. Detecting this at bind time is strictly better UX.
4. **Clean up stale routing state** — called unconditionally before save; safe no-ops on fresh binds:
   ```java
   backend.evict(channelId);                          // clears bindingCache + slackToChannel + threadCache (in-memory)
   threadCacheStore.deleteAllForChannel(channelId);   // clears DB thread cache
   ```
   **Why this is required:** On rebind (same channel, different slackChannelId), the old `slackToChannel["C1"]` entry survives without evict(), causing C1's inbound messages to continue routing to this channel. Also, old thread_ts values from C1 are cached for this channel — post() would call the C2 bot API with C1's thread timestamps, causing Slack API failures or unexpected thread nesting. The clean-slate approach is unconditional to also handle tombstone cases (binding deleted and re-created).
5. Persist: field-assignment construction (see entity section), `bindingStore.put(b)`
6. `gateway.initChannel(channelId, new ChannelRef(channelId, channel.name))` — fires `ChannelInitialisedEvent`; backend self-registers. Wrap in try-catch:
   ```java
   try {
       gateway.initChannel(channelId, new ChannelRef(channelId, channel.name));
   } catch (DuplicateParticipatingBackendException e) {
       // Race: a ChannelConnectorBinding was added between step 2 and step 5.
       // The binding save at step 4 is in the same @Transactional method — catching here
       // before the exception reaches the @Transactional interceptor boundary leaves the
       // transaction active (not RollbackOnly). Undo the save and return 409.
       bindingStore.delete(channelId);
       return Response.status(409)
           .entity("Channel already has a participating backend: " + e.getMessage())
           .build();
   }
   ```
   `DuplicateParticipatingBackendException extends IllegalStateException`. Catching it within
   the method (not at the `@Transactional` boundary) keeps the transaction active. The
   `bindingStore.delete()` call in the catch block undoes the save in the same transaction.
   The transaction commits cleanly with no net change to the binding table.
7. 200 `SlackBindingView`

### DELETE `/{channelId}` (ordering matters — read cache before DB delete)

```
1. CacheEntry entry = channelCache.get(channelId)         // capture slackChannelId before evict()
2. If entry == null AND DB has no binding → return 204 immediately (idempotent DELETE)
3. backend.evict(channelId)                               // stops inbound routing atomically (in-memory only)
4. gateway.deregisterBackend(channelId, BACKEND_ID)       // stops fanOut routing
5. bindingStore.delete(channelId)                         // DB binding
6. 204
// Note: SlackThreadEntry DB rows are NOT deleted here.
// After evict(), post() finds no CacheEntry and returns early — rows are functionally orphaned.
// TTL cleanup (SlackThreadCacheCleanupJob) handles them. Comment in implementation explains.
```

**`put()` is intentionally NOT `@Transactional`.** Steps 5 and 6 commit in independent transactions. This is a deliberate design decision — do not add `@Transactional` to `put()` without understanding the consequence: if `put()` were `@Transactional`, `onChannelInitialised()` (step 7 via `initChannel()`) would join that outer transaction. When `registerBackend()` throws `DuplicateParticipatingBackendException` inside the observer, the `@Transactional` interceptor marks the OUTER transaction rollback-only before re-throwing. The catch block that calls `bindingStore.deleteByChannelId()` would then execute inside a rollback-only transaction, throwing `RollbackException` — the save cannot be undone. The split-transaction design avoids this: the observer's own transaction rolls back (reads-only, clean no-op), the exception re-throws to the catch block, which runs `bindingStore.deleteByChannelId()` in a fresh independent transaction. The known cost is that steps 5 and 6 are not atomic (see Known Limitations).

**Idempotent:** DELETE returns 204 whether or not the binding exists. Desired end state (no binding) is achieved in both cases.  
`close()` is NOT called from `delete()` — `close()` has channel-deletion semantics and is only called by `ChannelGateway.closeChannel()`.

---

## `SlackThreadCacheCleanupJob`

`@ApplicationScoped`. Both the cleanup interval and the retention period are configurable:

```java
@ConfigProperty(name = "casehub.qhorus.slack-channel.thread-cache-ttl-days",
                defaultValue = "30")
int threadCacheTtlDays;

// Cleanup interval is configured via the @Scheduled expression — not injected as a field.
// casehub.qhorus.slack-channel.thread-cache-cleanup-interval (default: 24h)
@Scheduled(every = "${casehub.qhorus.slack-channel.thread-cache-cleanup-interval:24h}")
void cleanup() {
    threadCache.deleteOlderThan(Instant.now().minus(threadCacheTtlDays, ChronoUnit.DAYS));
}
```

Cleans up rows for commitments that expired without a terminal message reaching the backend (server crash mid-commitment, network partition during DONE delivery).

---

## ConnectorChannelBackend — WARN→DEBUG

In `ConnectorChannelBackend.onInboundMessage()`, the log for "No channel for connector=%s key=%s — discarding" is currently at WARN. With `slack-channel` on the classpath, every Slack inbound event before a binding is configured produces this WARN. Change to DEBUG universally.

**Rationale for blanket WARN→DEBUG:** The counter `inbound_messages_discarded_total{connector_id}` is the correct alerting surface for misconfigured connectors. Log-level WARN is a weak substitute for counter-based alerting. The WARN was appropriate when the counter didn't exist; now that it does, DEBUG is correct.

**Deployment requirement:** The `inbound_messages_discarded_total` counter alert must be configured in the operations runbook. An alert threshold of >0 over 5 minutes on any `connector_id` other than `slack-inbound` (which has a dedicated handler) signals genuine misconfiguration.

---

## Testing

**Unit (CDI-free, constructor injection):**

`SlackChannelBackendTest` — `InMemorySlackBotBindingStore`, `InMemorySlackThreadCacheStore`, `@InjectMock SlackBotClient`, `@InjectMock ChannelGateway`. Cover:
- `post()`: EVENT type → immediate return (no postMessage call), null content → immediate return, missing CacheEntry → ERROR log + return, first post with corrId (top-level + cache written to memory and DB), second post same corrId (thread reply using cached ts), DONE/FAILURE/DECLINE evict from memory and DB, RESPONSE does NOT evict, HANDOFF does NOT evict, failed Slack API call → WARN + no cache mutation
- `onInboundMessage()`: non-Slack connector → filtered, unknown slackChannelId → DEBUG + counter
  - New top-level message (no `slack-thread-ts`): in-memory anchor set first, DB put attempted (best-effort), corrId passed in InboundHumanMessage, normaliser returns QUERY
  - Thread reply, cache hit: corrId found via DB reverse lookup (`slack-thread-ts → corrId`), corrId passed through, normaliser returns RESPONSE
  - Thread reply, cache miss: corrId generated, rootTs = `slackThreadTs` (not `slackTs` — thread root, not reply ts), in-memory first then DB best-effort, normaliser returns QUERY
  - **`threadCacheStore.put()` throws**: in-memory update already succeeded (set before try-catch); `receiveHumanMessage()` is still called; WARN logged; assert gateway call happened and anchor is in memory but not in DB
  - Null content (media-only message): passes through to gateway with no NPE; normaliser returns QUERY with null content
- `onChannelInitialised()` — these paths must be unit-tested (NOT bypassed by pre-populating `bindingCache` directly, which is the antipattern that hid the `@ObservesAsync` bug):
  - `_registration`: fire `ChannelInitialisedEvent` for a channel with a binding → verify `channelCache` and `slackIndex` populated, `gateway.registerBackend()` called
  - `_restartRecovery`: binding exists, `threadCacheStore.findByChannelId()` returns rows → verify rows loaded into `threadCache` inner map (corrId → threadTs)
  - `_unknownChannel`: fire event for channel with no binding → verify no registration, `channelCache` empty

**Required `onInboundMessage()` unit tests** — these paths currently have zero coverage; the rootTs bug and ORDERING INVARIANT are unverified:
  - `_newTopLevelMessage`: send message with no `slack-thread-ts` → verify new corrId generated, `threadCacheStore.put(channelId, corrId, slackTs)` called **before** `gateway.receiveHumanMessage()` (mock ordering)
  - `_unknownThreadReply`: send thread reply where `getCorrIdByThreadTs()` returns empty → verify `rootTs = slackThreadTs` (not `slackTs`), new corrId generated, DB written with slackThreadTs
  - `_knownThreadReply`: send thread reply where `getCorrIdByThreadTs()` returns existing corrId → verify no new anchor written, existing corrId passed in `InboundHumanMessage`, normaliser returns RESPONSE

`SlackInboundNormaliserTest` — pure logic. Cover: slash command → COMMAND, slash command with null content → QUERY (no NPE), thread reply + corrId → RESPONSE, new message → QUERY, thread-ts == slack-ts (human thread root) → QUERY.

**Integration (`@QuarkusTest` + H2):**

`SlackChannelBackendIT` — `@InjectMock SlackBotClient`. Lifecycle: bind → init event → inbound new message → corrId generated and written to DB → agent RESPONSE → thread reply asserted. Assert `slackBotClient.postMessage()` called `times(1)` — the normaliser telemetry EVENT from `receiveHumanMessage()` returns immediately at the EVENT type guard. Assert `messageService` called `times(2)` per inbound message (content dispatch + telemetry EVENT).

`SlackBindingResourceTest` (unit, CDI-free with mocked dependencies) — the endpoint behaviors below need explicit test coverage:
  - PUT → 200: valid workspaceId, slackChannelId, and configured token → binding persisted, 200 `SlackBindingView` returned
  - PUT → 400 (missing key): `Config.getValue()` throws `NoSuchElementException` → 400 with key name in body
  - PUT → 400 (blank token): `Config.getValue()` returns `""` → 400 with "Credential ... is configured but empty"
  - PUT → 409 (generic binding exists): `channelBindingStore.findByChannelId()` returns a `ChannelConnectorBinding` → 409 before any DB write
  - PUT → 409 (DuplicateParticipatingBackendException): `gateway.initChannel()` throws → 409, `bindingStore.findByChannelId()` returns empty (binding save undone in same transaction)
  - DELETE → 204 (binding present): evict() called, deregisterBackend() called, bindingStore.delete() called, 204
  - DELETE → 204 (no binding, idempotent): no-op, 204 immediately
  - GET → 200: binding exists, token field absent from response
  - GET → 404: no binding

`FlywayMigrationSchemaTest` — plain-Java Flyway + H2. Runs V1–V24 + V2000. Asserts both tables with correct columns, constraints, indexes.

---

## Known limitations

**Recovery anchor / terminal eviction race in post():** `save()` (recovery anchor) and `delete()` (terminal eviction) in `post()` run in independent `@Transactional(REQUIRED)` transactions on virtual threads from `fanOut()`. If RESPONSE and DONE arrive for the same corrId in rapid succession, their virtual threads may interleave: if DONE's `delete()` runs before RESPONSE's `save()`, the anchor is inserted after the eviction — net: anchor present when it should be absent. In well-formed Qhorus conversations, RESPONSE always precedes DONE (agent sends RESPONSE before DONE — separate HTTP calls processed sequentially). This ordering guarantees RESPONSE's `post()` starts before DONE's `post()` starts, making the race vanishingly unlikely. For v1, rely on the FIFO guarantee from the agent protocol and document the assumption.

**RESPONSE-no-evict follow-up: protocol noise.** RESPONSE (agent answers human QUERY) does not evict the thread anchor. If the human follows up in the same Slack thread, `findCorrIdByThreadTs()` finds the original corrId and the normaliser returns `MessageType.RESPONSE`. The original QUERY commitment is already discharged. Whether Qhorus silently ignores the second RESPONSE or surfaces it to the agent depends on its commitment store validation. This is observable as unexpected RESPONSE messages for discharged commitments. Mitigation: DONE/FAILURE/DECLINE evict the anchor, so post-terminal follow-ups correctly start new QUERY commitments.

**Slack event subtype filtering is SlackInboundConnector's responsibility (connectors#22).** `SlackChannelBackend` receives `InboundMessage` events already filtered by `SlackInboundConnector`. The connector currently does NOT filter by `subtype`. Events with `type="message"` and no `bot_id` pass through regardless of subtype — including `message_changed` (edited messages), `message_deleted`, `channel_join`. Every message edit in an active workspace generates a new Qhorus COMMAND/QUERY, creating duplicate commitments and agent work. The fix is a one-line addition to `SlackInboundConnector.parseMessages()`: `if (event.getString("subtype", null) != null) return messages;`. This is tracked at casehub-connectors#22 and must be fixed before shipping to a real workspace.

**Reverse mutual exclusion not enforced.** `SlackBindingResource` rejects a Slack binding when a generic `ChannelConnectorBinding` exists (409). The inverse is not enforced — `connector-backend` must not depend on `slack-channel`. Follow-up issue.

**Thread cache miss creates a new top-level bot message, not a thread reply to the human.** On a cache miss in `post()` (e.g., TTL-evicted entry, `slackTs` was null on inbound), the recovery anchor posts as a new top-level Slack message. The commitment's subsequent posts (DONE, STATUS, further RESPONSE) thread under that bot message, not under the human's original message. The human's original question appears "unanswered" in Slack. This is the correct recovery path (avoids total loss of threading) but the UX is visibly different from the normal path. Monitoring: `slack_thread_miss_total{channel_id}` counter.

**Delivery gap during PUT (rebind).** Between step 4 (`evict()` clears in-memory state) and step 7 (`onChannelInitialised()` re-registers the backend), any `fanOut()` call to `post()` for the affected channel finds no `bindingCache` entry and silently returns early. Agent messages dispatched during this window are dropped with no error visible to the operator. The window is the latency of one synchronous `initChannel()` + `onChannelInitialised()` call — typically sub-second on a local deployment. For an admin rebind (deliberate reconfiguration), a narrow delivery gap is the expected trade-off. This is distinct from the non-atomicity of steps 5–6 (below).

**PUT steps 5 and 6 are not atomic.** `threadCacheStore.deleteAllForChannel()` (step 5) and `bindingStore.save()` (step 6) run in independent `@Transactional(REQUIRED)` transactions. If step 6 fails after step 5 commits, the DB thread cache is purged but the new binding is not saved — channel has no binding in DB and no in-memory state. On the next inbound Slack message, `slackToChannel` has no entry → silently discarded. Recovery: re-issue the PUT. DB failure at step 6 is rare; the operation is idempotent. For v1 this is acceptable. If atomicity is required in a future iteration, wrapping `put()` with `@Transactional` makes steps 5 and 6 join the same tx — if step 6 fails, step 5 rolls back. In-memory `evict()` (step 4) cannot participate in JPA transactions and is not rolled back, but this is self-healing: the next `onChannelInitialised()` call re-populates the in-memory state.

**`@Transactional` on `JpaSlackThreadCacheStore` write methods is required.** Without it, `put()` and `delete()` called from `post()` (virtual thread, no JTA tx) and from `onInboundMessage()` (`@ObservesAsync` ManagedExecutor, no active JTA tx) throw `TransactionRequiredException`. `fanOut()`'s `catch (Exception ex)` swallows these silently — the Slack post succeeds but anchors/evictions never fire.

---

## Thread lifecycle — data flow

The complete trace from a human Slack message to an agent thread reply:

**Inbound (human Slack message → Qhorus QUERY/RESPONSE):**
1. `SlackInboundConnector` receives Slack webhook → fires `InboundMessage(connectorId="slack-inbound", externalChannelRef=slackChannelId, metadata={"slack-ts": ts, "slack-thread-ts"?: threadTs, ...})`
2. `SlackChannelBackend.onInboundMessage()`:
   - Route to `ChannelRef` via `slackIndex` (O(1) — slackChannelId → Qhorus channelId)
   - Thread context resolution:
     - Thread reply + cache hit: `slackThreadTs` → DB reverse lookup → corrId (RESPONSE path)
     - New message or cache miss: generate `UUID corrId`, compute `rootTs = slackThreadTs ?? slackTs`
   - **ORDERING INVARIANT:** write `(channelId, corrId, rootTs)` to DB and memory **before** `receiveHumanMessage()`
   - Call `gateway.receiveHumanMessage(channelRef, InboundHumanMessage(corrId=corrId.toString()))`
3. `SlackInboundNormaliser.normalise()` — infers COMMAND / RESPONSE / QUERY; passes `raw.correlationId()` through
4. `MessageService.dispatch(MessageDispatch(correlationId=corrId.toString()))` → persists `Message`, writes ledger entry
5. `fanOut()` fires for the dispatched message and for the normaliser telemetry EVENT; both call `post()`. The EVENT returns immediately at the type guard. No net Slack API call for the inbound dispatch.

**Outbound (agent RESPONSE → Slack thread reply):**
6. Agent reads QUERY via `check_messages`, sees `correlationId=corrId`
7. Agent calls `send_message(type=RESPONSE, correlationId=corrId, inReplyTo=queryMessageId)`
8. `MessageService.dispatch()` → persists → `fanOut()` spawns virtual thread
9. `SlackChannelBackend.post(channel, OutboundMessage(type=RESPONSE, correlationId=UUID))`
10. Thread lookup: `corrIdStr → threadTs` from memory (loaded at channel init); DB fallback if memory miss (e.g. post-restart)
11. `slackBotClient.postMessage(token, slackChannelId, content, threadTs=rootTs)` → reply appears in the human's original Slack thread

**Key invariant the data flow protects:** The thread anchor (step 2 write) happens before `receiveHumanMessage()` (step 2 call), which happens before the message is visible to the agent (step 6). This ordering guarantees that when the agent sends RESPONSE and `post()` runs (step 9), the thread anchor already exists in both memory and DB.

---

## Design decisions and non-obvious invariants

This section captures decisions that would be non-obvious from the code alone.

| Decision | Why |
|---|---|
| Generate corrId in `onInboundMessage()`, not in `post()` | `post()` runs on a virtual thread from `fanOut()`. Generating corrId there would require a thread cache write AFTER the outbound message is dispatched — wrong direction. The corrId must exist before the inbound message reaches the agent. |
| `rootTs = slackThreadTs ?? slackTs` for cache-miss thread replies | Slack `thread_ts` must equal the ROOT message's timestamp. Storing the reply's `slackTs` would cause Slack to reject the `postMessage` or create a sub-sub-thread. |
| DB-backed thread cache, not in-memory only | Server restarts between a COMMAND and its RESPONSE would send the reply to channel root without DB persistence. The `onChannelInitialised()` recovery load restores the mapping. |
| Separate `close()` / `evict()` / `delete()` paths | `close()` is channel-deletion (total cleanup including DB rows). `evict()` is admin unbinding (in-memory only — in-flight commits can still resolve). `delete()` is the REST endpoint (calls `evict()`, then separately deletes DB thread rows because the binding is gone so orphaned rows serve no purpose). |
| UUID PK on `SlackThreadCache`, not BIGINT | All qhorus entities use UUID PKs; BIGINT would require a sequence in the DDL and `@GeneratedValue` in the entity, diverging from platform convention for no benefit at this scale. |
| Thread anchor = outbound only, not inbound corrId source | `slackThreadTs` determines WHERE the agent's reply appears in Slack (rootTs for post()), but never which corrId to reuse for inbound routing. Each inbound human message always generates a fresh corrId — per-message commitment model. Reusing a previous corrId would route a human follow-up to an already-closed or unrelated Qhorus commitment. The reverse lookup `(channelId, rootTs) → corrId` was removed: (1) it maps to multiple corrIds in the same Slack thread (non-unique); (2) its only coherent use case is a per-thread commitment model, which conflicts with per-message. No `idx_slack_thread_root_ts` index. |
| Per-binding credential reference (PP-20260617-per-binding-credential-ref) | Static `@ConfigProperty` is single-workspace (Tier 1). `EndpointRegistry.credentialRef` is not yet implemented (Tier 2 — deferred, platform#103). Per-binding reference stores a logical name in `credential_ref`, resolved from `casehub.qhorus.slack-channel.credentials.<name>` at call time — supports multiple Slack workspaces without a secrets backend. Token never stored in DB. |
| Explicit DONE/FAILURE/DECLINE enumeration for eviction (not `isTerminal()`) | `MessageType.isTerminal()` is wrong in BOTH directions for this use: (1) it returns true for HANDOFF — HANDOFF must NOT evict, because the delegated agent continues in the same Slack thread; (2) it returns false for DECLINE — DECLINE must evict, because the commitment is rejected and the conversation is done. Explicit `DONE \|\| FAILURE \|\| DECLINE` is the only correct predicate. |
| Blanket WARN→DEBUG in `ConnectorChannelBackend` | Every Slack inbound event before a binding is configured fires WARN in `ConnectorChannelBackend` (which has no `ChannelConnectorBinding`). The `inbound_messages_discarded_total{connector_id}` counter is the correct alerting surface. Coupling `ConnectorChannelBackend` to `SlackChannelBackend` to be selective would violate module independence. |

---

## Key protocols applied

| Protocol | Application |
|----------|-------------|
| `per-binding-credential-reference` (PP-20260617) | `credentialRef` in DB; token resolved from MP Config at call time; never stored or returned |
| `module-tier-structure` | Store SPI interfaces in main; JPA impls in main; in-memory impls in `testing/` |
| `flyway-version-range-allocation` Rule 4 + global V numbering | Scoped `db/slack-channel/migration/` path; V23/V24 avoid collision with V1–V22 in `db/qhorus/migration/` |
| `maven-submodule-folder-naming` | Folder `slack-channel/`, artifactId `casehub-qhorus-slack-channel` |
| Qhorus entity convention | UUID PKs, public fields, `@PrePersist` for ID/timestamp generation, no all-args constructors |
| `MessageType.isTerminal()` scope | Returns true for HANDOFF, DONE, FAILURE — not DECLINE. Never use for thread cache eviction; use explicit DONE/FAILURE/DECLINE enumeration |
| Thread anchoring ordering | corrId generated and cached to DB BEFORE `receiveHumanMessage()` — eliminates race with fast agent RESPONSE |
| Startup recovery | `onChannelInitialised()` loads all DB thread rows into in-memory cache — in-flight commitments survive restart |

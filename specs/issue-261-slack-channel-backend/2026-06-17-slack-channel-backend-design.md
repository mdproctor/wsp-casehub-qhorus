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
@Table(name = "slack_bot_binding")
public class SlackBotBinding extends PanacheEntityBase {

    @Id
    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(name = "credential_ref", nullable = false, length = 128)
    public String credentialRef;

    @Column(name = "slack_channel_id", nullable = false, length = 64)
    public String slackChannelId;
}
```

Construction (field assignment — no all-args constructor, matching qhorus entity convention):
```java
var b = new SlackBotBinding();
b.channelId = channelId;
b.credentialRef = request.credentialRef();
b.slackChannelId = request.slackChannelId();
bindingStore.put(b);
```

### `SlackThreadCache` — JPA entity, `qhorus` PU

UUID PK consistent with all other qhorus entities. `@PrePersist` generates the ID on first persist.

```java
@Entity
@Table(name = "slack_thread_cache",
       uniqueConstraints = @UniqueConstraint(
           name = "uq_slack_thread_corr",
           columnNames = {"channel_id", "correlation_id"}))
public class SlackThreadCache extends PanacheEntityBase {

    @Id
    @Column(name = "id", nullable = false)
    public UUID id;

    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(name = "correlation_id", nullable = false, length = 36)
    public String correlationId;   // UUID.toString() — 36 chars

    @Column(name = "thread_ts", nullable = false, length = 64)
    public String threadTs;        // Slack message timestamp, e.g. "1234567890.123456"

    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
        if (createdAt == null) createdAt = Instant.now();
    }
}
```

**Why UUID, not BIGINT:** All qhorus entities (`Channel`, `Message`, `Commitment`, `ChannelConnectorBinding`) use UUID PKs, application-generated. BIGINT identity would require a sequence, adding DDL complexity and diverging from the platform convention for no performance benefit at this scale.

---

## Flyway migrations

**Directory:** `slack-channel/src/main/resources/db/slack-channel/migration/`  
Separate scoped path per Rule 4 of `flyway-version-range-allocation.md`. Version numbers remain global across all locations for the same datasource. V23/V24 avoid collision with V1–V22 in `db/qhorus/migration/`.

### V23__slack_bot_binding.sql

```sql
CREATE TABLE slack_bot_binding (
    channel_id        UUID         NOT NULL,
    credential_ref    VARCHAR(128) NOT NULL,
    slack_channel_id  VARCHAR(64)  NOT NULL,
    CONSTRAINT pk_slack_bot_binding
        PRIMARY KEY (channel_id),
    CONSTRAINT fk_slack_binding_channel
        FOREIGN KEY (channel_id) REFERENCES channel(id)
);
```

### V24__slack_thread_cache.sql

```sql
CREATE TABLE slack_thread_cache (
    id              UUID        NOT NULL,
    channel_id      UUID        NOT NULL,
    correlation_id  VARCHAR(36) NOT NULL,
    thread_ts       VARCHAR(64) NOT NULL,
    created_at      TIMESTAMP   NOT NULL,
    CONSTRAINT pk_slack_thread_cache PRIMARY KEY (id),
    CONSTRAINT uq_slack_thread_corr  UNIQUE (channel_id, correlation_id)
);
CREATE INDEX idx_slack_thread_cache_ts ON slack_thread_cache (channel_id, thread_ts);
```

The `(channel_id, thread_ts)` index enables O(1) inbound reverse lookup (Slack `thread_ts` → `correlationId`) without a full table scan.

---

## Runtime config changes — `runtime/src/main/resources/application.properties`

Two additions required. Both are safe when the module is absent (no jar → no resources found, no packages discovered):

```properties
# 1. Include slack-channel Flyway migrations when jar is on classpath
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/slack-channel/migration

# 2. Extend Hibernate entity scan to cover io.casehub.qhorus.slack.channel package
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus,io.casehub.ledger.runtime
```

The `packages` expansion changes `io.casehub.qhorus.runtime` → `io.casehub.qhorus` — a superset that includes the new package while retaining all existing entities.

**Module test `application.properties`** must include both additions and use `drop-and-create` (no Flyway in test cycle).

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

**`JpaSlackThreadCacheStore`** — `@ApplicationScoped`, named `qhorus` PU.  
**`InMemorySlackThreadCacheStore`** — `@Alternative @Priority(1) @ApplicationScoped` in `testing/`.

---

## `SlackChannelBackend`

`@ApplicationScoped`, `BACKEND_ID = "slack-bot"`, implements `HumanParticipatingChannelBackend`. Constructor injection throughout, matching `ConnectorChannelBackend` convention.

### In-memory indexes

```java
// channelId → (credentialRef, slackChannelId, channelName)
private final ConcurrentHashMap<UUID, CacheEntry> channelCache = new ConcurrentHashMap<>();
// slackChannelId → ChannelRef  (inbound routing — O(1) reverse lookup)
private final ConcurrentHashMap<String, ChannelRef> slackIndex = new ConcurrentHashMap<>();
// channelId → (correlationId → threadTs)  (write-through over DB; populated at channel init)
private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, String>> threadCache = new ConcurrentHashMap<>();

private record CacheEntry(String credentialRef, String slackChannelId, String channelName) {}
```

### `@Observes ChannelInitialisedEvent` (sync)

Registration and **restart recovery** — both happen here:

1. `bindingStore.findByChannelId(channelId)` — if absent, skip (not a Slack-backed channel)
2. Populate `channelCache` and `slackIndex` from binding + event
3. **Load thread cache from DB** — critical for restart recovery:
   ```java
   ConcurrentHashMap<String, String> channelThread = new ConcurrentHashMap<>();
   threadCacheStore.findByChannelId(channelId)
       .forEach(e -> channelThread.put(e.correlationId, e.threadTs));
   threadCache.put(channelId, channelThread);
   ```
   After a server restart, in-flight commitments (COMMAND dispatched, RESPONSE not yet received) retain their thread context. Without this load, `post()` would miss the cached `thread_ts` and the agent's RESPONSE would go to the channel root rather than the original Slack thread.
4. `gateway.deregisterBackend(channelId, BACKEND_ID)` — dedup guard
5. `gateway.registerBackend(channelId, this, "human_participating")`

### `close(ChannelRef)` — channel deletion hook

Called only by `ChannelGateway.closeChannel()`. Total cleanup:

```java
CacheEntry entry = channelCache.remove(channel.id());
if (entry != null) slackIndex.remove(entry.slackChannelId());
threadCache.remove(channel.id());
threadCacheStore.deleteAllForChannel(channel.id());
bindingStore.delete(channel.id());
```

### `evict(UUID channelId)` — package-private, admin unbinding

Removes from in-memory indexes only. Does NOT touch DB thread cache rows — post-unbinding in-flight commitments may still resolve (agent RESPONSE arrives, `post()` finds no `CacheEntry`, logs DEBUG, returns; thread cache rows are cleaned by TTL job).

```java
CacheEntry entry = channelCache.remove(channelId);
if (entry != null) slackIndex.remove(entry.slackChannelId());
threadCache.remove(channelId);
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

String token = resolveToken(entry.credentialRef());

// OutboundMessage.correlationId is null for COMMAND, QUERY, EVENT;
// non-null for RESPONSE, DONE, FAILURE, DECLINE, STATUS.
String threadTs = null;
if (message.correlationId() != null) {
    String corrIdStr = message.correlationId().toString();
    // Memory-first lookup (in-memory cache populated at channel init, write-through on anchoring)
    Map<String, String> channelThread = threadCache.get(channel.id());
    threadTs = channelThread != null ? channelThread.get(corrIdStr) : null;
    if (threadTs == null) {
        // Fallback to DB (e.g. restart with no prior fanOut for this corrId)
        threadTs = threadCacheStore.findThreadTs(channel.id(), corrIdStr).orElse(null);
    }
}

PostResult result = slackBotClient.postMessage(token, entry.slackChannelId(),
                                                message.content(), threadTs);
if (!result.ok()) {
    LOG.warnf("Slack post failed on channel %s: %s", channel.name(), result.error());
    meterRegistry.counter("slack_post_failures_total",
                          "channel_id", channel.id().toString()).increment();
    return;
}

// Track thread misses — correlationId was set but no anchor exists in memory or DB.
// This should not happen in normal operation: onInboundMessage() writes the anchor before
// calling receiveHumanMessage(), so post() should always find it. A miss here indicates
// an edge case (slackTs was null on inbound, or a DB write failure in onInboundMessage()).
if (message.correlationId() != null && threadTs == null) {
    meterRegistry.counter("slack_thread_miss_total",
                          "channel_id", channel.id().toString()).increment();
}

// Anchor thread on first successful post for this correlationId
if (message.correlationId() != null && threadTs == null && result.ts() != null) {
    String corrIdStr = message.correlationId().toString();
    threadCache.computeIfAbsent(channel.id(), k -> new ConcurrentHashMap<>())
               .put(corrIdStr, result.ts());
    threadCacheStore.put(channel.id(), corrIdStr, result.ts());
}

// Evict on terminal commitment resolution.
// Do NOT use MessageType.isTerminal() — it returns true for HANDOFF (must not evict;
// delegated agent continues the same thread) and false for DECLINE (must evict).
// Use explicit enumeration.
if ((message.type() == DONE || message.type() == FAILURE || message.type() == DECLINE)
        && message.correlationId() != null) {
    String corrIdStr = message.correlationId().toString();
    Map<String, String> channelThread = threadCache.get(channel.id());
    if (channelThread != null) channelThread.remove(corrIdStr);
    threadCacheStore.delete(channel.id(), corrIdStr);
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

if (slackThreadTs != null && !slackThreadTs.equals(slackTs)) {
    // Thread reply — reverse-lookup the corrId that anchored this thread.
    // Missing: treat as new conversation (human replied to a thread we didn't create).
    corrId = threadCacheStore.findCorrelationId(channelRef.id(), slackThreadTs).orElse(null);
} else {
    corrId = null;
}

if (corrId == null) {
    // New top-level message OR unrecognised thread reply — generate corrId and anchor.
    UUID newCorrId = UUID.randomUUID();
    corrId = newCorrId.toString();

    // For a top-level message: slackTs is the root. For an unrecognised thread reply:
    // slackThreadTs is the root — the reply's own slackTs must NOT be used as thread_ts,
    // or Slack will reject the reply (thread_ts must equal the root message ts, not a reply ts).
    String rootTs = (slackThreadTs != null) ? slackThreadTs : slackTs;

    // ORDERING INVARIANT: write to thread cache BEFORE calling receiveHumanMessage().
    // receiveHumanMessage() dispatches the COMMAND/QUERY synchronously, which triggers
    // fanOut(). If a RESPONSE arrives at post() before this write completes, post()
    // misses the cache and posts to the channel root instead of the correct Slack thread.
    // The write must complete before the gateway call — this ordering is non-negotiable.
    if (rootTs != null) {
        threadCacheStore.put(channelRef.id(), corrId, rootTs);
        threadCache.computeIfAbsent(channelRef.id(), k -> new ConcurrentHashMap<>())
                   .put(corrId, rootTs);
    }
}

// content may be null for media-only Slack messages (images, files, voice).
// QUERY and RESPONSE accept null content — MessageDispatch.build() has no content
// requirement for these types.
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

        // Type inference — ordered by specificity:
        // 1. Slash command → COMMAND (content != null required; null content → QUERY)
        // 2. Thread reply with known corrId → RESPONSE
        // 3. Default → QUERY
        final MessageType type;
        if (content != null && content.startsWith("/")) {
            type = MessageType.COMMAND;
        } else if (slackThreadTs != null && !slackThreadTs.equals(slackTs)
                   && raw.correlationId() != null) {
            type = MessageType.RESPONSE;
        } else {
            type = MessageType.QUERY;
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

**Slash-command detection note:** COMMAND type is inferred from a leading `/`. The null guard (`content != null`) prevents NPE for media-only Slack messages. COMMAND requires neither correlationId nor inReplyTo in `MessageDispatch.build()` — the agent receives the COMMAND and responds independently.

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
public record SlackBindingRequest(String credentialRef, String slackChannelId) {}

/** GET response body — token never returned */
public record SlackBindingView(UUID channelId, String credentialRef, String slackChannelId) {
    static SlackBindingView from(UUID channelId, SlackBotBinding b) {
        return new SlackBindingView(channelId, b.credentialRef, b.slackChannelId);
    }
}
```

### PUT `/{channelId}`

1. `channelService.findById(channelId)` — 404 if channel absent
2. `channelBindingStore.findByChannelId(channelId)` — 409 if generic `ChannelConnectorBinding` exists
3. Validate credential: `Config.getValue("casehub.qhorus.slack-channel.credentials." + credentialRef)` — 400 with key name if `NoSuchElementException`
4. Persist: field-assignment construction (see entity section), `bindingStore.put(b)`
5. `gateway.initChannel(channelId, new ChannelRef(channelId, channel.name))` — fires `ChannelInitialisedEvent`; backend self-registers. Wrap in try-catch:
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
6. 200 `SlackBindingView`

### DELETE `/{channelId}` (ordering matters — read cache before DB delete)

```
1. CacheEntry entry = channelCache.get(channelId)    // must capture slackChannelId before evict()
2. 404 if entry == null and DB has no binding
3. backend.evict(channelId)                          // stops inbound routing atomically
4. gateway.deregisterBackend(channelId, BACKEND_ID)  // stops fanOut routing
5. threadCacheStore.deleteAllForChannel(channelId)   // purge thread history — after evict(), post()
                                                     // returns immediately on missing CacheEntry
                                                     // so rows are orphaned; delete is correct cleanup
6. bindingStore.delete(channelId)                    // DB binding last
7. 204
```

`close()` is NOT called from `delete()` — `close()` has channel-deletion semantics and is only called by `ChannelGateway.closeChannel()`.

---

## `SlackThreadCacheCleanupJob`

`@ApplicationScoped`. Both the cleanup interval and the retention period are configurable:

```java
@ConfigProperty(name = "casehub.qhorus.slack-channel.thread-cache-cleanup-interval",
                defaultValue = "24h")
String cleanupInterval;   // used in @Scheduled(every = "${...}")

@ConfigProperty(name = "casehub.qhorus.slack-channel.thread-cache-ttl-days",
                defaultValue = "30")
int threadCacheTtlDays;

@Scheduled(every = "${casehub.qhorus.slack-channel.thread-cache-cleanup-interval:24h}")
void cleanup() {
    threadCacheStore.deleteOlderThan(Instant.now().minus(threadCacheTtlDays, ChronoUnit.DAYS));
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
  - New top-level message (no `slack-thread-ts`): corrId generated, rootTs = `slackTs`, written to DB BEFORE gateway call (ordering invariant), corrId passed in InboundHumanMessage, normaliser returns QUERY
  - Thread reply, cache hit: corrId found via DB reverse lookup (`slack-thread-ts → corrId`), corrId passed through, normaliser returns RESPONSE
  - Thread reply, cache miss: corrId generated, rootTs = `slackThreadTs` (not `slackTs` — thread root, not reply ts), written to DB BEFORE gateway call, normaliser returns QUERY
  - Null content (media-only message): passes through to gateway with no NPE; normaliser returns QUERY with null content
- `onChannelInitialised()`: DB rows loaded into in-memory cache (restart recovery path)

`SlackInboundNormaliserTest` — pure logic. Cover: slash command → COMMAND, slash command with null content → QUERY (no NPE), thread reply + corrId → RESPONSE, new message → QUERY, thread-ts == slack-ts (human thread root) → QUERY.

**Integration (`@QuarkusTest` + H2):**

`SlackChannelBackendIT` — `@InjectMock SlackBotClient`. Lifecycle: bind → init event → inbound new message → corrId generated and written to DB → agent RESPONSE → thread reply asserted. Assert `slackBotClient.postMessage()` called `times(1)` — the normaliser telemetry EVENT from `receiveHumanMessage()` returns immediately at the EVENT type guard. Assert `messageService` called `times(2)` per inbound message (content dispatch + telemetry EVENT).

`SlackBindingResourceIT` — PUT (200), GET (200, no token), PUT unknown credentialRef (400 with key name), DELETE (204), double-DELETE (404), PUT on channel with generic `ChannelConnectorBinding` pre-check (409 from step 2), PUT with mocked `gateway.initChannel()` throwing `DuplicateParticipatingBackendException` → 409 with no net binding in store (tests the race-condition catch path: binding save is undone in the same transaction, `findByChannelId()` returns empty after the call).

`FlywayMigrationSchemaTest` — plain-Java Flyway + H2. Runs V1–V24 + V2000. Asserts both tables with correct columns, constraints, indexes.

---

## Known limitations

**Reverse mutual exclusion not enforced.** `SlackBindingResource` rejects a Slack binding when a generic `ChannelConnectorBinding` exists (409). The inverse is not enforced — `connector-backend` must not depend on `slack-channel`. Follow-up issue.

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
| Tier 1.5 credential reference | Tier 1 (`@ConfigProperty`) is single-workspace. Tier 2 (`EndpointRegistry.credentialRef`) is not yet implemented. Tier 1.5 stores a logical name in DB, resolved from `casehub.qhorus.slack-channel.credentials.<name>` at call time — supports multiple Slack workspaces without a secrets backend. |
| Explicit DONE/FAILURE/DECLINE enumeration for eviction | `MessageType.isTerminal()` returns true for HANDOFF — must NOT evict (delegated agent continues in same Slack thread). DECLINE is not in `isTerminal()` but must evict. Using `isTerminal()` would be wrong in both directions. |
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

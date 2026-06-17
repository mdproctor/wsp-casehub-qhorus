# casehub-qhorus-slack-channel — Design Spec

**Issue:** qhorus#261  
**Branch:** issue-261-slack-channel-backend  
**Date:** 2026-06-17  
**Revised:** 2026-06-17 (r3 — post second code-review)

---

## Overview

New optional Maven module `casehub-qhorus-slack-channel` (`slack-channel/`) in the qhorus project. Implements `HumanParticipatingChannelBackend` using `SlackBotClient` directly — bypassing the generic `ConnectorService` path — to enable thread-aware Slack delivery. Activates by classpath presence.

This is the qhorus-side counterpart to `casehub-connectors-slack-bot` (connectors#2, already shipped).

---

## Module Structure

```
qhorus/
  slack-channel/
    pom.xml                                         — artifact: casehub-qhorus-slack-channel
    src/main/java/io/casehub/qhorus/slack/
      SlackBotBinding.java                          — JPA entity
      SlackBotBindingStore.java                     — JPA repository
      SlackThreadCache.java                         — JPA entity
      SlackThreadCacheId.java                       — embedded composite PK
      SlackThreadCacheStore.java                    — JPA repository
      SlackChannelBackend.java                      — HumanParticipatingChannelBackend impl
      SlackInboundNormaliser.java                   — InboundNormaliser impl
      SlackBindingResource.java                     — REST /slack-channel/bindings
      SlackBindingRequest.java                      — request record
      SlackBindingDto.java                          — response record
      SlackThreadCacheCleanupJob.java               — @Scheduled TTL eviction
    src/main/resources/db/qhorus/migration/
      V23__slack_bot_binding.sql
      V24__slack_thread_cache.sql
```

**Dependencies (pom.xml):**
- `casehub-qhorus-api` — `HumanParticipatingChannelBackend`, `InboundNormaliser`, `OutboundMessage`
- `casehub-qhorus` (runtime) — `ChannelGateway`, `ChannelService`, JPA persistence, `@QuarkusPersistenceUnit("qhorus")`
- `casehub-connectors-slack-bot` 0.2-SNAPSHOT — `SlackBotClient`
- `casehub-connectors-core` — `InboundMessage`, `InboundConnectorIds`
- Test: `casehub-qhorus-testing`, `casehub-platform` (MockCurrentPrincipal), `quarkus-jdbc-h2`, `quarkus-junit5`, `quarkus-junit5-mockito`

---

## Schema

### V23__slack_bot_binding.sql

```sql
CREATE TABLE slack_bot_binding (
    channel_id       UUID         NOT NULL,
    slack_channel_id VARCHAR(32)  NOT NULL,
    workspace_id     VARCHAR(32)  NOT NULL,
    created_at       TIMESTAMP    NOT NULL,
    CONSTRAINT pk_slack_bot_binding      PRIMARY KEY (channel_id),
    CONSTRAINT fk_slack_binding_channel  FOREIGN KEY (channel_id) REFERENCES channel(id)
);
```

`workspace_id` is the Slack team/workspace ID (e.g. "T123ABC"). It doubles as the credential reference key: `casehub.qhorus.slack-channel.credentials.<workspaceId>=xoxb-...`. Using the workspace ID rather than an operator-invented name makes it structured, auto-discoverable from inbound metadata (`"workspace-id"` key), and verifiable.

### V24__slack_thread_cache.sql

```sql
CREATE TABLE slack_thread_cache (
    channel_id     UUID        NOT NULL,
    correlation_id UUID        NOT NULL,
    thread_ts      VARCHAR(32) NOT NULL,
    created_at     TIMESTAMP   NOT NULL,
    CONSTRAINT pk_slack_thread_cache  PRIMARY KEY (channel_id, correlation_id),
    CONSTRAINT uq_slack_thread_ts     UNIQUE (channel_id, thread_ts)
);
```

Migration location: `slack-channel/src/main/resources/db/qhorus/migration/`. Flyway merges V23 and V24 with runtime's V1–V22 and V2000 via classpath scanning of `db/qhorus/migration/`.  
No FK from `slack_thread_cache` to `channel` — avoids cascade delete of in-flight thread entries.

---

## Entities

### SlackBotBinding
```java
@Entity @Table(name = "slack_bot_binding")
public class SlackBotBinding {
    @Id public UUID channelId;
    public String slackChannelId;   // Slack channel ID, e.g. "C123ABC"
    public String workspaceId;      // Slack workspace ID, e.g. "T123ABC" — also the credential ref key
    public Instant createdAt;
}
```

Credential resolution (Tier 1.5 per `casehub/garden: docs/protocols/casehub/per-binding-credential-reference.md`):
```java
private String resolveToken(String workspaceId) {
    return ConfigProvider.getConfig()
        .getValue("casehub.qhorus.slack-channel.credentials." + workspaceId, String.class);
}
```
Operator sets: `casehub.qhorus.slack-channel.credentials.T123ABC=xoxb-...`  
Raw token never stored in DB or returned over REST.

### SlackThreadCache

```java
@Entity @Table(name = "slack_thread_cache")
public class SlackThreadCache {
    @EmbeddedId public SlackThreadCacheId id;   // (channelId, correlationId)
    public String threadTs;                      // Slack root message ts, e.g. "1718567890.123456"
    public Instant createdAt;
}

@Embeddable
public record SlackThreadCacheId(UUID channelId, UUID correlationId) {}
```

**Thread cache is DB-backed with in-memory write-through cache.** Entries are keyed by `(channelId, correlationId)` — one per in-flight commitment, not per channel. A single channel can have many concurrent commitments. DB persistence preserves thread continuity across server restarts: without it, a restart mid-commitment causes the next outbound message to create a new top-level Slack message rather than a thread reply, silently fragmenting the conversation. Overhead: one SELECT per channel on startup, one INSERT per new commitment, one DELETE on terminal state (DONE/FAILURE/DECLINE). Not per-message.

In-memory structure: `ConcurrentHashMap<UUID, ConcurrentHashMap<UUID, String>> threadCache` — outer key is channelId, inner key is correlationId, value is threadTs. Forward-direction only (corrId → threadTs). Reverse lookups (threadTs → corrId) use the DB `UNIQUE (channel_id, thread_ts)` index.

---

## REST Types

```java
/** Request body for PUT /slack-channel/bindings/{channelId} */
public record SlackBindingRequest(String slackChannelId, String workspaceId) {}

/** Response body for GET /slack-channel/bindings/{channelId} */
public record SlackBindingDto(UUID qhorusChannelId, String slackChannelId, String workspaceId) {
    static SlackBindingDto from(UUID channelId, SlackBotBinding b) {
        return new SlackBindingDto(channelId, b.slackChannelId, b.workspaceId);
    }
}
```

`workspaceId` is included in the response for operator confirmation. The resolved bot token is never included.

---

## SlackChannelBackend

`@ApplicationScoped`, implements `HumanParticipatingChannelBackend`. BACKEND_ID = `"slack-bot"`. Constructor injection throughout, matching `ConnectorChannelBackend` convention.

**Internal state:**
```java
// Forward: channelId → SlackBotBinding (for post())
private final ConcurrentHashMap<UUID, SlackBotBinding> bindingCache = new ConcurrentHashMap<>();
// Reverse: slackChannelId → ChannelRef (for onInboundMessage())
private final ConcurrentHashMap<String, ChannelRef> slackToChannel = new ConcurrentHashMap<>();
// Thread cache: channelId → (correlationId → threadTs), loaded from DB at channel init
private final ConcurrentHashMap<UUID, ConcurrentHashMap<UUID, String>> threadCache = new ConcurrentHashMap<>();
```

### onChannelInitialised(@Observes ChannelInitialisedEvent)
1. Look up binding via `bindingStore.findByChannelId(event.channelId())`
2. If absent: no-op (not a Slack-backed channel)
3. If present:
   - `bindingCache.put(channelId, binding)`
   - `slackToChannel.put(binding.slackChannelId, new ChannelRef(channelId, event.channelName()))`
   - Load all `slack_thread_cache` rows for channelId into `threadCache` inner map
   - Deregister stale self-registration; `gateway.registerBackend(channelId, this, "human_participating")`

### post(ChannelRef channel, OutboundMessage message)

```
// Skip EVENT — fanOut() fires for the normaliser telemetry EVENT unconditionally
if (message.type() == MessageType.EVENT) return;
// Safety net for future null-content types
if (message.content() == null) return;

SlackBotBinding binding = bindingCache.get(channel.id());
if (binding == null) { LOG.debugf("No binding for channel %s — skipping", channel.name()); return; }

String token = resolveToken(binding.workspaceId);

// Thread lookup: memory first, fallback to DB (e.g. post-restart)
String threadTs = null;
if (message.correlationId() != null) {
    Map<UUID, String> channelCache = threadCache.get(channel.id());
    threadTs = channelCache != null ? channelCache.get(message.correlationId()) : null;
    if (threadTs == null) {
        threadTs = threadCacheStore.findThreadTs(channel.id(), message.correlationId()).orElse(null);
    }
}

PostResult result = slackBotClient.postMessage(token, binding.slackChannelId, message.content(), threadTs);
if (!result.ok()) { LOG.warnf("Slack post failed on channel %s: %s", channel.name(), result.error()); return; }

// Cache new thread root (first post for this correlationId)
if (message.correlationId() != null && threadTs == null && result.ts() != null) {
    threadCache.computeIfAbsent(channel.id(), k -> new ConcurrentHashMap<>())
               .put(message.correlationId(), result.ts());
    threadCacheStore.save(channel.id(), message.correlationId(), result.ts());
}

// Evict on task completion — DONE/FAILURE/DECLINE only.
// HANDOFF: don't evict — conversation continues with the delegated agent in the same thread.
// RESPONSE: don't evict — human may follow up in the same Slack thread.
if ((message.type() == DONE || message.type() == FAILURE || message.type() == DECLINE)
        && message.correlationId() != null) {
    Map<UUID, String> channelCache = threadCache.get(channel.id());
    if (channelCache != null) channelCache.remove(message.correlationId());
    threadCacheStore.delete(channel.id(), message.correlationId());
}
```

**Threading model:** `ChannelGateway.fanOut()` spawns `Thread.ofVirtual().start(() -> backend.post(...))`. `SlackBotClient.postMessage()` blocks on HTTP and calls `Thread.sleep()` on HTTP 429 retry — safe on a virtual thread (unmounts from carrier during blocking; no carrier-thread starvation). No `@Blocking` needed.

**Note on `MessageType.isTerminal()`:** returns true for HANDOFF, DONE, FAILURE — not DECLINE. Do not use `isTerminal()` for eviction. HANDOFF must not evict (conversation continues with delegated agent). Use explicit type enumeration: DONE, FAILURE, DECLINE.

### onInboundMessage(@ObservesAsync InboundMessage msg) → CompletionStage&lt;Void&gt;

**CorrelationId is generated here, not in the normaliser.** The cache write must happen BEFORE `receiveHumanMessage()` — otherwise a race exists where the agent's RESPONSE arrives before the thread cache entry is populated.

```
if (!InboundConnectorIds.SLACK_INBOUND.equals(msg.connectorId())) {
    return CompletableFuture.completedFuture(null);
}
ChannelRef channelRef = slackToChannel.get(msg.externalChannelRef());
if (channelRef == null) {
    LOG.debugf("No Slack-backed channel for Slack channel %s — discarding", msg.externalChannelRef());
    return CompletableFuture.completedFuture(null);
}

String slackThreadTs = msg.metadata().get("slack-thread-ts");
String slackTs       = msg.metadata().get("slack-ts");
String corrIdStr;

if (slackThreadTs != null && !slackThreadTs.equals(slackTs)) {
    // Thread reply — reverse-lookup existing corrId from DB
    corrIdStr = threadCacheStore.findCorrelationId(channelRef.id(), slackThreadTs)
                                .map(UUID::toString).orElse(null);
} else {
    corrIdStr = null;
}

if (corrIdStr == null) {
    // New top-level message (or reply to an unknown thread) — generate corrId and cache
    UUID corrId = UUID.randomUUID();
    corrIdStr = corrId.toString();
    if (slackTs != null) {
        // DB write before gateway call — prevents race with fast agent RESPONSE
        threadCacheStore.save(channelRef.id(), corrId, slackTs);
        threadCache.computeIfAbsent(channelRef.id(), k -> new ConcurrentHashMap<>())
                   .put(corrId, slackTs);
    }
}

gateway.receiveHumanMessage(channelRef,
    new InboundHumanMessage(msg.externalSenderId(), msg.content(), msg.receivedAt(),
                             msg.metadata(), corrIdStr, null));
return CompletableFuture.completedFuture(null);
```

### close(ChannelRef channel) — called only by ChannelGateway.closeChannel() on channel deletion

```
SlackBotBinding binding = bindingCache.remove(channel.id());
if (binding != null) slackToChannel.remove(binding.slackChannelId);
threadCache.remove(channel.id());
threadCacheStore.deleteAllByChannelId(channel.id());   // DB cleanup — channel is gone
bindingStore.deleteByChannelId(channel.id());           // DB cleanup
```

### normaliser() → slackInboundNormaliser (constructor-injected)

### actorType() → ActorType.HUMAN

---

## SlackInboundNormaliser

`@ApplicationScoped`, implements `InboundNormaliser`. Constructor-injected into `SlackChannelBackend`.

`SlackChannelBackend.onInboundMessage()` resolves the correlationId and passes it in `InboundHumanMessage.correlationId()` before calling the gateway. The normaliser's only responsibility is type inference — matching `DefaultInboundNormaliser`'s pattern of passing `raw.correlationId()` through.

```java
public NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw) {
    String slackThreadTs = raw.metadata().get("slack-thread-ts");
    String slackTs       = raw.metadata().get("slack-ts");

    // RESPONSE: known thread reply with a resolved corrId
    // QUERY: new message, or reply to an unknown thread (corrId will be null or for new conversation)
    MessageType type = (slackThreadTs != null && !slackThreadTs.equals(slackTs)
                        && raw.correlationId() != null)
            ? MessageType.RESPONSE
            : MessageType.QUERY;

    return new NormalisedMessage(
            type,
            raw.content(),
            "human:" + raw.externalSenderId(),
            raw.correlationId(),   // pass through — resolved by SlackChannelBackend before gateway call
            null,                  // inReplyTo — v1: agent supplies from check_messages ledger ID
            null,
            null);
}
```

Metadata keys populated by `SlackInboundConnector`: `"slack-ts"`, `"slack-thread-ts"`, `"workspace-id"`.

---

## SlackBindingResource

```java
@Path("/slack-channel/bindings")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class SlackBindingResource { ... }
```

**Auth:** No auth annotations — consistent with all other qhorus REST resources. Network isolation is the current security boundary. Auth deferred to platform#103 / auth-retrofit-readiness.md.

| Method | Path | Action |
|---|---|---|
| PUT | `/{channelId}` | Validates `workspaceId` resolves to a credential (HTTP 400 + key name if `NoSuchElementException`); creates/replaces binding; fires `ChannelInitialisedEvent` to trigger backend re-registration without restart |
| GET | `/{channelId}` | Returns `SlackBindingDto` — `workspaceId` returned, resolved token never included |
| DELETE | `/{channelId}` | Removes binding and evicts in-memory state |

**DELETE ordering** (correctness-critical — read binding from cache BEFORE DB delete to access slackChannelId):
```
1. SlackBotBinding binding = bindingCache.get(channelId)          // capture before DB delete
2. bindingStore.deleteByChannelId(channelId)                       // primary DB delete
3. gateway.deregisterBackend(channelId, BACKEND_ID)               // stops future post() calls
4. bindingCache.remove(channelId)                                  // clear forward cache
5. if (binding != null) slackToChannel.remove(binding.slackChannelId)  // clear reverse index
6. threadCache.remove(channelId)                                   // clear in-memory thread cache
   // DB thread cache rows preserved — TTL cleanup handles them; in-flight commitments may still resolve
```

`close()` is NOT called from `delete()` — `close()` has channel-deletion semantics (deletes DB thread cache rows). Binding removal (channel survives) must not delete thread cache rows; in-flight commitments should be allowed to complete.

---

## SlackBotBindingStore

```java
@ApplicationScoped
public class SlackBotBindingStore {
    Optional<SlackBotBinding> findByChannelId(UUID channelId);
    void save(SlackBotBinding binding);
    void deleteByChannelId(UUID channelId);
}
```

---

## SlackThreadCacheStore

```java
@ApplicationScoped
public class SlackThreadCacheStore {
    Optional<String> findThreadTs(UUID channelId, UUID correlationId);       // forward — used in post()
    Optional<UUID> findCorrelationId(UUID channelId, String threadTs);        // reverse — for inbound
    List<SlackThreadCache> findByChannelId(UUID channelId);                   // bulk load at channel init
    void save(UUID channelId, UUID correlationId, String threadTs);
    void delete(UUID channelId, UUID correlationId);                          // terminal eviction
    void deleteAllByChannelId(UUID channelId);                                // channel deletion cleanup
    int deleteOlderThan(Instant threshold);                                   // TTL cleanup
}
```

---

## SlackThreadCacheCleanupJob

`@ApplicationScoped`, `@Scheduled(every = "24h")`. Calls `threadCacheStore.deleteOlderThan(Instant.now().minus(30, DAYS))`. Also removes corresponding in-memory entries from `threadCache`. Handles abandoned entries from commitments that expired without a terminal message reaching the backend.

---

## ConnectorChannelBackend Fix

In `connector-backend/ConnectorChannelBackend.post()`, change `errorf` to `debugf` for the "No cache entry for channel" path. Channels using `SlackChannelBackend` don't register with `ConnectorChannelBackend`; this path is unreachable during normal operation for Slack-backed channels. `errorf` is misleading.

---

## Thread Semantics — Summary

**Outbound (`post()`):**
- EVENT → skip immediately (primary guard — type, not null)
- null content → skip (safety net)
- null correlationId → top-level Slack message, no cache write
- correlationId, cache miss → top-level message, cache `(channelId, corrId) → result.ts()` in memory + DB
- correlationId, cache hit → thread reply using cached `thread_ts`
- DONE/FAILURE/DECLINE → post in thread, then evict from memory + DB
- RESPONSE/HANDOFF/STATUS → post in thread, NO eviction (human may follow up; delegated agent continues)

**Inbound (`onInboundMessage()` → `normalise()`):**

| Slack signal | corrId in InboundHumanMessage | NormalisedMessage type |
|---|---|---|
| No `slack-thread-ts` | Generated UUID, cached | QUERY |
| `slack-thread-ts == slack-ts` (thread root by human) | Generated UUID, cached | QUERY |
| `slack-thread-ts != slack-ts`, cache hit | Found UUID from DB reverse lookup | RESPONSE |
| `slack-thread-ts != slack-ts`, cache miss | Generated UUID, cached | QUERY |

The `UNIQUE (channel_id, thread_ts)` constraint on `slack_thread_cache` enables O(1) inbound reverse lookup (threadTs → corrId) without a separate reverse map.

**Why corrId is generated in `onInboundMessage()`, not in the normaliser:**  
The normaliser is called inside `receiveHumanMessage()`. If corrId were generated there, the cache write would happen after the gateway dispatch. An agent that responds immediately (within the same transaction or on a fast event loop tick) could send RESPONSE before the cache entry exists, causing `post()` to create a new top-level Slack message instead of a thread reply. Writing the cache entry before the gateway call eliminates this race.

---

## Testing

**Unit (CDI-free, constructor injection):**
- `SlackChannelBackendTest` — mock `SlackBotClient`, `SlackBotBindingStore`, `SlackThreadCacheStore`, `ChannelGateway`, `Config`. Cover:
  - EVENT type → returns immediately, `postMessage` not called
  - null content → returns immediately, `postMessage` not called
  - null binding → debug log, `postMessage` not called
  - null correlationId → top-level post, no cache write
  - First post with corrId → top-level post, cache written (memory + DB)
  - Second post same corrId → thread reply with cached `thread_ts`
  - DONE post → thread reply, then cache evicted
  - FAILURE, DECLINE → same eviction behaviour
  - RESPONSE post → thread reply, NO eviction
  - HANDOFF post → thread reply, NO eviction (not evicted even though `isTerminal()` returns true for HANDOFF)
  - Failed Slack API call → WARN logged, no cache mutation
- `SlackInboundNormaliserTest` — mock none (pure logic). Cover all four inbound paths from the table above. Verify RESPONSE type when corrId is non-null + thread-ts mismatch.
- `SlackThreadCacheStoreTest` — CDI-free in-memory stub. Forward/reverse lookup, eviction, deleteAllByChannelId, TTL delete.

**Integration (`@QuarkusTest` + H2):**
- `SlackBotBindingStoreIT` — JPA CRUD.
- `SlackChannelBackendIT` — `@InjectMock SlackBotClient`. Persist binding → fire `ChannelInitialisedEvent` → `post()` → verify `slackBotClient.postMessage()` args. Second call same corrId → `thread_ts` passed. DONE → evicts. Importantly: `ChannelGateway.receiveHumanMessage()` dispatches twice per inbound message (content message + normaliser telemetry EVENT). Integration tests asserting `messageService` call count must use `times(2)`. Tests asserting `slackBotClient.postMessage()` call count must use `times(1)` — the telemetry EVENT is caught by the type guard and never reaches Slack.
- `SlackBindingResourceIT` — PUT validates credential (HTTP 400 on missing key), GET returns workspaceId not token, DELETE removes binding.
- `FlywayMigrationSchemaTest` — plain-Java Flyway + H2 (PostgreSQL mode). Runs V1–V24 + V2000. Asserts both tables with correct columns, PK, UQ, FK.

---

## Platform Coherence

- Credential pattern: Tier 1.5 (`workspace_id` in DB, resolved from MicroProfile Config as credential key). Protocol: `casehub/garden: docs/protocols/casehub/per-binding-credential-reference.md`. Tier 2 migration: platform#103.
- Flyway: V23–V24 in domain range (V1–V999), scoped to `db/qhorus/migration/` per `flyway-repo-scoped-migration-path.md`.
- SPI usage: `HumanParticipatingChannelBackend` + `InboundNormaliser` both in `casehub-qhorus-api` per `qhorus-consumer-integration-pattern.md`.
- Cross-repo dep: `casehub-connectors-slack-bot → casehub-qhorus slack-channel` in PLATFORM.md dependency table.
- Mutual exclusivity with `ConnectorChannelBackend`: enforced naturally — channels with `slack_bot_binding` have no `channel_connector_binding`, so `ConnectorChannelBackend` ignores them.
- `SlackBindingResource`: no auth annotations — consistent with all other qhorus REST resources.

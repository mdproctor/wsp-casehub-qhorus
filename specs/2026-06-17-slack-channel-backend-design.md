# casehub-qhorus-slack-channel — Design Spec

**Issue:** qhorus#261  
**Branch:** issue-261-slack-channel-backend  
**Date:** 2026-06-17  
**Revised:** 2026-06-17 (post code-review)

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
      SlackThreadCacheCleanupJob.java               — @Scheduled TTL eviction
    src/main/resources/db/qhorus/migration/
      V23__slack_channel.sql
```

**Dependencies (pom.xml):**
- `casehub-qhorus-api` — `HumanParticipatingChannelBackend`, `InboundNormaliser`, `OutboundMessage`
- `casehub-qhorus` (runtime) — `ChannelGateway`, `ChannelService`, JPA persistence, `@QuarkusPersistenceUnit("qhorus")`
- `casehub-connectors-slack-bot` 0.2-SNAPSHOT — `SlackBotClient`
- `casehub-connectors-core` — `InboundMessage`, `InboundConnectorIds` (for connectorId filter)
- Test: `casehub-qhorus-testing`, `casehub-platform` (MockCurrentPrincipal), `quarkus-jdbc-h2`, `quarkus-junit5`, `quarkus-junit5-mockito`

---

## Schema — V23__slack_channel.sql

```sql
CREATE TABLE slack_bot_binding (
    channel_id       UUID         NOT NULL,
    slack_channel_id VARCHAR(32)  NOT NULL,
    credential_ref   VARCHAR(128) NOT NULL,
    created_at       TIMESTAMP    NOT NULL,
    CONSTRAINT pk_slack_bot_binding      PRIMARY KEY (channel_id),
    CONSTRAINT fk_slack_binding_channel  FOREIGN KEY (channel_id) REFERENCES channel(id)
);

CREATE TABLE slack_thread_cache (
    channel_id     UUID        NOT NULL,
    correlation_id UUID        NOT NULL,
    thread_ts      VARCHAR(32) NOT NULL,
    created_at     TIMESTAMP   NOT NULL,
    CONSTRAINT pk_slack_thread_cache  PRIMARY KEY (channel_id, correlation_id),
    CONSTRAINT uq_slack_thread_ts     UNIQUE (channel_id, thread_ts)
);
```

Migration location: `slack-channel/src/main/resources/db/qhorus/migration/V23__slack_channel.sql`.  
Flyway merges this with runtime's V1–V22 and V2000 via classpath scanning of `db/qhorus/migration/`.  
No FK from `slack_thread_cache` to `channel` — avoids cascade delete of in-flight thread entries.

---

## Entities

### SlackBotBinding
```java
@Entity @Table(name = "slack_bot_binding")
public class SlackBotBinding {
    @Id
    public UUID channelId;
    public String slackChannelId;   // Slack channel ID, e.g. "C123ABC"
    public String credentialRef;    // logical name resolved from MicroProfile Config
    public Instant createdAt;
}
```

Credential resolution (Tier 1.5 per `casehub/garden: docs/protocols/casehub/per-binding-credential-reference.md`):
```java
private String resolveToken(String credentialRef) {
    return ConfigProvider.getConfig()
        .getValue("casehub.qhorus.slack-channel.credentials." + credentialRef, String.class);
}
```
Operator sets: `casehub.qhorus.slack-channel.credentials.acme-workspace=xoxb-...`  
Raw token never stored in DB or returned over REST.

### SlackThreadCache

```java
@Entity @Table(name = "slack_thread_cache")
public class SlackThreadCache {
    @EmbeddedId
    public SlackThreadCacheId id;   // (channelId, correlationId)
    public String threadTs;         // Slack root message ts, e.g. "1718567890.123456"
    public Instant createdAt;
}

@Embeddable
public record SlackThreadCacheId(UUID channelId, UUID correlationId) {}
```

**Thread cache is DB-backed with in-memory write-through cache.** Rationale: entries are keyed by `(channelId, correlationId)` — one per in-flight commitment, not per channel. A single channel can have many concurrent commitments. More critically, a server restart mid-commitment loses in-memory state and causes the next outbound message to create a new top-level Slack message rather than a thread reply, silently fragmenting the conversation. DB persistence preserves thread continuity across restarts. Overhead is minimal: one SELECT per channel on startup (bounded by Slack-backed channel count), one INSERT per commitment start (bounded by new commitment rate, not message rate), one DELETE on terminal state.

In-memory representation: `ConcurrentHashMap<UUID, String> threadTsCache` keyed by correlationId, per-channel bucket loaded at `onChannelInitialised()`.

---

## SlackChannelBackend

`@ApplicationScoped`, implements `HumanParticipatingChannelBackend`. BACKEND_ID = `"slack-bot"`. Uses constructor injection throughout, matching `ConnectorChannelBackend` convention.

**Internal state:**
```java
// Forward: channelId → SlackBotBinding (for post())
private final ConcurrentHashMap<UUID, SlackBotBinding> bindingCache = new ConcurrentHashMap<>();
// Reverse: slackChannelId → ChannelRef (for onInboundMessage())
private final ConcurrentHashMap<String, ChannelRef> slackToChannel = new ConcurrentHashMap<>();
// Thread cache: channelId → (correlationId → threadTs), loaded from DB at channel init
private final ConcurrentHashMap<UUID, ConcurrentHashMap<UUID, String>> threadCache = new ConcurrentHashMap<>();
```

**`onChannelInitialised(@Observes ChannelInitialisedEvent event)`:**
1. Look up binding via `bindingStore.findByChannelId(event.channelId())`
2. If absent: no-op (not a Slack-backed channel)
3. If present:
   - Populate `bindingCache.put(channelId, binding)`
   - Populate `slackToChannel.put(binding.slackChannelId, new ChannelRef(channelId, event.channelName()))`
   - Load `slack_thread_cache` rows for this channel into `threadCache.computeIfAbsent(channelId, ...)`
   - Deregister stale self-registration, re-register via `gateway.registerBackend(channelId, this, "human_participating")`

**`post(ChannelRef channel, OutboundMessage message)`:**

```
// Guard: skip telemetry — fanOut() does not filter before calling backends
if (message.type() == MessageType.EVENT) return;
if (message.content() == null) return;  // safety net for future null-content types

SlackBotBinding binding = bindingCache.get(channel.id());
if (binding == null) { LOG.debugf(...); return; }

String token = resolveToken(binding.credentialRef);
String threadTs = (message.correlationId() != null)
    ? threadCache.getOrDefault(channel.id(), emptyMap).get(message.correlationId())
    : null;

PostResult result = slackBotClient.postMessage(token, binding.slackChannelId,
                                               message.content(), threadTs);
if (!result.ok()) { LOG.warnf(...); return; }

if (message.correlationId() != null && threadTs == null && result.ts() != null) {
    // First post on this correlationId — cache thread root
    threadCache.computeIfAbsent(channel.id(), k -> new ConcurrentHashMap<>())
               .put(message.correlationId(), result.ts());
    threadCacheStore.save(channel.id(), message.correlationId(), result.ts());
}

if (isTerminal(message.type()) && message.correlationId() != null) {
    threadCache.getOrDefault(channel.id(), emptyMap).remove(message.correlationId());
    threadCacheStore.delete(channel.id(), message.correlationId());
}
```

**Threading model:** `ChannelGateway.fanOut()` spawns `Thread.ofVirtual().start(() -> backend.post(...))`. `SlackBotClient.postMessage()` blocks on HTTP including `Thread.sleep()` on HTTP 429 retry — safe on a virtual thread (virtual threads are unmounted from their carrier during blocking operations; no carrier-thread starvation). No `@Blocking` annotation needed.

**`close(ChannelRef channel)`:**
```
SlackBotBinding binding = bindingCache.remove(channel.id());
if (binding != null) {
    slackToChannel.remove(binding.slackChannelId);
}
threadCache.remove(channel.id());
threadCacheStore.deleteAllByChannelId(channel.id());
bindingStore.deleteByChannelId(channel.id());
```
Called by `ChannelGateway.closeChannel()` on channel deletion. Self-contained cleanup — no stale state in DB or memory after deletion.

**`onInboundMessage(@ObservesAsync InboundMessage msg)`** — returns `CompletionStage<Void>`:
```
if (!InboundConnectorIds.SLACK_INBOUND.equals(msg.connectorId())) {
    return CompletableFuture.completedFuture(null);
}
ChannelRef channel = slackToChannel.get(msg.externalChannelRef());
if (channel == null) {
    LOG.debugf("No Slack-backed channel for Slack channel %s — discarding", msg.externalChannelRef());
    return CompletableFuture.completedFuture(null);
}
gateway.receiveHumanMessage(channel, new InboundHumanMessage(
    msg.externalSenderId(), msg.content(), msg.receivedAt(), msg.metadata(), null, null));
return CompletableFuture.completedFuture(null);
```

**`normaliser()`:** returns `slackInboundNormaliser` (constructor-injected).

**`actorType()`:** returns `ActorType.HUMAN`.

---

## SlackInboundNormaliser

`@ApplicationScoped`, implements `InboundNormaliser`. Constructor-injected into `SlackChannelBackend`.

**`normalise(ChannelRef channel, InboundHumanMessage raw)`:**

| Condition | MessageType | correlationId |
|---|---|---|
| No `slack-thread-ts` in metadata | QUERY | null |
| `slack-thread-ts == slack-ts` (human created thread root, not a reply) | QUERY | null |
| `slack-thread-ts != slack-ts`, cache hit | RESPONSE | found correlationId |
| `slack-thread-ts != slack-ts`, cache miss | QUERY | null |

Reverse lookup uses `threadCacheStore.findCorrelationId(channel.id(), slackThreadTs)` — backed by `UNIQUE (channel_id, thread_ts)` on `slack_thread_cache`.

Metadata keys populated by `SlackInboundConnector`: `"slack-ts"`, `"slack-thread-ts"`, `"workspace-id"`.

---

## SlackBindingResource

`@Path("/slack-channel/bindings") @Produces(APPLICATION_JSON) @Consumes(APPLICATION_JSON) @ApplicationScoped`

**Auth:** No auth annotations. Consistent with all other qhorus REST resources — network isolation is the current security boundary. Auth deferred to platform#103 / auth-retrofit-readiness.md.

| Method | Path | Body | Effect |
|---|---|---|---|
| PUT | `/{channelId}` | `{ "slackChannelId": "C123ABC", "credentialRef": "acme-workspace" }` | Validates credential resolves (HTTP 400 with key name if missing); creates/replaces binding; fires `ChannelInitialisedEvent` to trigger backend re-registration without restart |
| GET | `/{channelId}` | — | Returns `{ "slackChannelId": "...", "credentialRef": "..." }` — resolved token never returned |
| DELETE | `/{channelId}` | — | Removes binding; `close()` called at next channel lifecycle event or server restart |

---

## SlackThreadCacheStore

```java
@ApplicationScoped
public class SlackThreadCacheStore {
    Optional<String> findThreadTs(UUID channelId, UUID correlationId);      // forward — used at startup load
    Optional<UUID> findCorrelationId(UUID channelId, String threadTs);       // reverse — for inbound normaliser
    List<SlackThreadCache> findByChannelId(UUID channelId);                  // bulk load at channel init
    void save(UUID channelId, UUID correlationId, String threadTs);
    void delete(UUID channelId, UUID correlationId);                         // terminal eviction
    void deleteAllByChannelId(UUID channelId);                               // channel deletion cleanup
    int deleteOlderThan(Instant threshold);                                  // TTL cleanup
}
```

---

## SlackThreadCacheCleanupJob

`@ApplicationScoped`, `@Scheduled(every = "24h")`. Calls `threadCacheStore.deleteOlderThan(Instant.now().minus(30, DAYS))`. Handles abandoned entries from commitments that expired without a terminal message reaching the backend. Also clears corresponding in-memory entries — iterates `threadCache` and removes entries whose `createdAt` is beyond the threshold.

---

## ConnectorChannelBackend Fix

In `connector-backend/ConnectorChannelBackend.post()`, change log level from `errorf` to `debugf` for the "No cache entry for channel" path. Channels using `SlackChannelBackend` don't register with `ConnectorChannelBackend` — this path is unreachable for them during normal operation. `errorf` is misleading; `debugf` is correct.

---

## Thread Semantics — Summary

**Outbound (`post()`):**
- EVENT → skip (telemetry, never user-facing)
- null correlationId → top-level Slack message, no cache
- correlationId, cache miss → top-level Slack message, cache `(channelId, correlationId) → result.ts()`
- correlationId, cache hit → thread reply using cached `thread_ts`
- DONE/FAILURE/DECLINE → post in thread, then evict cache entry (memory + DB)

**Inbound (`onInboundMessage()` → `normalise()`):**
- Top-level Slack message (no `slack-thread-ts`) → QUERY
- Thread root posted by human (`slack-thread-ts == slack-ts`) → QUERY
- Thread reply (`slack-thread-ts != slack-ts`), cache hit → RESPONSE with correlationId
- Thread reply, cache miss → QUERY (human replied to a thread we didn't create)

The `UNIQUE (channel_id, thread_ts)` constraint serves double duty: data integrity (one correlationId per Slack thread root per channel) and O(1) index scan for inbound reverse lookup.

---

## Testing

**Unit (CDI-free), constructor injection throughout:**
- `SlackChannelBackendTest` — mock `SlackBotClient`, `SlackBotBindingStore`, `SlackThreadCacheStore`, `ChannelGateway`, `Config`. Cover: EVENT skip, null content skip, first post caches, second post uses cached ts, DONE evicts, null correlationId no-cache, failed post no mutation, missing binding logs debug.
- `SlackInboundNormaliserTest` — mock `SlackThreadCacheStore`. All four normalise() paths including `slack-thread-ts == slack-ts` edge case.
- `SlackThreadCacheStoreTest` — CDI-free in-memory stub implementing the store interface. Forward lookup, reverse lookup, eviction, deleteAllByChannelId, TTL delete.

**Integration (`@QuarkusTest` + H2):**
- `SlackBotBindingStoreIT` — JPA CRUD.
- `SlackChannelBackendIT` — `@InjectMock SlackBotClient`. Persist binding → fire `ChannelInitialisedEvent` → post() → verify `slackBotClient.postMessage()` args. Second call same correlationId → thread_ts passed. DONE → evicts. Integration test asserting `messageService` call count from `receiveHumanMessage()` must use `times(2)` — gateway dispatches twice per inbound message (normalised content + normaliser telemetry EVENT).
- `SlackBindingResourceIT` — REST PUT validates credential (400 on missing), GET returns ref not token, DELETE removes.
- `FlywayMigrationSchemaTest` — plain-Java Flyway + H2 (PostgreSQL mode). Runs V1–V23 + V2000. Asserts both tables with correct columns, PK, UQ, FK.

---

## Platform Coherence

- Credential pattern: Tier 1.5 (`credential_ref` in DB, resolved from MicroProfile Config). Protocol: `casehub/garden: docs/protocols/casehub/per-binding-credential-reference.md`. Tier 2 migration: platform#103.
- Flyway: V23 in domain range (V1–V999), scoped to `db/qhorus/migration/` per `flyway-repo-scoped-migration-path.md`.
- SPI usage: `HumanParticipatingChannelBackend` + `InboundNormaliser` both in `casehub-qhorus-api` per `qhorus-consumer-integration-pattern.md`.
- Cross-repo dep: `casehub-connectors-slack-bot → casehub-qhorus slack-channel` added to PLATFORM.md dependency table.
- Mutual exclusivity with `ConnectorChannelBackend`: enforced naturally — channels with `slack_bot_binding` have no `channel_connector_binding`, so `ConnectorChannelBackend` doesn't register for them.
- `SlackBindingResource`: no auth annotations — consistent with all other qhorus REST resources.

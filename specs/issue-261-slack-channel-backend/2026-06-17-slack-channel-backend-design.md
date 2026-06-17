# casehub-qhorus-slack-channel — Design Spec

**Issue:** casehubio/qhorus#261  
**Date:** 2026-06-17  
**Revised:** 2026-06-18 (r4)  
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

### `SlackBotBinding` (JPA entity, `qhorus` PU)

```
slack_bot_binding
  channel_id        UUID         PK (Qhorus channel UUID — not generated)
  credential_ref    VARCHAR(128) NOT NULL   — logical config key name, e.g. "acme-workspace"
  slack_channel_id  VARCHAR(64)  NOT NULL   — Slack channel ID, e.g. "C123ABC"

  CONSTRAINT fk_slack_binding_channel FOREIGN KEY (channel_id) REFERENCES channel(id)
```

One row per Qhorus channel. `channel_id` is the PK — enforces one binding per channel structurally.

### `SlackThreadCache` (JPA entity, `qhorus` PU)

```
slack_thread_cache
  id              BIGINT       PK (generated)
  channel_id      UUID         NOT NULL
  correlation_id  VARCHAR(255) NOT NULL   — Qhorus correlationId as String (UUID.toString())
  thread_ts       VARCHAR(64)  NOT NULL   — Slack message ts, e.g. "1234567890.123456"
  created_at      TIMESTAMP    NOT NULL   — for TTL GC

  UNIQUE (channel_id, correlation_id)     — uq_slack_thread_corr
  INDEX  (channel_id, thread_ts)          — idx_slack_thread_cache_ts (reverse lookup)
```

Persists the correlationId ↔ thread_ts mapping. Survives restarts. Works in multi-node deployments. One row per in-flight commitment.

---

## Store SPIs

### `SlackBotBindingStore`

```java
Optional<SlackBotBinding> findByChannelId(UUID channelId);
void put(SlackBotBinding binding);
void delete(UUID channelId);
```

**`JpaSlackBotBindingStore`** — `@ApplicationScoped`, named `qhorus` PU, Panache entity ops.  
**`InMemorySlackBotBindingStore`** — `@Alternative @Priority(1) @ApplicationScoped` in `testing/`.

### `SlackThreadCacheStore`

```java
Optional<String> findThreadTs(UUID channelId, String correlationId);
Optional<String> findCorrelationId(UUID channelId, String threadTs);  // reverse — inbound routing
void put(UUID channelId, String correlationId, String threadTs);
void deleteAllForChannel(UUID channelId);  // called on binding deletion and channel close
int deleteOlderThan(Instant threshold);    // TTL cleanup
```

**`JpaSlackThreadCacheStore`** — `@ApplicationScoped`, named `qhorus` PU.  
**`InMemorySlackThreadCacheStore`** — `@Alternative @Priority(1) @ApplicationScoped` in `testing/`.

---

## `SlackChannelBackend`

`@ApplicationScoped`, `BACKEND_ID = "slack-bot"`, implements `HumanParticipatingChannelBackend`. Constructor injection throughout.

### In-memory indexes (populated at startup, maintained on binding changes)

```java
ConcurrentHashMap<UUID, CacheEntry>   channelCache  // channelId → (credentialRef, slackChannelId, channelName)
ConcurrentHashMap<String, ChannelRef> slackIndex    // slackChannelId → ChannelRef  (inbound routing)

private record CacheEntry(String credentialRef, String slackChannelId, String channelName) {}
```

### Registration — `@Observes ChannelInitialisedEvent` (sync)

1. `bindingStore.findByChannelId(channelId)` — if absent, skip silently
2. Populate both maps from binding + event data
3. `gateway.deregisterBackend(channelId, BACKEND_ID)` then `gateway.registerBackend(channelId, this, "human_participating")`

### Teardown — `close(ChannelRef)`

Called by `ChannelGateway.closeChannel()` on channel deletion. Total cleanup:
1. Remove from both in-memory maps
2. `threadCacheStore.deleteAllForChannel(channelId)` — DB rows deleted (channel is gone)
3. `bindingStore.delete(channelId)`

### `evict(UUID channelId)` — package-private

Removes from both in-memory maps immediately. Does NOT touch the DB. Called by `SlackBindingResource.delete()` before `gateway.deregisterBackend()` to atomically stop inbound routing.

### `normaliser()`

Returns the injected `SlackInboundNormaliser` bean (not null). The normaliser is needed for QUERY/RESPONSE type inference based on Slack thread metadata.

### Outbound — `post(ChannelRef, OutboundMessage)`

```
// 1. Guard: EVENT messages have null content and must not be posted to Slack.
//    fanOut() does not filter before calling backends.
if (message.type() == MessageType.EVENT) return;  // primary: semantic intent
if (message.content() == null) return;             // safety net: future null-content types

// 2. Look up binding
CacheEntry entry = channelCache.get(channel.id());
if (entry == null) {
    LOG.errorf("No binding cache entry for channel %s — cannot post", channel.name());
    return;
}

// 3. Resolve token at call time (never cached — supports rotation without restart)
String token = resolveToken(entry.credentialRef());

// 4. Determine threadTs — correlationId is null for COMMAND, QUERY, EVENT; non-null for
//    RESPONSE, DONE, FAILURE, DECLINE, STATUS (per OutboundMessage contract)
String threadTs = null;
if (message.correlationId() != null) {
    threadTs = threadCacheStore.findThreadTs(channel.id(), message.correlationId().toString())
                               .orElse(null);
}

// 5. Post to Slack
PostResult result = slackBotClient.postMessage(token, entry.slackChannelId(),
                                                message.content(), threadTs);
if (!result.ok()) {
    LOG.warnf("Slack post failed on channel %s: %s", channel.name(), result.error());
    meterRegistry.counter("slack_post_failures_total", "channel_id", channel.id().toString()).increment();
    return;
}

// 6. Anchor thread on first post for this correlationId
if (message.correlationId() != null && threadTs == null && result.ts() != null) {
    threadCacheStore.put(channel.id(), message.correlationId().toString(), result.ts());
}

// 7. Evict on terminal commitment types.
//    DONE, FAILURE, DECLINE → commitment closed; evict.
//    RESPONSE → do not evict (human may follow up in the same Slack thread).
//    HANDOFF → do not evict (delegated agent continues in the same thread).
//    MessageType.isTerminal() returns true for HANDOFF, DONE, FAILURE — not DECLINE.
//    Do not use isTerminal() here; use explicit enumeration.
if ((message.type() == DONE || message.type() == FAILURE || message.type() == DECLINE)
        && message.correlationId() != null) {
    threadCacheStore.deleteAllForChannel(channel.id()); // no — wrong scope
    // Correct: delete only this correlationId's entry, not all entries for the channel
    // threadCacheStore.delete(channel.id(), message.correlationId().toString())
}
```

**Note on `SlackThreadCacheStore.delete(UUID channelId, String correlationId)`:** the store interface above shows `deleteAllForChannel()` but also needs a per-entry `delete(UUID channelId, String correlationId)` for terminal eviction. Add this method.

**Threading model:** `ChannelGateway.fanOut()` spawns `Thread.ofVirtual().start(() -> backend.post(...))`. `SlackBotClient.postMessage()` blocks on HTTP and may `Thread.sleep()` on HTTP 429 retry — both are safe on a virtual thread. No `@Blocking` needed.

### Inbound — `@ObservesAsync InboundMessage` → `CompletionStage<Void>`

(`CompletionStage<Void>` return mirrors `ConnectorChannelBackend` — lets tests `.join()` before asserting)

```
// 1. Filter by connector
if (!InboundConnectorIds.SLACK_INBOUND.equals(msg.connectorId())) {
    return CompletableFuture.completedFuture(null);
}

// 2. Route to Qhorus channel
ChannelRef channelRef = slackIndex.get(msg.externalChannelRef());
if (channelRef == null) {
    LOG.debugf("No Slack binding for Slack channel %s — discarding", msg.externalChannelRef());
    meterRegistry.counter("slack_inbound_discarded_total").increment();
    return CompletableFuture.completedFuture(null);
}

String slackThreadTs = msg.metadata().get("slack-thread-ts");
String slackTs       = msg.metadata().get("slack-ts");
String corrId;

if (slackThreadTs != null && !slackThreadTs.equals(slackTs)) {
    // Thread reply — reverse-lookup the existing corrId that anchored this thread.
    // If not found (human replied to a thread we didn't create), treat as new conversation.
    corrId = threadCacheStore.findCorrelationId(channelRef.id(), slackThreadTs).orElse(null);
} else {
    corrId = null;
}

if (corrId == null) {
    // New top-level message (or reply to unknown thread):
    // Generate corrId and anchor the thread BEFORE calling the gateway.
    // Pre-gateway write eliminates the race where a fast agent RESPONSE arrives at post()
    // before the thread cache entry exists — without this, RESPONSE posts as top-level.
    UUID newCorrId = UUID.randomUUID();
    corrId = newCorrId.toString();
    if (slackTs != null) {
        threadCacheStore.put(channelRef.id(), corrId, slackTs);
    }
}

// 3. Forward to gateway. content may be null for media-only Slack messages (images, files,
//    voice). Null content is valid for QUERY — MessageDispatch.build() has no content
//    requirement for QUERY. NormalisedMessage.content is nullable.
gateway.receiveHumanMessage(channelRef,
    new InboundHumanMessage(msg.externalSenderId(), msg.content(), msg.receivedAt(),
                             msg.metadata(), corrId, null));
return CompletableFuture.completedFuture(null);
```

---

## `SlackInboundNormaliser`

`@ApplicationScoped`, implements `InboundNormaliser`. Returned by `SlackChannelBackend.normaliser()`.

The correlationId is resolved by `SlackChannelBackend.onInboundMessage()` before the gateway call and passed in `InboundHumanMessage.correlationId()`. The normaliser's only job is type inference and pass-through.

```java
@ApplicationScoped
public class SlackInboundNormaliser implements InboundNormaliser {

    @Override
    public NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw) {
        String slackThreadTs = raw.metadata().get("slack-thread-ts");
        String slackTs       = raw.metadata().get("slack-ts");

        // RESPONSE: reply to a Slack thread where we have a corrId — human is continuing
        //           a conversation the agent started (or vice versa).
        // QUERY:    new top-level message, or reply to an unknown thread.
        //           corrId is non-null for both (generated in onInboundMessage), but
        //           QUERY is the correct type for a new conversation.
        MessageType type = (slackThreadTs != null
                            && !slackThreadTs.equals(slackTs)
                            && raw.correlationId() != null)
                ? MessageType.RESPONSE
                : MessageType.QUERY;

        // content may be null for media-only Slack messages. QUERY and RESPONSE accept null
        // content — MessageDispatch.build() has no content requirement for these types.
        return new NormalisedMessage(
                type,
                raw.content(),
                "human:" + raw.externalSenderId(),
                raw.correlationId(),   // pass through — set by SlackChannelBackend before gateway
                null,                  // inReplyTo — agent supplies from ledger messageId
                null,
                null);
    }
}
```

**Slash-command detection deferred.** The obvious extension (`content.startsWith("/")` → COMMAND type) is a v2 concern. Adding it in v1 with no consumer of COMMAND from a human context adds complexity with no value. If added later, the null guard is required: `content != null && content.startsWith("/")`.

---

## `SlackBindingResource`

Path prefix: `/qhorus/slack/bindings`

```
PUT    /qhorus/slack/bindings/{channelId}   — create or replace
GET    /qhorus/slack/bindings/{channelId}   — read (never returns token)
DELETE /qhorus/slack/bindings/{channelId}   — remove + cleanup
```

**Request/response types:**

```java
/** PUT request body */
public record SlackBindingRequest(String credentialRef, String slackChannelId) {}

/** GET response body — token is never included */
public record SlackBindingView(UUID channelId, String credentialRef, String slackChannelId) {
    static SlackBindingView from(UUID channelId, SlackBotBinding b) {
        return new SlackBindingView(channelId, b.credentialRef, b.slackChannelId);
    }
}
```

**PUT flow:**
1. `channelService.findById(channelId)` — 404 if channel not found
2. `channelBindingStore.findByChannelId(channelId)` — 409 Conflict if generic `ChannelConnectorBinding` exists (mutual exclusion: a channel is Slack bot OR generic connector, not both)
3. Validate credential: `Config.getValue("casehub.qhorus.slack-channel.credentials." + credentialRef)` — 400 "credentialRef not configured: casehub.qhorus.slack-channel.credentials.<ref>" if key is absent (fail-fast; `NoSuchElementException`)
4. `bindingStore.put(...)` — persists or replaces
5. `gateway.initChannel(channelId, new ChannelRef(channelId, channel.name))` — fires `ChannelInitialisedEvent`; backend self-registers with new binding
6. 200 `SlackBindingView`

**DELETE flow** (ordering matters — read cache before DB delete):
1. 404 if no binding in `channelCache` or DB
2. `backend.evict(channelId)` — removes both in-memory maps **before** DB delete; stops inbound routing atomically
3. `gateway.deregisterBackend(channelId, BACKEND_ID)` — removes from fanOut routing
4. `threadCacheStore.deleteAllForChannel(channelId)` — purges thread history; correct because after evict() the binding's slackChannelId is gone from the index and post() returns early on missing CacheEntry anyway
5. `bindingStore.delete(channelId)` — DB delete last
6. 204

`close()` is NOT called from `delete()`. `close()` has channel-deletion semantics (called by `ChannelGateway.closeChannel()`). Binding removal (`delete()`) is an admin operation; the channel survives.

---

## Flyway

**Migration directory:** `slack-channel/src/main/resources/db/slack-channel/migration/`  
Separate scoped path per Rule 4 of `flyway-version-range-allocation.md`. Version numbers are still global across all configured Flyway locations for the same datasource — V1 would collide with `V1__initial_schema.sql` in `db/qhorus/migration/`. Use V23/V24.

```
V23__slack_bot_binding.sql    — slack_bot_binding table
V24__slack_thread_cache.sql   — slack_thread_cache + uq_slack_thread_corr + idx_slack_thread_cache_ts
```

**Required config change — `runtime/src/main/resources/application.properties`:**

```properties
# Extend to include slack-channel migrations (only active when jar is on classpath)
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/slack-channel/migration
```

**Required config change — `runtime/src/main/resources/application.properties`:**

```properties
# Expand from io.casehub.qhorus.runtime to cover io.casehub.qhorus.slack.channel
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus,io.casehub.ledger.runtime
```

The package expansion is safe when `slack-channel` is absent: the Jandex index for its entities is missing, so Hibernate finds nothing in that package and nothing breaks. When present, entities are discovered automatically.

**Module test `application.properties`:** must include both config changes and use `drop-and-create` (no Flyway in test cycle — H2 schema from drop-and-create).

---

## `SlackThreadCacheCleanupJob`

`@ApplicationScoped`, `@Scheduled(every = "24h")`. Calls `threadCacheStore.deleteOlderThan(Instant.now().minus(30, DAYS))`. Handles abandoned entries from commitments that expired without a terminal message reaching the backend (server crash mid-commitment, network partition during DONE delivery).

---

## Ancillary change — `ConnectorChannelBackend` WARN→DEBUG

In `connector-backend/ConnectorChannelBackend.java`, the `onInboundMessage` log when no channel is found for an inbound message currently logs at WARN. With `slack-channel` on the classpath, every Slack inbound event before a binding is configured produces a spurious WARN. Change to DEBUG. Single-line fix in a separate commit referencing #261.

---

## Testing strategy

**Unit tests (CDI-free):**  
`SlackChannelBackendTest` — `InMemorySlackBotBindingStore`, `InMemorySlackThreadCacheStore`, `@InjectMock SlackBotClient`. Covers:
- `post()`: EVENT guard (returns before postMessage), null-content guard, missing CacheEntry (ERROR log), first post with corrId (top-level → cache anchored), second post same corrId (thread reply), DONE/FAILURE/DECLINE (evicts), RESPONSE (no eviction), HANDOFF (no eviction — `isTerminal()` is true for HANDOFF but must not evict), Slack API failure (WARN, no cache mutation)
- `onInboundMessage()`: non-Slack connector (filtered), unknown slackChannelId (DEBUG + counter), new top-level message (corrId generated, cache written before gateway, passed through), thread reply with cache hit (corrId found, RESPONSE type), thread reply with cache miss (new corrId, QUERY type), null content (passes through, no NPE)

`SlackInboundNormaliserTest` — pure logic, no mocks. Cover: RESPONSE path (thread-ts mismatch + non-null corrId), QUERY path (no thread-ts), QUERY path (thread-ts == slack-ts), null content (returns NormalisedMessage with null content, no NPE).

**Integration tests (`@QuarkusTest`, H2):**  
`SlackChannelBackendIT` — `@InjectMock SlackBotClient`. Full lifecycle: bind → register → inbound new message → corrId generated and cached → agent RESPONSE → thread reply verified. `ChannelGateway.receiveHumanMessage()` dispatches twice per inbound message (content + normaliser telemetry EVENT). Integration tests asserting `messageService` call count use `times(2)`. Tests asserting `slackBotClient.postMessage()` call count use `times(1)` — the EVENT dispatch returns immediately at the type guard.

`SlackBindingResourceIT` — REST PUT (200), GET (200, no token), PUT with unknown credentialRef (400), DELETE (204), double-DELETE (404), PUT on channel with existing generic binding (409).

`FlywayMigrationSchemaTest` — plain-Java Flyway + H2. Runs V1–V24 + V2000. Asserts both tables exist with correct columns, constraints, and indexes.

In-memory stores activate via `@Alternative @Priority(1)` — no `@TestProfile` needed.

---

## Known limitations

**Reverse mutual exclusion not enforced.** `SlackBindingResource` rejects a Slack binding if a `ChannelConnectorBinding` exists (409). The inverse — `connector-backend` blocking creation of a generic binding when a `SlackBotBinding` exists — is NOT enforced (connector-backend must not depend on slack-channel). Tracked as follow-up.

**Slash-command detection deferred to v2.** All inbound Slack messages are QUERY or RESPONSE. Slash commands (`/command`) are treated as QUERY. Adding COMMAND type inference requires a null guard on content and a consumer for COMMAND from a human actor.

---

## Store interface addition

`SlackThreadCacheStore` needs a per-entry delete for terminal eviction in `post()`:

```java
void delete(UUID channelId, String correlationId);   // add — terminal commitment eviction
```

`deleteAllForChannel()` remains for binding deletion and channel close. `delete()` is for per-commitment cleanup in `post()`.

---

## Key protocols applied

| Protocol | Application |
|----------|-------------|
| `per-binding-credential-reference` (PP-20260617) | `credentialRef` stored in DB; token from MP Config at call time |
| `module-tier-structure` | Store SPI interfaces + JPA impls + in-memory in `testing/` |
| `flyway-version-range-allocation` Rule 4 + global V numbering | Scoped `db/slack-channel/migration/` path; V23/V24 to avoid collision with `db/qhorus/migration/V1` |
| `maven-submodule-folder-naming` | Folder `slack-channel/`, artifactId `casehub-qhorus-slack-channel` |
| Mutual exclusion design | REST enforces: generic `ChannelConnectorBinding` → 409 on Slack bind attempt |
| Thread anchoring ordering | corrId generated and cached BEFORE `receiveHumanMessage()` — eliminates race with fast agent RESPONSE |
| `MessageType.isTerminal()` scope | Returns true for HANDOFF, DONE, FAILURE (not DECLINE). Never use for eviction; use explicit DONE/FAILURE/DECLINE enumeration. |

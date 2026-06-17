# casehub-qhorus-slack-channel — Design Spec

**Issue:** qhorus#261  
**Branch:** issue-261-slack-channel-backend  
**Date:** 2026-06-17

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
- `casehub-connectors-core` — `InboundMessage` (for normaliser metadata keys)
- Test: `casehub-qhorus-testing`, `casehub-platform` (MockCurrentPrincipal), `quarkus-jdbc-h2`, `quarkus-junit5`

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
// SlackChannelBackend
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
    public String threadTs;         // Slack root message timestamp, e.g. "1718567890.123456"
    public Instant createdAt;
}

@Embeddable
public record SlackThreadCacheId(UUID channelId, UUID correlationId) {}
```

---

## SlackChannelBackend

`@ApplicationScoped`, implements `HumanParticipatingChannelBackend`. BACKEND_ID = `"slack-bot"`.

**Registration:** observes `@Observes ChannelInitialisedEvent`. If `SlackBotBindingStore.findByChannelId(channelId)` returns a binding, deregisters any stale self-registration then re-registers via `gateway.registerBackend()`. Channels without a `slack_bot_binding` are ignored — `ConnectorChannelBackend` may still register for them if they have a `channel_connector_binding`.

**`post(ChannelRef, OutboundMessage)`:**

1. Load binding → resolve token from Config
2. Look up `threadCacheStore.findThreadTs(channelId, correlationId)` if `correlationId != null`
3. Call `slackBotClient.postMessage(token, slackChannelId, content, threadTs)`
4. On failure (`!result.ok()`): log WARN, return. No cache mutation.
5. On success, cache miss, non-null correlationId: save `(channelId, correlationId) → result.ts()`
6. On success, terminal type (DONE / FAILURE / DECLINE): evict `(channelId, correlationId)` after posting

**`normaliser()`:** returns `slackInboundNormaliser` (injected CDI bean).

**`actorType()`:** returns `ActorType.HUMAN`.

---

## SlackInboundNormaliser

`@ApplicationScoped`, implements `InboundNormaliser`. Injected into `SlackChannelBackend.normaliser()`.

**`normalise(ChannelRef, InboundHumanMessage)`:**

| Condition | MessageType | correlationId |
|---|---|---|
| No `slack-thread-ts` in metadata | QUERY | null |
| `slack-thread-ts == slack-ts` (human created thread root) | QUERY | null |
| `slack-thread-ts != slack-ts`, cache hit | RESPONSE | found correlationId |
| `slack-thread-ts != slack-ts`, cache miss | QUERY | null |

Metadata keys populated by `SlackInboundConnector`: `"slack-ts"`, `"slack-thread-ts"`, `"workspace-id"`.

---

## SlackBindingResource

`@Path("/slack-channel/bindings") @ApplicationScoped`

| Method | Path | Body | Effect |
|---|---|---|---|
| PUT | `/{channelId}` | `{ "slackChannelId": "C123ABC", "credentialRef": "acme-workspace" }` | Create/replace binding; fires `ChannelInitialisedEvent` to trigger backend re-registration |
| GET | `/{channelId}` | — | Returns binding (omits resolved token; returns `credentialRef` name only) |
| DELETE | `/{channelId}` | — | Removes binding; backend deregisters on next `ChannelInitialisedEvent` or restart |

PUT validates that the `credentialRef` resolves (config key exists) before persisting — fast feedback if the operator forgot to set the env var. If `Config.getValue(...)` throws `NoSuchElementException`, return HTTP 400 with a message identifying the missing key.

---

## SlackThreadCacheCleanupJob

`@ApplicationScoped` `@Scheduled` job, runs daily. Calls `threadCacheStore.deleteOlderThan(Instant.now().minus(30, DAYS))`. Handles abandoned entries from commitments that expired without a terminal message reaching the backend (server crash, network partition during DONE delivery).

---

## ConnectorChannelBackend Fix

In `connector-backend/ConnectorChannelBackend.post()`, change log level from `errorf` to `debugf` for the "No cache entry for channel" path. When `SlackChannelBackend` is registered for a channel, `ConnectorChannelBackend` is not — but startup sequencing can occasionally fire this path before the cache is warm. ERROR is misleading; DEBUG is correct.

---

## Thread Semantics — Summary

**Outbound (post):** `correlationId` is the thread key. First post on a correlationId → top-level Slack message (no `thread_ts`), cache returned `ts`. Subsequent posts → reply using cached `ts`. DONE/FAILURE/DECLINE → post in thread, then evict cache entry.

**Inbound (normalise):** `slack-thread-ts` in metadata indicates a thread reply. Reverse-lookup `(channelId, slack-thread-ts) → correlationId` via the `UQ` index on `slack_thread_cache`. Hit → RESPONSE. Miss or absent → QUERY.

The `UNIQUE (channel_id, thread_ts)` constraint on `slack_thread_cache` serves double duty: data integrity guarantee (one correlationId per Slack thread root, one thread root per correlationId per channel) and O(1) index scan for inbound reverse lookup.

---

## Testing

**Unit (CDI-free):**
- `SlackChannelBackendTest` — mock `SlackBotClient`, stores, `Config`. All post() paths: first post (caches), second post (uses cache), terminal type (evicts), null correlationId (no cache), failed post (no cache mutation).
- `SlackInboundNormaliserTest` — mock `SlackThreadCacheStore`. All four normalise() paths.
- `SlackThreadCacheStoreTest` — CDI-free via in-memory stub. Forward/reverse lookup, eviction, TTL delete.

**Integration (@QuarkusTest + H2):**
- `SlackBotBindingStoreIT` — JPA CRUD.
- `SlackChannelBackendIT` — `@InjectMock SlackBotClient`. Persist binding → fire event → post() → verify `slackBotClient.postMessage()` args. Second call same correlationId → thread_ts passed.
- `SlackBindingResourceIT` — REST PUT/GET/DELETE lifecycle.
- `FlywayMigrationSchemaTest` — plain-Java Flyway + H2 (PostgreSQL mode). Runs V1–V23 + V2000. Asserts `slack_bot_binding` and `slack_thread_cache` tables with correct columns and constraints.

---

## Platform Coherence

- Credential pattern: Tier 1.5 (`credential_ref` in DB, resolved from MicroProfile Config). See `casehub/garden: docs/protocols/casehub/per-binding-credential-reference.md`. Tier 2 migration tracked in platform#103.
- Flyway: V23 in domain range (V1–V999), scoped to `db/qhorus/migration/` per `flyway-repo-scoped-migration-path.md`.
- SPI usage: `HumanParticipatingChannelBackend` + `InboundNormaliser` both in `casehub-qhorus-api` — correct integration points per `qhorus-consumer-integration-pattern.md`.
- Cross-repo dep: `casehub-connectors-slack-bot → casehub-qhorus slack-channel` added to PLATFORM.md dependency table.
- Mutual exclusivity with `ConnectorChannelBackend`: enforced by binding lookup at `ChannelInitialisedEvent` — only one backend registers per channel.

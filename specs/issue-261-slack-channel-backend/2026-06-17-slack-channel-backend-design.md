# casehub-qhorus-slack-channel — Design Spec

**Issue:** casehubio/qhorus#261  
**Date:** 2026-06-17  
**Status:** Approved

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
- `casehub-qhorus-api` — `HumanParticipatingChannelBackend`, `ChannelRef`, `InboundHumanMessage`, `OutboundMessage`, `ChannelInitialisedEvent`
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
  channel_id        UUID  PK (Qhorus channel UUID — not generated)
  credential_ref    VARCHAR(128)  NOT NULL   — logical config key name, e.g. "acme-workspace"
  slack_channel_id  VARCHAR(64)   NOT NULL   — Slack channel ID, e.g. "C123ABC"
```

One row per Qhorus channel. `channel_id` is the PK — enforces one binding per channel structurally.

### `SlackThreadCache` (JPA entity, `qhorus` PU)

```
slack_thread_cache
  id              BIGINT       PK (generated)
  channel_id      UUID         NOT NULL
  correlation_id  VARCHAR(255) NOT NULL   — Qhorus correlationId (String UUID)
  thread_ts       VARCHAR(64)  NOT NULL   — Slack ts, e.g. "1234567890.123456"
  created_at      TIMESTAMP    NOT NULL   — for future TTL GC; unused now

  UNIQUE (channel_id, correlation_id)     — uq_slack_thread_corr
  INDEX  (channel_id, thread_ts)          — idx_slack_thread_cache_ts (reverse lookup)
```

Persists the correlationId ↔ thread_ts mapping. Survives restarts. Works in multi-node deployments.

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
void deleteAllForChannel(UUID channelId);  // called on binding deletion
```

**`JpaSlackThreadCacheStore`** — `@ApplicationScoped`, named `qhorus` PU.  
**`InMemorySlackThreadCacheStore`** — `@Alternative @Priority(1) @ApplicationScoped` in `testing/`.

---

## `SlackChannelBackend`

`@ApplicationScoped`, `BACKEND_ID = "slack-bot"`, implements `HumanParticipatingChannelBackend`.

### In-memory indexes (populated at startup, maintained on binding changes)

```java
ConcurrentHashMap<UUID, CacheEntry>   channelCache  // channelId → (credentialRef, slackChannelId, channelName)
ConcurrentHashMap<String, ChannelRef> slackIndex    // slackChannelId → ChannelRef  (inbound routing)
private record CacheEntry(String credentialRef, String slackChannelId, String channelName) {}
```

### Registration — `@Observes ChannelInitialisedEvent` (sync)

- `bindingStore.findByChannelId(channelId)` — if absent, skip silently
- Populate both maps from binding + event data
- `gateway.deregisterBackend(channelId, BACKEND_ID)` then `gateway.registerBackend(channelId, this, "human_participating")`

### Teardown — `close(ChannelRef)`

Removes from both maps.

### `evict(UUID channelId)` — package-private

Removes from both maps immediately. Called by `SlackBindingResource` on DELETE before `gateway.deregisterBackend()` to ensure inbound routing stops atomically.

### Outbound — `post(ChannelRef, OutboundMessage)`

1. Look up `CacheEntry` — if absent, log ERROR and return
2. Resolve token at call time (never cached — supports token rotation without restart):
   `Config.getValue("casehub.qhorus.slack-channel.credentials." + credentialRef)`
3. Determine `threadTs`:
   - `correlationId != null` → `threadCacheStore.findThreadTs(channelId, correlationId.toString())`
   - No match or null correlationId → top-level post (`threadTs = null`)
4. `slackBotClient.postMessage(token, slackChannelId, content, threadTs)`
5. If `ok` AND `correlationId != null` AND no prior ts existed → `threadCacheStore.put(channelId, correlationId.toString(), result.ts())` — anchors the thread for all subsequent replies
6. If `!ok` → WARN + increment `slack_post_failures_total{channel_id}` counter

### Inbound — `@ObservesAsync InboundMessage` → `CompletionStage<Void>`

(`CompletionStage<Void>` return mirrors `ConnectorChannelBackend` — lets tests `.join()` before asserting)

1. Filter: `!InboundConnectorIds.SLACK_INBOUND.equals(msg.connectorId())` → return immediately
2. `slackIndex.get(msg.externalChannelRef())` — if absent, DEBUG + `slack_inbound_discarded_total` counter + return
3. Reverse thread lookup: `msg.metadata().get("slack-thread-ts")` → `threadCacheStore.findCorrelationId(channelId, threadTs)` → `correlationId` (null if no match)
4. Build `InboundHumanMessage(externalSenderId, content, receivedAt, metadata, correlationId, null)`
5. `gateway.receiveHumanMessage(channelRef, msg)`

---

## `SlackBindingResource`

Path prefix: `/qhorus/slack/bindings`

```
PUT    /qhorus/slack/bindings/{channelId}   — create or replace
GET    /qhorus/slack/bindings/{channelId}   — read (never returns token)
DELETE /qhorus/slack/bindings/{channelId}   — remove + cleanup
```

**DTOs:**
```java
record SlackBindingRequest(String credentialRef, String slackChannelId) {}
record SlackBindingView(UUID channelId, String credentialRef, String slackChannelId)
```

**PUT flow:**
1. `channelService.findById(channelId)` — 404 if channel not found
2. `channelBindingStore.findByChannelId(channelId)` — 409 Conflict if generic `ChannelConnectorBinding` exists (mutual exclusion: a channel is Slack bot OR generic connector, not both)
3. Validate credential: `Config.getValue("casehub.qhorus.slack-channel.credentials." + credentialRef)` — 400 "credentialRef not configured" if key is absent (fail-fast)
4. `bindingStore.put(...)` — persists or replaces
5. `gateway.initChannel(channelId, new ChannelRef(...))` — fires `ChannelInitialisedEvent` so backend self-registers with new binding in place
6. 200 `SlackBindingView`

**DELETE flow:**
1. 404 if no binding
2. `threadCacheStore.deleteAllForChannel(channelId)` — purge thread history
3. `backend.evict(channelId)` — removes both in-memory maps immediately
4. `gateway.deregisterBackend(channelId, "slack-bot")` — removes from fanOut routing
5. `bindingStore.delete(channelId)`
6. 204

---

## Flyway

Scoped migration directory: `src/main/resources/db/slack-channel/migration/`  
Version numbers are module-local (start at V1, no global reservation needed — Rule 4 of flyway-version-range-allocation).

```
V1__slack_bot_binding.sql    — slack_bot_binding table
V2__slack_thread_cache.sql   — slack_thread_cache + uq_slack_thread_corr + idx_slack_thread_cache_ts
```

Consumers must extend their Flyway locations config:
```properties
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/slack-channel/migration
```

Module tests use `drop-and-create` on H2 — no Flyway needed in the test cycle.

---

## Testing strategy

**Unit tests (CDI-free):** `SlackChannelBackendTest` — `InMemorySlackBotBindingStore`, `InMemorySlackThreadCacheStore`, hand-rolled mock `SlackBotClient` via direct field assignment. Covers: registration, outbound with/without correlationId thread anchoring, inbound routing, inbound reverse thread lookup, missing credentialRef (token resolution failure), `evict()` cleanup.

**Integration tests (`@QuarkusTest`, H2, `@InjectMock ChannelGateway`):** `SlackBindingResourceTest` — tests REST PUT/GET/DELETE lifecycle, 404/409/400 error paths. Gateway is mocked to decouple from full ledger dependency.

In-memory stores activate in all test profiles via `@Alternative @Priority(1)` — no `@TestProfile` needed.

---

## Ancillary change — `ConnectorChannelBackend` WARN→DEBUG

In `connector-backend/ConnectorChannelBackend.java`, the `onInboundMessage` log when no channel is found for an inbound message currently logs at WARN. With `slack-channel` on the classpath, every Slack inbound event before a binding is configured produces a spurious WARN. Change to DEBUG. Single-line fix in a separate commit referencing #261.

---

## PU package registration

`runtime/src/main/resources/application.properties` currently declares:
```
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime
```

`SlackBotBinding` and `SlackThreadCache` are in `io.casehub.qhorus.slack.channel` — outside the scanned packages, so they'd be assigned to the default PU (wrong). Fix: expand the base package from `io.casehub.qhorus.runtime` to `io.casehub.qhorus`:

```
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus,io.casehub.ledger.runtime
```

This is safe: if `slack-channel` is not on the classpath, its Jandex index is absent — no entities are found, nothing breaks. If it is on the classpath, its entities are picked up automatically. Change applies to `runtime/src/main/resources/application.properties`.

The `slack-channel` module's test `application.properties` must also use the expanded base package:
```
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus,io.casehub.ledger.runtime
```

---

## Known limitations

**Reverse mutual exclusion not enforced.** `SlackBindingResource` rejects a Slack binding if a `ChannelConnectorBinding` exists (409). The inverse — `connector-backend` blocking creation of a generic binding when a `SlackBotBinding` exists — is NOT enforced, because `connector-backend` must not depend on `slack-channel`. An operator can create a `ChannelConnectorBinding` after a `SlackBotBinding`, resulting in both backends registering and duplicate delivery. Tracked as a follow-up issue; operators should be aware.

**Thread cache has no TTL GC.** `created_at` on `slack_thread_cache` is present but unused. Long-running channels accumulate rows indefinitely. TTL-based cleanup is a follow-up concern.

---

## Key protocols applied

| Protocol | Application |
|----------|-------------|
| `per-binding-credential-reference` (PP-20260617) | `credentialRef` stored in DB; token from MP Config at call time |
| `module-tier-structure` | Store SPI interfaces + JPA impls + in-memory in `testing/` |
| `flyway-version-range-allocation` Rule 4 | Scoped `db/slack-channel/migration/` path, V1 local |
| `maven-submodule-folder-naming` | Folder `slack-channel/`, artifactId `casehub-qhorus-slack-channel` |
| GE-20260517-f28d15 | `InboundNormaliser` SPI NOT implemented; Slack inbound handled internally in backend |
| Mutual exclusion design | REST enforces: generic `ChannelConnectorBinding` → 409 on Slack bind attempt |

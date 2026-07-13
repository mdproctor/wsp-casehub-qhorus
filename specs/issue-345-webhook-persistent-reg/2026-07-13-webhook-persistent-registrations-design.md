# Webhook Observer: Persistent Registrations (JPA)

**Issue:** #345
**Date:** 2026-07-13
**Status:** Approved

## Problem

The webhook observer module (from #163) uses an in-memory `ConcurrentHashMap` for webhook registrations. Registrations are lost on server restart. Production deployments need durable webhook subscriptions.

## Design

### Credential Security

The existing `WebhookRegistration.secret` field stores a plaintext HMAC signing key. This is replaced with `secretRef` — a reference resolved via `CredentialResolver` at POST time. The actual secret never appears in the database or in-memory data structures. This follows the same pattern as `SlackChannelBackend.resolveToken()`.

The secret is extracted via `creds.get(CredentialPropertyKeys.SIGNING_SECRET)`. A new constant `SIGNING_SECRET = "signing-secret"` is added to `CredentialPropertyKeys` in `casehub-platform-api`.

If `secretRef` is set but resolution fails (missing credential or missing `signing-secret` key), the POST is **skipped entirely** and logged at ERROR. The presence of `secretRef` declares that signing is mandatory for this registration — silent degradation to unsigned delivery is not acceptable.

Breaking change to the REST API: `RegisterRequest.secret` becomes `RegisterRequest.secretRef`. No migration needed — the module shipped in #163 with explicit "in-memory only, lost on restart" documentation. No durable consumers exist.

### JPA Entity

`WebhookRegistrationEntity` in `io.casehub.qhorus.webhook`:

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUID | PK |
| `channelId` | UUID | Nullable (null = global webhook), FK → channel |
| `url` | String | NOT NULL, max 2048 |
| `secretRef` | String | Nullable (no signature if omitted) |
| `headers` | String | Nullable, JSON-serialized Map via `MapToJsonConverter` |
| `tenancyId` | String | NOT NULL, default DEFAULT_TENANT_ID |
| `createdAt` | Instant | NOT NULL |

UNIQUE constraint: `(url, channel_id, tenancy_id)` — prevents duplicate registrations for channel-specific webhooks. A partial unique index `(url, tenancy_id) WHERE channel_id IS NULL` covers global webhooks (SQL NULL ≠ NULL in standard UNIQUE constraints).

#### MapToJsonConverter

`MapToJsonConverter implements AttributeConverter<Map<String, String>, String>` in `io.casehub.qhorus.webhook`. Follows the same pattern as `ArtefactRefListConverter`. Null or empty maps convert to `null` in the database column. Non-empty maps serialize to JSON via Jackson `ObjectMapper`.

### Store

`WebhookRegistrationStore` — `@ApplicationScoped`, injects `@PersistenceUnit("qhorus") EntityManager`:

- `findById(UUID)` → `Optional<WebhookRegistrationEntity>`
- `findByChannelId(UUID)` → `List<WebhookRegistrationEntity>` (channel-specific only)
- `findGlobal()` → `List<WebhookRegistrationEntity>` (channelId is null)
- `findAll()` → `List<WebhookRegistrationEntity>`
- `save(WebhookRegistrationEntity)` — `em.merge()`
- `delete(UUID)` → `boolean`
- `deleteByChannelId(UUID)` — channel deletion cleanup

Injects `CurrentPrincipal`. All runtime queries (`findById`, `findByChannelId`, `findGlobal`, `save`, `delete`) filter by `CurrentPrincipal.tenancyId()` via JPQL WHERE clause. Two methods are deliberately cross-tenant:
- `findAll()` — used only by startup reload, which runs outside a request context
- `deleteByChannelId(UUID)` — channel deletion is authoritative; all registrations for a channel share the same tenant via the channel's tenancy, so ambient `CurrentPrincipal` filtering is unnecessary and may fail outside request context

No `InMemoryWebhookRegistrationStore` — optional module with its own store. Tests use H2.

### Registry Changes

`WebhookRegistry` becomes the coordination layer between the in-memory cache and the JPA store:

In-memory maps:
- `channelHooks`: `Map<UUID, Set<WebhookRegistration>>` — keyed by channelId. Channel UUIDs are globally unique and tenant-scoped, so this is inherently tenant-correct.
- `globalHooks`: `Map<String, Set<WebhookRegistration>>` — keyed by **tenancyId**. Prevents cross-tenant leakage of global webhooks.
- `byId`: `Map<UUID, WebhookRegistration>` — keyed by registration ID (globally unique).

Operations:
- **Startup reload:** `@Observes StartupEvent` calls `store.findAll()` (cross-tenant) and loads all registrations into the in-memory maps, keying global hooks by `tenancyId`.
- **`register()`** — NOT `@Transactional`. Calls `store.save()` (which is `@Transactional` and commits on return), then populates in-memory maps. If `save()` throws, in-memory maps are not updated — consistency preserved.
- **`deregister()`** — NOT `@Transactional`. Calls `store.delete()` (commits on return), then removes from in-memory maps.
- **Channel deletion:** `@Observes ChannelClosedEvent` — removes entries from `channelHooks` and `byId` for the closed channel, then calls `store.deleteByChannelId()` to clean the DB. `deleteByChannelId()` is cross-tenant (no `CurrentPrincipal` filter) — channel deletion is authoritative and all registrations for a given channel share the same tenant.
- **`findForChannel(UUID channelId, String tenancyId)`** — returns `globalHooks.get(tenancyId) + channelHooks.get(channelId)`. Tenant-scoped on the hot path.
- **`listAll(String tenancyId)`** — filters `byId.values()` by `tenancyId`. Used by the REST endpoint.
- **`findByChannelId(UUID channelId)`** — unchanged (channel UUIDs are globally unique, inherently tenant-correct).

The in-memory `ConcurrentHashMap` remains the runtime lookup for `WebhookMessageObserver.onMessage()`. JPA is purely for durability and reload.

#### Prerequisite: ChannelClosedEvent

A new `ChannelClosedEvent(UUID channelId, String channelName)` CDI event is added to `io.casehub.qhorus.api.gateway`, fired by `ChannelGateway.closeChannel()` after calling `close()` on all registered backends. This mirrors the existing `ChannelInitialisedEvent` pattern and enables any observer (not just backends) to react to channel deletion.

### WebhookRegistration Record

Becomes a DTO/API type. `secret` field replaced by `secretRef`:

```java
public record WebhookRegistration(
        UUID id,
        UUID channelId,
        String tenancyId,
        String url,
        String secretRef,
        Map<String, String> headers,
        Instant createdAt) { ... }
```

### WebhookMessageObserver

Injects `CredentialResolver`. The `onMessage(MessageReceivedEvent event)` call passes `event.tenancyId()` through to `registry.findForChannel(event.channelId(), event.tenancyId())` for tenant-scoped webhook lookup.

At POST time, if `secretRef` is non-null, resolves the actual secret via `credentialResolver.resolve(secretRef)` and extracts `creds.get(CredentialPropertyKeys.SIGNING_SECRET)`. Same pattern as `SlackChannelBackend.resolveToken()` which extracts `CredentialPropertyKeys.BEARER_TOKEN`.

Resolution failure (missing credential or missing `signing-secret` key) logs ERROR and **skips the POST entirely**. A `secretRef` declares that signing is mandatory — sending unsigned is not an acceptable fallback.

### REST API

`WebhookRegistryResource` injects `CurrentPrincipal` and passes `tenancyId` to registry methods:
- `list()` calls `registry.listAll(currentPrincipal.tenancyId())` — tenant-scoped
- `list(channelId)` calls `registry.findByChannelId(channelId)` — inherently tenant-correct via globally unique channel UUIDs

`RegisterRequest` changes:
- `secret` → `secretRef` (breaking, acceptable — no durable consumers)
- All other fields unchanged

### Flyway Migration — V35

```sql
CREATE TABLE webhook_registration (
    id UUID PRIMARY KEY,
    channel_id UUID,
    url VARCHAR(2048) NOT NULL,
    secret_ref VARCHAR(255),
    headers TEXT,
    tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT uq_webhook_url_channel_tenant UNIQUE (url, channel_id, tenancy_id),
    CONSTRAINT fk_webhook_channel FOREIGN KEY (channel_id)
        REFERENCES channel(id)
);

-- NULL ≠ NULL in SQL UNIQUE constraints — partial index covers global webhooks
CREATE UNIQUE INDEX uq_webhook_global ON webhook_registration (url, tenancy_id)
    WHERE channel_id IS NULL;
```

### Protocol Compliance

- **optional-module-jpa-package-registration (PP-20260618-d9aeef):** Consumers adding `casehub-qhorus-webhook-observer` must append `io.casehub.qhorus.webhook` to `quarkus.hibernate-orm.qhorus.packages`.
- **qhorus-flyway-consumer-versioning (PP-20260521-0ba358):** V35 is in the domain sequence (V1–V999), not the ledger subclass range (V2000+). Correct.

## Testing

| Test | Type | What it covers |
|------|------|----------------|
| `WebhookRegistrationStoreTest` | `@QuarkusTest` + H2 | CRUD, tenancy filtering, channel deletion cleanup (deleteByChannelId) |
| `WebhookRegistryTest` | CDI-free unit | In-memory lookup, tenant-scoped global hooks, cross-tenant isolation |
| `WebhookMessageObserverTest` | CDI-free unit | Credential resolution, HMAC with resolved secret, missing-credential graceful skip |
| `WebhookFlywaySchemaTest` | Plain Java | V35 migration produces correct schema |

Test `application.properties` must include:
- `quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime,io.casehub.qhorus.webhook`
- Standard qhorus H2 datasource config
- `casehub.qhorus.delivery.enabled=false`

## Scope

**In scope:** JPA entity, store, registry reload, secretRef migration, V35, tests.

**Out of scope:**
- Retry/backoff for failed webhook POSTs (#1)
- Health monitoring for failing webhooks (#2)

### Prerequisite Changes (other modules)

- **casehub-platform-api:** Add `SIGNING_SECRET = "signing-secret"` to `CredentialPropertyKeys`.
- **casehub-qhorus (api):** Add `ChannelClosedEvent(UUID channelId, String channelName)` record to `io.casehub.qhorus.api.gateway`.
- **casehub-qhorus (runtime):** Fire `ChannelClosedEvent` from `ChannelGateway.closeChannel()` after closing all backends.

### Dependency Changes (webhook-observer/pom.xml)

New dependencies:
- `io.quarkus:quarkus-hibernate-orm` (for `@Entity`, `EntityManager`, `@PersistenceUnit`)
- `io.quarkus:quarkus-jdbc-h2` (test scope, for `@QuarkusTest` with H2)

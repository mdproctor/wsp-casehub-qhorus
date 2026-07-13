# Webhook Observer: Persistent Registrations (JPA)

**Issue:** #345
**Date:** 2026-07-13
**Status:** Approved

## Problem

The webhook observer module (from #163) uses an in-memory `ConcurrentHashMap` for webhook registrations. Registrations are lost on server restart. Production deployments need durable webhook subscriptions.

## Design

### Credential Security

The existing `WebhookRegistration.secret` field stores a plaintext HMAC signing key. This is replaced with `secretRef` — a reference resolved via `CredentialResolver` at POST time. The actual secret never appears in the database or in-memory data structures. This follows the same pattern as `SlackChannelBackend.resolveToken()`.

Breaking change to the REST API: `RegisterRequest.secret` becomes `RegisterRequest.secretRef`. No migration needed — the module shipped in #163 with explicit "in-memory only, lost on restart" documentation. No durable consumers exist.

### JPA Entity

`WebhookRegistrationEntity` in `io.casehub.qhorus.webhook`:

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUID | PK |
| `channelId` | UUID | Nullable (null = global webhook), FK → channel ON DELETE CASCADE |
| `url` | String | NOT NULL, max 2048 |
| `secretRef` | String | Nullable (no signature if omitted) |
| `headers` | String | Nullable, JSON-serialized Map via JPA @Converter |
| `tenancyId` | String | NOT NULL, default DEFAULT_TENANT_ID |
| `createdAt` | Instant | NOT NULL |

UNIQUE constraint: `(url, channel_id, tenancy_id)` — prevents duplicate registrations.

### Store

`WebhookRegistrationStore` — `@ApplicationScoped`, injects `@PersistenceUnit("qhorus") EntityManager`:

- `findById(UUID)` → `Optional<WebhookRegistrationEntity>`
- `findByChannelId(UUID)` → `List<WebhookRegistrationEntity>` (channel-specific only)
- `findGlobal()` → `List<WebhookRegistrationEntity>` (channelId is null)
- `findAll()` → `List<WebhookRegistrationEntity>`
- `save(WebhookRegistrationEntity)` — `em.merge()`
- `delete(UUID)` → `boolean`
- `deleteByChannelId(UUID)` — channel deletion cleanup

All queries filter by `tenancyId` from `CurrentPrincipal`.

No `InMemoryWebhookRegistrationStore` — optional module with its own store (same pattern as `SlackBotBindingStore`). Tests use H2.

### Registry Changes

`WebhookRegistry` becomes the coordination layer between the in-memory cache and the JPA store:

- **Startup reload:** `@Observes StartupEvent` loads all registrations from the store into the in-memory maps.
- **`register()`** — `@Transactional`: persists to store, then populates in-memory maps.
- **`deregister()`** — `@Transactional`: removes from store, then removes from in-memory maps.
- **Lookup methods** (`findForChannel`, `findByChannelId`, `listAll`) — unchanged, still read from in-memory maps only (hot path).

The in-memory `ConcurrentHashMap` remains the runtime lookup for `WebhookMessageObserver.onMessage()`. JPA is purely for durability and reload.

### WebhookRegistration Record

Becomes a DTO/API type. `secret` field replaced by `secretRef`:

```java
public record WebhookRegistration(
        UUID id,
        UUID channelId,
        String url,
        String secretRef,
        Map<String, String> headers) { ... }
```

### WebhookMessageObserver

Injects `CredentialResolver`. At POST time, if `secretRef` is non-null, resolves the actual secret via `credentialResolver.resolve(secretRef)` and uses it for HMAC-SHA256 signing. Same pattern as `SlackChannelBackend.resolveToken()`.

Resolution failure (missing credential) logs WARN and skips the signature — the webhook POST still fires, just unsigned.

### REST API

`WebhookRegistryResource.RegisterRequest` changes:
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
    tenancy_id VARCHAR(255) NOT NULL DEFAULT 'DEFAULT',
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT uq_webhook_url_channel_tenant UNIQUE (url, channel_id, tenancy_id),
    CONSTRAINT fk_webhook_channel FOREIGN KEY (channel_id)
        REFERENCES channel(id) ON DELETE CASCADE
);
```

### Protocol Compliance

- **optional-module-jpa-package-registration (PP-20260618-d9aeef):** Consumers adding `casehub-qhorus-webhook-observer` must append `io.casehub.qhorus.webhook` to `quarkus.hibernate-orm.qhorus.packages`.
- **qhorus-flyway-consumer-versioning (PP-20260521-0ba358):** V35 is in the domain sequence (V1–V999), not the ledger subclass range (V2000+). Correct.

## Testing

| Test | Type | What it covers |
|------|------|----------------|
| `WebhookRegistrationStoreTest` | `@QuarkusTest` + H2 | CRUD, tenancy filtering, cascade delete |
| `WebhookRegistryTest` | CDI-free unit | In-memory lookup (updated: `secretRef` not `secret`) |
| `WebhookMessageObserverTest` | CDI-free unit | Credential resolution, HMAC with resolved secret, missing-credential graceful skip |
| `WebhookFlywaySchemaTest` | Plain Java | V35 migration produces correct schema |

Test `application.properties` must include:
- `quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime,io.casehub.qhorus.webhook`
- Standard qhorus H2 datasource config
- `casehub.qhorus.delivery.enabled=false`

## Scope

**In scope:** JPA entity, store, registry reload, secretRef migration, V35, tests.

**Out of scope:** Retry/backoff for failed webhook POSTs. Health monitoring for failing webhooks. These are separate concerns if needed later.

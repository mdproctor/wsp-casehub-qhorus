# Mesh Service Auth — OIDC Integration + API Key Fallback

**Issue:** casehubio/qhorus#508
**Parent:** casehubio/qhorus#475 (distributed mesh epic)
**Related:** casehubio/eidos#215 (external identity service — future)

## Problem

The standalone mesh service has zero authentication. Any network-reachable
client can register agents, send messages, create channels, and read all
data. `QhorusInboundCurrentPrincipal` returns `"anonymous"` for every caller.
`X-Tenancy-ID` is a routing header, not a security boundary
(PP-20260610-9487d3).

## Solution

Two authentication mechanisms that coexist via Quarkus's
`HttpAuthenticationMechanism` priority chain:

1. **OIDC** — add `casehub-platform-oidc` dependency. Zero code changes.
   `SecurityIdentityCurrentPrincipal @Alternative @Priority(100)`
   auto-displaces `QhorusInboundCurrentPrincipal`. JWT bearer tokens
   carry actorId, tenancyId, and roles.

2. **API key** — new `HttpAuthenticationMechanism` with an
   `ApiKeyIdentityResolver` SPI. Config-based default impl for dev/edge.
   Eidos provides dynamic resolution later (eidos#215).

Both mechanisms are active simultaneously. OIDC bearer token is tried
first (higher priority). API key header is the fallback.

## Architecture

### Component Overview

```
HTTP Request
    │
    ├─ Authorization: Bearer <jwt>  ──→  quarkus-oidc mechanism
    │                                         │
    │                                    SecurityIdentity
    │                                         │
    ├─ X-Api-Key: <key>  ──→  ApiKeyAuthenticationMechanism
    │                              │
    │                         ApiKeyIdentityResolver SPI
    │                              │
    │                    ┌─────────┴──────────┐
    │              ConfigResolver        (eidos #215)
    │              (@DefaultBean)        (future SPI impl)
    │                    │
    │               SecurityIdentity
    │                    │
    └───────────────────→ SecurityIdentityCurrentPrincipal
                              │
                         CurrentPrincipal
                         (actorId, tenancyId, groups)
```

### New Components

**1. `ApiKeyIdentityResolver` — SPI in `qhorus-api`**

```java
package io.casehub.qhorus.api.identity;

public interface ApiKeyIdentityResolver {
    Optional<ResolvedIdentity> resolve(String apiKey);
}

public record ResolvedIdentity(
    String actorId,
    String tenancyId,
    Set<String> roles
) {}
```

Placed in `api/` so both `runtime/` and `mesh/` can see it, and
external modules (eidos) can implement it without depending on
`casehub-qhorus` runtime.

**2. `ApiKeyAuthenticationMechanism` — in `mesh/`**

```java
package io.casehub.qhorus.mesh.auth;

@ApplicationScoped
public class ApiKeyAuthenticationMechanism
        implements HttpAuthenticationMechanism {

    @Inject ApiKeyIdentityResolver resolver;

    @Override
    public Uni<SecurityIdentity> authenticate(
            RoutingContext context,
            IdentityProviderManager identityProviderManager) {
        String key = context.request().getHeader("X-Api-Key");
        if (key == null || key.isBlank()) {
            return Uni.createFrom().optional(Optional.empty());
        }
        return Uni.createFrom().item(() -> resolver.resolve(key))
            .map(opt -> opt.map(this::toSecurityIdentity).orElse(null));
    }

    @Override
    public Uni<HttpCredentialTransport> getCredentialTransport(
            RoutingContext context) {
        return Uni.createFrom().item(
            new HttpCredentialTransport(
                HttpCredentialTransport.Type.OTHER_HEADER,
                "X-Api-Key"));
    }

    @Override
    public int getPriority() {
        return 2000; // lower than OIDC (default ~1000)
    }
}
```

Lives in `mesh/` (not `runtime/`) because this is a standalone-service
concern. Embedded qhorus consumers manage their own auth.

**3. `ConfigApiKeyIdentityResolver` — `@DefaultBean` in `runtime/`**

```java
package io.casehub.qhorus.runtime.identity;

@DefaultBean
@ApplicationScoped
public class ConfigApiKeyIdentityResolver
        implements ApiKeyIdentityResolver {

    @Inject QhorusConfig config;

    @Override
    public Optional<ResolvedIdentity> resolve(String apiKey) {
        return config.apiKeys().entrySet().stream()
            .filter(e -> e.getValue().key().equals(apiKey))
            .findFirst()
            .map(e -> new ResolvedIdentity(
                e.getKey(),
                config.defaultTenancyId()
                    .orElse(TenancyConstants.DEFAULT_TENANT_ID),
                e.getValue().roles().orElse(Set.of())));
    }
}
```

Config structure in `application.properties`:

```properties
casehub.qhorus.api-keys.agent-alpha.key=sk-abc123
casehub.qhorus.api-keys.agent-alpha.roles=agent
casehub.qhorus.api-keys.agent-beta.key=sk-def456
casehub.qhorus.default-tenancy-id=my-tenant
```

The map key is the actorId. Single global tenant from
`default-tenancy-id`. Roles are optional.

### OIDC Wiring

Add to `mesh/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-oidc</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

This transitively brings `quarkus-oidc`. Add OIDC config to
`mesh/src/main/resources/application.properties`:

```properties
# OIDC — configure via env vars in production
quarkus.oidc.auth-server-url=${QHORUS_OIDC_AUTH_SERVER_URL:}
quarkus.oidc.client-id=${QHORUS_OIDC_CLIENT_ID:}
quarkus.oidc.credentials.secret=${QHORUS_OIDC_CLIENT_SECRET:}
quarkus.oidc.token.audience=${QHORUS_OIDC_AUDIENCE:}

# When no OIDC provider configured, allow anonymous + API key
quarkus.oidc.tenant-enabled=${QHORUS_OIDC_ENABLED:false}
```

When `QHORUS_OIDC_ENABLED=false` (default), the OIDC mechanism is
inactive and API key auth is the only path. When `true`, both coexist.

### Endpoint Security Policy

All endpoints require authentication when any auth mechanism is
configured. Quarkus HTTP security policy in `application.properties`:

```properties
quarkus.http.auth.permission.default.paths=/*
quarkus.http.auth.permission.default.policy=authenticated
quarkus.http.auth.permission.internal.paths=/internal/*
quarkus.http.auth.permission.internal.policy=permit
quarkus.http.auth.permission.health.paths=/q/*
quarkus.http.auth.permission.health.policy=permit
```

Health endpoints (`/q/health`, `/q/metrics`) remain open.
`/internal/*` paths are permitted at the HTTP auth policy level because
`InternalSecretFilter @Priority(50)` already guards them with
`X-Internal-Secret`. The filter doesn't set a `SecurityIdentity`,
so the `authenticated` policy would reject valid inter-node requests.
The two auth layers are orthogonal: Quarkus HTTP security handles
external clients, `InternalSecretFilter` handles inter-node.

**When no API keys and no OIDC are configured:** Quarkus's anonymous
authentication mechanism applies, and the `authenticated` policy still
requires _some_ identity. We need to handle the "no auth configured"
case — either permit anonymous, or fail fast at startup.

**Decision: fail fast.** If neither OIDC nor any API keys are
configured, log a WARN at startup. In dev mode, auto-register a
default dev key (`dev-key` → actorId `dev-agent`) so the mesh is
immediately usable. In prod profile, the WARN is sufficient — the
admin must configure auth.

### Dev Mode

```properties
%dev.casehub.qhorus.api-keys.dev-agent.key=dev-key
%dev.casehub.qhorus.api-keys.dev-agent.roles=admin
```

### Inter-Node Auth Unchanged

`InternalSecretFilter` at `@Priority(50)` continues to guard
`/internal/*` paths with `X-Internal-Secret`. This is orthogonal to
external client auth and remains unchanged.

## Testing

- **Unit tests:** `ApiKeyAuthenticationMechanism` with mock resolver.
  `ConfigApiKeyIdentityResolver` with test config.
- **Integration tests:** `@QuarkusTest` with API key config —
  verify `CurrentPrincipal.actorId()` and `tenancyId()` are populated
  from the resolved key.
- **OIDC integration:** Not tested in qhorus — platform-oidc has its
  own test suite. Verify the dependency wiring compiles and
  `SecurityIdentityCurrentPrincipal` displaces correctly.
- **No-auth startup:** Verify dev mode auto-registers dev key. Verify
  prod mode logs WARN when unconfigured.

## Migration

No database changes. No Flyway migration needed.

## Future

- **eidos#215:** Dynamic credential registry implements
  `ApiKeyIdentityResolver` SPI with DB-backed CRUD, key rotation,
  and self-organising society credential registration.
- **Role-based authorization:** Future issue can add `@RolesAllowed`
  on admin endpoints using the roles from `ResolvedIdentity`.

## References

- `cluster/src/main/java/.../InternalSecretFilter.java` — existing inter-node auth pattern
- `runtime/src/main/java/.../identity/QhorusInboundCurrentPrincipal.java` — current default principal
- `platform/oidc/src/main/java/.../SecurityIdentityCurrentPrincipal.java` — OIDC principal that auto-displaces
- PP-20260610-9487d3 — X-Tenancy-ID is not a security boundary
- PP-20260610-85e6a4 — ApplicationScoped principal pattern
- `runtime/src/main/java/.../instance/RegistryBackedInstanceManager.java` — precedent for SPI + @DefaultBean pattern
- casehubio/eidos#215 — future dynamic identity service

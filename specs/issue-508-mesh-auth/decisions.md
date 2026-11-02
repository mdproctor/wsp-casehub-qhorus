## D1: Auth mode coexistence

**Choice:** OIDC and API key mechanisms coexist simultaneously — no config switch needed
**Alternatives:**
- Explicit config switch (oidc|api-key|none) — simpler mental model but less flexible
- Coexist with disable switch — unnecessary complexity, dev can just not configure keys
**Rationale:** Quarkus supports multiple HttpAuthenticationMechanisms natively. Presence of credentials determines the path — OIDC bearer tried first, API key header as fallback.
**Trade-offs:** Slightly harder to reason about "which auth is active" in logs, but Quarkus handles the priority chain transparently.
**Sources:** Quarkus security architecture, SecurityIdentityCurrentPrincipal.java
**Exploration:** quick
**Status:** captured

## D2: API key authentication mechanism

**Choice:** Custom HttpAuthenticationMechanism with SPI resolver
**Alternatives:**
- JAX-RS ContainerRequestFilter (InternalSecretFilter pattern) — doesn't integrate with Quarkus security subsystem, SecurityIdentityCurrentPrincipal won't see the identity
- SecurityIdentityAugmentor — can't create identity from scratch, only enriches existing
**Rationale:** Integrates with Quarkus security natively, coexists with OIDC, SPI boundary is exactly what eidos #215 needs for dynamic credential resolution.
**Trade-offs:** More framework coupling than a simple filter, but that coupling is the point — we want the identity to flow through the standard CurrentPrincipal path.
**Sources:** InternalSecretFilter.java, SecurityIdentityCurrentPrincipal.java, PP-20260610-9487d3 (X-Tenancy-ID not security boundary), PP-20260610-85e6a4 (ApplicationScoped principal pattern)
**Exploration:** quick
**Status:** captured

## D3: API key identity resolution — SPI with config default

**Choice:** ApiKeyIdentityResolver SPI with @DefaultBean config-file implementation, single-tenant
**Alternatives:**
- Database-backed registry — more operational surface, needs migration and admin API; deferred to eidos #215
- External identity service — belongs in eidos, filed as eidos#215
**Rationale:** Config-based default is sufficient for dev/edge. SPI seam lets eidos provide dynamic resolution later without changing qhorus.
**Trade-offs:** Static config requires restart to add/remove keys. Multi-tenant not supported in default impl — requires OIDC or eidos #215.
**Depends on:** D2 (mechanism determines how resolver is invoked)
**Sources:** RegistryBackedInstanceManager.java (precedent for SPI+default pattern), eidos#215
**Exploration:** quick
**Status:** captured

## D4: Endpoint auth scope

**Choice:** All endpoints require auth — MCP, REST, A2A, WebSocket
**Alternatives:**
- REST/A2A/WebSocket only — leaves MCP open with local trust model
**Rationale:** MCP-over-SSE carries HTTP headers, so bearer/API key works naturally. Consistent auth boundary across all transports.
**Trade-offs:** Dev setup requires configuring at least one API key (or disabling auth via empty config).
**Sources:** MeshService.java (MCP tools), A2AResource.java, ChannelResource.java
**Exploration:** quick
**Status:** captured

## D5: Authorization scope

**Choice:** Authentication only — no endpoint-level role guards
**Alternatives:**
- Basic @RolesAllowed on admin endpoints — more secure but broader scope
**Rationale:** Existing channel ACLs (allowedWriters, adminInstances) already handle authorization. Adding role guards is a separate concern for a future issue.
**Trade-offs:** Any authenticated user can hit any endpoint. Channel-level ACLs still enforce write access.
**Sources:** ChannelService.java (allowedWriters), AllowedWritersPolicy.java
**Exploration:** quick
**Status:** captured

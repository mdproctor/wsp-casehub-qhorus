# A2A, AgentCard, and Ledger Tenant Scoping — Design Spec

**Date:** 2026-06-10  
**Issues:** #265 (A2A tenant scoping), #264 (AgentCard per-tenant), #263 (MessageLedgerEntryRepository tenant scoping)  
**Branch:** `issue-265-a2a-tenant-scoping`

---

## Problem

Every HTTP request to Qhorus — A2A, MCP REST tools, AgentCard — runs with `MockCurrentPrincipal @DefaultBean @ApplicationScoped`, which returns `DEFAULT_TENANT_ID` unconditionally. In a multi-tenant deployment without OIDC, there is no mechanism for a caller to specify their tenant. Three consequences:

1. **A2A messages always land in the default tenant.** `MessageService.dispatch()` resolves `tenancyId` from `CurrentPrincipal`; A2A has no auth, so multi-tenant A2A routing is impossible.
2. **AgentCard carries no tenant identity.** A2A orchestrators discovering the card cannot determine which tenant's mesh they are talking to.
3. **`MessageLedgerEntryRepository` queries are cross-tenant.** JPQL queries filter by `channelId` and `correlationId` but not `tenancyId` — ledger data leaks across tenants.

---

## Design

### Foundation — Inbound HTTP Tenant Principal (addresses #265)

Three new classes in `runtime/src/main/java/io/casehub/qhorus/runtime/identity/`:

#### `InboundTenancyContext` — `@RequestScoped`

Plain CDI holder populated by the JAX-RS filter and read by the principal bean.

```java
@RequestScoped
public class InboundTenancyContext {
    private String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
    public String tenancyId() { return tenancyId; }
    public void set(String t) { this.tenancyId = t != null ? t : TenancyConstants.DEFAULT_TENANT_ID; }
}
```

#### `TenancyContextFilter` — `@Provider @Priority(100)`

Standard JAX-RS `ContainerRequestFilter`. Runs before resource method dispatch, inside active request scope.

```java
@Provider
@Priority(100)
public class TenancyContextFilter implements ContainerRequestFilter {
    @Inject InboundTenancyContext ctx;

    @Override
    public void filter(ContainerRequestContext req) {
        ctx.set(req.getHeaderString("X-Tenancy-ID"));
    }
}
```

Fallback: `InboundTenancyContext.set(null)` resolves to `DEFAULT_TENANT_ID`.

#### `QhorusInboundCurrentPrincipal` — `@RequestScoped @Alternative @Priority(1)`

Reads from `InboundTenancyContext`. Implements `CurrentPrincipal`.

```java
@RequestScoped
@Alternative
@Priority(1)
public class QhorusInboundCurrentPrincipal implements CurrentPrincipal {
    @Inject InboundTenancyContext ctx;

    @Override public String actorId()           { return "anonymous"; }
    @Override public Set<String> groups()       { return Set.of(); }
    @Override public String tenancyId()         { return ctx.tenancyId(); }
    @Override public boolean isCrossTenantAdmin() { return false; }
}
```

**CDI priority ladder** (highest wins):

| Bean | Priority | When active |
|------|----------|-------------|
| `OidcCurrentPrincipal` | 100 | When `casehub-platform-oidc` on classpath |
| `QhorusInboundCurrentPrincipal` | 1 | All HTTP requests, no OIDC |
| `MockCurrentPrincipal @DefaultBean` | -1 | CDI-free tests, background tasks |

**No changes required** to `A2AResource`, `A2AChannelBackend`, `MessageService`, or any store. Every tenant-scoped path already reads `CurrentPrincipal`; the new bean provides the correct value.

**No build-time gating** (`@IfBuildProperty`/`@UnlessBuildProperty`). `@RequestScoped` CDI beans and JAX-RS `@Provider` filters work in both blocking and reactive stacks. Background tasks (Watchdog, Quartz) use `@CrossTenant` stores and/or set `tenancyId` explicitly on `MessageDispatch` — they do not depend on `CurrentPrincipal`.

### #264 — AgentCard per-tenant

`AgentCard` record gains a `tenancyId` field:

```java
public record AgentCard(
        String name, String description, String url, String version,
        List<AgentSkill> skills, AgentCapabilities capabilities,
        String tenancyId)  // reflects CurrentPrincipal.tenancyId()
```

Both `AgentCardResource` and `ReactiveAgentCardResource` inject `CurrentPrincipal` and pass `currentPrincipal.tenancyId()` when constructing the card.

**Behaviour:**
- Request with `X-Tenancy-ID: tenant-a` → `AgentCard.tenancyId = "tenant-a"`
- Request without header → `AgentCard.tenancyId = DEFAULT_TENANT_ID`
- OIDC request → `AgentCard.tenancyId` reflects the JWT claim

This makes the card self-describing: an A2A orchestrator reads the card, notes `tenancyId`, then sends messages with `X-Tenancy-ID: <tenancyId>`.

### #263 — `MessageLedgerEntryRepository` tenant scoping

Add `String tenancyId` as the final parameter to every query method that currently lacks it, and append `AND e.tenancyId = :tenancyId` to the JPQL predicate.

**Methods updated (blocking + reactive mirrors):**

| Method | Tenant predicate added |
|--------|----------------------|
| `findByChannelId(channelId, tenancyId)` | `AND e.tenancyId = :tid` |
| `listEntries(..., tenancyId)` (both overloads) | `AND e.tenancyId = :tid` |
| `findAllByCorrelationId(channelId, corrId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findAncestorChain(channelId, entryId, tenancyId)` | per-entry `e.tenancyId` check in loop |
| `findStalledCommands(channelId, olderThan, tenancyId)` | `AND c.tenancyId = :tid` (both subqueries) |
| `countByOutcome(channelId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findByActorIdInChannel(channelId, actorId, limit, tenancyId)` | `AND e.tenancyId = :tid` |
| `findEventsSince(channelId, since, tenancyId)` | `AND e.tenancyId = :tid` |
| `findLatestByCorrelationId(channelId, corrId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findEarliestWithSubjectByCorrelationId(corrId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findByCorrelationIdAcrossChannels(corrId, limit, tenancyId)` | `AND e.tenancyId = :tid` |

**Unchanged (by surrogate PK — unique within datasource):**
- `findByMessageId(messageId)`
- `findByMessageIds(messageIds)`

**Callers — source of `tenancyId`:**

| Caller | Source |
|--------|--------|
| `LedgerWriteService.record()` | `dispatch.tenancyId()` — non-null by this point (set by `MessageService.dispatch()`) |
| `QhorusMcpTools` (ledger query tools) | `currentPrincipal.tenancyId()` — already injected since #260 |
| `ReactiveQhorusMcpTools` | same |
| `ReactiveLedgerWriteService.record()` | `dispatch.tenancyId()` |

**`StubMessageLedgerEntryRepository`** (test stub, `runtime/src/test/`) — update all overriding methods to match the new signatures.

**Test call sites** that directly call repo methods pass `TenancyConstants.DEFAULT_TENANT_ID`. Test setup does not change — no new tables, no new migrations.

---

## What Doesn't Change

- `A2AResource`, `ReactiveA2AResource`, `A2AChannelBackend` — no code changes (tenant flows through `CurrentPrincipal` automatically)
- `MessageService`, `JpaMessageStore`, `JpaChannelStore`, `JpaCommitmentStore` — already inject `CurrentPrincipal`; no changes
- `QhorusLedgerEntryRepository` — already has full tenancy; no changes
- Flyway migrations — no new columns; `tenancyId` columns exist from #260
- `@CrossTenant` stores used by Watchdog/background — unaffected

---

## Testing

### #265 / #264
- `A2ATenantScopingTest @QuarkusTest` — sends `POST /a2a/message:send` with `X-Tenancy-ID: test-tenant`; asserts message's `tenancyId = "test-tenant"` in the store
- `AgentCardTenantTest @QuarkusTest` — `GET /.well-known/agent-card.json` with and without `X-Tenancy-ID` header; asserts `tenancyId` field in response
- Without header: `tenancyId = DEFAULT_TENANT_ID` in both

### #263
- `MessageLedgerEntryRepositoryTest` — extend existing tests to call all methods with explicit `tenancyId` parameter; assert cross-tenant data is not returned
- `LedgerWriteService` integration test — dispatch a message, verify ledger entry lands in correct tenant bucket
- `StubMessageLedgerEntryRepository` — updated signatures, all override methods pass through or no-op

---

## Protocol coherence

- Follows `unconditional-tenancy-filtering` (PP-20260520-439daf) — all queries now filter by `tenancyId`
- Follows `tenancyId-in-data-access-only` (PP-20260520-e6a5f0) — tenancyId not in domain model, only in persistence layer
- `QhorusInboundCurrentPrincipal @Alternative @Priority(1)` follows `persistence-backend-cdi-priority` pattern — activates by presence, displaced by higher-priority impl
- No `@CrossTenant` queries in `MessageLedgerEntryRepository` — all are per-tenant (contrast with `CrossTenantLedgerEntryRepository` in casehub-ledger which is an explicit separate interface)

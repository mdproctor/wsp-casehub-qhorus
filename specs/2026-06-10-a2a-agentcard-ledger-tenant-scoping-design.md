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

## Security boundary

**`X-Tenancy-ID` is not a security boundary.** Any HTTP caller can claim any tenant by including this header. The mechanism is appropriate only for deployments where network isolation (firewall, mTLS, gateway policy) enforces tenant trust — a caller inside the network boundary is trusted. This is consistent with the Qhorus trust posture documented in PLATFORM.md: internal foundation services carry no auth annotations and rely on network policy.

**Production multi-tenant deployments that require cross-tenant isolation must use `casehub-platform-oidc`**, which provides `OidcCurrentPrincipal @Priority(100)` that displaces `QhorusInboundCurrentPrincipal`. With OIDC active, `X-Tenancy-ID` has no effect — the tenant is set from the JWT claim.

This distinction must be understood before deploying Qhorus in a multi-tenant environment: `X-Tenancy-ID` routes, it does not authenticate.

---

## Design

### Foundation — Inbound HTTP Tenant Principal (addresses #265)

Three new classes in `runtime/src/main/java/io/casehub/qhorus/runtime/identity/`:

#### `InboundTenancyContext` — `@RequestScoped`

Plain CDI holder populated by the JAX-RS filter and read by the principal bean. Scoped to the HTTP request lifetime.

```java
@RequestScoped
public class InboundTenancyContext {
    private String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
    public String tenancyId() { return tenancyId; }
    public void set(String t) { this.tenancyId = t != null && !t.isBlank() ? t : TenancyConstants.DEFAULT_TENANT_ID; }
}
```

#### `TenancyContextFilter` — `@Provider @PreMatching @Priority(100)`

Standard JAX-RS `ContainerRequestFilter`. `@PreMatching` ensures it runs before resource matching, guaranteeing tenant context is set for all requests including those that may not match a resource method. Runs inside an active CDI request scope.

```java
@Provider
@PreMatching
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

Implements `CurrentPrincipal`. Reads from `InboundTenancyContext`.

```java
@RequestScoped
@Alternative
@Priority(1)
public class QhorusInboundCurrentPrincipal implements CurrentPrincipal {
    @Inject InboundTenancyContext ctx;

    @Override
    public String actorId() { return "anonymous"; }

    @Override
    public Set<String> groups() { return Set.of(); }

    @Override
    public String tenancyId() {
        try {
            return ctx.tenancyId();
        } catch (ContextNotActiveException e) {
            // Background threads (Scheduled, ObservesAsync, StartupEvent) have no request
            // scope. Per PP-20260609-scheduled-service-cross-tenant-stores, background code
            // should use @CrossTenant stores and never reach here through per-tenant stores.
            // This catch is a safety net for any code that accidentally calls a per-tenant
            // store from a non-HTTP context — DEFAULT_TENANT_ID is the correct fallback.
            return TenancyConstants.DEFAULT_TENANT_ID;
        }
    }

    @Override
    public boolean isCrossTenantAdmin() { return false; }
}
```

**Why `ContextNotActiveException` catch is needed:** Introducing `@RequestScoped @Alternative @Priority(1)` changes the `CurrentPrincipal` CDI proxy resolution behaviour. Previously, `MockCurrentPrincipal @ApplicationScoped @DefaultBean` resolved safely from any thread (application scope is always active). With `QhorusInboundCurrentPrincipal @RequestScoped` winning, any background thread that holds a reference to a per-tenant store (`JpaChannelStore`, `JpaMessageStore`, `JpaCommitmentStore`, `JpaWatchdogStore`) and calls a method that reads `currentPrincipal.tenancyId()` will throw `ContextNotActiveException`. The existing protocol (`scheduled-service-cross-tenant-stores.md`) requires background code to use `@CrossTenant`-qualified stores instead — Watchdog and ChannelGateway startup already comply. The `catch` is a safety net for future code that doesn't follow the protocol.

**Behavioural delta from `MockCurrentPrincipal`:** `MockCurrentPrincipal.actorId()` defaults to `"system"` (via `casehub.platform.principal.actorId`), making `isAuthenticated()` return `true`. `QhorusInboundCurrentPrincipal.actorId()` returns `"anonymous"`, making `isAuthenticated()` return `false`. This is semantically correct — an HTTP caller with no auth token is genuinely anonymous. No qhorus code currently gates on `isAuthenticated()`. Consumers that do gate on it will observe this change when they add qhorus to the classpath.

**CDI principal resolution:**

| Bean | How it's selected | When active |
|------|------------------|-------------|
| `OidcCurrentPrincipal` from `casehub-platform-oidc` | `@Alternative @Priority(100)` — highest priority; wins over all others | OIDC configured |
| `QhorusInboundCurrentPrincipal` | `@Alternative @Priority(1)` — beats the `@DefaultBean`; displaced by OIDC | Any HTTP request, no OIDC |
| `MockCurrentPrincipal @DefaultBean` | Suppressed by any `@Alternative` bean; active only when no `@Alternative` is present | CDI-free unit tests, non-HTTP CDI contexts |

Note: `@DefaultBean` is a CDI suppression qualifier, not a numeric priority — it loses to any non-default bean regardless of priority value.

**No build-time gating** (`@IfBuildProperty`/`@UnlessBuildProperty`). `@RequestScoped` CDI beans and JAX-RS `@Provider @PreMatching` filters work in both blocking and reactive stacks.

**Source logic unchanged** for `A2AResource`, `ReactiveA2AResource`, `A2AChannelBackend`, `MessageService`, and all stores. Every tenant-scoped path already reads `CurrentPrincipal`; the new bean provides the correct value. (`A2AResource.getTask()` and `ReactiveA2AResource.getTask()` require Javadoc additions — see getTask() tenancy requirement below.)

#### getTask() tenancy requirement

`GET /a2a/tasks/{id}` calls `messageService.findAllByCorrelationId(taskId)` which filters by tenant via `CurrentPrincipal`. A task submitted with `X-Tenancy-ID: tenant-a` but retrieved without the header will resolve to `DEFAULT_TENANT_ID` and return HTTP 404, even though the task exists. **A2A orchestrators must include `X-Tenancy-ID` consistently on both `sendMessage` and `getTask` calls for the same task.** Document this hazard in the `A2AResource.getTask()` and `ReactiveA2AResource.getTask()` Javadoc — these are the authoritative source for callers.

---

### #264 — AgentCard per-tenant

`AgentCard` record gains a `tenancyId` field:

```java
public record AgentCard(
        String name, String description, String url, String version,
        List<AgentSkill> skills, AgentCapabilities capabilities,
        String tenancyId)  // Qhorus extension — not in A2A spec; reflects CurrentPrincipal.tenancyId()
```

Both `AgentCardResource` and `ReactiveAgentCardResource` inject `CurrentPrincipal` and pass `currentPrincipal.tenancyId()` when constructing the card.

**Conformance note:** `tenancyId` is a Qhorus-specific extension to the A2A AgentCard schema. It is not defined in the Google A2A spec. A top-level field (rather than an `extensions` wrapper) is used because:
1. The A2A spec does not define an `extensions` map either — wrapping in an undefined `extensions` field solves no conformance problem.
2. Top-level access is simpler for consumers (`card.tenancyId` vs `card.extensions().get("casehub:tenancyId")`).
3. Qhorus's card already contains `mcp: true` in capabilities, also non-standard.

Most A2A validators tolerate unknown fields. Strict schema validators that reject unknown top-level fields will also reject an unknown `extensions` object. The conformance implication is the same either way.

**Behaviour:**
- Request with `X-Tenancy-ID: tenant-a` → `AgentCard.tenancyId = "tenant-a"`
- Request without header → `AgentCard.tenancyId = DEFAULT_TENANT_ID`
- OIDC request → `AgentCard.tenancyId` reflects the JWT-provided tenancyId

This makes the card self-describing: an A2A orchestrator reads the card, notes `tenancyId`, then sends messages with `X-Tenancy-ID: <tenancyId>`.

---

### #263 — `MessageLedgerEntryRepository` tenant scoping

Add `String tenancyId` as the final parameter to every query method that currently lacks it, and append `AND e.tenancyId = :tenancyId` to the JPQL predicate.

#### Blocking `MessageLedgerEntryRepository` — methods updated

| Method | Tenant predicate added |
|--------|----------------------|
| `findByChannelId(channelId, tenancyId)` | `AND e.tenancyId = :tid` |
| `listEntries(..., tenancyId)` (6-param and 8-param overloads become 7-param and 9-param) | `AND e.tenancyId = :tid` in the 9-param JPQL; 7-param delegates to 9-param and must thread `tenancyId` through the call explicitly — `return listEntries(channelId, messageTypes, afterSequence, agentId, since, null, false, limit, tenancyId)` |
| `findAllByCorrelationId(channelId, corrId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findAncestorChain(channelId, entryId, tenancyId)` | guard: `!tenancyId.equals(entry.tenancyId)` added to loop break condition |
| `findStalledCommands(channelId, olderThan, tenancyId)` | `AND c.tenancyId = :tid` on both outer and correlated subquery |
| `countByOutcome(channelId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findByActorIdInChannel(channelId, actorId, limit, tenancyId)` | `AND e.tenancyId = :tid` |
| `findEventsSince(channelId, since, tenancyId)` | `AND e.tenancyId = :tid` |
| `findLatestByCorrelationId(channelId, corrId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findEarliestWithSubjectByCorrelationId(corrId, tenancyId)` | `AND e.tenancyId = :tid` |
| `findByCorrelationIdAcrossChannels(corrId, limit, tenancyId)` | `AND e.tenancyId = :tid` |

**Unchanged (surrogate PK — unique within datasource, no tenant ambiguity):**
- `findByMessageId(messageId)`
- `findByMessageIds(messageIds)`

#### Reactive `ReactiveMessageLedgerEntryRepository` — methods updated

The reactive repo has 4 methods. Three get tenancyId:

| Reactive method | Change |
|----------------|--------|
| `findByChannelId(channelId, tenancyId)` | `AND subjectId = ?1 AND tenancyId = ?2` |
| `findLatestByCorrelationId(channelId, corrId, tenancyId)` | add `AND tenancyId = ?3` |
| `findEarliestWithSubjectByCorrelationId(corrId, tenancyId)` | add `AND tenancyId = ?2` |
| `findByMessageId(messageId)` | **unchanged** — PK-based |

#### Known limitations — cross-tenant delegation

**`findEarliestWithSubjectByCorrelationId`:** Adding tenant scoping here means a HANDOFF that delegates an obligation to an agent in a **different tenant** will silently fail to propagate `subjectId` to the child correlation thread. The child's `LedgerWriteService.record()` call will not find a matching seed entry across the tenant boundary and will fall back to using `channelId` as the `subjectId`. This is acceptable for current use cases (cross-tenant delegation is not yet a use case) but is now an explicit architectural constraint baked into the implementation. Any future cross-tenant delegation feature must revisit this method and either make it cross-tenant or introduce an explicit cross-tenant subjectId propagation path.

**`findByCorrelationIdAcrossChannels`:** Same constraint. `get_obligation_activity` will show only the originating tenant's entries for a correlationId. If an obligation is delegated via HANDOFF to an agent in a different tenant, the cross-channel trace breaks at the tenant boundary. State as a known limitation in the MCP tool's documentation.

#### Callers — source of `tenancyId`

| Caller | Source |
|--------|--------|
| `LedgerWriteService.record()` | `dispatch.tenancyId()` — guaranteed non-null at this point (set by `MessageService.dispatch()` before calling `record()`) |
| `ReactiveLedgerWriteService.record()` | `dispatch.tenancyId()` |
| `QhorusMcpTools.listLedgerEntries` | `currentPrincipal.tenancyId()` |
| `QhorusMcpTools.getObligationChain` | `currentPrincipal.tenancyId()` |
| `QhorusMcpTools.getCausalChain` | `currentPrincipal.tenancyId()` |
| `QhorusMcpTools.listStalledObligations` | `currentPrincipal.tenancyId()` — calls `findStalledCommands` |
| `QhorusMcpTools.getObligationStats` | `currentPrincipal.tenancyId()` — calls `countByOutcome` + `findStalledCommands` |
| `QhorusMcpTools.getTelemetrySummary` | `currentPrincipal.tenancyId()` — calls `findEventsSince` |
| `QhorusMcpTools.getObligationActivity` | `currentPrincipal.tenancyId()` — calls `findByCorrelationIdAcrossChannels` |
| `ReactiveQhorusMcpTools` (blocking mirrors of all above) | `currentPrincipal.tenancyId()` |

**Files with `@Override` methods that must have signatures updated:**

- `StubMessageLedgerEntryRepository` (`runtime/src/test/.../ledger/`) — stub has 3 `@Override` methods: `findByMessageId` (PK-based, **unchanged**), `findEarliestWithSubjectByCorrelationId` (gains `tenancyId`), `findLatestByCorrelationId` (gains `tenancyId`).
- `LedgerQueryRepoTest.CapturingRepo` (`runtime/src/test/.../ledger/LedgerQueryRepoTest.java`, line 43) — package-private static inner class; not a stub, a pure in-memory capture device for CDI-free unit tests. Has 7 `@Override` methods that all change signature: `findAllByCorrelationId`, `findAncestorChain`, `findStalledCommands`, `countByOutcome`, `findByActorIdInChannel`, `findEventsSince`, `listEntries` (the 8-param overload, which becomes 9-param). The `save()` and `findLatestBySubjectId()` helpers in `CapturingRepo` are NOT `@Override` — they don't change.

**Test and example files with direct call sites that will fail to compile — all pass `TenancyConstants.DEFAULT_TENANT_ID`:**

| File | Methods called (that change signature) | CI? |
|------|---------------------------------------|-----|
| `runtime/src/test/.../ledger/MessageLedgerEntryRepositoryTest.java` (pkg `io.casehub.qhorus.ledger`) | `findByChannelId`, `listEntries` (6-param), `findLatestByCorrelationId` | ✅ |
| `runtime/src/test/.../runtime/ledger/MessageLedgerEntryRepositoryTest.java` (pkg `io.casehub.qhorus.runtime.ledger`) | `findEarliestWithSubjectByCorrelationId` | ✅ |
| `runtime/src/test/.../ledger/MessageLedgerCaptureTest.java` | `findByChannelId` (26 call sites across all 9 message-type test methods) + `listEntries` 6-param (4 call sites: lines 318, 331, 344, 356) | ✅ |
| `examples/type-system/src/test/.../LedgerCaptureExampleTest.java` | `findByChannelId` (lines 94, 123) | ✅ runs without flags |
| `runtime/src/test/.../ledger/LedgerAttestationIntegrationTest.java` (pkg `io.casehub.qhorus.ledger`) | `findAllByCorrelationId` (lines 68, 96, 117, 135, 159, 177, 196) + `findByActorIdInChannel` (line 210) | ✅ |
| `runtime/src/test/.../ledger/LedgerQueryRepoTest.java` (plain `@Test`, no Quarkus) | test-body call sites for all 7 methods that CapturingRepo overrides: `findAllByCorrelationId`, `findAncestorChain`, `findStalledCommands`, `countByOutcome`, `findByActorIdInChannel`, `findEventsSince`, `listEntries` (both 6-param and 8-param overloads called from test bodies) | ✅ |
| `examples/agent-communication/src/test/.../LedgerObligationTrailTest.java` | unknown — injects `MessageLedgerEntryRepository`; audit required | ⚠️ behind `-Pwith-llm-examples` |

No new tables or migrations.

---

## What Doesn't Change

- `A2AChannelBackend` — no code changes (tenant flows through `CurrentPrincipal` automatically)
- `A2AResource` — source logic unchanged; Javadoc addition required on `getTask()` (see getTask() tenancy requirement above)
- `ReactiveA2AResource` — source logic unchanged; Javadoc addition required on `getTask()` (see getTask() tenancy requirement above)
- `MessageService`, `JpaMessageStore`, `JpaChannelStore`, `JpaCommitmentStore` — already inject `CurrentPrincipal`; no changes
- `QhorusLedgerEntryRepository` — already has full tenancy; no changes
- Flyway migrations — no new columns; `tenancyId` columns exist from #260
- `@CrossTenant` stores used by Watchdog/ChannelGateway startup — unaffected

---

## Testing

### #265 / #264

- **`A2ATenantScopingTest @QuarkusTest`** — `POST /a2a/message:send` with `X-Tenancy-ID: test-tenant`; assert message's `tenancyId = "test-tenant"` in the store; assert `GET /a2a/tasks/{id}` with same header returns the task; assert `GET /a2a/tasks/{id}` without header returns 404 (tenancy asymmetry).
- **`A2ATenantScopingTest` — no-header case** — `POST /a2a/message:send` without `X-Tenancy-ID`; assert message lands in `DEFAULT_TENANT_ID`.
- **`AgentCardTenantTest @QuarkusTest`** — `GET /.well-known/agent-card.json` with `X-Tenancy-ID: tenant-a`; assert `tenancyId = "tenant-a"` in response. Without header: assert `tenancyId = DEFAULT_TENANT_ID`.
- **`QhorusInboundCurrentPrincipalTest` (CDI-free `@Test`)** — directly instantiate `QhorusInboundCurrentPrincipal` wiring a stub `InboundTenancyContext` that throws `ContextNotActiveException` on `tenancyId()`; call `.tenancyId()` on the principal; assert `DEFAULT_TENANT_ID` is returned and no exception propagates. This is the definitive test of the catch: no `@QuarkusTest`, no CDI container, no HTTP stack — it directly exercises the safety net code path that would otherwise require a background thread race to trigger.
- **`BackgroundDispatchTenancyTest @QuarkusTest`** — invoke `messageService.dispatch()` from a `@Scheduled` method with explicit `tenancyId` set on `MessageDispatch.builder()`; assert the message lands in the correct tenant. This tests that background dispatch with explicit tenancyId is not broken by the new bean. Note: this test does NOT reach `currentPrincipal.tenancyId()` (dispatch short-circuits when `dispatch.tenancyId()` is non-null); it confirms the background path is tenant-correct, not that the catch works.

### #263

- **`io.casehub.qhorus.ledger.MessageLedgerEntryRepositoryTest`** (`runtime/src/test/.../ledger/`) — extend `listEntries`, `findByChannelId`, `findLatestByCorrelationId` tests: call with explicit `tenancyId`; add a cross-tenant isolation case (dispatch two messages in different tenants on the same channel; assert each tenant's query returns only its own entries).
- **`io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepositoryTest`** (`runtime/src/test/.../runtime/ledger/`) — update `findEarliestWithSubjectByCorrelationId` call sites to pass `tenancyId`.
- **`LedgerWriteService` integration test** — dispatch a message; verify the ledger entry's `tenancyId` matches the dispatch's `tenancyId`.
- **`StubMessageLedgerEntryRepository`** — updated signatures compile cleanly.
- **`LedgerQueryRepoTest.CapturingRepo`** — all 7 `@Override` methods updated; tests pass `DEFAULT_TENANT_ID`.
- **`MessageLedgerCaptureTest`** — 26 `findByChannelId` call sites updated + 4 `listEntries` 6-param call sites (lines 318, 331, 344, 356) updated; all pass `DEFAULT_TENANT_ID`.
- **`LedgerCaptureExampleTest`** (`examples/type-system/`) — 2 `findByChannelId` call sites updated; pass `DEFAULT_TENANT_ID`.

---

## Protocol coherence

- Follows `unconditional-tenancy-filtering` (PP-20260520-439daf) — all queries now filter by `tenancyId`.
- Follows `tenancyId-in-data-access-only` (PP-20260520-e6a5f0) — tenancyId is in persistence layer, not domain model.
- Follows `scheduled-service-cross-tenant-stores` (PP-20260609) — `ContextNotActiveException` catch is the safety net; background code should use `@CrossTenant` stores as the primary pattern.
- `QhorusInboundCurrentPrincipal @Alternative @Priority(1)` follows `persistence-backend-cdi-priority` pattern — activates by presence, displaced by higher-priority impl.
- No `@CrossTenant` queries in `MessageLedgerEntryRepository` — all per-tenant (contrast with `CrossTenantLedgerEntryRepository` in casehub-ledger which is an explicit separate interface for tools that need cross-tenant access).

# Design Journal — issue-265-a2a-tenant-scoping

### 2026-06-10 · §Services · §Component Structure · §MCP Tool Surface

Three-issue tenant scoping closes the gap left by #260's entity-level multi-tenancy.

**§Component Structure:** A new `runtime/identity/` sub-package holds the HTTP-layer tenant plumbing: `InboundTenancyContext @RequestScoped` (per-request holder), `TenancyContextFilter @Provider @PreMatching` (reads `X-Tenancy-ID` header), and `QhorusInboundCurrentPrincipal @ApplicationScoped @Alternative @Priority(1)` (implements `CurrentPrincipal`, delegates to the holder). The bean is `@ApplicationScoped` rather than `@RequestScoped` by design: with `@RequestScoped` the CDI proxy throws `ContextNotActiveException` before entering the method body, making the safety-net catch unreachable from background threads. With `@ApplicationScoped`, the catch fires correctly when the delegate (request-scoped holder) is unavailable.

**§Services:** `MessageLedgerEntryRepository` — all 11 query methods gain a `String tenancyId` parameter and `AND e.tenancyId = :tid` JPQL predicate. The two PK-based methods (`findByMessageId`, `findByMessageIds`) are unchanged since surrogate PKs carry no cross-tenant ambiguity. The 6→7-param `listEntries` overload delegates to 9-param, threading `tenancyId` explicitly. `LedgerWriteService.record()` was restructured to resolve `tenancyId` before the first `messageRepo` call, since `findEarliestWithSubjectByCorrelationId` now requires it. The reactive stack (`ReactiveMessageLedgerEntryRepository`, `ReactiveLedgerWriteService`) mirrors the same changes with tenancyId captured before Uni lambdas to avoid request-scope loss on worker threads.

**§MCP Tool Surface:** All ledger query MCP tools (`list_ledger_entries`, `get_obligation_chain`, `get_causal_chain`, `list_stalled_obligations`, `get_obligation_stats`, `get_telemetry_summary`, `get_obligation_activity`) pass `currentPrincipal.tenancyId()` to the repository. `AgentCard` response gains a `tenancyId` field reflecting the current request's tenant — Qhorus-specific extension to the A2A spec, not a security boundary.

Known constraint: `findEarliestWithSubjectByCorrelationId` and `findByCorrelationIdAcrossChannels` are now tenant-scoped. Cross-tenant HANDOFF delegation would silently lose `subjectId` propagation at the tenant boundary. This is the correct tradeoff for current use cases (cross-tenant delegation not yet supported).

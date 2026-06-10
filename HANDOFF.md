# CaseHub Qhorus — Session Handover
**Date:** 2026-06-10 — #265, #264, #263 shipped (HTTP tenant scoping + ledger query isolation)

---

## What Was Done This Session

**#265/#264/#263 — Three-issue tenant scoping, closed and landed:**

- `QhorusInboundCurrentPrincipal @ApplicationScoped @Alternative @Priority(1)`: reads `X-Tenancy-ID` header via `InboundTenancyContext @RequestScoped` + `TenancyContextFilter @PreMatching`. Key design: `@ApplicationScoped` (not `@RequestScoped`) so the `ContextNotActiveException` catch in `tenancyId()` is reachable for background threads — CDI proxy throws before method entry for narrower scopes. See GE-20260610-f1982c, PP-20260610-85e6a4.
- `AgentCard` record: `tenancyId` field reflecting `currentPrincipal.tenancyId()`. Qhorus-specific A2A extension.
- `MessageLedgerEntryRepository`: all 11 query methods (except PK-based `findByMessageId`, `findByMessageIds`) gain `String tenancyId`. All callers updated: `LedgerWriteService`, `ReactiveLedgerWriteService`, `QhorusMcpTools`, `ReactiveQhorusMcpTools`. `LedgerWriteService.record()` now resolves `tenancyId` before first `messageRepo` call.
- Review feedback across 8 iterations caught: `BackgroundContextSafetyTest` not exercising the catch, `CapturingRepo` ignoring tenancyId, `LedgerAttestationIntegrationTest` missing from call-site inventory, `StubMessageLedgerEntryRepository` silently not filtering.
- Protocols: PP-20260610-85e6a4 (`@ApplicationScoped` outer bean requirement), PP-20260610-9487d3 (`X-Tenancy-ID` security boundary documentation).
- Garden: GE-20260610-f1982c (`@ApplicationScoped` vs `@RequestScoped` CDI proxy catch reachability).
- Branch closed: squashed 6→2 commits, pushed to mdproctor/qhorus + casehubio/qhorus.

## Immediate Next Step

Clean main. Run `/work` to pick up the next issue.

## What's Left

None — #265, #264, #263 all closed and shipped. GitHub issues #267 and #268 filed (minor/test followup, not blocking).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| casehub-ledger#126 | Full EVENT telemetry decoupling from message.content | M | Med | Follow-up from #257; not blocking anything |
| casehubio/parent#221 | Sync PLATFORM.md — add HTTP tenant routing to Capability Ownership | XS | Low | Filed this session; peer-repo |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-10-mdp01-the-catch-that-wasnt-reachable.md` |
| Design spec | `docs/specs/2026-06-10-a2a-agentcard-ledger-tenant-scoping-design.md` (project) |
| Protocol: HTTP principal scope | PP-20260610-85e6a4 |
| Protocol: X-Tenancy-ID boundary | PP-20260610-9487d3 |
| Garden: CDI proxy scope gotcha | GE-20260610-f1982c |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

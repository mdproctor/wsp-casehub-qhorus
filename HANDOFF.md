# CaseHub Qhorus — Session Handover
**Date:** 2026-07-05 — #316 closed (API interface taxonomy protocol).

---

## Immediate Next Step

Main is clean. Both remotes at `4ed6d7d`. Issue #316 merged and closed.

Cross-repo follow-up still pending (carried from #314/#315):
- `MessageReceivedEvent` constructor changed (added `Instant occurredAt`) — Claudony (3 sites) and engine (7 sites) need `Instant.now()` as 7th arg
- Store SPI imports moved from `runtime/store/` to `api/store/` — casehub-engine (actor-state), casehub-ops (drift checker), casehub-drafthouse (4 files), casehub-clinical (1 file)
- Parent repo doc sync: casehubio/parent#330 (from #314), casehubio/parent#341 (from #315)

New deferred issues from this session:
- qhorus#320 — ARC42STORIES.MD §5 stale (api module table lists 5 packages, 9 exist)
- parent#348 — Create `module-tier-structure.md` protocol (dangling PLATFORM.md reference)

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Cross-repo Store SPI migration (engine, ops, drafthouse, clinical) | M | Low | Mechanical — update imports from `runtime/store/` to `api/store/` + `MessageReceivedEvent` constructor |
| openclaw#57 | Override deliveryGuarantee → AT_LEAST_ONCE on OpenClawChannelBackend | XS | Low | Propagation from #132 |
| #320 | ARC42STORIES.MD §5 alignment | XS | Low | From design review of #316 |
| parent#348 | module-tier-structure.md protocol | S | Low | Dangling PLATFORM.md reference |

## What Was Done This Session

**API interface taxonomy protocol (#316):** Created `api-interface-taxonomy.md` in garden documenting the four categories of `api/` interfaces (store, SPI, gateway, service facade) with placement rules and decision flowchart. Design-reviewed (5 rounds, 13 issues, 11 verified, 2 accepted). Fixed dangling `consumer-spi-placement.md` reference in PLATFORM.md. Also committed PLATFORM.md update to parent `issue-331-docs-sync-batch` branch.

## References

| What | Path |
|------|------|
| Protocol | `~/.hortora/garden/docs/protocols/casehub/api-interface-taxonomy.md` |
| Design spec | `docs/specs/issue-316-service-facade-protocol/` (promoted to project) |
| PLATFORM.md update | `casehub-parent` on branch `issue-331-docs-sync-batch` |
| Previous session | `git show HEAD~1:HANDOFF.md` |

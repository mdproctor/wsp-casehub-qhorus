# CaseHub Qhorus — Session Handover

**Date:** 2026-07-28 — #375 shipped. Notification bridge migrated to platform subscription engine.

---

## Immediate Next Step

Pick from the cross-repo backlog below — all qhorus-local issues are complete.

## What Was Done

#375: migrated `notification-bridge/` from direct `NotificationStore.store()` to firing `QhorusObligationEvent` POJOs into the platform subscription engine via `DataSourceRegistry.resolveSource()`. Added `QhorusSubscriptionBootstrap` for 5 default SYSTEM-scope subscriptions at startup. Deleted `NotificationCategories`. Also fixed the jetbrains-index-mcp-plugin PR #254→#268 (MCP Kotlin SDK migration broke `CreateModuleTool` — rebased, fixed types, CI green, awaiting maintainer merge).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #358 | Supervisor + friction interventions | L | High | Cross-repo: engine |
| #359 | Summarisation → Qhorus integration | M | Med | Cross-repo: blocks |
| #361 | CBR routing + coordination memory | M | High | Cross-repo: blocks/neocortex |

### Epics

| # | Epic | Children |
|---|------|----------|
| #349 | Coordination Resilience | ~~#353~~, ~~#354~~, ~~#362~~, ~~#363~~, ~~#368~~ — **CLOSED** |
| #350 | Channel Intelligence | ~~#355~~, ~~#357~~ — **CLOSED** |
| #351 | Verification & Trust | ~~#356~~ — **CLOSED** |
| #352 | Cross-Repo Suggestions | #358, #359, ~~#360~~, #361, ~~#364~~ |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*

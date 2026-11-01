# HANDOFF — casehub-qhorus

## Last Session (2026-11-01)

**Branch:** `issue-502-org-governance-phase2` — scaffolded, ready to plan and execute.

Five issues queued, ordered quick-fixes first then M-scale features:

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| 506 | fix: MessageDispatcher.broadcast() throws UnsupportedOperationException | S | Low | Wire or remove the default impl |
| 507 | test: add integration test for ORG_DELEGATION advisory | S | Low | Deferred from #503 code review |
| 502 | feat: protocol inheritance from org units to channels | M | Med | Phase 2 org governance |
| 505 | feat: orgUnitId on ledger entries | M | Med | Phase 2 — compliance reporting per org boundary |
| 504 | feat: unit-level watchdog aggregation | M | Med | Phase 2 — alert per org unit not per agent |

**To resume:** `work continue` → brainstorm/plan for the batch, then execute.

**Key context:**
- `OrgAwareDispatch` API (in casehub-eidos-api) has: `escalate()`, `canDelegateTo()`, `teamMembers()`, `orgDistance()` — all already used in RoutingBridge
- Phase 1 landed on main: routing preference (#494), escalation (#500), redistribution verification (#501), delegation authorisation (#503)
- #502 will need a new SPI or extension to `OrgAwareDispatch` for protocol lookup by org unit
- #505 needs a new column on `MessageLedgerEntry` (next Flyway migration V2005) + stamping in `LedgerWriteService.record()`
- #504 needs `WatchdogEvaluationService` to group evaluations by org unit

## Previous Session (2026-10-31)

Landed Phase 1 org governance batch on `issue-500-org-governance-phase1-3`:
- #500: org-aware escalation routing in RoutingBridge
- #501: verified redistribution inherits org-awareness (no code changes)
- #503: delegation authorisation for HANDOFF messages
- Refactored: extracted `tryOrgEscalation()` helper, fixed FQN import
- Blog: revised "Governance Follows the Org Chart" with escalation SVG
- Filed #507 (deferred integration test)
- Closed #500, #501, #503 on GitHub

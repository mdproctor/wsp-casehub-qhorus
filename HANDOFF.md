# CaseHub Qhorus — Session Handover

**Date:** 2026-08-22 — #398 and #399 complete, #400 next in queue, #407 (tutorial example) added.

---

## Immediate Next Step

Run `work next` to advance from #399 to #400 (Active governance policies — enforcement modes). After #400 lands, implement #407 (tutorial example covering all three Phase 1 epics + text rendering of causal graphs). The tutorial should include #400's enforcement scenarios once that epic's implementation shape is known.

## What Was Done

### E1: Cross-channel causal graphs (#398) — CLOSED

Built the query layer over existing ledger data for cross-channel attribution:
- `CausalGraphService` with `CausalGraph`/`GraphNode`/`GraphEdge` DTOs — graph building algorithm with BFS depth, outcome precedence (FAILED > DECLINED > FULFILLED), truncation detection
- `get_causal_chain` MCP tool — `channel` now optional; omitting crosses channel boundaries via `findAncestorChainCrossChannel`; `CausalChainEntry` gained `channelId`, `channelName`, `content` fields
- `get_causal_graph(correlation_id)` — new MCP tool returning structured nodes+edges graph
- `CausalGraphResource` — REST at `/api/causal-graph/{correlationId}` and `/api/causal-graph/attribution/{entryId}`
- 50-hop depth limit on both single-channel and cross-channel ancestor walks
- 42 new tests (repo + service + MCP + REST)

Light design review (3 dimensions): 17 findings accepted, 2 rejected (visual rendering out of scope, ACL is write-only).

### E2: Cascade containment (#399) — implementation complete, issue not yet closed

Connected existing watchdog detection to existing containment primitives:
- `WatchdogAction` enum (ALERT, PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE) — persisted per-watchdog, V44 migration
- `AlertContext.affectedAgentIds()` default method on sealed interface — 4 overrides for agent-specific contexts
- `executeContainmentAction()` in `WatchdogEvaluationService.fireAlert()` — PAUSE_CHANNEL pauses + expires commitments, DEREGISTER_AGENT marks instance offline, QUARANTINE does both + containment EVENT
- Safety guards: error isolation (try-catch), notification channel excluded, null channelId handled
- `CommitmentService.expireByChannel()` + `InstanceService.markOffline()` — new containment primitives
- `register_watchdog` MCP tool gained `action` parameter
- 15 new tests

Light design review (3 dimensions): 11 findings accepted, 2 rejected (separate ContainmentService premature, recovery path out of scope). Key accepted finding: DEREGISTER_AGENT uses instance registry (markOffline), not gateway backend deregistration.

### Infrastructure fix: .plan state corruption

Fixed a bug where `.plan` files created outside `scaffold.py` had no `state:` field:
- `plan_manager.py build_plan_content` — ensures `state: active` when writing `## State` section
- `scaffold.py` — self-healing: patches `state: active` when existing `.plan` has `## State` but no `state:` key

## Queue

| # | Title | Status |
|---|-------|--------|
| #398 | E1: Cross-channel causal graphs | CLOSED |
| #399 | E2: Cascade containment | Done, not yet closed |
| #400 | E3: Active governance policies | Next |
| #407 | Tutorial example — governance in action | After #400 |

## What's Next

| Item | Scale | Complexity | Notes |
|------|-------|-----------|-------|
| #400 Active governance policies | S | Low | Channel enforcement mode (ADVISORY/BLOCKING/QUARANTINE); protocol violations become rejections or trigger containment |
| #407 Tutorial example | S | Low | `CausalGraphRenderer` text output + `examples/governance-tutorial/` module covering #398, #399, #400 scenarios; update after #400 lands to include enforcement scenarios |

## Cross-Module

None — Phase 1 epics are entirely qhorus-internal, zero cross-repo dependencies.

## References

- Specs: `specs/issue-398-roadmap-phase1/2026-08-21-causal-graphs-design.md`, `specs/issue-398-roadmap-phase1/2026-08-22-cascade-containment-design.md`
- Plans: `plans/2026-08-21-causal-graphs.md`, `plans/2026-08-22-cascade-containment.md`
- Roadmap: `docs/roadmap-epics-2026.md`

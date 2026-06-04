# CaseHub Qhorus — Session Handover
**Date:** 2026-06-04 — batch S/XS fixes shipped (#248, #244, #240, #239, #238, parent#163)

---

## What Was Done This Session

**Shipped 6 S/XS issues on `issue-248-batch-s-xs`:**

- **qhorus#248** — `FindOrCreateResult(channel, wasCreated)` return type on `ChannelService.findOrCreateWithBinding()`; counter in `ConnectorChannelBackend.tryAutoCreate()` only fires when `wasCreated == true`. Fixes concurrent first-contact counter double-increment.
- **qhorus#244** — `set_channel_type_constraints(channel, allowed_types?, denied_types?)` MCP tool; full-replacement semantics; validation via `MessageType.parseTypes()` + overlap check; `@Blocking` on reactive path (calls `resolveChannel()`).
- **qhorus#240** — `list_projections()` MCP tool exposes `ProjectionRegistry.registeredNames()` (sorted).
- **qhorus#239** — `project_channel` gains optional `max_messages` param; folds first N messages in insertion order (not most-recent).
- **qhorus#238** — Protocol doc `PP-20260604-dualid` (qhorus channel dual identity) written to parent/docs/protocols/casehub/.
- **parent#163** — `docs/repos/casehub-qhorus.md` oversight channel corrected: `deniedTypes=EVENT` not `allowedTypes=COMMAND,RESPONSE`.

Also: garden entry GE-20260604-96d82a (`@Blocking` gotcha on `resolveChannel()`) and protocol PP-20260604-995096 (reactive MCP tool @Blocking rule).

## Immediate Next Step

Start `qhorus#244`-related work for Claudony (claudony#142 oversight channel fix may now unblock once CI publishes the 0.2-SNAPSHOT).

## Cross-Module

**We're blocking (less urgent now):**
- `claudony` — waiting on CI to publish `0.2-SNAPSHOT` to GitHub Packages (claudony#142). Also needs `set_channel_type_constraints` to update oversight channel config.

## What's Left

- **qhorus#236** — slug enforcement on channel names · M · Low
- **qhorus#237** — MCP tool migration from channel_name to UUID-or-slug · L · Low
- **qhorus#238** — closed (protocol written this session)
- **qhorus#239** — closed (max_messages shipped)
- **qhorus#240** — closed (list_projections shipped)
- **qhorus#244** — closed (set_channel_type_constraints shipped)
- **qhorus#248** — closed (FindOrCreateResult shipped)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| qhorus#236 | Slug enforcement on channel names | M | Low | V17 migration + ChannelService validation |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-04-mdp02-finding-and-blocking.md` |
| Batch spec | `docs/specs/2026-06-04-batch-sx-fixes-design.md` (project) |
| Garden: @Blocking gotcha | GE-20260604-96d82a |
| Protocol: reactive @Blocking rule | PP-20260604-995096 |
| Protocol: channel dual identity | PP-20260604-dualid |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

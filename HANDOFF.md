# CaseHub Qhorus — Session Handover

**Date:** 2026-07-12 — Branch `issue-339-conversation-followups` closed. #339, #336, #335, #340, #333 implemented. #341 filed.

---

## Immediate Next Step

All conversation model followups done. #333 (Presence) and #334 (Space) were the remaining #328 children — #333 is now closed. #334 (Space — channel hierarchy) is the last major piece. Start with `/work` when ready.

## What Was Done

Five issues on one branch: `get_reactions_batch` MCP tool (#339), `mergeTopics` (#336), `moveTopic` with commitment gate (#335), `ArtefactType.DEBATE` (#340), `Presence` with Caffeine cache and heartbeat degradation (#333). Design review: 5 rounds, 22 issues, 16 verified. Full build green across 13 modules. 7 commits squashed to 5.

## Cross-Module

**We're blocking:**
- `connectors` — needs topic merge/move + presence + artefactRef + membership APIs (connectors#65, #68, #80)
- `engine` — needs topic field for choreography context (engine#701)
- `blocks` — needs all for end-to-end integration (blocks#49)

**Cross-repo follow-ups still open:**
- claudony#169 — OutboundMessage correlationId UUID→String
- drafthouse#102 — redundant `.toString()` on correlationId

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #334 | Space — channel hierarchy | L | High | Last #328 child |
| #335 | moveTopic | — | — | CLOSED this session |
| #337 | Topic-aware digest | M | Med | |
| #338 | Topic-aware projections | M | Med | |
| #341 | Escape topic/channel names in telemetry JSON | XS | Low | Filed this session |

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/issue-339-conversation-followups/2026-07-12-conversation-followups-design.md` |
| Blog | `blog/2026-07-12-mdp02-five-followups-one-branch.md` |
| Design review | `~/adr/casehub-qhorus/conversation-followups-20260712-151919/tracker.md` |

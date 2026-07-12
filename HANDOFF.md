# CaseHub Qhorus — Session Handover

**Date:** 2026-07-12 — Epic #328 sessions 1+2 complete. Branch closed. #329-#332 implemented, #333-#334 remain.

---

## Immediate Next Step

Branch `issue-328-conversation-model-enrichments` is closed. #333 (Presence) and #334 (Space) are open — start a new branch via `/work` when ready.

## What Was Done

**Session 1 (#329 Topic + #330 Reactions):** Hybrid Topic entity with denormalized string on Message. Reactions as non-normative UI metadata with CDI event.

**Session 2 (#331 Rich ArtefactRef + #332 Channel Membership):** Replaced `List<UUID>` artefactRefs with structured `ArtefactRef(uri, type, label, scope)` across all 8 pipeline types. Added `ChannelMembership` with roles, ID-based unread tracking, lazy auto-membership on first human interaction.

Adversarial design review passed (4 rounds, 18 issues, all verified). Full build green across 13 modules. 19 commits squashed to 4.

## Cross-Module

**We're blocking:**
- `connectors` — needs topic + reaction + artefactRef + membership APIs (connectors#65, #68, #80)
- `engine` — needs topic field for choreography context (engine#701)
- `blocks` — needs all for end-to-end integration (blocks#49)

**Cross-repo follow-ups still open:**
- claudony#169 — OutboundMessage correlationId UUID→String
- drafthouse#102 — redundant `.toString()` on correlationId

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #333 | Presence — availability with heartbeat | M | High | Depends on #332 |
| #334 | Space — channel hierarchy | L | High | Most complex |
| #335 | moveTopic | S | Med | Deferred |
| #336 | mergeTopics | S | Med | Deferred |
| #337 | Topic-aware digest | M | Med | |
| #338 | Topic-aware projections | M | Med | |
| #339 | get_reactions_batch | S | Low | |

## References

| What | Path |
|------|------|
| Session 1 spec | `docs/specs/issue-328-conversation-model-enrichments/2026-07-10-topic-reactions-design.md` |
| Session 2 spec | `docs/specs/issue-328-conversation-model-enrichments/2026-07-11-artefactref-membership-design.md` |
| Blog | `blog/2026-07-10-topics-and-reactions.md` |

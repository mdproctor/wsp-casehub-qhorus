# CaseHub Qhorus — Session Handover

**Date:** 2026-07-12 — Branch `issue-334-space-channel-hierarchy` closed. #334 implemented and pushed.

---

## Immediate Next Step

#334 (Space) is done. The last #328 child is closed. Pick from What's Next — #337 (topic-aware digest) and #338 (topic-aware projections) are the natural next pair. Start with `/work` when ready.

## What Was Done

Space — recursive organizational channel hierarchy. Space entity with adjacency-list nesting (parentSpaceId, MAX_DEPTH=10). SpaceService with create, findByName (ambiguity resolution), rename, updateDescription, moveSpace (cycle detection), moveChannelToSpace (tenancy enforcement). 9 MCP tools with resolveSpace dual-identity. ChannelDetail.spaceName batch enrichment. V33/V34 migrations. 30 store contract tests + 32 service tests. Design review: 5 rounds, 16 issues, 13 verified. Full build green across 13 modules. 16 commits squashed to 1.

## Cross-Module

**We're blocking:**
- `connectors` — needs Space API for space-aware channel grouping (connectors#67)
- `engine` — needs Space for normative channel layout integration (case → space mapping)
- `blocks` — needs all for end-to-end integration (blocks#49)

**Cross-repo follow-ups still open:**
- claudony#169 — OutboundMessage correlationId UUID→String
- drafthouse#102 — redundant `.toString()` on correlationId

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #337 | Topic-aware digest | M | Med | |
| #338 | Topic-aware projections | M | Med | |
| #341 | Escape topic/channel names in telemetry JSON | XS | Low | Filed last session |

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/issue-334-space-channel-hierarchy/2026-07-12-space-channel-hierarchy-design.md` |
| Blog | `blog/2026-07-12-mdp03-spaces-replace-naming-conventions.md` |
| Design review | `~/adr/casehub-qhorus/space-channel-hierarchy-20260712-211404/tracker.md` |

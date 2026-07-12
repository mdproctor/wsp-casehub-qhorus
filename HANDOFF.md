# CaseHub Qhorus — Session Handover

**Date:** 2026-07-10 — Epic #328 session 1 (Topic #329 + Reactions #330) implemented. Branch open.

---

## Immediate Next Step

Branch `issue-328-conversation-model-enrichments` is open for #328. Run `/work` to resume.

Session 2 scope: #331 (Rich ArtefactRef) + #332 (Channel Membership). Design spec covers sessions 1–3. Read `specs/issue-328-conversation-model-enrichments/2026-07-10-topic-reactions-design.md` for the full context — the Future Directions and Cross-Repo Integration sections are relevant for sessions 2–3.

## What Was Done This Session

**Topic (#329) + Reactions (#330) fully implemented.** Hybrid Topic entity + denormalized `topic` string on Message (avoiding Zulip's architectural regret). Topic recorded immutably on `MessageLedgerEntry` (CQRS pattern). Reactions as non-normative UI metadata with own CDI event. 8 implementation commits, full test suite green across 13 modules. Flyway V28–V30 + V2001. CLAUDE.md updated.

Design decisions documented in `design/JOURNAL.md` (§Session-1). Cross-repo integration issues filed: engine#701, connectors#80, blocks#49. Deferred items: #335 (moveTopic), #336 (mergeTopics), #337 (topic digest), #338 (topic projections), #339 (reactions batch MCP).

Two garden entries: GE-20260711-bf1d9a (Flyway V-ordering gotcha), GE-20260711-40e102 (bulk API migration technique).

## Cross-Module

**We're blocking:**
- `connectors` — needs qhorus topic + reaction API for chat UI (connectors#80) · M · Med
- `engine` — needs qhorus topic field for choreography context (engine#701) · S · Low
- `blocks` — needs all three for end-to-end integration (blocks#49) · L · High

**Cross-repo follow-ups still open:**
- claudony#169 — OutboundMessage correlationId UUID→String · XS · Low
- drafthouse#102 — redundant `.toString()` on correlationId · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #331 | Rich ArtefactRef — typed, anchored document references | M | Med | Session 2 — restructures existing `List<UUID>` |
| #332 | Channel Membership — who's in the channel, distinct from ACL | M | Med | Session 2 — foundation for #333 |
| #333 | Presence — availability status with heartbeat degradation | M | High | Session 3 — depends on #332 |
| #334 | Space — organizational channel hierarchy | L | High | Session 3 — most complex piece |

## References

| What | Path |
|------|------|
| Design spec | `specs/issue-328-conversation-model-enrichments/2026-07-10-topic-reactions-design.md` |
| Implementation plan | `plans/2026-07-10-topic-reactions.md` |
| Design journal | `design/JOURNAL.md` |
| Blog entry | `blog/2026-07-10-topics-and-reactions.md` |
| Previous session | `git show HEAD~1:HANDOFF.md` |

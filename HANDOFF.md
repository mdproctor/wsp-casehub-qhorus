# CaseHub Qhorus — Session Handover
**Date:** 2026-06-28 — #309, #308 lifecycle + credential migration; #287 redirected to ops

---

## Immediate Next Step

Main is clean. Both remotes at `dc946b6`. Three issues closed (#309, #308, #287→ops#14).

Cross-repo follow-up still pending from last session: `MessageReceivedEvent` constructor changed (added `Instant occurredAt`). Claudony (3 test sites) and engine (7 test sites) will fail at next compile — mechanical fix (`Instant.now()` as 7th arg).

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #310 | Sweep `!isTerminal()` → `isActive()` + resolveToken direct test | XS | Low | Follow-up from code review |
| #169 | Extract persistence-memory/ module from testing/ | M | Low | Standalone |
| ops#14 | Enrich ChannelDriftChecker — full field comparison, tenancy fix | S | Low | Cross-repo (casehub-ops) |

## What Was Done This Session

**CommitmentState isActive (#309):** Added `isActive()` with explicit enumeration of OPEN and ACKNOWLEDGED. `CommitmentStateTest` verifies exhaustiveness invariant (every state classified by exactly one method). LIFECYCLE.md updated in parent repo.

**Slack credential migration (#308):** Replaced `Config.getValue("casehub.qhorus.slack-channel.credentials.<id>")` with `CredentialResolver.resolve(workspaceId).get(BEARER_TOKEN)`. Error contract preserved — `resolveToken()` now throws explicitly since CredentialResolver never throws. Config namespace: `casehub.credentials.<id>`.

**#287 redirected:** Original bridge module proposal was a layering violation (Foundation→Integration upward coupling). Closed qhorus#287, filed casehub-ops#14 with 6 requirements including tenancy gap fix and CSV set comparison.

**Code review:** 0 Critical/Important. 3 Minor noted → filed #310.

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-06-27-xs-s-fixes-design.md` |
| Previous session | `git show HEAD~1:HANDOFF.md` |

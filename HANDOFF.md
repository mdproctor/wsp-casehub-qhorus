# CaseHub Qhorus — Session Handover
**Date:** 2026-06-29 — #310 closed (isActive sweep + resolveToken tests)

---

## Immediate Next Step

Main is clean. Both remotes at `9e35142`. #310 closed — code review follow-up from last session complete.

Cross-repo follow-up still pending: `MessageReceivedEvent` constructor changed (added `Instant occurredAt`). Claudony (3 test sites) and engine (7 test sites) will fail at next compile — mechanical fix (`Instant.now()` as 7th arg).

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #169 | Extract persistence-memory/ module from testing/ | M | Low | Standalone |
| ops#14 | Enrich ChannelDriftChecker — full field comparison, tenancy fix | S | Low | Cross-repo (casehub-ops) |

## What Was Done This Session

**isActive() sweep (#310):** Replaced 11 `!isTerminal()` call sites with `isActive()` across CommitmentService, JpaCommitmentStore, ReactiveJpaCommitmentStore, InMemoryCommitmentStore, InMemoryCrossTenantCommitmentStore. Added 2 direct unit tests for `SlackChannelBackend.resolveToken()` error paths.

## References

| What | Path |
|------|------|
| Previous session | `git show HEAD~1:HANDOFF.md` |

# CaseHub Qhorus — Session Handover
**Date:** 2026-06-17 — Three cross-repo issues shipped (#282, #280, #281 closed)

*Updated: parent#265, parent#268 closed — removed from backlog.*

---

## Immediate Next Step

Run `/work` to start #261 (casehub-qhorus-slack-channel module). Main is clean, both repos aligned.

## What Was Done This Session

**Three issues shipped on branch `issue-282-fix-reactive-jpa-channel-jpql`:**

**#282 / claudony#155 — ReactiveJpaChannelStore JPQL fix:**
`repo.update(query, Object...)` in Hibernate Reactive converts positional `?N` params to named
params derived from adjacent field names at runtime. `?3` near `tenancyId` became `:tenancyId`
internally but was never bound → `QueryParameterException`. Fixed with `Parameters.with()` named
params. Added `@WithTransaction` (was missing). Unblocks Claudony's 6 failing
`MeshResourceInterjectionTest` tests. GE-20260617-54b75b captured in garden.

**#280 — MessageLedgerEntryTestFactory moved to casehub-qhorus-testing:**
Factory now in `io.casehub.qhorus.testing` — accessible to Claudony, devtown, any consumer of
`casehub-qhorus-testing`. Runtime module can't depend on testing (build cycle), so runtime tests
carry a local `buildEntry()` helper instead. Unblocks claudony#94.

**#281 — CommitmentExpiredEvent CDI event:**
New record in `api/`, fired by `CommitmentService.expireOverdue()` after all saves complete.
Two-phase design (save then fire) with per-event try-catch prevents observer exceptions from
rolling back the expiry batch. Includes `expiresAt` for stall-duration computation.
Unblocks engine#504 (OutcomePolicy.onExpired) and devtown#14.

Key technique: CDI batch events should fire after the save loop completes, with per-event
try-catch, so observer failures don't corrupt the transaction.

## What's Left

*(nothing outstanding)*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #261 | casehub-qhorus-slack-channel module | L | Med | Next up; needs brainstorm |

## References

| What | Path |
|------|------|
| Garden entry | `GE-20260617-54b75b` — Hibernate Reactive Panache positional param bug |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

# CaseHub Qhorus — Session Handover
**Date:** 2026-05-29 — issue-212 XS/S batch closed

---

## What Was Done This Session

**#212 batch complete.** All 5 issues closed and pushed to upstream (casehubio/qhorus):
- #211 (XS) — null guard for `lastActivityAt` in watchdog evaluation
- #210 (S) — BARRIER_STUCK + CHANNEL_IDLE test coverage
- #204 (S) — `registerBackend()` dedup guard (synchronized block extended to all types)
- #150 (S) — A2A: `ErrorResponse` record (M1), max-priority `fromMessageHistory` (M2)
- #199 (S) — `TrustGateService` wired into `dispatch()` — `casehub.qhorus.commitment.min-obligor-trust`

Key discoveries: `casehub.ledger.datasource=qhorus` required in `QuarkusTestProfile.getConfigOverrides()` on context restart (now PP-20260529-6047d2 + garden REVISE GE-20260424-6b88a0). `distinctSendersByChannel` excludes the passed type — now in CLAUDE.md.

Filed **#213** — `ObligorTrustPolicy` SPI to replace the colon-heuristic target guard in the trust gate.

## Immediate Next Step

Audit the 4 stale open epic branches (10–11 days, no closure marker):
- `epic-142-flyway-versioning`
- `epic-153-cdi-message-event`
- `epic-154-inbound-correlationid`
- `epic-a2a-lifecycle-cleanup`

Close with `work-end` or confirm still active. Then pick up #213 or #132.

## What's Left

- **claudony#135** — `deadline` + `correlationId` as first-class params to `postToChannel()` SPI · S · Low _(Claudony work; qhorus ready)_
- **#193** — ReactiveMessageService enforcement parity · M · Med _(deferred, @Disabled)_
- **#203** — Qhorus dispatch to DraftHouse on successful publish · ? · ?

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Audit 4 stale epic branches | XS | Low | Do first |
| #213 | ObligorTrustPolicy SPI — replace colon heuristic | M | Med | Follow-on from #199 |
| #132 | Delivery guarantees (retry + dead-letter) | L | High | Main feature item |
| #193 | ReactiveMessageService enforcement parity | M | Med | Needs reactive enforcement beans |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-29-mdp03-five-fixes-three-assumptions.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

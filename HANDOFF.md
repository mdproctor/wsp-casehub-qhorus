# CaseHub Qhorus — Session Handover
**Date:** 2026-05-29 — qhorus#207 #208 #209 watchdog store follow-ups

---

## What Was Done This Session

**#207 + #208 + #209** completed and closed. Routed `w.lastFiredAt` through `WatchdogStore.put(w)` (store-seam compliance), added Javadoc to `ReactiveMessageStore.count()` and improved `InMemoryMessageStore.count()` comment, added negative tests + debounce test + ID capture fixes to `WatchdogEvaluationServiceTest`. 2 squashed commits on upstream main. Also filed #210 (BARRIER_STUCK/CHANNEL_IDLE zero coverage) and #211 (NPE on `ch.lastActivityAt` in `evaluateBarrierStuck()`).

**Garden:** GE-20260529-9f3557 — `@TestTransaction` + JPA auto-flush makes store-seam compliance tests pass before the fix is applied.

**CLAUDE.md:** documented unconditional pre-push hook (`.githooks/pre-push`) — blocks every push post-squash, requires `--no-verify` after git-squash approval.

## Immediate Next Step

Address the 4 stale open epic branches (10–11 days, no EPIC-CLOSED.md):
- `epic-142-flyway-versioning`
- `epic-153-cdi-message-event`
- `epic-154-inbound-correlationid`
- `epic-a2a-lifecycle-cleanup`

Either close them (work-end) or confirm they're still active. Then pick up #132 (delivery guarantees).

## What's Left

- **claudony#135** — add `deadline` + `correlationId` as first-class params to `postToChannel()` SPI · S · Low _(Claudony work; qhorus ready)_
- **#193** — ReactiveMessageService full enforcement parity · M · Med _(deferred — service @Disabled)_
- **#203** — Qhorus dispatch to DraftHouse on successful publish · ? · ?
- **#210** — BARRIER_STUCK / CHANNEL_IDLE zero test coverage in WatchdogEvaluationServiceTest · S · Low
- **#211** — `evaluateBarrierStuck()` NPE on `ch.lastActivityAt` when channel created with no messages · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Audit 4 stale epic branches | XS | Low | Close or confirm active before new work |
| #210+#211 | Watchdog BARRIER_STUCK/CHANNEL_IDLE coverage + NPE fix | XS–S | Low | Small batch; natural follow-on to this session |
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| #193 | ReactiveMessageService enforcement parity | M | Med | Deferred; needs reactive enforcement beans |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-29-mdp02-the-test-that-passed-too-soon.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

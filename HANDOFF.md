# CaseHub Qhorus — Session Handover
**Date:** 2026-05-29 — qhorus#205 store seam + qhorus#206 router comment

---

## What Was Done This Session

**qhorus#205 + #206** completed and closed. Fixed three Panache static-call bypasses in `WatchdogEvaluationService` (CommitmentStore, InstanceStore, MessageStore.count), extracted `MessageQueryJpql` to eliminate scan/count predicate duplication across JPA stores, added `MessageStore.count(MessageQuery)` to all store implementations with contract tests. Documented `ConfiguredWatchdogAlertRouter.route()` v1 fan-out semantics. 1,384 tests, 0 failures. Subagent-driven, full spec+review cycle. Pushed squashed (14→9 commits) to fork and upstream.

**Filed during review:** #207 (`w.lastFiredAt` WatchdogStore bypass — out of scope this branch), #208 (ReactiveMessageStore.count() Javadoc + InMemoryMessageStore comment wording), #209 (WatchdogEvaluationServiceTest minor improvements).

## Immediate Next Step

Batch **#207 + #208 + #209** on one branch — all XS/S, all watchdog/store follow-ups from this session. Run `work-start`.

## What's Left

- **claudony#135** — add `deadline` + `correlationId` as first-class params to `postToChannel()` SPI · S · Low _(Claudony work; qhorus ready)_
- **#193** — ReactiveMessageService full enforcement parity · M · Med _(deferred — service @Disabled)_
- **#203** — Qhorus dispatch to DraftHouse on successful publish (cross-repo gap) · ? · ?
- **#207** — `w.lastFiredAt = now` in `evaluateAll()` bypasses `WatchdogStore` — should go through `watchdogStore.put(w)` · XS · Low
- **#208** — `ReactiveMessageStore.count()` Javadoc missing; `InMemoryMessageStore.count()` comment wording · XS · Low
- **#209** — WatchdogEvaluationServiceTest: explicit ID capture, negative cases, richer assertion messages · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #207+#208+#209 | Watchdog follow-ups (batch) | XS–S | Low | Roll together on one branch |
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| #193 | ReactiveMessageService enforcement parity | M | Med | Deferred; needs reactive enforcement beans |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-29-mdp01-the-seam-that-was-half-closed.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

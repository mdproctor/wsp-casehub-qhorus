# CaseHub Qhorus — Session Handover
**Date:** 2026-05-28 — qhorus#200 WatchdogAlertEvent + ConnectorAlertBridge

---

## What Was Done This Session

**qhorus#200** completed end-to-end. Designed (3 spec review rounds — sealed AlertContext over Map, WatchdogAlertRouter SPI with @DefaultBean, fireAsync-before-dispatch ordering), implemented with TDD subagents (1207 tests, 0 failures), work-end'd. New module: `casehub-qhorus-connectors`. ADR-0010 (sealed AlertContext), ADR-0011 (WatchdogAlertRouter SPI). Blog entry written. Garden entry GE-20260527-cad5ba, protocol PP-20260528-6b1d80 captured.

**qhorus#201** (`QhorusDashboardService` canonical types) was previously merged; now pushed to origin/main as part of this session's squash push.

**Filed during review:** #205 (evaluateApprovalPending/evaluateAgentStale bypass store seam), #206 (ConfiguredWatchdogAlertRouter.route() comment gap).

## Immediate Next Step

Start **#205 + #206** on one branch — small, low-complexity follow-ups from #200. Run `work-start` with "do qhorus#205 and qhorus#206".

## What's Left

- **claudony#135** — add `deadline` + `correlationId` as first-class params to `postToChannel()` SPI · S · Low _(Claudony work; qhorus ready)_
- **#193** — ReactiveMessageService full enforcement parity (ACL, rate limit, type policy, LAST_WRITE, ledger, fanOut) · M · Med _(deferred — service @Disabled)_
- **#198** — Two deferred review items: double channel-read in `sendHumanMessage()`, reactive deadline test · S · Low
- **#203** — Qhorus dispatch to DraftHouse on successful publish (cross-repo gap) · ? · ?
- **#205** — evaluateApprovalPending + evaluateAgentStale bypass store seam, use direct Panache calls · M · Low
- **#206** — ConfiguredWatchdogAlertRouter.route() comment: explain fan-out behaviour and SPI override mechanism · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #205+#206 | Store seam fix + router comment (batch) | S | Low | Roll together on one branch |
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| #193 | ReactiveMessageService enforcement parity | M | Med | Deferred; needs reactive enforcement beans |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-28-mdp01-teaching-watchdog-to-reach-outside.md` |
| Spec (#200) | `specs/2026-05-27-watchdog-alert-event-design.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

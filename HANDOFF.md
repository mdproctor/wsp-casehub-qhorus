# CaseHub Qhorus — Session Handover
**Date:** 2026-06-08 — #257 #258 #259 shipped (quarkmind Layer 3 integration bugs)

---

## What Was Done This Session

Shipped three bugs from quarkmind Layer 3 integration on branch `issue-257-event-observer-fixes`:

**#257 — MessageDispatch.telemetry field + EVENT content fail-fast:**
- Root cause: EVENT content silently nulled by `MessageObserverDispatcher`; real fix required decoupling ledger telemetry from `message.content`. Partial fix: add `String telemetry` (13th field) to `MessageDispatch`; `Builder.build()` throws on EVENT+content; `LedgerWriteService.populateTelemetry()` reads `dispatch.telemetry()` not `dispatch.content()`. Follow-up: `casehub-ledger#126` for full decoupling.
- 57 files changed (all production callers, test callers, examples migrated); `QhorusEntityMapper.toTimelineEntry()` gained a ledger-first overload (N+1 tracking: #262).
- Protocol: PP-20260608-054090 (EVENT content-free rule).

**#258 — ChannelSlugValidator dot-notation error:**
- `segment.contains(".")` branch produces actionable error with hyphen and slash suggestions.

**#259 — @Any on Instance<MessageObserver>:**
- Added `@Any` to both `MessageService` and `ReactiveMessageService` — qualified observer beans were silently excluded.
- Protocol: PP-20260608-07daa6 (observer test transaction discipline).
- Garden: GE-20260608-038af4 (@TestTransaction silently skips observers; use `QuarkusTransaction.requiringNew()`).

4 stale workspace branches stamped `chore: branch closed`. All 3 issues closed. CLAUDE.md updated.

## Immediate Next Step

On main, clean. Run `/work` to pick up next issue — natural start is **#256** (move sequence assignment to `LedgerSequenceAllocator`, prereq for #255).

## What's Left

- **#255** — Use `JpaLedgerEntryRepository` from casehub-ledger directly (blocked by #256) · S · Low
- **#256** — Move sequence assignment to `LedgerSequenceAllocator` · M · Med
- **#262** — Batch `findByMessageIds()` for `getChannelTimeline()` N+1 fix · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #256 | Move sequence assignment to LedgerSequenceAllocator | M | Med | Prereq for #255; casehub-ledger coordination |
| #255 | Use JpaLedgerEntryRepository from casehub-ledger — drop LedgerEntryJpaRepository | S | Low | Blocked by #256 |
| #262 | Batch findByMessageIds() to fix N+1 in getChannelTimeline() | S | Low | Performance, not blocking |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-08-mdp01-the-guard-that-revealed-the-callers.md` |
| Design spec | `docs/specs/2026-06-07-observer-fixes-257-258-259.md` (project) |
| Protocol: EVENT content-free | PP-20260608-054090 |
| Protocol: observer test discipline | PP-20260608-07daa6 |
| Garden: @TestTransaction observer | GE-20260608-038af4 |
| Ledger follow-up | casehub-ledger#126 |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

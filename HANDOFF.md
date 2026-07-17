# CaseHub Qhorus — Session Handover

**Date:** 2026-07-17 — #353, #362, #363 implemented (correlation integrity, QUERY tracking, context telemetry). PR #367 open on casehubio/qhorus.

---

## Immediate Next Step

Pick from the backlog below. #354 (coordination pathology watchdog, M/Med) is the recommended next pick — it builds on the correlation integrity signals just shipped.

## What Was Done

**Implemented three S-scale issues on one branch.** Root finding: #362's premise was wrong — QUERYs already create Commitment records. The real gap was type-blind obligation resolution, which unified #353 and #362 into one design.

- **#353 CorrelationIntegrityChecker:** New `@ApplicationScoped` bean in the dispatch gate. Four advisory checks: inReplyTo validation, resolution type matching (RESPONSE↔COMMAND, DONE↔QUERY), obligor identity, cross-channel resolution. All advisory — WARN + `DispatchResult.advisories()`.
- **#362 Default QUERY deadline:** `casehub.qhorus.commitment.default-query-deadline` config. Applied to Commitment.expiresAt only (Message.deadline stays null). Existing `expireOverdue()` + notification bridge handle the rest.
- **#363 Context window telemetry:** `contextWindowPct` nullable SMALLINT on `MessageLedgerEntry` (V2002). `CONTEXT_PRESSURE` watchdog condition. `ContextPressureContext` added to sealed `AlertContext` hierarchy.

PR #367 open on casehubio/qhorus. 2 squashed commits, 1759 tests pass.

## Backlog — Prioritised by Effort-to-Impact

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-17-correlation-query-telemetry-design.md` |
| Blog entry | `blog/2026-07-17-mdp02-obligation-model-was-already-there.md` |
| PR | `casehubio/qhorus/pull/367` |
| Previous references | *Unchanged — `git show HEAD~1:HANDOFF.md`* |

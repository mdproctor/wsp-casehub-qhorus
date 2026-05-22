# CaseHub Qhorus — Session Handover
**Date:** 2026-05-22 — #126/#184 + branch close + work-end skill fix

---

## What Was Done This Session

Shipped qhorus#184 (`MessageDispatch` builder API): replaced the 9-param positional `send()` with `dispatch(MessageDispatch)`, returning `DispatchResult` with `ledgerEntryId`, `subjectId`, `causedByEntryId`. Every `MessageLedgerEntry` now records the domain aggregate as `subjectId` (not the channel). Two-priority propagation: `subjectId` from correlation root, `causedByEntryId` from `inReplyTo`. Builder enforces speech-act protocol at `build()` — 62 latent protocol violations surfaced and fixed across the codebase. Full build: 1149 tests, 0 failures. Design validated by AML team across 3 rounds. ADR-0009 written.

claudony#126 done and on claudony main: removed stale `quarkus.arc.selected-alternatives` block.

Branch retention policy formalised: 14-day minimum hold, always prompt, never auto-delete silently.

Branch `issue-126-184-dispatch-builder` closed. Issue #184 closed. work-end skill fixed (`BLOG_COUNT` → `BLOG_HAS_ENTRIES`).

## Immediate Next Step

Pick up claudony#129 (replace local DTO copies with api types, S · Low) or file a qhorus issue for Plan B (Category B `@Blocking` tools in `ReactiveQhorusMcpTools` → `Uni<T>`) before starting it.

## Cross-Module

**We're blocking:**
- `casehubio/aml#30` — Layer 4 FinCEN audit trail was designed against this API. #184 is now done; AML can proceed. No further action from qhorus.

## What's Left

- **claudony#129** — replace local DTO copies with api types · S · Low _(wait for #175 on casehubio/qhorus if needed)_
- **Plan B** — Category B `@Blocking` tools in `ReactiveQhorusMcpTools` → `Uni<T>` · L · High _(file issue first)_
- **#185–189** — batched minor review findings filed as issues; low priority

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| Plan B | Category B reactive tool conversion | L | High | File issue first |
| claudony#129 | Replace local DTO copies with api types | S | Low | After #175 on casehubio/qhorus |
| — | A2AChannelBackend enforcement gaps (#188) | S | Low | Rate limiter + ACL bypass documented |

## Notes

- Push workflow: `git push origin main` only (mdproctor/qhorus); promote to casehubio/qhorus manually
- `backup/pre-squash-main-20260522` exists locally (project) — safe to delete after next session confirms clean
- `remotes/origin/claude/agent-argument-graphs-DWlFm` — unknown provenance remote branch, left untouched

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-22-mdp03-ledger-was-lying.md` |
| ADR-0009 | `docs/adr/0009-message-dispatch-builder-audit-chain.md` (project) |
| Spec | `docs/specs/2026-05-22-message-dispatch-builder-design.md` (project) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

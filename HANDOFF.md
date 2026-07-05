# CaseHub Qhorus — Session Handover
**Date:** 2026-07-05 — #321, #320 closed (CDI persistence-memory cleanup + ARC42STORIES alignment).

---

## Immediate Next Step

Main is clean. Both remotes at `37569773`. Issues #321 and #320 merged and closed.

Cross-repo follow-up issues filed and pending:
- claudony#167 — 9 test files: `io.casehub.qhorus.testing.InMemory*` → `io.casehub.qhorus.persistence.memory.InMemory*`
- devtown#140 — remove ghost `exclude-types` for deleted classes + CrossTenantProducer
- openclaw#62 — 1 test file: same import change as claudony

Ledger API drift from prior sessions was fixed on this branch (imports + JOINED inheritance). casehub-ledger was rebuilt from source — consumers resolving 0.2-SNAPSHOT locally will get the correct version.

Garden entries submitted: GE-20260705-2a5555 (LedgerAttestation MappedSuperclass vs Entity), GE-20260705-a910c0 (JOINED inheritance break when parent becomes MappedSuperclass). Push to garden GitHub failed (auth) — committed locally, push on next session.

## What Was Done This Session

**CDI persistence-memory cleanup (#321):** Deleted 40 duplicate InMemory stores from testing/ (copied not moved during extraction). Eliminated CrossTenantProducer, @CrossTenant qualifier, QhorusSystemCurrentPrincipal, @QhorusSystem — qualifier was redundant on distinct types, admin assertion was tautological. Fixed MessageLedgerEntry to extend JpaLedgerEntry (JOINED inheritance broke when LedgerEntry became @MappedSuperclass). Fixed LedgerAttestation creation to use runtime @Entity class. Updated 3 protocols, ARC42STORIES.MD, DESIGN.md, CLAUDE.md. Design-reviewed (6 rounds, 16 issues, 16 verified).

**ARC42STORIES.MD alignment (#320):** Expanded §5 api module table from 5 to 9 packages, added persistence-memory/ module, removed CrossTenantProducer from §4-§13. Updated in workspace.

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/issue-321-cdi-persistence-memory/` |
| Garden entries | `GE-20260705-2a5555`, `GE-20260705-a910c0` |
| Cross-repo issues | claudony#167, devtown#140, openclaw#62 |
| Previous session | `git show HEAD~1:HANDOFF.md` |

# CaseHub Qhorus — Session Handover
**Date:** 2026-06-09 — #256, #255, #262 shipped (ledger sequence refactor + batch timeline fix)

---

## What Was Done This Session

**#256/#255/#262 — Ledger sequence refactor + repository cleanup + batch timeline:**

- `QhorusSequenceAllocator` (`@REQUIRES_NEW` CDI bean): MERGE SQL commits atomically before concurrent callers can race the H2 INSERT — eliminates TOCTOU race. `QhorusLedgerEntryRepository.save()` holds `synchronized(this)` across the REQUIRES_NEW call so T2 blocks until T1's commit is visible.
- `QhorusLedgerEntryRepository` (implements `LedgerEntryRepository`, non-`@Alternative`): replaces deleted `LedgerEntryJpaRepository`. Full Merkle chain, actorId tokenisation, null-safe tenancyId. `quarkus.arc.selected-alternatives` is unreliable for library beans in Quarkus extension test augmentation — subclass pattern is the correct CDI solution.
- `QhorusLedgerMerkleFrontierRepository`: thin non-`@Alternative` subclass of `JpaLedgerMerkleFrontierRepository`.
- `ReactiveLedgerEntryJpaRepository.save()`: first reactive Merkle chain. MERGE sequence + actorId tokenisation before `leafHash` + frontier update via `createMutationQuery`.
- `LedgerWriteService` + `ReactiveLedgerWriteService`: `findLatestBySubjectId` + `sequenceNumber` computation removed.
- `LedgerEntryJpaRepository.java` + `LedgerEntryJpaRepositoryTest.java` deleted.
- `StubLedgerEntryJpaRepository` → `StubLedgerEntryRepository`.
- `MessageLedgerEntryRepository.findByMessageIds(Collection<Long>)`: batch IN query.
- `getChannelTimeline()` (blocking): N+1 eliminated. Reactive path was returning null telemetry for all EVENTs — fixed with same batch pattern.
- Protocol PP-20260609-e5ac14 (ledger_subject_sequence SQL init required in all test modules with ledger enabled + Flyway disabled).
- Garden: GE-20260609-62a1a7 (H2 concurrent MERGE race + synchronized+REQUIRES_NEW fix); REVISE on GE-20260417-c59817 (extension test caveat); REVISE on GE-20260607-ad3d62 (sql-load-script alternative fix).

## Immediate Next Step

On main, clean. Run `/work` to pick up next issue.

## What's Left

None — #256, #255, #262 all closed and shipped.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| casehub-ledger#126 | Full EVENT telemetry decoupling from message.content | M | Med | Follow-up from #257; not blocking anything |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-09-mdp02-the-row-that-wouldnt-lock.md` |
| Design spec | `specs/2026-06-09-ledger-sequence-repo-cleanup.md` (workspace); `docs/specs/` (project) |
| Protocol: ledger sequence SQL init | PP-20260609-e5ac14 |
| Garden: H2 concurrent MERGE race | GE-20260609-62a1a7 |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

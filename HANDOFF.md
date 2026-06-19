# CaseHub Qhorus — Session Handover
**Date:** 2026-06-19 — #290 shipped (Merkle frontier tenancyId fixes), unblocks claudony#155

---

## Immediate Next Step

Main is clean. Both remotes aligned at `8a554b8`. Pick the next issue — #279 (CloudEvent adapter) is the easiest standalone entry point.

## What Was Done This Session

**#290 — Merkle frontier tenancyId fixes (two commits):**

Claudony's `MeshResourceInterjectionTest` failures traced through several false leads before a broad classpath scan of all `:tenancyId` references found the actual source:

1. `LedgerMerkleFrontier.deleteBySubjectAndLevel` named query had a `:tenancyId` predicate added in `casehub-ledger` but the call site in `ReactiveLedgerEntryJpaRepository.save()` never bound it → `QueryParameterException` at runtime.

2. `LedgerMerkleTree.append()` creates frontier nodes without tenancyId. Column is NOT NULL → INSERT failed after fix 1.

Both fixed with one line each. Build passes, both remotes aligned at `8a554b8`.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #279 | CloudEvent adapter for `MessageReceivedEvent` | S | Low | Independent, standalone |
| #233 | Complete ARC42STORIES.MD | L | High | Docs; ~20 blog entries as source |
| #218 | Consolidate `ChannelService.create()` overloads | M | Med | Refactor, deferred |

## Cross-Module

**Claudony:** `claudony#155` now has the full fix chain: `cd46e30` (no-op InMemory), `a29e461` (bind `:tenancyId` in named query), `8a554b8` (propagate tenancyId to frontier nodes). Claudony needs a SNAPSHOT after `8a554b8` — all three commits required.

## References

| What | Path |
|------|------|
| Garden entries | `GE-20260619-fe34fc` (named query cross-repo drift), `GE-20260619-479b69` (entity factory tenancy) |
| Protocol | `PP-20260618-100368` — InMemory store mutation prohibition |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

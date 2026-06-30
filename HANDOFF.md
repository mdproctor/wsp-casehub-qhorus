# CaseHub Qhorus — Session Handover
**Date:** 2026-06-30 — #314 in progress. Branch `issue-314-store-spi-to-api`.

---

## Immediate Next Step

Branch does not compile. Resume with `/work`, then complete Task 4: apply the JpaChannelStore conversion pattern to the remaining 17 JPA stores, ~19 InMemory stores, ~9 services, and ~95 tests. The pattern is established — `JpaChannelStore.java` is the reference implementation (committed). See `design/JOURNAL.md` for the full remaining-work breakdown.

## What Was Done This Session

**Design:** brainstorming → spec → 8-round adversarial design review ($22.80, 18 issues, all resolved). Spec at `specs/issue-314-store-spi-to-api/2026-06-30-store-spi-to-api-design.md`.

**Implementation (4 commits on branch):**
1. `41cdd29` — renamed 10 JPA entities to `*Entity` (253 files)
2. `3f0e48f` — 9 domain records in api/, moved ChannelCreateRequest+ChannelSlugValidator (63 files)
3. `e18e16e` — fromDomain()/toDomain() conversion layer on all entities (37 files)
4. `b45b0cd` — moved 23 Store SPI + Query types to api/store/, updated SPI signatures to domain records (210 files) — **WIP, does not compile**

**Cross-repo issues filed:** engine#608 (store imports), parent#331 (doc sync). Remaining to file: ops, drafthouse, clinical.

**Garden:** `GE-20260630-91be72` — IntelliJ MCP ide_refactor_rename partial failure gotcha (committed locally, push failed).

## What's Left

- Complete Task 4: mechanical entity→record updates across ~140 files · L · Low
- Task 5: remove testing/ duplicate InMemory stores · S · Low
- Task 6: CLAUDE.md, ADR-0017, cross-repo issues for ops/drafthouse/clinical · M · Low
- Garden push (GE-20260630-91be72) · XS · Low

## References

| What | Path |
|------|------|
| Design spec | `specs/issue-314-store-spi-to-api/2026-06-30-store-spi-to-api-design.md` |
| Implementation plan | `~/.claude/plans/declarative-pondering-sifakis.md` |
| Design journal | `design/JOURNAL.md` |
| JPA store reference impl | `runtime/.../store/jpa/JpaChannelStore.java` (committed) |
| Previous session | `git show HEAD~1:HANDOFF.md` |

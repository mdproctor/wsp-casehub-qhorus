# CaseHub Qhorus — Session Handover

**Date:** 2026-08-04 — #388 shipped. Thread summary storage for blocks#59.

---

## Immediate Next Step

Pick from the cross-repo backlog — all qhorus-local issues are complete. #358, #359, #361 are independent cross-repo items.

## What Was Done

#388: added `ThreadSummary` record + `ThreadSummaryStore` SPI + `ThreadSummaryUpdatedEvent` in qhorus-api, `InMemoryThreadSummaryStore` in persistence-memory (6 tests), JPA entity + `JpaThreadSummaryStore` in runtime. Three commits on main. All backward-compatible — new types only. Blocks#59 consumes these for push-based thread summary integration.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #358 | Supervisor + friction interventions | L | High | Cross-repo: engine |
| #359 | Summarisation → Qhorus integration | M | Med | Cross-repo: blocks (slot 47 archived) |
| #361 | CBR routing + coordination memory | M | High | Cross-repo: blocks/neocortex (slot 48 ready to land) |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*

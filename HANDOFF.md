# CaseHub Qhorus — Session Handover
**Date:** 2026-06-18 — #289 shipped (InMemory store Hibernate dirty-flush fix) + push alignment

---

## Immediate Next Step

Main is clean. Both remotes aligned at `45fad37`. Pick the next issue from the backlog — #279 (CloudEvent adapter) is the easiest standalone entry point.

## What Was Done This Session

**Push alignment:** `mdproctor/qhorus` was missing 2 commits (slack-channel + CDI regression guard) from the previous session. Pushed to align. Push order convention locked in: origin (mdproctor) always first, then upstream (casehubio) — enforced by work-end skill, memory updated.

**#289 — InMemory store Hibernate dirty-flush fix:** `InMemoryChannelStore.updateLastActivity()` and `InMemoryReactiveChannelStore.updateLastActivity()` were modifying `channel.lastActivityAt` in-place within `Panache.withSession()`. Hibernate's bytecode dirty tracker flagged the field write as a managed entity mutation and generated a JPA UPDATE at session flush — even though `InMemoryReactiveChannelStore` was correctly selected by CDI (confirmed by direct injection test). Fix: both methods are now no-ops. Unblocks `claudony#155` (6/9 `MeshResourceInterjectionTest` failures). Shipped to both remotes.

**Protocol + CLAUDE.md:** `PP-20260618-100368` formalised the rule. Garden entry `GE-20260618-d81cef` submitted.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #279 | CloudEvent adapter for `MessageReceivedEvent` | S | Low | Independent, standalone |
| #233 | Complete ARC42STORIES.MD | L | High | Docs; ~20 blog entries as source |
| #218 | Consolidate `ChannelService.create()` overloads | M | Med | Refactor, deferred |

## Cross-Module

**Claudony unblocked:** `claudony#155` should close once Claudony picks up a SNAPSHOT after `cd46e30`. Claudony needs to bump its Qhorus dependency to a build that includes the `fix(#289)` commit.

## References

| What | Path |
|------|------|
| Protocol — InMemory store mutation | `docs/protocols/casehub/inmemory-store-no-entity-mutation-in-session.md` |
| Garden entry | `GE-20260618-d81cef` (jvm/ domain) |

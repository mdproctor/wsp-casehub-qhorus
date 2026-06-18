# CaseHub Qhorus — Session Handover
**Date:** 2026-06-18 — #261 shipped (SlackChannelBackend) + CDI regression guard

---

## Immediate Next Step

Main is clean. Both repos aligned. Pick the next issue from the backlog.

## What Was Done This Session

**CDI regression guard (#288):** `StoreCdiAlternativesTest` in `examples/type-system` verifies `InMemoryReactive*Store` beans are selected over JPA reactive stores in consumer test contexts. Documents that consumers enabling `casehub.qhorus.reactive.enabled=true` must list reactive InMemory stores in `quarkus.arc.selected-alternatives`. Addresses root cause of claudony#155.

**#261 — casehub-qhorus-slack-channel:** New optional module. `SlackChannelBackend` implements `HumanParticipatingChannelBackend` via `SlackBotClient`. Thread continuity via composite in-memory + DB cache (V23/V24). Key design: correlationId written to cache BEFORE `gateway.receiveHumanMessage()` to prevent race with fast agent responses. Credential pattern: Tier 1.5 (workspaceId in DB, token from MicroProfile Config). Code review found and fixed: `post()` now catches `NoSuchElementException` from `resolveToken()`; `onChannelInitialised()` now `@Transactional`.

Squashed to 2 commits on main. Both pushed to fork (origin) and upstream (casehubio/qhorus). All tests pass.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #279 | CloudEvent adapter for MessageReceivedEvent | S | Low | Independent, standalone |
| #233 | Complete ARC42STORIES.MD | L | High | Docs; ~20 blog entries as source |
| #218 | Consolidate `ChannelService.create()` overloads | M | Med | Refactor, deferred |

## References

| What | Path |
|------|------|
| Slack backend spec | `docs/specs/2026-06-17-slack-channel-backend-design.md` (project) |
| Protocol — JPA packages | `docs/protocols/casehub/optional-module-jpa-package-registration.md` |
| Protocol — reactive InMemory | `docs/protocols/casehub/reactive-inmemory-store-selected-alternatives.md` |
| Diary entry | `blog/2026-06-18-mdp02-slack-thread-cache.md` |
| Peer issues filed | casehubio/parent#278 (qhorus deep-dive), casehubio/parent#279 (PLATFORM.md capability row) |

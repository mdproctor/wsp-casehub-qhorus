# CaseHub Qhorus — Session Handover
**Date:** 2026-05-14 — ledger#75 + qhorus#135 both shipped and closed

---

## What Was Done This Session

**ledger#75** — `ActorTypeResolver` A2A rules. `"agent"` → AGENT (was HUMAN via catch-all, silent governance bug). `"user"` → HUMAN (now explicit). Two `if` statements + Javadoc. Committed and pushed by ledger Claude.

**qhorus#135** — Robust A2A integration. Full stack:
- `V9__add_actor_type_to_message.sql` — `actor_type VARCHAR(10) NOT NULL`
- `MessageService.send()` — 3 overloads → 1 canonical with required `ActorType actorType`; ~90 call sites updated
- `QhorusMcpTools` — injects `InstanceActorIdProvider`; LAST_WRITE path now updates actorType (code review catch)
- `LedgerWriteService` — reads `message.actorType` directly; no more string re-derivation
- `A2AActorResolver` — 6-step chain: header → Instance registry → agentCardUrl → ActorTypeResolver(agentId) → default HUMAN
- `A2AChannelBackend implements ChannelBackend` (not `AgentChannelBackend` — CDI ambiguity with QhorusChannelBackend); `ensureRegistered()` via `ConcurrentHashMap.newKeySet().add()`
- `A2AResource` — thin adapter; delegates to backend; `getTask()` uses `CommitmentStore` for durable state; history always included
- `CommitmentService.findByCorrelationId()` — new public query method
- 7 follow-on issues filed: #145 (rate limiting), #146 (artefacts), #147 (SSE), #148 (LAST_WRITE), #149 (post() log guard), #150 (batch minors), #151 (DELEGATED state mapping)

1035 tests, 0 failures. Both issues closed. Pushed.

Platform: `qhorus-actor-type-mapping.md` updated (A2A chain documented). `casehub-qhorus.md` deep dive updated. `CLAUDE.md` updated with MessageService.send() actorType convention. Protocol PP-20260514-c80d4c (gateway backend registration ordering).

## Current State

- **Branch:** `main` — clean, fully pushed
- **Test count:** 1035 passing, 44 skipped (reactive JPA, need Docker)
- **Untracked:** `docs/squash-plan-*.md` (stale, can delete)

## Immediate Next Steps

Priority order:
1. **#141** (bug) — Gate `quarkus-hibernate-reactive-panache` behind `casehub.qhorus.reactive.enabled` build-time flag
2. **#142** (bug) — Flyway V2 conflict with casehub-work when on same classpath
3. **#132** (feat) — Delivery guarantees for registered backends (retry + dead-letter) — now unblocked since #135 shipped
4. **#151** (bug) — DELEGATED → "completed" in A2A is semantically wrong; fix `toA2AState(DELEGATED)` → `"working"`

## Key Architecture Facts (new this session)

- `A2AChannelBackend` registers itself lazily via `ensureRegistered()` — not at channel creation time
- `A2AActorResolver` is separate from `A2AChannelBackend` for unit testability (no gateway dependency)
- `message.actorType` is the source of truth — never re-derive from sender string
- `InstanceActorIdProvider` enrichment (`actorIdProvider.resolve(sender)`) must happen BEFORE `messageService.send()` in QhorusMcpTools — not inside LedgerWriteService
- Gateway backend registration ordering: `open()` BEFORE `registerBackend()` (PP-20260514-c80d4c)
- `WatchdogEvaluationService` sender is now `"system:watchdog"` (was `"watchdog"` — latent bug fixed)

## References

| What | Path |
|---|---|
| A2A design spec | `wksp/specs/2026-05-13-a2a-integration-design.md` |
| A2A impl plan | `wksp/plans/2026-05-13-a2a-integration.md` |
| Latest blog | `blog/2026-05-14-mdp01-making-actortype-explicit.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

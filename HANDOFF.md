# CaseHub Qhorus — Session Handover
**Date:** 2026-05-05 — Channel gateway shipped (#131 core done), #140 closed, garden swept

---

## What Was Done This Session

- **Channel backend abstraction (#131) — full design + implementation + docs + tests**
  - Brainstorm: coherence invariant (at most one `HumanParticipatingChannelBackend`), `ActorType` alignment, A2A as protocol bridge, `InboundNormaliser` SPI
  - `ChannelGateway`, `QhorusChannelBackend`, `DefaultInboundNormaliser`, `Senders` (moved to api module)
  - SPI hierarchy in `casehub-qhorus-api`: `AgentChannelBackend`, `HumanParticipatingChannelBackend`, `HumanObserverChannelBackend`, `InboundNormaliser`
  - MCP tools: `send_message` fans out via `channelGateway.fanOut()`; `create_channel` auto-registers; `delete_channel` closes backends; `list_backends`, `deregister_backend`, `register_backend`
  - Code review caught: TOCTOU race (synchronized), `closeChannel` ordering (after persistence), `Senders` moved to api, dead `post()` (package-private)
  - 1011 tests, 0 failures. ADR-0006, full doc sweep, CLAUDE.md, parent repo docs + actor-type convention
- **#124 (InstanceActorIdProvider) — closed** both sides (Qhorus SPI + Claudony#107 implementation)
- **#140 (register_backend MCP tool) — implemented and closed** — CDI `Instance<ChannelBackend>` lookup by `backendId()`
- **Created:** #135 (A2A bridge), casehubio/ledger#75 (ActorTypeResolver fix), Claudony#107 (closed)
- **All pushed to main**

## Current State

- **Branch:** `main` — clean, fully pushed (`.claude/settings.local.json` untracked, ignore)
- **Open issues:** #135 (A2A bridge — blocked on ledger#75), #132 (delivery guarantees — now unblocked), #131 (epic tracker), #98 (LLM baseline, machine-intensive)
- **Test count:** 1011 passing, 44 skipped (reactive JPA, need Docker)

## Immediate Next Step

**Check ledger#75** — `ActorTypeResolver` explicit A2A rules (`"user"` → `HUMAN`, `"agent"` → `AGENT`). Ledger Claude was working on other issues (#56–#59) this session — may or may not have landed. If done, proceed to **#135** (`A2AChannelBackend` protocol bridge). If not, either do it in qhorus session or wait for ledger Claude.

Then **#132** — delivery guarantees (retry + dead-letter for registered backends). Now unblocked since #131 gateway is shipped.

## Key Architecture Facts (new this session)

- `ChannelGateway.fanOut()` dispatches to external backends only (not `QhorusChannelBackend`). `post()` is package-private, test-only — calling it in production double-persists.
- `Senders.HUMAN = "human"` is in `io.casehub.qhorus.api.gateway.Senders` (not runtime).
- `registerBackend()` human_participating check-then-add is synchronized on the channel's backend list.
- `@WrapBusinessError` tests must catch `ToolCallException`, not `IllegalArgumentException` — the interceptor wraps at CDI proxy boundary.
- Backend registrations are in-memory — lost on restart (tracked in #132).
- A2A `role: "user"` is not always a human — may be an AI orchestrator. Requires actor identity resolution chain (6 steps) in `A2AChannelBackend`. Tracked in #135.

## References

| What | Path |
|---|---|
| Design spec | `docs/superpowers/specs/2026-05-04-channel-backend-abstraction-design.md` |
| ADR | `docs/adr/ADR-0006-channel-backend-abstraction.md` |
| A2A bridge issue | casehubio/qhorus#135 |
| ActorTypeResolver fix | casehubio/ledger#75 |
| Latest blog | `blog/2026-05-05-mdp01-the-coherence-invariant.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

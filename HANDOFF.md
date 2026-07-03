# CaseHub Qhorus — Session Handover
**Date:** 2026-07-03 — #315 closed (MessageDispatcher and ChannelManager SPIs extracted).

---

## Immediate Next Step

Main is clean. Both remotes at `b906782`. #315 merged and closed.

Cross-repo follow-up pending (carried from #314):
- `MessageReceivedEvent` constructor changed (added `Instant occurredAt`) — Claudony (3 sites) and engine (7 sites) need `Instant.now()` as 7th arg
- Store SPI imports moved from `runtime/store/` to `api/store/` — casehub-engine (actor-state), casehub-ops (drift checker), casehub-drafthouse (4 files), casehub-clinical (1 file)
- Parent repo doc sync: casehubio/parent#330 (from #314), casehubio/parent#341 (from #315)

New follow-up issues from #315:
- #317: findOrCreate name-based concurrency recovery fails on PostgreSQL (PersistenceException retry within rollback-marked REQUIRES_NEW tx)
- #318: ReactiveChannelService.create() does not call channelGateway.initChannel()

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Cross-repo Store SPI migration (engine, ops, drafthouse, clinical) | M | Low | Mechanical — same entity→record pattern |
| #317 | Fix findOrCreate PostgreSQL race recovery | S | Med | Move catch outside REQUIRES_NEW boundary |
| #318 | Reactive create() missing gateway.initChannel() | XS | Low | One-line addition |
| ops#14 | Enrich ChannelDriftChecker — full field comparison, tenancy fix | S | Low | Simplifies with domain record typed fields |
| openclaw#57 | Override deliveryGuarantee → AT_LEAST_ONCE on OpenClawChannelBackend | XS | Low | Propagation from #132 |

## What Was Done This Session

**Service facade SPI extraction (#315):** Four interfaces (`MessageDispatcher`, `ReactiveMessageDispatcher`, `ChannelManager`, `ReactiveChannelManager`) in api/ domain packages. `ChannelCreateRequest` CSV→List<String> type alignment. `findOrCreate` generalised to dual-mode lookup. Dead code removed. Garden entry GE-20260703-30313f submitted (PostgreSQL transaction gotcha).

## References

| What | Path |
|------|------|
| Design spec | `specs/issue-315-message-dispatcher-channel-lifecycle-spi/` (promoted to project) |
| Blog entry | `blog/2026-07-03-mdp01-the-fourth-category.md` |
| Garden entry | `GE-20260703-30313f` (jvm) — PersistenceException H2 vs PostgreSQL |
| Previous session | `git show HEAD~1:HANDOFF.md` |

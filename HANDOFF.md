# CaseHub Qhorus — Session Handover
**Date:** 2026-07-04 — #319, #317, #318 closed (Channel null lists, findOrCreate race, reactive create parity).

---

## Immediate Next Step

Main is clean. Both remotes at `4b995e9`. All three issues merged and closed.

Cross-repo follow-up pending (carried from #314/#315):
- `MessageReceivedEvent` constructor changed (added `Instant occurredAt`) — Claudony (3 sites) and engine (7 sites) need `Instant.now()` as 7th arg
- Store SPI imports moved from `runtime/store/` to `api/store/` — casehub-engine (actor-state), casehub-ops (drift checker), casehub-drafthouse (4 files), casehub-clinical (1 file)
- Parent repo doc sync: casehubio/parent#330 (from #314), casehubio/parent#341 (from #315)

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Cross-repo Store SPI migration (engine, ops, drafthouse, clinical) | M | Low | Mechanical — update imports from `runtime/store/` to `api/store/` + `MessageReceivedEvent` constructor |
| openclaw#57 | Override deliveryGuarantee → AT_LEAST_ONCE on OpenClawChannelBackend | XS | Low | Propagation from #132 |

## What Was Done This Session

**Three Channel creation fixes (#319, #317, #318):** Null list normalization (compact constructors default to `List.of()`), ChannelCreateHelper with `REQUIRES_NEW` for PostgreSQL race recovery, ReactiveChannelService.create() brought to full parity (binding, initChannel, race recovery). Design-reviewed spec (11 issues, all resolved). Protocol PP-20260704-d0b9f3 captured (`@TestTransaction` + `REQUIRES_NEW` helper unique names).

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/issue-319-channel-null-lists-and-fixes/` (promoted to project) |
| Blog entry | `blog/2026-07-04-mdp01-the-null-that-bit-every-caller.md` |
| Protocol | `PP-20260704-d0b9f3` (universal) — @TestTransaction + REQUIRES_NEW unique names |
| Protocol updated | `PP-20260609-fe1300` — channel-create-requires-init-channel (both paths now internal) |
| Previous session | `git show HEAD~1:HANDOFF.md` |

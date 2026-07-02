# CaseHub Qhorus — Session Handover
**Date:** 2026-07-01 — #314 closed (Store SPI migration complete).

---

## Immediate Next Step

Main is clean. Both remotes at `10d2816`. #314 merged and closed.

Cross-repo follow-up pending:
- `MessageReceivedEvent` constructor changed (added `Instant occurredAt`) — Claudony (3 sites) and engine (7 sites) need `Instant.now()` as 7th arg
- Store SPI imports moved from `runtime/store/` to `api/store/` — casehub-engine (actor-state), casehub-ops (drift checker), casehub-drafthouse (4 files), casehub-clinical (1 file) need mechanical import + field-access updates
- Parent repo doc sync: casehubio/parent#330

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Cross-repo Store SPI migration (engine, ops, drafthouse, clinical) | M | Low | Mechanical — same entity→record pattern |
| ops#14 | Enrich ChannelDriftChecker — full field comparison, tenancy fix | S | Low | Simplifies with domain record typed fields |
| openclaw#57 | Override deliveryGuarantee → AT_LEAST_ONCE on OpenClawChannelBackend | XS | Low | Propagation from #132 |

## What Was Done This Session

**Store SPI migration (#314):** Completed the full migration — 336 files, 12 modules. Domain records in api/, Store SPIs in api/store/, JPA entities renamed to *Entity in runtime/. Conversion at JPA store boundary via fromDomain()/toDomain(). All tests pass.

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/issue-314-store-spi-to-api/2026-06-30-store-spi-to-api-design.md` |
| Blog entry | `blog/2026-06-30-mdp02-the-roots-not-the-leaves.md` |
| Journal | `design/JOURNAL.md` — §store-spi-boundary |
| Previous session | `git show HEAD~1:HANDOFF.md` |

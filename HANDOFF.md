# CaseHub Qhorus — Session Handover
**Date:** 2026-06-30 — #311, #312, #313, #169 closed. #314 filed (Store SPI → api/).

---

## Immediate Next Step

Main is clean. Both remotes at `84bba77`. Four issues closed, one follow-up filed.

Cross-repo follow-up still pending from prior session: `MessageReceivedEvent` constructor changed (added `Instant occurredAt`). Claudony (3 test sites) and engine (7 test sites) will fail at next compile — mechanical fix (`Instant.now()` as 7th arg).

Parent repo doc sync issue filed: casehubio/parent#330 (persistence-memory, delivery metrics, LAST_WRITE version in deep-dive).

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #314 | Move Store SPI interfaces from runtime/ to api/ (Tier 1) | L | Med | Filed this session. Cross-repo migration. |
| ops#14 | Enrich ChannelDriftChecker — full field comparison, tenancy fix | S | Low | Cross-repo (casehub-ops) |
| openclaw#57 | Override deliveryGuarantee → AT_LEAST_ONCE on OpenClawChannelBackend | XS | Low | Propagation from #132 |

## What Was Done This Session

**CI dispatch (#311):** Added `blocks` to publish.yml downstream dispatch list.

**persistence-memory extraction (#169):** Created `persistence-memory/` module. Moved 18 InMemory store implementations + 23 test files from `testing/`. Package rename `io.casehub.qhorus.testing` → `io.casehub.qhorus.persistence.memory`. testing/ depends on persistence-memory/ transitively.

**Delivery metrics (#312):** 4 Micrometer metrics on DeliveryService — `messages.delivered` counter, `failures` counter, `backends.unhealthy` gauge, `cursor.lag` gauge. BatchResult changed from enum to record with deliveredCount. MeterRegistry via `Instance<>` for optional injection.

**LAST_WRITE delivery (#313):** Version counter on Message, `lastDeliveredVersion` on DeliveryCursor, version-aware batch query, delivery signal on overwrite via post-commit TSR. V26 migration.

## References

| What | Path |
|------|------|
| Design specs | `docs/specs/2026-06-30-*.md` (3 specs) |
| Blog entry | `blog/2026-06-30-mdp01-the-module-that-wasnt-a-module.md` |
| Garden entry | `GE-20260630-6c2515` — CDI Instance<T> provided scope classloading gotcha |
| Parent doc sync | `casehubio/parent#330` |
| Previous session | `git show HEAD~1:HANDOFF.md` |

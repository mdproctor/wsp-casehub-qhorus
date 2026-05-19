# CaseHub Qhorus — Session Handover
**Date:** 2026-05-19 — epic-a2a-lifecycle-cleanup closed; 6 issues resolved; all @QuarkusTest tests unblocked

---

## What Was Done This Session

**epic-a2a-lifecycle-cleanup** — six issues, all closed and merged to main:

- **#151** — already done from previous session; closed on discovery
- **#157** — removed dead `pending_reply` table from V1 schema; added V10 commitment migration (`fk_commitment_channel`, `fk_commitment_parent`, `idx_commitment_state_expires`); `FlywayMigrationSchemaTest` proves Flyway migrations directly (bypasses drop-and-create)
- **#167** — `MessageObserverDispatcher` now uses `Instance.handles()` with `finally`-close; any CDI scope valid on `MessageObserver`; `@ApplicationScoped`-only constraint retired (protocol PP-20260518-837246 updated)
- **#160** — E2E commitment auto-fulfillment test via `ChannelGatewayCommitmentE2ETest` with `@QuarkusTestProfile`; `StubReactiveLedgerEntryRepository` (`@DefaultBean`) fixes the pre-existing CDI env issue — all `@QuarkusTest` tests now run
- **#155/#156** — Flyway directory-scoping documented in PLATFORM.md; V9→V1003 gap explained in V1003 header

**Side-effects:**
- Protocol PP-20260519-1f5e2c: `@DefaultBean` stub pattern for cross-module reactive CDI deps in tests
- Garden: 4 new entries (InstanceHandle.close(), Instance.handles() wildcard, Flyway baselineVersion, Flyway migration TDD)
- `#171` filed: `delete_channel` must call `commitmentStore.deleteAll(channelId)` before `channelStore.delete(channelId)` (fk_commitment_channel has no CASCADE)

## Current State

- **Both repos:** `main` — epic merged
- **Epic branches retained:** epic-142, epic-153, epic-154, epic-a2a-lifecycle-cleanup (all with `EPIC-CLOSED.md`; epic-a2a deletion: 2026-06-02)
- **Tests:** 40 passing (including 4 new `@QuarkusTest` E2E tests now unblocked); `LedgerQueryE2ETest` still fails to load (separate pre-existing CDI issue, unrelated)
- **Next Flyway domain migration:** V11 (V10 is commitment table)

## Immediate Next Steps

1. **#132** — Delivery guarantees for registered backends (retry + dead-letter) — main feature item
2. **clinical#16** — PiResponseListener workaround removal — unblocked since #154
3. **claudony#117** — `ClaudonyChannelBackend` — unblocked since qhorus#131 (gateway SPI) and qhorus#153 (MessageObserver SPI)
4. **#171** — `delete_channel` must also delete commitments before channel deletion

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-19-mdp02-what-drop-and-create-hides.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

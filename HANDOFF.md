# CaseHub Qhorus — Session Handover
*Updated: claudony#167, devtown#140 closed — removed from backlog.*
**Date:** 2026-07-06 — #162 closed (cross-node backend delivery via ChannelActivityBroadcaster SPI).

---

## Immediate Next Step

Main is clean. Both remotes at `a13c5ee2`. Issue #162 merged and closed.

Cross-repo follow-up issues filed and pending:
- parent#353 — update PLATFORM.md capability ownership for cross-node backend delivery
- claudony#168 — migrate from FleetMessageRelayObserver to casehub-qhorus-postgres-broadcaster
- qhorus#325 — review cleanup follow-ups (test sleeps→Awaitility, parseCorrelationUuid duplication, exponential backoff)

Prior session follow-ups still open:
- openclaw#62 — 1 test file: same import change

Garden entry GE-20260706-b56877 committed locally (Collections.synchronizedSet compound operations gotcha). Push to garden GitHub failed (auth) — push on next session (also GE-20260705-2a5555, GE-20260705-a910c0 from prior session).

## What Was Done This Session

**Cross-node backend delivery (#162):** Established that shared PostgreSQL is the only valid multi-node topology (ADR-0017). Built `ChannelActivityBroadcaster` SPI in `api/gateway/`, `NoOpChannelActivityBroadcaster` @DefaultBean, `ChannelGateway.deliverRemote()` with lazy channel init, wired broadcaster into both MessageService and ReactiveMessageService (normal + LAST_WRITE paths). Created `postgres-broadcaster/` module with PostgreSQL LISTEN/NOTIFY, self-notification filter, virtual thread offload, connection reconnection. Fixed pre-existing gap where LAST_WRITE overwrites skipped fanOut(). Design-reviewed (3 rounds, 18 issues, all resolved). Full branch code review — 1 Important fixed (correlationId UUID parse). Blog entry written.

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/issue-162-cross-node-delivery-gap/` |
| ADR | `docs/adr/0017-shared-database-multi-node-prerequisite.md` |
| Garden entry | `GE-20260706-b56877` (Collections.synchronizedSet compound ops) |
| Cross-repo issues | parent#353, claudony#168, qhorus#325 |
| Previous session | `git show HEAD~1:HANDOFF.md` |

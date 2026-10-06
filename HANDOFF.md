# Session Handover — 2026-10-06

## What happened

Implemented Phase 1 runtime safety net for distributed mesh (#475) — 3 surgical changes adding DB-level locks to subsystems that break under concurrent multi-node writes:

1. **Merkle Frontier Locking** — `QhorusLedgerMerkleFrontierRepository.findBySubjectIdForUpdate()` with `SELECT FOR UPDATE`; wired into `QhorusLedgerEntryRepository.save()` to prevent concurrent frontier corruption.

2. **LAST_WRITE Pessimistic Locking** — `MessageReader.findLastMessageForUpdate()` with `SELECT FOR UPDATE`; wired into `MessageService.dispatch()` LAST_WRITE path to prevent lost updates during ownership transfer.

3. **Commitment Pessimistic Locking** — `CommitmentReader.findByCorrelationIdForUpdate()` with `SELECT FOR UPDATE`; wired into all 6 `CommitmentService` state-transition methods (acknowledge/fulfill/decline/fail/delegate/extendDeadline) to prevent double-fulfillment.

Design note: the original plan called for `@Version` (optimistic locking) on Batch 2, but `em.merge()` with `@Version` doesn't work with the detached-entity-from-domain-record pattern used by `JpaMessageStore.put()` — Hibernate throws `StaleObjectStateException` even without concurrency. Switched to pessimistic locking (consistent with Batches 1 and 3).

Full project build green — all modules compile, 2073+ runtime tests pass.

## Key decisions

- All three subsystems use pessimistic locking (`SELECT FOR UPDATE`) for consistency
- Existing `synchronized` keywords kept — they protect the single-JVM fast path; `FOR UPDATE` is the multi-node safety net
- New interface methods use `default` delegation to non-locking versions for backward compatibility (InMemory stores get locking for free via delegation)

## Next action

Phase 1 complete. Next: Phase 2 planning — standalone mesh service, channel partitioning, transport facades.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-475-distributed-mesh/2026-10-06-distributed-mesh-design.md` |
| Decisions | `specs/issue-475-distributed-mesh/decisions.md` |
| Phase 1 plan | `plans/2026-10-06-distributed-mesh-phase1-runtime-safety-net.md` |
| Diary entry | `blog/2026-10-06-mdp01-the-lock-that-doesnt-lock.md` |
| Epic issue | casehubio/qhorus#475 |

## Project state

- **Project branch:** `issue-475-distributed-mesh` — 3 commits (Phase 1 implementation)
- **Workspace branch:** `issue-475-distributed-mesh`
- Build: green (`mvn clean install` — all modules pass)

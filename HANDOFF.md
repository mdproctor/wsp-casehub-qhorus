# Session Handover — 2026-10-07 (Session 2)

## What happened

Continued #484 (audit and e2e cluster testing). Completed all 6 batches and validated e2e tests against real Podman clusters. Discovered 4 production bugs in the distributed mesh through e2e testing.

### Commits

| Commit | What |
|--------|------|
| e3f25454 | CDI wiring and config gate integration tests (Batch 4) |
| 7a9cf6aa | e2e-cluster module with 3 Podman e2e scenarios (Batches 5-6) |
| 475996f8 | Jandex indexes for cluster/cache + relay enabled in mesh |
| 92f9a507 | e2e harness fixes — parallel startup, API field names, deps |
| b3ab7760 | e2e timeout tuning — proxy timeout, dead detection timing |

### E2E validation results

| Test | Result | What it proves |
|------|--------|----------------|
| DispatchRoutingE2ETest (3 tests) | GREEN | Message routing, cross-node DB visibility, cluster health |
| NodeFailure: baseline + dead detection + writes | GREEN | DEAD detection works (3s proxy timeout × 3 misses = ~12s), surviving node accepts writes |
| NodeFailure: restart rejoin | BLOCKED | Container startup fails — Flyway migration + HeartbeatService never re-probes DEAD peers |
| Quorum: full cluster writes | GREEN | 3-node cluster accepts writes |
| Quorum: minority rejects | PARTIAL | Dead detection works (24.5s for 2 peers), but write NOT rejected (200 instead of 400+) |
| Quorum: majority restored | BLOCKED | Depends on minority test |

### Production bugs discovered

1. **Cluster/cache modules had no Jandex index** — CDI beans invisible as library deps. The mesh relay was running WITHOUT clustering or caching. FIXED in 475996f8.

2. **HeartbeatService skips DEAD peers permanently** — `tick()` has `if (ps.state() == NodeState.DEAD) continue`. Once a peer is DEAD, it's never re-probed. Node restarts depend on the restarted node probing the surviving nodes (reverse heartbeat). This works but is fragile.

3. **Cross-node proxy dispatch silently drops messages** — `WriteRoutingDecorator` proxies to owner via `WriteProxyClient.dispatch()`, returns 200, but the message doesn't appear in the shared database. `InternalMeshResource.dispatch()` receives the proxy but the dispatch fails silently.

4. **Quorum enforcement doesn't block REST writes** — `WriteRoutingDecorator.canServeWrites()` check is present but writes via REST still return 200 in minority partition. Either the REST path bypasses the decorator, or the `QuorumViolationException` is caught/swallowed.

## Next action

File GitHub issues for the 4 bugs above, then close #484 via work-end.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-475-distributed-mesh/2026-10-07-audit-e2e-design.md` |
| Implementation plan | `plans/2026-10-07-audit-e2e-cluster.md` |
| Issue | #484 — audit and e2e cluster testing |
| Epic | #475 — distributed mesh |

## Project state

- **Project branch:** `issue-475-distributed-mesh` — 43 commits
- **Workspace branch:** `issue-475-distributed-mesh`
- **.plan queue:** #484 (active, all batches done, e2e validated)
- Build: green (full build passes, 94 cluster tests, e2e profile-gated)

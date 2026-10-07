# Session Handover — 2026-10-07 (Session 2)

## What happened

Continued #484 (audit and e2e cluster testing). Completed all 6 batches — Batch 4 (CDI wiring integration tests), Batches 5-6 (e2e-cluster module with Podman scenario tests), plus critical infrastructure fixes discovered during e2e validation.

### Implementation — Batch 4: CDI Wiring Integration Tests

| Commit | What |
|--------|------|
| e3f25454 | CDI wiring and config gate integration tests — first @QuarkusTest in cluster module |

### Implementation — Batches 5-6: E2E Cluster Module

| Commit | What |
|--------|------|
| 7a9cf6aa | e2e-cluster module with ClusterTestHarness + 3 Podman e2e scenarios |

### Infrastructure Fixes (discovered during e2e validation)

| Commit | What |
|--------|------|
| 475996f8 | Add Jandex indexes to cluster/cache + enable relay in mesh — without these, cluster and cache beans were invisible to the mesh module's Quarkus augmentation |
| 92f9a507 | Fix e2e harness — parallel startup (dead-peer race), API field names, deps |

### Key discoveries

1. **Cluster and cache modules had no Jandex index** — their CDI beans were completely invisible when used as library dependencies. All `@IfBuildProperty`-gated beans (relay, heartbeat, ownership, cache) were silently excluded from the mesh relay node. This means the mesh module was running WITHOUT clustering or caching in production.

2. **`@IfBuildProperty` is build-time only** — must be in `application.properties` at augmentation time. Runtime env vars (`CASEHUB_QHORUS_RELAY_ENABLED`) cannot activate build-time gates. Added `casehub.qhorus.relay.enabled=true` to mesh module.

3. **HeartbeatService skips DEAD peers** — once a peer is marked DEAD (via missed heartbeats), it's never re-probed. Sequential container startup causes permanent DEAD state. Fixed in e2e with parallel startup via `Startables.deepStart()`. The heartbeat skip-DEAD design is a production concern for node restarts — should be tracked separately.

4. **Cross-node proxy dispatch bug** — messages dispatched via `WriteRoutingDecorator` proxy (node-b → node-a) return 200 but don't appear in the shared database. The InternalMeshResource receives the proxy request but the dispatch silently fails. Needs investigation — tracked separately.

5. **`@IfBuildProperty` unregisters `@ConfigMapping`** — when the property gate removes all beans that inject a `@ConfigMapping` interface, the config mapping is unregistered. Any remaining properties under that prefix fail validation. Disabled test profiles must NOT set properties under the gated prefix.

## Next action

1. **File issues** for the two production bugs discovered:
   - HeartbeatService skipping DEAD peers (prevents node recovery)
   - Cross-node proxy dispatch silently dropping messages
2. **Run NodeFailure and Quorum e2e tests** — these haven't been validated yet
3. **Close #484** via work-end once e2e validation is complete

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-475-distributed-mesh/2026-10-07-audit-e2e-design.md` |
| Implementation plan | `plans/2026-10-07-audit-e2e-cluster.md` |
| Decisions | `specs/issue-475-distributed-mesh/decisions.md` (D39-D47) |
| Issue | #484 — audit and e2e cluster testing |
| Epic | #475 — distributed mesh |

## Project state

- **Project branch:** `issue-475-distributed-mesh` — 42 commits
- **Workspace branch:** `issue-475-distributed-mesh`
- **.plan queue:** #484 (active, all 6 batches done — e2e validation in progress)
- Build: pending verification (full build running)
- E2E: DispatchRoutingE2ETest 3/3 green, NodeFailure/Quorum not yet run

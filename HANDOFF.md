# Session Handover — 2026-10-06

## What happened

This session covered the full Phase 2 implementation and significant architecture evolution for the distributed mesh epic (#475).

### Phase 2 implementation (8 commits)

Built `casehub-qhorus-cluster` module — conversation multiplexer infrastructure:
- **ConsistentHashRing** — SHA-256 consistent hashing, 128 virtual nodes, immutable snapshots
- **ClusterManager** — peer state machine (ALIVE/SUSPECT/DEAD), quorum enforcement, graceful shutdown
- **HeartbeatService** — peer polling, failure detection, ring hash mismatch warning
- **WriteRoutingDecorator + ChannelManagerDecorator** — CDI decorators with two-gate activation (Level 3 pass-through, Level 4 routing) and fallback-to-local on proxy failure
- **InternalMeshResource + ClusterHealthResource** — internal RPC and health/topology endpoints
- **RelayConfig + RelayProducer** — `@IfBuildProperty` gated CDI wiring

One API change to existing qhorus: `preAssignedId` field on `ChannelCreateRequest`.

41 cluster module tests. Full project build green.

### Architecture evolution (D18-D24)

Post-implementation discussion reframed the architecture around a **topology maturity ladder**:

| Level | Topology | Config required |
|-------|----------|----------------|
| 1. Dev | Embedded, LLMs direct | Datasource only (zero qhorus config) |
| 2. Dedicated server | Separate qhorus, LLMs direct | Datasource only |
| 3. Relays | Any-channel read/write | `relay.enabled` + peers |
| 4. Channel ownership | Dynamic ownership on relays | Level 3 + `relay.routing=dynamic` |

Key decisions:
- **D23:** Relay is a **conversation multiplexer** — batches and compresses agent conversations over shared PostgreSQL
- **D20:** Fallback-to-local on proxy failure — HA gap eliminated
- **D21:** Self-healing — agents reconnect to any relay on failure
- **D22:** Connection multiplexing — relay as implicit connection pooler
- **D24:** Shallow/full relay depth modes — full mirrors complete history for search offloading

### Consolidated spec + design review

Wrote `2026-10-06-distributed-mesh-consolidated.md` superseding two earlier specs. Standard design review found 6 issues (4 major, 2 minor) — all fixed. Old specs marked ⛔ SUPERSEDED.

### Phase 3-4 implementation plan

Wrote `plans/2026-10-06-distributed-mesh-phase3-4.md`:
- **Batch 1:** InstanceResource — agent registration via REST
- **Batch 2:** Message listing — GET /api/channels/{id}/messages
- **Batch 3:** Mesh wiring — PostgreSQL, cluster module integration, Dockerfile
- **Deferred:** SSE events endpoint (separate observer module); shallow caching (needs own brainstorming cycle)

## Next action

Execute the Phase 3-4 plan: `plans/2026-10-06-distributed-mesh-phase3-4.md`

Start with Batch 1 (InstanceResource) — invoke `executing-plans`.

After Phase 3-4: brainstorm shallow caching (Phase 5) as a separate spec → plan cycle.

## References

| Artifact | Path |
|----------|------|
| Consolidated spec | `specs/issue-475-distributed-mesh/2026-10-06-distributed-mesh-consolidated.md` |
| Decisions D1-D24 | `specs/issue-475-distributed-mesh/decisions.md` |
| Phase 2 plan (done) | `plans/2026-10-06-cluster-module-phase2.md` |
| Phase 3-4 plan (next) | `plans/2026-10-06-distributed-mesh-phase3-4.md` |
| Blog entry | `blog/2026-10-06-mdp03-the-ring-that-routes.md` |
| Epic issue | casehubio/qhorus#475 |

## Project state

- **Project branch:** `issue-475-distributed-mesh` — 12 commits (Phase 1 safety net + Phase 2 cluster module + post-discussion fixes)
- **Workspace branch:** `issue-475-distributed-mesh`
- Build: green (`mvn clean install` — all modules, all tests pass)
- Cluster module: 41 tests, 15 source files

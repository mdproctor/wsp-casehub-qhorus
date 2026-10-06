# Session Handover — 2026-10-06

## What happened

Implemented Phase 2 of the distributed mesh epic (#475) — the `casehub-qhorus-cluster` module — then significantly evolved the architecture through design discussion.

### Phase 2 implementation (7 commits)

Built the cluster module with:
- **ConsistentHashRing** — SHA-256 consistent hashing, 128 virtual nodes, immutable snapshots
- **ClusterManager** — peer state machine (ALIVE/SUSPECT/DEAD), quorum enforcement, graceful shutdown
- **HeartbeatService** — peer polling, failure detection, ring hash mismatch warning
- **WriteRoutingDecorator + ChannelManagerDecorator** — CDI decorators for write routing with two-gate activation and fallback-to-local
- **InternalMeshResource + ClusterHealthResource** — internal RPC and health/topology endpoints
- **RelayConfig + RelayProducer** — `@IfBuildProperty` gated CDI wiring

One API change to existing qhorus: `preAssignedId` field on `ChannelCreateRequest` (optional, backward-compatible).

41 cluster module tests. Full project build green.

### Architecture evolution (D18-D24)

Post-implementation discussion fundamentally reframed the architecture:

**D18 — Topology maturity ladder:** Four deployment levels, each additive. DB locks (Phase 1) are the correctness constant. Each level adds operational/performance optimisation without changing correctness.

| Level | Topology | Config required |
|-------|----------|----------------|
| 1. Dev | Embedded, LLMs direct | Datasource only |
| 2. Dedicated server | Separate qhorus, LLMs direct | Datasource only |
| 3. Relays | Any-channel read/write | `relay.enabled` + peers |
| 4. Channel ownership | Dynamic ownership on relays | Level 3 + `relay.routing=dynamic` |

**D20 — Fallback-to-local:** Proxy failure → local dispatch with DB locks. HA gap eliminated.

**D21 — Self-healing:** Agents reconnect to any relay on failure. No coordination needed.

**D22 — Connection multiplexing:** Relay batches DB connections, pg_notify subscriptions, local fan-out.

**D23 — Relay purpose:** Conversation multiplexer over shared PostgreSQL, not a router or correctness layer.

**D24 — Relay depth modes:** Shallow (cache active conversations) and full (mirror complete history for search offloading). New full nodes start shallow, backfill in background, transition once caught up.

### Post-implementation code changes

Three changes aligning the cluster module with the evolved architecture:
1. Config prefix renamed `casehub.qhorus.cluster` → `casehub.qhorus.relay`
2. Two-gate activation: `relay.enabled` for Level 3, `relay.routing` for Level 4
3. Fallback-to-local in `WriteRoutingDecorator` — catches proxy errors, dispatches locally

### Consolidated spec

Wrote `2026-10-06-distributed-mesh-consolidated.md` superseding the two earlier specs. Marked old specs with ⛔ SUPERSEDED headers.

## Key decisions

- The relay is a **conversation multiplexer**, not a router or correctness layer (D23)
- DB locks are the correctness constant across all four topology levels (D18)
- Channel ownership (Level 4) is a performance optimisation, not core functionality
- Dev topology must be zero-config beyond datasource (D18 Level 1)
- `preAssignedId` on `ChannelCreateRequest` may be unnecessary for dynamic ownership — evaluate when implementing Level 4

## Next action

The consolidated spec (§11) defines the updated phase ordering:

| Priority | Phase | Why |
|----------|-------|-----|
| 1 | REST API gaps | Level 2 needs these — LLMs can't connect without REST endpoints |
| 2 | Relay infrastructure split | Separate Level 3 (peer awareness, health) from Level 4 (routing) cleanly |
| 3 | Shallow caching | The relay's primary purpose — conversation multiplexing IS the product |
| 4 | Mesh wiring + Dockerfile | Level 2-3 deployment artifact |
| 5 | Dynamic ownership | Level 4 performance optimisation |

Start with Phase 3 (REST API gaps) — these are the five endpoints that currently only exist as MCP tools.

## References

| Artifact | Path |
|----------|------|
| Consolidated spec | `specs/issue-475-distributed-mesh/2026-10-06-distributed-mesh-consolidated.md` |
| Decisions D1-D24 | `specs/issue-475-distributed-mesh/decisions.md` |
| Phase 2 plan | `plans/2026-10-06-cluster-module-phase2.md` |
| Blog entry | `blog/2026-10-06-mdp03-the-ring-that-routes.md` |
| Old design spec | `specs/issue-475-distributed-mesh/2026-10-06-distributed-mesh-design.md` (⛔ superseded) |
| Old cluster spec | `specs/issue-475-distributed-mesh/2026-10-06-cluster-module-design.md` (⛔ superseded) |
| Epic issue | casehubio/qhorus#475 |

## Project state

- **Project branch:** `issue-475-distributed-mesh` — 11 commits (Phase 1 + Phase 2 + post-discussion fixes)
- **Workspace branch:** `issue-475-distributed-mesh`
- Build: green (`mvn clean install` — all modules, all tests pass)
- Cluster module: 41 tests, 15 source files

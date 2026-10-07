# Distributed Qhorus Mesh — Consolidated Design Specification

**Issue:** casehubio/qhorus#475
**Date:** 2026-10-06
**Status:** Draft (supersedes `2026-10-06-distributed-mesh-design.md` and `2026-10-06-cluster-module-design.md`)

## 1. What This Is

Qhorus is a governance layer for multi-agent AI communication. The relay is a **conversation multiplexer** — it batches and compresses multiplexed agent conversations over shared PostgreSQL. Correctness lives in the database (row-level locks). The relay adds efficiency: connection pooling, pg_notify batching, local fan-out, and optional conversation caching.

The relay is never a required intermediary. Every agent can write directly to PostgreSQL with DB locks. The relay is optional infrastructure that improves performance and operational visibility.

## 2. Topology Ladder

Four deployment levels. Each is additive — no level requires the next. Every level degrades gracefully to the one below it.

### Level 1: Dev (embedded)

```yaml
# application.properties — that's it
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:qhorus
```

The LLM application includes `casehub-qhorus` as a Maven dependency. Qhorus runs in-process. H2 in-memory for dev, PostgreSQL for anything persistent. No relay, no clustering, no configuration beyond the datasource.

**This is the default.** A developer adds the dependency, writes their agent, and runs it. Zero qhorus-specific configuration required.

### Level 2: Dedicated server

```yaml
# On the qhorus server
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://db:5432/qhorus
```

A standalone qhorus process (`MeshApp`) running on its own. LLMs connect to it over HTTP (REST, MCP-over-SSE, A2A JSON-RPC, or WebSocket). The server is the existing `mesh/` module — `@QuarkusMain`, PostgreSQL-backed, all protocol adapters active.

**What it adds over Level 1:** Operational separation (qhorus lifecycle independent of the LLM), connection pooling (one DB pool shared by all connected agents), health endpoints.

**Configuration:** Datasource. That's it. No clustering config, no peer lists.

### Level 3: Relays

```yaml
# On each relay
casehub.qhorus.relay.enabled=true
casehub.qhorus.relay.peers=relay-1:8080,relay-2:8080,relay-3:8080
casehub.qhorus.relay.node-id=relay-1
```

Multiple relay instances, all reading and writing to the same PostgreSQL. Any relay serves any channel — no ownership, no routing. Each relay multiplexes its connected agents' conversations through a shared DB connection pool.

**What it adds over Level 2:** Horizontal scaling of the conversation multiplexer. Local fan-out (messages between agents on the same relay skip pg_notify). Heartbeat-based health monitoring across relays. Topology endpoints for ops.

**Self-healing:** When a relay dies, its agents reconnect to any other relay and continue immediately. No coordination needed — any relay serves any channel. When the relay recovers, agents reconnect and the relay resumes multiplexing. No state to rebuild — PostgreSQL holds everything.

### Level 4: Channel ownership

```yaml
# On each relay (in addition to Level 3 config)
casehub.qhorus.relay.routing=dynamic
```

Relays earn ownership of channels they write to most frequently. The owning relay handles writes without DB lock contention (the `synchronized` keyword works within a single JVM). Non-owner relays proxy writes to the owner.

**What it adds over Level 3:** Uncontended write locks on hot channels. Reduced proxy hops for agents that consistently use the same relay.

**Fallback-to-local:** When the owning relay is unreachable, the calling relay executes the write locally with DB locks. The Phase 1 `SELECT FOR UPDATE` changes guarantee correctness during the brief overlap. The HA gap disappears — agents never see 503 for a dead relay.

**Dynamic ownership:** Ownership is earned, not assigned by a hash ring. The first or most-frequent writer to a channel becomes the owner. Ownership can drift as workload patterns change. The hash ring remains available as a static alternative for predictable routing.

## 3. Configuration Summary

The guiding principle: **each level adds one block of config to the previous level.** Nothing is removed, nothing changes meaning.

| Level | Config required | Everything else |
|-------|----------------|-----------------|
| 1. Dev | `datasource` | Defaults |
| 2. Server | `datasource` | Defaults |
| 3. Relays | `datasource` + `relay.enabled` + `relay.peers` + `relay.node-id` | Defaults |
| 4. Ownership | Level 3 + `relay.routing=dynamic` | Defaults |

### Full configuration reference

```java
@ConfigMapping(prefix = "casehub.qhorus.relay")
public interface RelayConfig {

    @WithDefault("false")
    boolean enabled();                    // Level 3+: activate relay mode

    Optional<List<String>> peers();       // Level 3+: host:port list of peer relays

    Optional<String> nodeId();            // Level 3+: this relay's identity (default: hostname)

    @WithDefault("none")
    String routing();                     // Level 4: "none" | "dynamic" | "hash-ring"

    @WithDefault("shallow")
    String depth();                       // "shallow" | "full" (see §5)

    @WithDefault("128")
    int virtualNodes();                   // Hash ring vnodes (Level 4, hash-ring mode only)

    @WithDefault("3s")
    Duration heartbeatInterval();         // Level 3+: peer polling interval

    @WithDefault("2")
    int heartbeatMissThreshold();         // Level 3+: misses before DEAD

    @WithDefault("30s")
    Duration drainTimeout();              // Level 3+: graceful shutdown drain period

    @WithDefault("true")
    boolean quorumEnforced();             // Level 4: minority partition write rejection

    @WithDefault("10s")
    Duration proxyTimeout();              // Level 4: proxy call timeout
}
```

### Environment variable mapping

Quarkus maps `casehub.qhorus.relay.*` to `CASEHUB_QHORUS_RELAY_*` automatically:

```bash
# Level 3 relay
CASEHUB_QHORUS_RELAY_ENABLED=true
CASEHUB_QHORUS_RELAY_PEERS=relay-1:8080,relay-2:8080,relay-3:8080
CASEHUB_QHORUS_RELAY_NODE_ID=relay-1

# Level 4 with dynamic ownership
CASEHUB_QHORUS_RELAY_ROUTING=dynamic

# Full-depth caching relay
CASEHUB_QHORUS_RELAY_DEPTH=full
```

## 4. Write Model

### The constant: DB locks

Phase 1 added three `SELECT FOR UPDATE` changes to the qhorus runtime:

| Subsystem | Lock target | Purpose |
|-----------|------------|---------|
| Merkle hash chain | `LedgerMerkleFrontier` row | Prevents concurrent frontier corruption |
| LAST_WRITE channels | Last message for sender | Prevents lost updates during ownership transfer |
| Commitment transitions | Commitment by correlationId | Prevents double-fulfillment |

These locks are row-level and per-channel. One channel's lock does not block another's. At conversation pace (messages per minute), contention is unmeasurable. These locks serve all four topology levels — they are the correctness foundation that never changes.

### The optimization: channel ownership (Level 4 only)

When `routing=dynamic`, the `WriteRoutingDecorator` intercepts `MessageDispatcher.dispatch()` and `ChannelManager` mutations. It routes writes to the channel owner for uncontended locks. When the owner is unreachable, the decorator falls back to local execution with DB locks.

```
dispatch(channelId):
  1. Check quorum (Level 4 only)
  2. Find owner
  3. If local → delegate to MessageService
  4. If remote → try proxy to owner
  5. If proxy fails → delegate to local MessageService (DB locks catch it)
```

At Levels 1-2, no routing decorator exists — writes go directly to the service layer. At Level 3, decorators exist but pass through directly (`routing=none` — no ownership check, no proxy). At Level 4, decorators actively route writes to the channel owner. DB locks handle correctness at every level.

## 5. Relay Depth Modes

Same binary, different config. The depth mode determines what the relay caches locally.

### Shallow (default)

Caches active conversations. Pages in history from PostgreSQL on demand. Optimised for real-time multiplexing — low memory footprint, instant startup.

- Recent messages in active channels cached in-memory
- History requests beyond the cache window query PostgreSQL
- pg_notify subscriptions for channels with connected agents
- Cache eviction: LRU per channel, configurable max entries

### Full

Mirrors complete conversation history from PostgreSQL. Serves search, analytics, and history queries locally without hitting the primary database.

- Background sync from PostgreSQL on startup (channel by channel, batched)
- Real-time messages arrive via pg_notify (no gap between historical and live)
- Relay starts as shallow immediately (serves real-time traffic from moment one)
- Transitions to full once historical sync catches up
- Serves read queries from local state; writes still go through PostgreSQL

**Sync strategy for a new full relay:**

```
1. Start as shallow → immediately serving real-time traffic
2. Background: SELECT channels ORDER BY last_activity DESC
3. For each channel: batch-copy messages (1000 at a time, oldest first)
4. pg_notify delivers new messages in real-time during sync
5. When all channels synced → mark as full, begin serving history queries
6. No cluster pause. No network disruption. Rest of cluster unaware.
```

**Sync status:** The relay's health endpoint reports `depth_status: syncing` during background sync, `depth_status: ready` once complete. Search queries during sync return a `X-Qhorus-Sync-Status: partial` response header so clients know results may be incomplete. Relays in `syncing` state are excluded from desired-state's search-relay pool until they reach `ready`.

**Ops scaling knob:** Need more search throughput? Spin up full relays. Need more real-time multiplexing? Spin up shallow relays. Same image, different `CASEHUB_QHORUS_RELAY_DEPTH` value.

## 6. Connection Multiplexing

Without relays, N agents each maintain their own PostgreSQL connection pool (5-10 connections each). At 100 agents, that's 500-1000 DB connections — beyond PostgreSQL's default `max_connections` (100-200).

With relays, all agents on a machine share one relay's connection pool (~20-30 connections). The relay also batches pg_notify subscriptions — one LISTEN per channel per relay instead of per agent.

| Resource | Without relay (100 agents) | With 3 relays |
|----------|---------------------------|---------------|
| DB connections | 500-1000 | 60-90 |
| LISTEN subscriptions | 1 per channel per agent | 1 per channel per relay |
| Message fan-out | pg_notify for every pair | Local for co-located, pg_notify for cross-relay |

## 7. Self-Healing

The architecture self-heals at every level because the relay is never a correctness requirement.

| Failure | What happens | Recovery |
|---------|-------------|----------|
| Relay dies, others available | Agents reconnect to a different relay, continue immediately | Automatic — any relay serves any channel |
| All relays die (embedded agents) | Agents with embedded qhorus fall back to direct PostgreSQL writes | Automatic — DB locks handle correctness |
| All relays die (REST-only agents) | Agents retry relay connections until a relay recovers | Automatic — no direct DB fallback for HTTP-only clients |
| Relay recovers | Agents reconnect, relay resumes multiplexing | Automatic — no state to rebuild |
| Network partition (Level 4) | Minority partition rejects writes (quorum); majority continues | Automatic — fallback-to-local for proxied writes |
| PostgreSQL down | Everything stops — PostgreSQL is the single source of truth | Manual — restore database, all relays and agents resume |

Agent relay discovery can be:
- Static configuration (agent knows relay addresses)
- DNS-based (relay addresses resolved via service discovery)
- Desired-state managed (ops provisions relay addresses into agent config)

## 8. Desired-State Integration

Each topology level maps to a desired-state resource schema. Ops declares the topology; `casehub-desiredstate` provisions and reconciles it.

```yaml
# Level 1: Dev — no resource needed, qhorus is embedded as a library dependency

# Level 2: Dedicated server — relay.enabled not set (default false)
kind: QhorusMesh
spec:
  replicas: 1
  datasource:
    host: db.internal
    port: 5432
    database: qhorus

# Level 3: Relays — relay.enabled=true, relay.routing=none (default)
kind: QhorusMesh
spec:
  replicas: 3
  placement: per-cluster        # or per-machine
  relay:
    enabled: true
    depth: shallow
    peers: auto                 # desired-state populates from replica addresses
  datasource:
    host: db.internal
    port: 5432
    database: qhorus

# Level 4: Relays with ownership — relay.routing=dynamic
kind: QhorusMesh
spec:
  replicas: 3
  placement: per-cluster
  relay:
    enabled: true
    depth: shallow
    routing: dynamic
    peers: auto
  datasource:
    host: db.internal
    port: 5432
    database: qhorus

# Full-depth relay for search offloading
kind: QhorusMesh
spec:
  replicas: 1
  placement: per-cluster
  relay:
    enabled: true
    depth: full
    peers: auto
  datasource:
    host: db.internal
    port: 5432
    database: qhorus
```

The `spec.relay.*` fields map directly to `casehub.qhorus.relay.*` config properties — no translation layer between desired-state and runtime config.

Level transitions are schema changes — update `routing` or `depth`, desired-state reconciles. No redeployment, no data migration.

## 9. Protocol Adapters

Four protocol adapters, all stateless, all delegating to the same service layer. Available at Levels 2+. An agent picks whichever protocol suits its capabilities.

| Protocol | Endpoint | Use case |
|----------|----------|----------|
| REST + SSE | `POST /api/channels/{id}/messages`, `GET /api/channels/{id}/events` | Universal — any HTTP client |
| MCP-over-SSE | `quarkus-mcp-server-sse` | Claudony fleet — MCP tool-calling |
| A2A JSON-RPC | `POST /a2a` | Interop — A2A ecosystem agents |
| WebSocket | `/qhorus/ws/channels/{channelId}` | Real-time UI — browser dashboards |

At Level 1 (embedded), agents call the service layer directly via CDI. No HTTP endpoints needed.

## 10. Health and Topology Endpoints

Available when the relay is a standalone process.

| Endpoint | Level | Returns |
|----------|-------|---------|
| `GET /health/live` | 2+ | Process running |
| `GET /health/ready` | 2+ | Database connected, serving requests (503 if minority partition at Level 4) |
| `GET /health/cluster` | 3+ | Relay status, cluster size, ring hash, peer states, depth_status |
| `GET /admin/topology` | 3+ | Full node list with status, addresses, last heartbeat |

Desired-state reads these for convergence checks.

## 11. What's Built

### Phase 1: Runtime safety net (complete)

Three `SELECT FOR UPDATE` changes to `casehub-qhorus` runtime. Committed on `issue-475-distributed-mesh`. Serves all four topology levels.

### Phase 2: Cluster module (complete)

`casehub-qhorus-cluster` module with:
- `ConsistentHashRing` — SHA-256 consistent hashing, 128 virtual nodes, immutable snapshots
- `ClusterManager` — peer state machine (ALIVE/SUSPECT/DEAD), quorum, ring management
- `HeartbeatService` — peer polling, failure detection, ring hash mismatch warning
- `WriteRoutingDecorator` + `ChannelManagerDecorator` — CDI decorators for write routing
- `WriteProxyClient` — proxy stub (HTTP wiring in Phase 4)
- `InternalMeshResource` — `/internal/dispatch`, `/internal/heartbeat`, `/internal/leave`
- `ClusterHealthResource` — `/health/cluster`, `/admin/topology`
- `RelayConfig` + `RelayProducer` — `@IfBuildProperty` gated CDI wiring

41 unit tests. Full project build green.

Config prefix is `casehub.qhorus.relay` (aligned with the relay framing). Two-gate activation: `relay.enabled` for Level 3 infrastructure, `relay.routing` for Level 4 routing. `WriteRoutingDecorator` includes fallback-to-local on proxy failure (D20).

### Phase 3: REST API gaps (next)

Operations that currently exist only as MCP tools need REST equivalents for Level 2+ (non-embedded) usage:

| Operation | Current | Needed |
|-----------|---------|--------|
| Register agent | MCP `meshRegister` | `POST /api/instances` |
| Deregister agent | MCP `meshDeregister` | `DELETE /api/instances/{id}` |
| Send message | MCP `meshSendMessage` | `POST /api/channels/{id}/messages` |
| Check messages | MCP `meshCheckMessages` | `GET /api/channels/{id}/messages` |
| Discover peers | MCP `meshDiscoverPeers` | `GET /api/instances` |

### Phase 4: Mesh service wiring (after Phase 3)

Wire the cluster module into `mesh/` with PostgreSQL config, Dockerfile, production `application.properties`, and end-to-end multi-node testing.

### Phase 5: Relay depth modes (after Phase 4)

Shallow caching (LRU per channel) and full mirroring (background sync from PostgreSQL).

### Phase 6: Dynamic ownership (after Phase 5)

Ownership heuristics — first writer or most-frequent writer earns channel ownership. Replaces static hash ring as the default Level 4 strategy.

## 12. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Configuration complexity at higher levels | Each level adds one config block — nothing changes meaning between levels |
| PostgreSQL as single point of failure | Accepted — PostgreSQL replication and backup are ops concerns, not qhorus concerns |
| Full relay sync overwhelming PostgreSQL | Batch-copy with configurable batch size and throttle; sync is background, not blocking |
| Dynamic ownership oscillation (two relays fighting for ownership) | Hysteresis — ownership transfer requires sustained write majority, not a single message |
| Level 4 quorum too aggressive for small clusters | `quorum-enforced=false` disables it; 2-node clusters should use Level 3 |

## References

- Decisions D1-D24: `specs/issue-475-distributed-mesh/decisions.md`
- Phase 1 commits: `ccd4d0a0`, `bcdea284`, `236e9a63` on `issue-475-distributed-mesh`
- Phase 2 commits: `ba13d961` through `8932f24e` on `issue-475-distributed-mesh`
- `PostgresChannelActivityBroadcaster`: existing cross-node LISTEN/NOTIFY delivery
- `ChannelGateway.deliverRemote()`: cross-node message delivery entry point
- `MeshApp` + `MeshService`: existing standalone mesh relay
- Consistent hashing: Karger et al., "Consistent Hashing and Random Trees" (1997)
- `casehub-desiredstate`: ops reconciliation module

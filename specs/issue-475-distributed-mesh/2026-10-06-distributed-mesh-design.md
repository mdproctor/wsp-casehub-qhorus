# ⛔ SUPERSEDED — Do not use

> **This spec is superseded by [`2026-10-06-distributed-mesh-consolidated.md`](2026-10-06-distributed-mesh-consolidated.md).**
> Retained for git history only. The consolidated spec reframes the architecture
> around the topology maturity ladder and relay-as-multiplexer model (D18-D24).

---

# Distributed Qhorus Mesh — Design Specification

**Issue:** casehubio/qhorus#475
**Date:** 2026-10-06
**Status:** Superseded

## 1. Problem Statement

Qhorus is currently an embedded Quarkus extension library. To use it, an application adds `casehub-qhorus` as a Maven dependency and runs the mesh in-process. This works for Claudony (which embeds qhorus directly) but prevents:

- **Independent scaling** — the mesh scales with the host application, not independently
- **Multi-application participation** — agents in different JVMs share channels only via the database, without real-time push
- **Non-JVM clients** — any agent must run inside a Quarkus app
- **Operational independence** — the mesh lifecycle is coupled to the host

The distributed mesh transforms qhorus into a standalone network service that agents connect to over standard protocols.

## 2. Architecture Overview

The distributed mesh is a standalone Quarkus service that owns channels, messages, commitments, and the normative ledger. Agents connect over HTTP. The service layer is the existing qhorus runtime — no rewrite.

### 2.1 Layers

```
┌─────────────────────────────────────────────────────────────┐
│                     Mesh Service Node                       │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Protocol Adapter Layer                    │  │
│  │  REST+SSE │ MCP-over-SSE │ A2A JSON-RPC │ WebSocket  │  │
│  └─────────────────────┬─────────────────────────────────┘  │
│                        │                                    │
│  ┌─────────────────────▼─────────────────────────────────┐  │
│  │              Service Layer (unchanged)                 │  │
│  │  MessageService │ ChannelService │ CommitmentService   │  │
│  │  InstanceService │ LedgerWriteService │ DataService    │  │
│  └─────────────────────┬─────────────────────────────────┘  │
│                        │                                    │
│  ┌──────────┐  ┌───────▼──────────┐  ┌─────────────────┐  │
│  │ Cluster  │  │   Store SPIs     │  │   Delivery      │  │
│  │ Manager  │  │ (PostgreSQL JPA) │  │   Subsystem     │  │
│  └──────────┘  └──────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Key Principle: Shared PostgreSQL, Partitioned Writes

All nodes connect to the same PostgreSQL instance (or cluster). There is no custom state transfer protocol. Data lives in one place — the database. The distributed aspect is about **write routing** (which node handles writes to which channel) and **subscriber fan-out** (which node pushes to which connected clients).

This is not a distributed database. It is a distributed application layer over a shared database.

## 3. Protocol Adapter Layer

Four protocol adapters, all stateless, all delegating to the same service layer. An agent picks whichever protocol suits its capabilities.

### 3.1 REST + SSE (universal path)

**Writes:** RESTful endpoints for all mutations — `POST /api/channels`, `POST /api/channels/{id}/messages`, `POST /api/instances`, etc. These are the existing `ChannelResource`, `CausalGraphResource`, `AgentCardResource`, plus new endpoints that currently only exist as MCP tools.

**Reads:** Standard GET endpoints with pagination (`afterId`, `limit`).

**Real-time push:** SSE endpoint `GET /api/channels/{id}/events` — the agent opens a persistent connection and receives `MessageReceivedEvent` as server-sent events. Catch-up via `Last-Event-ID` header.

**Contract:** OpenAPI spec auto-generated from Quarkus REST annotations. This is the canonical API definition — all other protocols are mappings of this surface.

### 3.2 MCP-over-SSE (Claudony fleet path)

The existing `MeshApi` @McpDomain tools — `meshRegister`, `meshSendMessage`, `meshCheckMessages`, etc. Runs on `quarkus-mcp-server-sse`. No change needed. Claudony agents connect here and use MCP tool-calling to interact with channels.

MCP-over-SSE is HTTP+SSE with MCP JSON-RPC framing. Not a separate transport — runs on the same HTTP server.

### 3.3 A2A JSON-RPC (interop path)

The existing `A2AResource` at `POST /a2a`. Handles `message/send`, `tasks/get`, `tasks/cancel`. SSE streaming for long-poll task monitoring. Agent Cards at `/.well-known/agent.json`.

### 3.4 WebSocket (real-time UI path)

The existing `websocket-observer` module. Browser dashboards and monitoring UIs connect here. Catch-up replay via `lastEventId` query parameter. Server-side buffering during catch-up.

### 3.5 Gap Analysis — New REST Endpoints Needed

The following operations currently exist only as MCP tools and need REST equivalents:

| Operation | MCP tool | New REST endpoint |
|-----------|----------|-------------------|
| Register agent | `meshRegister` | `POST /api/instances` |
| Deregister agent | `meshDeregister` | `DELETE /api/instances/{id}` |
| Send message | `meshSendMessage` | `POST /api/channels/{id}/messages` |
| Check messages | `meshCheckMessages` | `GET /api/channels/{id}/messages` |
| Discover peers | `meshDiscoverPeers` | `GET /api/instances` |
| Correct message | `correctMessage` | `POST /api/channels/{id}/messages/{msgId}/correct` |
| Retract message | `retractMessage` | `POST /api/channels/{id}/messages/{msgId}/retract` |
| Erase content | `eraseMessageContent` | `POST /api/ledger/{entryId}/erase` |

Existing REST endpoints (`ChannelResource`, `AgentCardResource`, `A2AResource`, `CausalGraphResource`) remain unchanged.

## 4. Cluster Manager

The cluster manager is the core new subsystem. It handles ring management, write routing, heartbeat, and failure recovery.

### 4.1 Ring Management

**Configuration:** Peer list provided via environment variable:
```
QHORUS_PEERS=mesh-1.internal:8080,mesh-2.internal:8080,mesh-3.internal:8080
QHORUS_NODE_ID=mesh-1
```

**Hash ring construction:** Each node computes the same ring from the sorted peer list. Virtual nodes (configurable, default 128 per physical node) distribute channels evenly. The hash function is SHA-256 truncated to 64 bits, applied to `channelId.toString()`.

**Deterministic:** Given the same peer list, every node computes the same ring. No coordination needed for ring agreement — the peer list is the single source of truth.

### 4.2 Write Routing

When a protocol adapter receives a write request (message dispatch, channel creation, commitment mutation):

1. Compute the owning node for the target channel: `ring.owner(channelId)`
2. If this node is the owner → execute locally via the service layer
3. If another node owns it → proxy the request to the owner via internal HTTP

```
Agent → Node 2 (non-owner)
         │
         ├─ ring.owner(channelId) → Node 1
         │
         └─ HTTP POST http://node-1:8080/internal/dispatch
              │
              └─ Node 1 executes MessageService.dispatch()
                   │
                   └─ Response flows back: Node 1 → Node 2 → Agent
```

**The agent doesn't know or care which node owns the channel.** Any node accepts any request. The proxy is transparent.

**Internal HTTP path:** `/internal/*` endpoints are not exposed externally. They use the same serialisation as the REST API but bypass authentication (node-to-node trust within the cluster).

### 4.3 Heartbeat Protocol

**Mechanism:** Each node HTTP-polls every other node's health endpoint at a configurable interval (default 3 seconds).

```
GET /internal/heartbeat
→ 200 OK { "nodeId": "mesh-1", "timestamp": "...", "ringHash": "..." }
```

The `ringHash` is the SHA-256 of the sorted peer list — detects ring disagreement (e.g., one node got a config update the others didn't).

**Failure detection state machine:**

```
ALIVE ──(miss 1)──→ SUSPECT ──(miss 2)──→ DEAD ──(ring recalculation)
  ↑                     │
  └──(heartbeat ok)─────┘
```

- **SUSPECT** (1 missed heartbeat): logged, no action. Transient network blips are common.
- **DEAD** (2 consecutive misses, ~6-10 seconds): node removed from ring, channels reassigned. Logged at WARN. Ops notified via health endpoint status change.
- **ALIVE** (heartbeat received from a previously DEAD node): node re-added to ring, channels rebalanced back. This handles container restarts — ops restarts the container, the node comes back, rejoins the ring.

**Configurable:**
- `QHORUS_HEARTBEAT_INTERVAL_MS` — default 3000
- `QHORUS_HEARTBEAT_MISS_THRESHOLD` — default 2 (DEAD after 2 misses)

### 4.4 Ownership Transfer

When a node transitions to DEAD:

1. Surviving nodes recalculate the ring excluding the dead node
2. Channels previously owned by the dead node are now owned by their next node on the ring (consistent hashing property — only the dead node's channels move)
3. The new owner begins serving those channels immediately — no state transfer needed because PostgreSQL holds all state
4. Connected subscribers on the dead node are lost — clients reconnect to another node and re-establish their SSE/WebSocket connections
5. The delivery subsystem (`DeliveryService`) on the new owner picks up undelivered messages via cursor reconciliation

**Graceful shutdown:**

1. Node receives SIGTERM
2. Announces departure via `POST /internal/leave` to all peers
3. Peers recalculate ring immediately (no heartbeat delay)
4. Node drains in-flight requests (configurable timeout, default 30 seconds)
5. Node exits

### 4.5 Split-Brain Prevention

**Quorum rule:** A node serves write requests only if it can reach a majority of the configured peer list (> N/2). A minority partition returns `503 Service Unavailable` for all writes, with a `Retry-After` header.

Reads are always served — stale reads are acceptable (eventual consistency for cross-channel queries).

**Single-node mode:** When `QHORUS_PEERS` is not set or contains only this node, quorum checks are disabled. The mesh runs as a single node with no clustering overhead.

## 5. Write Model — Hybrid Hash Ring + DB Safety Net

### 5.1 The Problem

Code trace of the qhorus runtime revealed three subsystems that require single-writer-per-channel semantics:

1. **Merkle hash chain** (`QhorusLedgerEntryRepository.save()`): A `synchronized` method that reads the Merkle frontier, computes the append, and replaces the frontier. Two JVMs writing concurrently to the same channel would read the same frontier and produce divergent Merkle trees — corrupting the tamper-evidence chain.

2. **LAST_WRITE channel semantic** (`MessageService.dispatch()`): Reads the last message for the sender, checks version, overwrites. Read-modify-write without DB-level optimistic locking.

3. **Correction count enforcement**: Counts existing corrections, checks against limit, inserts. TOCTOU race under concurrent writes.

Additionally, commitment state transitions (`findByCorrelationId → check state → transition`) lack `SELECT FOR UPDATE`, creating a theoretical double-fulfillment race.

### 5.2 The Solution: Belt and Suspenders

**Hash ring (suspenders):** Routes all writes to the channel owner. In normal operation (99.9% of the time), only one node writes to any given channel. The existing JVM-local `synchronized` and in-process serialisation work correctly. No DB lock contention, no extra latency.

**DB-level locks (belt):** Three surgical changes to the qhorus runtime add DB-level protection for the edge cases where two nodes briefly write to the same channel (ownership transfer, split-brain healing):

| Subsystem | Change | Purpose |
|-----------|--------|---------|
| `QhorusLedgerEntryRepository.save()` | `SELECT FOR UPDATE` on `LedgerMerkleFrontier` before read-modify-write | Prevents concurrent frontier corruption |
| `MessageService.dispatch()` LAST_WRITE path | Add `@Version` to `MessageEntity`, WHERE clause with version check, retry on `OptimisticLockException` | Prevents concurrent overwrite of last message |
| `CommitmentStore` | New `findByCorrelationIdForUpdate()` with `SELECT FOR UPDATE` for the fulfill/decline path | Prevents double-fulfillment |

### 5.3 Why Both?

- **Hash ring without DB locks:** Normal operation is fine. But during ownership transfer (node A dies, node B takes over), there's a brief window where both might write. The Merkle chain would corrupt.

- **DB locks without hash ring:** Works correctly but every write pays the DB lock acquisition cost. Under load, channels with frequent writes see lock contention in `pg_stat_activity`. And every future read-modify-write pattern in qhorus needs the same audit — ongoing maintenance burden.

- **Both together:** Hash ring eliminates contention in normal operation (locks uncontended = free). DB locks catch edge cases. The two are independently verifiable — you can prove correctness from the DB locks alone, and prove performance from the hash ring alone.

### 5.4 Subsystem Concurrency Summary

| Subsystem | Multi-node safe without changes? | Fix | Risk if unfixed |
|-----------|--------------------------------|-----|-----------------|
| Message.id (PostgreSQL sequence) | Yes | None | — |
| Sequence allocator (MERGE INTO) | Yes (PostgreSQL row lock) | None | — |
| Merkle hash chain | **No** | `SELECT FOR UPDATE` on frontier | Merkle tree corruption |
| LAST_WRITE semantic | **No** | `@Version` optimistic lock | Silent message loss |
| Correction count | **No** (TOCTOU) | Advisory — accept the race | Exceeds correction limit by 1 |
| Commitment transitions | Marginal | `findByCorrelationIdForUpdate()` | Double-fulfillment |

## 6. Subscriber Fan-Out

When a message is written to a channel, it needs to reach all agents subscribed to that channel — including agents connected to other nodes.

### 6.1 Local Subscribers

The owning node fans out to its locally-connected SSE and WebSocket subscribers via the existing `MessageObserverDispatcher` and `ChannelGateway.fanOut()`. No change needed.

### 6.2 Remote Subscribers

For subscribers connected to other nodes, the `postgres-broadcaster` module (already in qhorus) sends a PostgreSQL `NOTIFY` signal. Remote nodes receive the `LISTEN` callback, load the message from PostgreSQL, and push to their local subscribers.

```
Node 1 (owner)                    Node 2 (subscriber connected here)
    │                                  │
    ├─ MessageService.dispatch()       │
    ├─ messageStore.put()              │
    ├─ fanOut() → local subscribers    │
    ├─ pg_notify('channel_activity')   │
    │                                  │
    │         ┌── LISTEN callback ─────┤
    │         │                        │
    │         └── load message ────────┤
    │                                  ├─ fanOut() → local subscribers
    │                                  │
```

This is the existing `PostgresChannelActivityBroadcaster` — already production-ready with self-notification filtering.

### 6.3 Delivery Guarantees

- **SSE/WebSocket:** Best-effort with catch-up. If a subscriber disconnects and reconnects, they provide `Last-Event-ID` and receive missed messages.
- **AT_LEAST_ONCE backends** (A2A outbound, push notification, Slack): The `DeliveryService` pump handles per-backend cursor tracking and retry. Each node runs its own delivery pump — the pump serves channels owned by that node.

## 7. Ops Interface

### 7.1 Health Endpoints

**Liveness:** `GET /health/live` — process is running, accepting connections.

**Readiness:** `GET /health/ready` — database connected, ring computed, node serving requests. Returns 503 if in minority partition (split-brain).

**Cluster status:** `GET /health/cluster`
```json
{
  "nodeId": "mesh-1",
  "status": "UP",
  "clusterSize": 3,
  "expectedSize": 3,
  "ring": {
    "members": ["mesh-1", "mesh-2", "mesh-3"],
    "suspectNodes": [],
    "deadNodes": [],
    "ringHash": "a1b2c3..."
  },
  "partitions": {
    "ownedChannels": 142,
    "totalChannels": 428
  },
  "connections": {
    "sseSubscribers": 23,
    "websocketClients": 5,
    "mcpSessions": 12
  }
}
```

### 7.2 Topology API

`GET /admin/topology` — full partition map for drift detection:
```json
{
  "nodes": [
    {
      "nodeId": "mesh-1",
      "address": "mesh-1.internal:8080",
      "status": "ALIVE",
      "ownedChannelCount": 142,
      "connectedAgents": 15,
      "lastHeartbeat": "2026-10-06T00:30:00Z"
    }
  ],
  "rebalanceHistory": [
    {
      "timestamp": "2026-10-05T23:45:00Z",
      "event": "NODE_DEPARTED",
      "nodeId": "mesh-4",
      "channelsReassigned": 37
    }
  ]
}
```

### 7.3 Environment Variable Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `QHORUS_NODE_ID` | hostname | This node's identity in the cluster |
| `QHORUS_PEERS` | (none) | Comma-separated `host:port` peer list. Empty = single-node mode |
| `QHORUS_HEARTBEAT_INTERVAL_MS` | 3000 | Heartbeat poll interval |
| `QHORUS_HEARTBEAT_MISS_THRESHOLD` | 2 | Missed heartbeats before DEAD |
| `QHORUS_VIRTUAL_NODES` | 128 | Virtual nodes per physical node on hash ring |
| `QHORUS_DRAIN_TIMEOUT_SECONDS` | 30 | Graceful shutdown drain period |
| `QHORUS_AUTH_MODE` | oidc | `oidc` or `api-key` |
| Standard Quarkus DB config | — | `quarkus.datasource.*` for PostgreSQL |

## 8. Authentication

### 8.1 Platform OIDC (primary)

The mesh service declares `quarkus-oidc` as a dependency. Agents authenticate with bearer tokens from the organisation's identity provider. The existing `OidcCurrentPrincipal` (already in qhorus, `@Priority(300+)`) extracts identity and tenancy from the token. No new auth code.

Configuration: standard Quarkus OIDC properties (`quarkus.oidc.auth-server-url`, `quarkus.oidc.client-id`).

### 8.2 API Key Fallback (development/scripts)

For agents that can't perform OIDC flows (dev scripts, simple CLI tools), the mesh accepts a static API key via `Authorization: Bearer <key>` header. Keys are configured via environment variable or database table.

A simple `ApiKeyCurrentPrincipal` resolves the key to a principal with a fixed tenancy. Lower priority than `OidcCurrentPrincipal` — OIDC tokens take precedence when both are present.

### 8.3 Node-to-Node Trust

Internal endpoints (`/internal/*`) are not authenticated. They are not exposed externally — network-level isolation (firewall rules, Kubernetes network policies) provides the trust boundary. Ops controls network access.

## 9. Changes to Existing Qhorus Code

The distributed mesh requires three surgical changes to the qhorus runtime and one new module.

### 9.1 Runtime Changes (casehub-qhorus)

**9.1.1 Merkle frontier locking:**
File: `QhorusLedgerEntryRepository.save()`
Change: Before the `findBySubjectId` call, execute `SELECT ... FOR UPDATE` on the `LedgerMerkleFrontier` row. This serialises concurrent frontier updates at the DB level. In single-node mode (current), the existing `synchronized` already prevents concurrency — the DB lock is uncontended and adds negligible overhead.

**9.1.2 LAST_WRITE optimistic locking:**
File: `MessageEntity.java`
Change: Add `@Version` annotation to the existing `version` field. In `MessageService.dispatch()` LAST_WRITE path, catch `OptimisticLockException` and retry (bounded, max 3 retries).

**9.1.3 Commitment pessimistic locking:**
File: `CommitmentStore` interface + `JpaCommitmentStore`
Change: Add `findByCorrelationIdForUpdate(String correlationId)` method that uses `SELECT ... FOR UPDATE`. Called from the fulfill/decline/fail paths in `CommitmentService`.

### 9.2 New Module: casehub-qhorus-cluster

A new Maven module in the qhorus project containing:

- `ClusterManager` — ring management, heartbeat, failure detection
- `ConsistentHashRing` — hash ring implementation with virtual nodes
- `WriteRouter` — proxy layer for non-owner writes
- `ClusterHealthResource` — `/health/cluster` and `/admin/topology` endpoints
- `HeartbeatService` — scheduled heartbeat poller

This module depends on `casehub-qhorus-api` only. It does not depend on `casehub-qhorus` runtime — it wires into the runtime via CDI events and configuration.

### 9.3 Mesh Module Updates (casehub-qhorus-mesh)

The existing `mesh/` module gains:

- Dependency on `casehub-qhorus-cluster`
- PostgreSQL driver dependency (replacing H2 for production)
- Production `application.properties` with env var substitution
- Dockerfile for container image builds

## 10. Deployment Scenarios

### 10.1 Single Node (development/small teams)

```
QHORUS_PEERS=          # empty = single-node mode
QHORUS_AUTH_MODE=api-key
```

No clustering overhead. Hash ring disabled. All writes local. Equivalent to the current embedded model but running as a standalone process.

### 10.2 Three-Node Cluster (production)

```
# On each node:
QHORUS_PEERS=mesh-1:8080,mesh-2:8080,mesh-3:8080
QHORUS_NODE_ID=mesh-N  # unique per node
```

All nodes connect to the same PostgreSQL. Channels distributed across nodes via consistent hashing. Heartbeat monitors health. One node failure → channels rebalance to survivors within ~6 seconds.

### 10.3 Scaling Up

Add the new node to `QHORUS_PEERS` on all nodes (ops handles this via desired-state). Nodes detect the new peer via heartbeat, add it to the ring, and rebalance channels. Only ~1/N channels move to the new node (consistent hashing property).

## 11. Testing Strategy

### 11.1 Unit Tests

- `ConsistentHashRing` — ring construction, lookup, node add/remove, virtual node distribution
- `WriteRouter` — owner detection, proxy decision, retry on failure
- `HeartbeatService` — state machine (ALIVE → SUSPECT → DEAD → ALIVE)

### 11.2 Integration Tests

- Single-node mode — all existing qhorus tests run unchanged against the mesh service
- Two-node cluster (H2, same JVM) — verify proxy routing, subscriber fan-out
- Ownership transfer — simulate node death, verify channel reassignment and continued delivery

### 11.3 Chaos Tests

- Kill a node mid-write — verify Merkle chain integrity via DB lock safety net
- Network partition (majority/minority) — verify minority refuses writes
- Rapid ring changes — add/remove nodes in quick succession, verify convergence

## 12. Implementation Phases

### Phase 1: Runtime Safety Net
Add the three DB-level lock changes to `casehub-qhorus` runtime. These are safe to land independently — they add correctness without changing behavior in single-node mode.

### Phase 2: Cluster Module
Build `casehub-qhorus-cluster` — hash ring, heartbeat, write router. Test in isolation with unit tests.

### Phase 3: REST API Gaps
Add the missing REST endpoints (Section 3.5) to the runtime. These are independently useful even without clustering.

### Phase 4: Mesh Service
Wire cluster module into the mesh, add PostgreSQL config, Dockerfile, health endpoints. End-to-end testing with a multi-node cluster.

### Phase 5: Ops Integration
Health/topology endpoints, graceful shutdown, env var configuration. Ops wraps the container image with desired-state infrastructure.

## References

- Kafka partition leader pattern — [Partition Leaders and Followers | AlgoMaster](https://algomaster.io/learn/kafka/partition-leaders-and-followers)
- Consistent hashing — [Consistent Hashing: Architecture Blueprint | CodingPanCake](https://www.codingpancake.com/2026/09/consistent-hashing-distributed-systems.html)
- PostgreSQL MVCC and concurrency — [PostgreSQL Concurrency with MVCC | Heroku](https://devcenter.heroku.com/articles/postgresql-concurrency)
- Database-centric event architecture — [Practical Event-Driven Microservices | ResearchGate](https://www.researchgate.net/publication/398564040)
- Serializable transactions — [Serializable Transactions in Distributed Systems | Aerospike](https://aerospike.com/blog/serializable-transactions-distributed-systems/)
- `QhorusLedgerEntryRepository.save()` — synchronized read-modify-write on Merkle frontier
- `QhorusSequenceAllocator` — REQUIRES_NEW MERGE with JVM-local synchronization
- `MessageService.dispatch()` — LAST_WRITE, correction, commitment enforcement gates
- `PostgresChannelActivityBroadcaster` — existing cross-node delivery via LISTEN/NOTIFY
- Ops team feedback — boundary contract for container/health/topology interface

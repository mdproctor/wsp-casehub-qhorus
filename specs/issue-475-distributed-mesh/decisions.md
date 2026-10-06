# Distributed Mesh — Design Decisions

## D1: Transport model

**Choice:** Multi-protocol over shared service layer — REST+SSE as foundation, MCP-over-SSE, A2A JSON-RPC, and WebSocket as protocol layers on the same HTTP server.
**Alternatives:**
- WebSocket only — lower latency but poor proxy/LB support, harder for stateless tool-use
- MCP-over-SSE only — limits adoption to MCP-capable clients
**Rationale:** MCP-over-SSE is HTTP+SSE with a specific message format. Not a separate transport. All protocols are facades over the same MessageDispatcher/ChannelManager. REST/GraphQL is the universal client interface — any language calls it directly.
**Trade-offs:** Multiple protocol endpoints to maintain and test, but they share the service layer so the incremental cost is low.
**Sources:** Existing A2AResource, ChannelResource, MeshApi, WebSocket observer modules
**Exploration:** quick
**Status:** captured

## D2: Deployment model

**Choice:** Single node + PostgreSQL to start. Architecture designed for multi-node from day one. Ops manages provisioning via desired-state infrastructure.
**Alternatives:**
- Multi-node cluster from day one — higher reliability but significantly more complexity upfront
- Embedded sidecar per agent host — maximises locality but complicates consistency
**Rationale:** Start simple, scale when needed. postgres-broadcaster already provides the cross-node primitive.
**Trade-offs:** Single point of failure until multi-node is implemented. Acceptable for initial deployment.
**Sources:** Ops team feedback: "Build the mesh, ops will wrap it"
**Exploration:** quick
**Status:** captured

## D3: Storage abstraction

**Choice:** All persistence behind SPIs (Store interfaces). PostgreSQL as default production implementation.
**Alternatives:** None considered — this is the existing qhorus pattern.
**Rationale:** Already proven across 15+ Store interfaces. InMemory implementations for testing, JPA for production.
**Trade-offs:** None — continuing established pattern.
**Sources:** Existing *Store interfaces in api/store/
**Exploration:** quick
**Status:** captured

## D4: Consistency model

**Choice:** Eventual consistency + total order per channel. Single writer per channel via partition ownership.
**Alternatives:**
- Strong consistency everywhere — linearizable but kills throughput (cross-node coordination per message)
- Best-effort with client-side dedup — simplest server-side but pushes complexity to clients
**Rationale:** Conversation order within a channel is what matters. Cross-channel ordering is not semantically meaningful for agent communication.
**Trade-offs:** Cross-channel queries (e.g., "all messages from agent X across channels") may see temporarily inconsistent views.
**Sources:** Channel-per-conversation design in qhorus
**Exploration:** quick
**Status:** captured

## D5: Channel partitioning

**Choice:** Consistent hashing on channel ID.
**Alternatives:**
- Explicit assignment via config — full control but doesn't scale, requires manual management
- Dynamic load-based rebalancing — best throughput distribution but complex (migration protocol, state transfer, split-brain prevention)
**Rationale:** Deterministic, no coordination needed to route. Adding/removing nodes reassigns a fraction of channels. Well-understood (Dynamo, Cassandra, Kafka pattern).
**Trade-offs:** Hot channels (high-traffic single channel) can't be split across nodes. Acceptable since agent channels are typically low-throughput.
**Sources:** Consistent hashing literature (Karger et al.)
**Exploration:** quick
**Status:** captured

## D6: Replication strategy

**Choice:** Messages partitioned (live on owning node's PostgreSQL). Channel metadata, instance registry, and commitment state replicated to all nodes.
**Alternatives:**
- Everything partitioned — lightest replication but discovery/routing requires cross-node queries
- Everything replicated — strongest consistency but write amplification scales linearly with cluster size
**Rationale:** Metadata is small and changes infrequently (channel creation, agent registration). Messages are large and append-only. Replicating metadata lets any node route and answer discovery without cross-node hops.
**Trade-offs:** Metadata replication adds a background sync protocol. Stale metadata during network partitions could route to the wrong node (client retries handle this).
**Sources:** postgres-broadcaster module (existing cross-node primitive)
**Exploration:** quick
**Status:** captured

## D7: Failure handling

**Choice:** Heartbeat between nodes + ownership transfer. When a node is unresponsive, surviving nodes recalculate the hash ring and the next node absorbs the dead node's channels. PostgreSQL-backed state survives — new owner reads from the same DB.
**Alternatives:**
- Passive detection only — clients get errors, ops restarts, channels unavailable until restart
- Raft-based leader election — strongest guarantees but heavyweight for a messaging system
**Rationale:** Heartbeat is simple and fast. PostgreSQL durability means no state transfer needed — just ring recalculation. Ops restarts the container; qhorus handles the membership change.
**Trade-offs:** Brief unavailability during ring recalculation (seconds). False positives from network glitches could trigger unnecessary transfers (configurable timeout mitigates).
**Sources:** Ops team feedback: "qhorus handles cluster membership changes, ops restarts containers"
**Exploration:** quick
**Status:** captured

## D8: Authentication

**Choice:** Platform OIDC primary, API key fallback for development/scripting. Both resolve to CurrentPrincipal.
**Alternatives:**
- API key only — simpler but weaker security model
- Mutual TLS — strongest but complex certificate lifecycle
- No auth initially — fastest to ship but insecure
**Rationale:** casehub-platform already has OIDC support. OidcCurrentPrincipal already exists in qhorus. No reason to ship a weaker mechanism. API key fallback for agents that can't do OIDC flows (dev scripts, simple tools).
**Trade-offs:** Requires identity provider infrastructure. Mitigated by API key fallback for development.
**Sources:** OidcCurrentPrincipal in qhorus runtime, casehub-platform OIDC
**Exploration:** quick
**Status:** captured

## D9: Client interface

**Choice:** Java API module (existing) as the typed client. REST and GraphQL APIs as the universal interface for any language. Non-Java SDKs deferred — any agent can call REST/GraphQL directly.
**Alternatives:**
- Python + TypeScript SDKs at launch — wider reach but premature without clear non-JVM agent use cases
- OpenAPI-generated SDKs — mechanical but still maintenance burden
**Rationale:** Non-casehub agents wouldn't understand speech acts or commitments anyway. The protocol intelligence lives in the LLM's context (via Claudony) or in the Java library. REST/GraphQL is sufficient as a transport layer for any client.
**Trade-offs:** Non-Java developers write raw HTTP calls. Mitigated by OpenAPI spec for documentation and future generation.
**Sources:** Discussion about speech act protocol awareness requirements
**Exploration:** quick
**Status:** captured

## D10: Ops interface contract

**Choice:** Container image accepting env vars (peer list, storage config, transport ports). Health endpoints reporting cluster membership status. Topology API for drift detection (expected vs actual cluster size, partition health).
**Alternatives:** None — this is the contract ops specified.
**Rationale:** Ops team explicitly defined the interface: "Don't design for ops integration. Design the best distributed messaging system you can. Ops will wrap whatever you build."
**Trade-offs:** None — this is a boundary agreement, not a design choice.
**Sources:** Ops team feedback
**Exploration:** quick
**Status:** captured

## D11: Write model — hybrid hash ring + DB safety net

**Choice:** Hash ring routes writes to the channel owner (single-writer fast path). DB-level pessimistic locks (`SELECT FOR UPDATE`) added to Merkle frontier and LAST_WRITE reads as a correctness safety net for edge cases (ownership transfer, split-brain healing).
**Alternatives:**
- Pure single-writer (no DB locks) — works in normal operation but brief overlap during ownership transfer could corrupt Merkle chain
- Pure any-node-writes (DB serialisation only) — achieves same serialisation via DB locks but at higher latency (lock acquisition round-trip), contention under load, and ongoing audit burden for every new read-modify-write pattern
**Rationale:** Code trace revealed three hard blockers for multi-node writes: (1) Merkle hash chain — `synchronized save()` protects read-modify-write on frontier, JVM-local only; (2) LAST_WRITE semantic — read-modify-write with version but no DB-level CAS; (3) Correction count — TOCTOU race. Hash ring eliminates contention in normal operation. DB locks catch edge cases. The two are independently verifiable.
**Trade-offs:** Hash ring adds a proxy hop for non-owner writes (~1ms). DB locks add overhead only when contended (edge cases). Three surgical changes to existing qhorus code: (1) `SELECT FOR UPDATE` on `LedgerMerkleFrontier`, (2) `@Version` on `MessageEntity` for LAST_WRITE, (3) `findByCorrelationIdForUpdate()` for commitment transitions.
**Depends on:** D5 (consistent hashing), D4 (total order per channel)
**Sources:** QhorusLedgerEntryRepository.save():90, MessageService.dispatch():371-432, QhorusSequenceAllocator:19-27; Kafka partition leader pattern; PostgreSQL MVCC/row locking docs
**Exploration:** deep-analysis (code trace + internet research + first-principles verification)
**Status:** captured

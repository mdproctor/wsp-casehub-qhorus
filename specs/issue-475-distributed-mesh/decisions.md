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

**Choice:** All data in shared PostgreSQL. Writes partitioned by channel ownership (hash ring determines the single writer). All nodes can read all data — channel metadata, instance registry, and commitment state are globally visible without custom replication.
**Alternatives:**
- Per-node databases with custom replication — stronger data locality but requires state transfer protocol and creates consistency headaches
- Everything replicated via application-level sync — unnecessary when all nodes share one database
**Rationale:** Shared PostgreSQL eliminates the need for a custom replication protocol entirely. "Partitioned" means write-ownership (which node does the INSERT), not data locality. Every node reads from the same tables. This is a distributed application layer over a shared database, not a distributed database.
**Trade-offs:** Single PostgreSQL is a bottleneck at extreme scale. Mitigated by PostgreSQL read replicas for read-heavy workloads, and by the fact that agent messaging is conversation-pace, not streaming-pace.
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

---

# Phase 2 — Cluster Module Implementation Decisions

## D12: Write routing interception point

**Choice:** CDI decorator on `MessageDispatcher` and `ChannelManager`. All callers (REST, MCP, A2A) get routing transparently — single interception point, no adapter changes needed. The decorator checks channel ownership via `ClusterManager.owner(channelId)` before delegating to the real service.
**Alternatives:**
- JAX-RS filter on write endpoints — catches HTTP but MCP tools bypass JAX-RS (direct CDI calls), creating two interception points
- Explicit routing in each protocol adapter — most control but every adapter must be modified and future adapters could forget to route
**Rationale:** CDI decorator is the standard Quarkus interception mechanism. It sits at the service layer boundary, catching all callers regardless of protocol. Single place to maintain, impossible to bypass accidentally.
**Trade-offs:** Decorator adds one method call per dispatch even for local writes (isLocal check). Negligible overhead — a hash lookup and string comparison.
**Depends on:** D5 (consistent hashing), D11 (hybrid write model)
**Sources:** Quarkus CDI decorator documentation, existing MessageDispatcher interface in api/message/
**Exploration:** quick
**Status:** captured

## D13: Internal node-to-node transport

**Choice:** Quarkus REST Client with type-safe JAX-RS interface. The owning node exposes `/internal/dispatch` and `/internal/channel` endpoints. Serialization via Jackson (consistent with existing REST). Quarkus generates the client at build time.
**Alternatives:**
- Plain Java HttpClient — no build-time generation, more boilerplate, harder to evolve the internal API
- gRPC with protobuf — lower latency but adds protobuf dependency, .proto files, separate port; overkill for conversation-pace throughput
**Rationale:** Quarkus REST Client is idiomatic, type-safe, and consistent with the existing stack. The internal API surface is small (dispatch, channel create, heartbeat) so the client interface is trivial. Build-time generation eliminates boilerplate.
**Trade-offs:** JSON serialization overhead vs gRPC binary (~0.1ms per message at expected payload sizes). Acceptable.
**Depends on:** D12 (decorator proxies to remote node via this client)
**Sources:** Quarkus REST Client documentation, InternalMeshClient interface design
**Exploration:** quick
**Status:** captured

## D14: Module structure

**Choice:** Plain library module (`casehub-qhorus-cluster`), not a Quarkus extension. Standard Maven module with runtime CDI beans, activated by classpath presence + config gate. No deployment module needed — no build-time processing (`@BuildStep`), no native image registration.
**Alternatives:**
- Quarkus extension (runtime + deployment) — follows core qhorus pattern, deployment module can validate config at build time, but adds a deployment module with boilerplate that isn't needed for this module's concerns
**Rationale:** The cluster module's behavior is purely runtime — hash ring computation, heartbeat polling, proxy forwarding. No build-time code generation or native image configuration needed. Follows the pattern of `connector-backend`, `slack-channel`, and other optional modules.
**Trade-offs:** No build-time config validation — misconfigured peer lists detected at startup, not build time. Acceptable since cluster config comes from environment variables at deployment time.
**Sources:** Existing optional modules: connector-backend/, slack-channel/, a2a-outbound/
**Exploration:** quick
**Status:** captured

## D15: Routing scope

**Choice:** Route all channel mutations (dispatch + create + delete + pause/resume + config changes) to the channel owner. Instance registration, data service, and other global operations remain unrouted (any node serves them).
**Alternatives:**
- Route MessageDispatcher.dispatch() only — simpler decorator, but channel config mutations during ring transitions could cause brief inconsistencies (one node modifying allowedWriters while another dispatches)
**Rationale:** Channel ownership means owning ALL writes to that channel. Creating a channel on one node but dispatching to it on another creates a window where the channel exists in the DB but the owning node hasn't initialized its gateway registry for it. Routing creation to the eventual owner eliminates this.
**Trade-offs:** Two decorators to maintain (MessageDispatcher + ChannelManager) instead of one. The ChannelManager decorator is thin — same pattern, same proxy client.
**Depends on:** D5 (consistent hashing), D12 (decorator approach)
**Sources:** ChannelCreateHelper.java (creation + gateway init coupling), ChannelGateway.initChannel()
**Exploration:** quick
**Status:** captured

## D16: Channel creation routing key

**Choice:** Pre-generate UUID on the routing node, route by it. The decorator generates a UUID before routing, passes it to the owning node, which uses it as the channel ID. The channel lands on the right node from birth.
**Alternatives:**
- Route by channel name hash — the channel name is the stable routing key, but name-based routing diverges from ID-based routing for messages, creating two different hash ring lookups and potential owner mismatch
**Rationale:** All post-creation routing uses channelId (UUID). If creation routes by a different key (name), the channel could be created on node A but owned by node B for all future writes. Pre-generating the UUID ensures the channel is created on its permanent owner.
**Trade-offs:** Requires `ChannelCreateRequest` to accept an optional pre-assigned ID. Minor API surface change.
**Depends on:** D15 (routing scope includes creation), D5 (consistent hashing on channelId)
**Sources:** ChannelCreateRequest.java, ChannelCreateHelper.java
**Exploration:** quick
**Status:** captured

## D17: Cluster activation mechanism

**Choice:** Config-gated CDI beans. `casehub.qhorus.cluster.enabled=true` activates clustering. When absent or false, all cluster beans are disabled via `@IfBuildProperty` — the decorator, heartbeat, health endpoints don't exist. Zero overhead in single-node mode.
**Alternatives:**
- Classpath presence only — adding the jar activates clustering; simpler but the decorator always wraps dispatch even in single-node mode, adding a code path that's never needed
**Rationale:** The mesh app always includes the cluster module on its classpath, but not every deployment needs clustering (dev, small teams). Config gate ensures zero overhead when clustering isn't needed — no decorator, no heartbeat scheduler, no health endpoints.
**Trade-offs:** Build-time property (`@IfBuildProperty`) means clustering can't be toggled at runtime — requires restart. Acceptable since cluster membership is a deployment-time decision.
**Sources:** QhorusConfig pattern, existing @IfBuildProperty usage in qhorus
**Exploration:** quick
**Status:** captured

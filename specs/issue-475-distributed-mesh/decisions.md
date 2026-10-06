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

---

# Phase 2b — Topology and HA Decisions

## D18: Topology maturity ladder

**Choice:** Four deployment levels, each additive. DB locks (Phase 1) are the correctness constant across all levels. Each level adds operational and performance optimisation without changing the correctness model.

| Level | Topology | What it adds |
|-------|----------|-------------|
| 1. Dev | Local server, LLMs direct | Simplest — single JVM, `synchronized` + DB locks |
| 2. Dedicated server | Separate qhorus server, LLMs connect directly | Operational separation, connection pooling |
| 3. Relays | Relay per machine or cluster, any-channel read/write | Local fan-out, caching, fewer DB connections, health endpoints |
| 4. Channel ownership | Dynamic ownership on relays | Uncontended locks, no proxy hops for owner |

No level requires the next. Every level degrades gracefully to the one below it. The relay is optional infrastructure — never a required intermediary for correctness. Channel ownership (level 4) is a performance optimisation that activates only when configured, with fallback-to-local on proxy failure so the HA gap disappears.
**Alternatives:**
- Fixed "all nodes must cluster" architecture — forces level 4 complexity on dev/small deployments
- Relay as mandatory intermediary — creates an availability dependency the system doesn't need
**Rationale:** The workload is conversation-pace. At that scale, DB locks handle correctness with unmeasurable contention. Relays add operational value (health, topology, local fan-out) without being correctness-critical. Channel ownership earns its complexity only at scale.
**Trade-offs:** More deployment configurations to test and document. Mitigated by each level being a strict superset of the previous.
**Sources:** Session discussion re-examining hash ring necessity and HA trade-offs
**Exploration:** deep-analysis (session dialogue challenging the Phase 2 architecture)
**Status:** captured

## D19: Desired-state schema per topology level

**Choice:** Each topology level maps to a desired-state resource schema. Ops declares the topology they want; `casehub-desiredstate` provisions and reconciles it. Level transitions are schema changes, not redeployments.

| Level | Schema key | Ops provisions |
|-------|-----------|----------------|
| 1. Dev | `mode: embedded` | Nothing — qhorus is a library dependency |
| 2. Dedicated server | `mode: server` | One qhorus container + PostgreSQL |
| 3. Relays | `mode: relay`, peer list, placement | N relay containers, shared PostgreSQL, health monitoring |
| 4. Channel ownership | `mode: relay`, `routing: dynamic` | Same as 3 + ownership heuristic config |

Health and topology endpoints are available at levels 2+. Desired-state reads them for convergence.
**Alternatives:**
- Manual ops configuration without desired-state schemas — viable but doesn't integrate with the casehub ops model
**Rationale:** The ops contract ("build the mesh, ops will wrap it") is best served by declarative schemas that the existing `casehub-desiredstate` reconciliation loop can manage. Each level's schema is configuration, not architecture.
**Depends on:** D18 (topology maturity ladder), D10 (ops interface contract)
**Sources:** casehub-desiredstate module, session discussion
**Exploration:** quick
**Status:** captured

## D20: Relay fallback-to-local on proxy failure

**Choice:** When `WriteRoutingDecorator` fails to proxy a write to the channel owner (connection error), it executes the write locally using DB locks instead of returning 503. The hash ring is the performance fast path; DB locks are the correctness fallback. The HA gap (6-10s of write failures during ownership transfer) disappears.
**Alternatives:**
- Return 503 and let client retry — simpler decorator but creates an availability gap during node failure
- Primary + secondary ownership — instant failover but creates concurrent-writer complexity that DB locks already solve
**Rationale:** Phase 1 DB locks were designed for the concurrent-writer edge case. The fallback-to-local path exercises exactly that case. Row-level locks are per-channel — one channel's contention doesn't block another's. At conversation pace, the contention during the fallback window (~6-10s) is unmeasurable.
**Depends on:** D11 (hybrid write model), D18 (topology ladder — level 4 only)
**Sources:** Phase 1 SELECT FOR UPDATE implementation, session HA discussion
**Exploration:** deep-analysis (session dialogue)
**Status:** captured

## D21: Self-healing relay topology

**Choice:** LLMs are not bound to a specific relay. When a relay dies, connected LLMs reconnect to any available relay and continue immediately. At level 3 (any-channel relays), no ownership transfer or coordination is needed — the new relay serves the channel directly. At level 4 (channel ownership), the new relay writes locally with DB lock fallback until the ring recalculates. If all relays die, LLMs degrade to direct PostgreSQL writes (level 2). When relays recover, LLMs reconnect and resume relay-mediated communication. No state to rebuild — PostgreSQL holds everything.
**Alternatives:**
- Sticky relay assignment (LLM bound to a specific relay) — simpler client config but creates a single point of failure per LLM
**Rationale:** The relay is optional infrastructure. Making relay selection dynamic means the topology self-balances under failure and recovery without operator intervention. Desired-state only needs to ensure enough relays are running — LLMs find them.
**Depends on:** D18 (topology maturity ladder), D20 (fallback-to-local)
**Sources:** Session discussion on HA and self-healing
**Exploration:** quick
**Status:** captured

## D22: Relay as connection multiplexer

**Choice:** Relays multiplex LLM connections to PostgreSQL. Without relays, N LLMs each maintain their own connection pool (5-10 connections each), hitting PostgreSQL's `max_connections` limit at scale. With a local relay per machine, all LLMs share one relay's pool (~20-30 connections). Relays also batch pg_notify subscriptions (one LISTEN per channel per relay, not per LLM) and cache recent messages for local fan-out without DB round-trips.
**Alternatives:**
- PgBouncer as infrastructure — achieves connection pooling but doesn't get the pg_notify batching or message caching benefits
- Increase PostgreSQL max_connections — scales to a point but degrades performance (each connection consumes ~10MB of shared memory)
**Rationale:** Connection multiplexing is a natural consequence of the relay architecture, not an add-on. It provides the same value as PgBouncer (connection pooling) plus application-level optimisations (LISTEN batching, message cache) that a generic connection pooler cannot offer.
**Depends on:** D18 (topology maturity ladder — level 3+)
**Sources:** PostgreSQL max_connections documentation, PgBouncer comparison, session discussion
**Exploration:** quick
**Status:** captured

## D23: Relay purpose statement

**Choice:** The relay is a **conversation multiplexer** — not a router, not a correctness layer. Its role is to batch and compress multiplexed conversations between agents and the shared PostgreSQL. Many LLM-to-channel conversations are compressed into fewer DB connections, fewer LISTEN subscriptions, and local fan-out for co-located agents. Correctness lives in PostgreSQL (DB locks). Channel ownership (level 4) is an optional write-path optimisation, not the relay's defining function.
**Alternatives:**
- "Distributed application layer" — the original framing; accurate but doesn't capture the relay's specific value
- "Write router" — describes level 4 only, misses the multiplexing value that justifies levels 2-3
**Rationale:** A clear purpose statement prevents scope creep. Every relay feature should pass the test: "does this improve conversation multiplexing?" If not, it belongs elsewhere.
**Depends on:** D18 (topology maturity ladder), D22 (connection multiplexing)
**Sources:** Session discussion distilling relay purpose
**Exploration:** quick
**Status:** captured

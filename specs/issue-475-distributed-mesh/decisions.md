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

**Choice:** Eventual consistency + total order per channel. Single writer per channel via partition ownership under normal operation; concurrent writers serialized by DB locks during network partitions (D11, D20).
**Alternatives:**
- Strong consistency everywhere — linearizable but kills throughput (cross-node coordination per message)
- Best-effort with client-side dedup — simplest server-side but pushes complexity to clients
**Rationale:** Conversation order within a channel is what matters. Cross-channel ordering is not semantically meaningful for agent communication. The "single writer" property is a performance optimisation via the hash ring, not a correctness invariant — DB locks (D11) guarantee serialisation regardless of how many nodes write concurrently.
**Trade-offs:** Cross-channel queries (e.g., "all messages from agent X across channels") may see temporarily inconsistent views. During network partitions, fallback-to-local (D20) creates concurrent writers on the same channel — correctness is maintained by DB locks, but callers should not assume single-writer as an invariant.
**Sources:** Channel-per-conversation design in qhorus
**Exploration:** quick
**Status:** revised (decision review R1-08)

## D5: Channel partitioning

**Choice:** Consistent hashing on channel ID.
**Alternatives:**
- Explicit assignment via config — full control but doesn't scale, requires manual management
- Dynamic load-based rebalancing — best throughput distribution but complex (migration protocol, state transfer, split-brain prevention)
**Rationale:** Deterministic, no coordination needed to route. Adding/removing nodes reassigns a fraction of channels. Well-understood (Dynamo, Cassandra, Kafka pattern).
**Trade-offs:** Hot channels (high-traffic single channel) can't be split across nodes. Acceptable since agent channels are typically low-throughput.
**Scope:** Applies at topology Level 4 (channel ownership) only. Levels 1–3 use any-node writes with DB lock serialisation (D18). D12, D15, and D16 depend on D5 but are only active when routing is enabled at Level 4.
**Sources:** Consistent hashing literature (Karger et al.)
**Exploration:** quick
**Status:** revised (decision review R1-13)

## D6: Replication strategy

**Choice:** All data in shared PostgreSQL. Writes partitioned by channel ownership (hash ring determines the single writer). All nodes can read all data — channel metadata, instance registry, and commitment state are globally visible without custom replication.
**Alternatives:**
- Per-node databases with custom replication — stronger data locality but requires state transfer protocol and creates consistency headaches
- Everything replicated via application-level sync — unnecessary when all nodes share one database
**Rationale:** Shared PostgreSQL eliminates the need for a custom write replication protocol. "Partitioned" means write-ownership (which node does the INSERT), not data locality. Every node reads from the same tables. This is a distributed application layer over a shared database, not a distributed database. Read-side caching (D31 background sync) is an application-level optimisation that populates local caches from PostgreSQL — it is a one-directional read cache, not a correctness replication protocol.
**Trade-offs:** Single PostgreSQL is a bottleneck at extreme scale. Mitigated by PostgreSQL read replicas for read-heavy workloads, and by the fact that agent messaging is conversation-pace, not streaming-pace.
**Sources:** postgres-broadcaster module (existing cross-node primitive)
**Exploration:** quick
**Status:** revised (decision review R1-14)

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
**Trade-offs:** Accepting the ops contract means qhorus cannot impose deployment constraints: minimum node count for quorum (D47), deployment ordering, or health prerequisites before accepting writes. The topology maturity ladder (D18) mitigates this by making each level self-contained and gracefully degradable.
**Sources:** Ops team feedback
**Exploration:** quick
**Status:** revised (decision review R1-18)

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

**Choice:** `@Alternative @Priority` bean displacement on `MessageDispatcher` and `ChannelManager` via CDI producer methods in `RelayProducer`. All callers (REST, MCP, A2A) get routing transparently — single interception point, no adapter changes needed. The replacement bean checks channel ownership via `ClusterManager.owner(channelId)` before delegating to the real service.
**Alternatives:**
- JAX-RS filter on write endpoints — catches HTTP but MCP tools bypass JAX-RS (direct CDI calls), creating two interception points
- CDI `@Decorator` with `@Delegate` — composes naturally via decoration chain, supports multiple decorators with `@Priority` ordering, but requires the delegate interface to be explicitly declared as a decorated type
- Explicit routing in each protocol adapter — most control but every adapter must be modified and future adapters could forget to route
**Rationale:** `@Alternative @Priority` bean displacement via `RelayProducer` producer methods is the standard Quarkus pattern for conditional bean replacement gated by `@IfBuildProperty`. It sits at the service layer boundary, catching all callers regardless of protocol. Single place to maintain, impossible to bypass accidentally.
**Trade-offs:** Bean displacement adds one method call per dispatch even for local writes (isLocal check). Negligible overhead. Unlike CDI `@Decorator` (which composes via decoration chain), `@Alternative @Priority` replaces the bean entirely — if a second wrapper is needed (e.g., metrics), the producer must explicitly compose the wrappers rather than relying on CDI decorator ordering.
**Depends on:** D5 (consistent hashing), D11 (hybrid write model)
**Sources:** RelayProducer.java (producer methods), existing MessageDispatcher interface in api/message/
**Exploration:** quick
**Status:** revised (decision review R1-11)

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
**Trade-offs:** Two bean replacements to maintain (MessageDispatcher + ChannelManager) instead of one. The ChannelManager replacement is thin — same pattern, same proxy client. `findOrCreate()` currently bypasses routing — it always calls the local delegate; this is acceptable because name-based lookup is idempotent and the first-writer creates the channel regardless of node. Remote config mutation proxying (pause/resume, constraint changes) is not yet wired — throws `UnsupportedOperationException` for non-local channels. This is Phase A audit scope (#484).
**Depends on:** D5 (consistent hashing), D12 (bean displacement approach)
**Sources:** ChannelCreateHelper.java (creation + gateway init coupling), ChannelGateway.initChannel()
**Exploration:** quick
**Status:** revised (decision review R1-05)

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

**Choice:** Config-gated CDI beans. `casehub.qhorus.relay.enabled=true` activates relay functionality. When absent or false, all relay beans are disabled via `@IfBuildProperty` — the routing decorator, heartbeat, health endpoints don't exist. Zero overhead in single-node mode.
**Alternatives:**
- Classpath presence only — adding the jar activates clustering; simpler but the decorator always wraps dispatch even in single-node mode, adding a code path that's never needed
**Rationale:** The mesh app always includes the cluster module on its classpath, but not every deployment needs relay functionality (dev, small teams). Config gate ensures zero overhead when relay is disabled — no routing decorator, no heartbeat scheduler, no health endpoints.
**Trade-offs:** Build-time property (`@IfBuildProperty`) means relay mode can't be toggled at runtime — requires restart. Acceptable since relay membership is a deployment-time decision.
**Sources:** RelayConfig (`@ConfigMapping(prefix = "casehub.qhorus.relay")`), RelayProducer (`@IfBuildProperty(name = "casehub.qhorus.relay.enabled")`)
**Exploration:** quick
**Status:** revised (decision review R1-10)

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

## D24: Relay depth modes — shallow and full

**Choice:** Two relay depth modes, same binary, different config.

| Mode | Caches | Serves | Startup | Use case |
|------|--------|--------|---------|----------|
| Shallow | Active conversations, pages in history on demand | Real-time multiplexing, local fan-out | Instant | Co-located with LLMs, conversation-pace workloads |
| Full | Complete conversation history, mirrored from PostgreSQL | Search, analytics, history queries | Background sync, then ready | Read-heavy workloads, reducing DB read pressure |

A new full relay starts as shallow (immediately useful), backfills historical data from PostgreSQL in background batches (channel by channel), receives real-time messages via pg_notify (no gap), and transitions to full once caught up. No cluster pause needed — the relay joins the cluster immediately and upgrades its role transparently.

Writes always go through PostgreSQL regardless of depth. The full relay is a read replica at the application layer — not a write replica. Correctness stays in the DB.
**Alternatives:**
- PostgreSQL streaming replication for read replicas — achieves read scaling but lacks application-aware caching (channel-scoped queries, message-type filtering, conversation-aware eviction)
- All relays full — simpler but wastes resources on relays whose value is real-time multiplexing, not history serving
**Rationale:** Different operational pressures require different relay profiles. Real-time agent coordination needs low-latency shallow relays. Search and analytics need full-history relays. Same binary means ops provisions one image and configures the role.
**Depends on:** D18 (topology maturity ladder — level 3+), D23 (relay as conversation multiplexer + cache)
**Sources:** Session discussion on read scaling and relay caching depth
**Exploration:** quick
**Status:** captured

---

# Phase 5 — Relay Depth Modes Implementation Decisions

## D25: Cache interception point

**Choice:** CDI decorator on `MessageStore` (wrapping both read and write paths). All callers — REST, MCP, A2A, WebSocket, projections — get caching transparently. Same interception pattern as `WriteRoutingDecorator` on `MessageDispatcher`. Cache miss falls through to the JPA store. Write interception captures dispatched messages for inline cache population.
**Alternatives:**
- Core/Resource layer interception — simpler scope but MCP tools and direct service-layer callers bypass the cache, requiring multiple interception points
- Dedicated `CachedMessageService` — explicit opt-in by callers gives most control but requires changing all read call sites and makes caching non-transparent
**Rationale:** The decorator pattern is proven in this codebase (`WriteRoutingDecorator`). Decorating `MessageStore` captures both reads (cache hits) and writes (cache population) in one place. Every caller that reads or writes messages goes through `MessageStore`, so the cache is comprehensive without caller changes.
**Trade-offs:** Decorator must handle all `MessageStore` methods, including `count()` and `distinctSendersByChannel()` which aren't cache-friendly. These pass through to JPA unchanged.
**Depends on:** D24 (relay depth modes)
**Sources:** WriteRoutingDecorator.java, MessageReader.java (api/store/), MessageStore.java (api/store/)
**Exploration:** quick
**Status:** captured

## D26: Cache data structure — per-channel message ring buffer

**Choice:** Caffeine cache keyed by `UUID` (channelId), value is a bounded ordered `NavigableMap<Long, Message>` (keyed by message ID). LRU eviction at the channel level — inactive channels evict before active ones. Supports `afterId` pagination natively via `tailMap(afterId, false)`. Configurable `max-messages-per-channel` (default 200) and `max-channels` (default 1000).
**Alternatives:**
- Full `MessageQuery` result cache (keyed by query hash) — higher hit rate for repeated identical queries but harder invalidation: any write to a channel must invalidate all cached query results touching that channel. Combinatorial explosion of cache keys.
- Per-channel complete state (all messages cached) — simplest consistency model but unbounded memory for channels with long history. Only appropriate for full mode.
**Rationale:** Per-channel ring buffer matches the access pattern: agents read recent messages in a channel, paginating forward. The `NavigableMap` supports the critical `afterId` query directly. Channel-level LRU means the cache naturally holds the hot working set — channels with active conversations stay cached, dormant channels evict.
**Trade-offs:** Complex queries (topic filter, type filter, content pattern) cannot be served purely from cache — they require post-filter on the ring buffer or fall through to JPA. Acceptable since these queries are uncommon in the real-time path.
**Depends on:** D25 (MessageStore decorator)
**Sources:** PresenceService.java (Caffeine pattern), MessageQuery.java (afterId pagination)
**Exploration:** quick
**Status:** captured

## D27: Cache population — post-commit for local writes

**Choice:** The decorator intercepts `MessageStore.put()` and registers a JTA `afterCompletion(STATUS_COMMITTED)` callback to add the message to the ring buffer after the transaction commits. This ensures the cache never contains uncommitted data — rollbacks (ledger write failure, enforcement gate, commitment conflict) do not pollute the cache.
**Alternatives:**
- Inline population (add to cache immediately in `put()`) — zero latency but cache may contain phantom messages from rolled-back transactions, violating the invariant that JPA is authoritative (identified in decision review R1-08)
- Populate via pg_notify only — simpler single mechanism but adds round-trip latency for local reads
**Rationale:** The relay that dispatches a message should serve it from cache as soon as possible, but never serve uncommitted data. `afterCompletion` fires immediately after commit, before the dispatch response returns — the latency cost is negligible while maintaining correctness.
**Trade-offs:** The decorator intercepts both read and write paths, making it a `MessageStore` decorator. Brief window between `put()` and commit where the message is not yet in cache — acceptable since `dispatch()` commits immediately.
**Depends on:** D25 (decorator approach), D26 (ring buffer structure)
**Sources:** MessageService.dispatch(), MessageObserverDispatcher (afterCompletion pattern — PP-20260608-07daa6)
**Exploration:** quick
**Status:** revised (decision review R1-08). Implementation gap: `CachingMessageStore.put()` uses inline population; `afterCompletion` callback not yet applied (Phase A audit #484 scope)

## D28: Remote cache invalidation — MessageObserver (CLUSTER scope)

**Choice:** The cache module implements `MessageObserver` with scope `CLUSTER`. When `deliverRemote()` fires CLUSTER-scoped observers, the cache observer receives the `MessageReceivedEvent` and adds the message to the ring buffer. This uses the existing observer infrastructure — no new notification channel or invalidation mechanism.

Note: the original design proposed piggybacking on `deliverRemote()`'s `messageStore.find()` call, but `deliverRemote()` uses `CrossTenantMessageStore.find()` — a separate interface hierarchy that a `MessageStore` decorator does not intercept (identified in decision review R1-02).
**Alternatives:**
- Decorate `CrossTenantMessageStore` as well — two decorators on unrelated interfaces, more surface area
- Proactive cache push via internal RPC — lower latency but couples relays at the cache layer
- `MessageStore.find()` piggyback — broken because `deliverRemote` uses `CrossTenantMessageStore`, not `MessageStore`
**Rationale:** CLUSTER-scoped observers already fire for every remote write via `deliverRemote()` → `MessageObserverDispatcher.dispatchClusterOnly()`. The observer fires after transaction commit, so the cache never contains uncommitted data. The `MessageReceivedEvent` carries all fields needed to construct a cache entry.
**Trade-offs:** `MessageReceivedEvent` does not carry the full `Message` object. The cache observer must load the full `Message` from JPA via `MessageStore.find(messageId)` rather than constructing an incomplete object from the event — incomplete cache entries (null `inReplyTo`, empty `artefactRefs`, zero `version`) cause incorrect `MessageQuery.matches()` filtering against cached remote messages. Loading from JPA on the observer path is acceptable — it's a single-row primary key read.
**Depends on:** D27 (post-commit for local), D26 (ring buffer)
**Sources:** MessageObserver.java (api/gateway/), MessageObserverDispatcher, ChannelGateway.deliverRemote(), CrossTenantMessageStore (decision review R1-02)
**Exploration:** quick
**Status:** revised (decision review R1-02, R1-12). Implementation gap: `CachePopulationObserver` constructs incomplete Message from event fields; JPA load not yet applied (Phase A audit #484 scope)

## D29: Cache miss behaviour — range check then fall-through

**Choice:** For `scan(MessageQuery)`, the decorator checks if the channel's ring buffer covers the requested range:
- **Hit:** `afterId` is within the buffer range (or absent, requesting latest) → serve from cache, apply filters in-memory
- **Miss:** `afterId` is before the buffer's earliest entry (requesting older history) → fall through to JPA store
- **Full mode exception:** after background sync completes, the cache holds all messages. Misses during sync fall through to JPA with a log warning.

For other methods: `find(Long id)` checks cache first, falls through on miss. `findRecent()` served from cache when buffer has enough entries. `count()`, `distinctSendersByChannel()`, `countByChannel()` pass through to JPA unchanged.
**Alternatives:**
- Always fall through on any filter beyond afterId — simpler but misses the common unfiltered read
- Negative cache — useful for full mode but adds complexity
**Rationale:** Range check is O(1). In-memory filtering on small buffers (200 messages) is fast. The decorator never serves stale data — JPA serves the authoritative answer on cache miss.
**Trade-offs:** In-memory filtering may return fewer results than `limit` requests. The caller paginates again, potentially hitting JPA. This is correct behaviour.
**Depends on:** D26 (ring buffer structure), D24 (shallow vs full modes)
**Sources:** MessageQuery.java (afterId, limit, filter fields), MessageQuery.matches()
**Exploration:** quick
**Status:** captured

## D30: Module structure — casehub-qhorus-cache

**Choice:** New Maven module `casehub-qhorus-cache`. Activated by classpath presence + config gate (`casehub.qhorus.cache.enabled`, default true). Independent of the cluster module.

Config prefix `casehub.qhorus.cache`:
- `enabled` (boolean, default true)
- `max-channels` (int, default 1000)
- `max-messages-per-channel` (int, default 200)
- `full-sync-batch-size` (int, default 1000)
- `full-sync-interval` (Duration, default 5s)
**Alternatives:**
- Inside cluster module — forces clustering onto Level 2 servers that only want caching
- Inside runtime module — always active, adds memory overhead to embedded Level 1 deploys
**Rationale:** Caching is orthogonal to clustering. A Level 2 server benefits from caching without clustering. Separate modules let ops compose features per deployment.
**Trade-offs:** Additional Maven module. Mitigated by established optional module pattern.
**Depends on:** D14 (module structure pattern), D24 (relay depth modes)
**Sources:** connector-backend/, slack-channel/, a2a-outbound/ (optional module pattern)
**Exploration:** quick
**Status:** captured

## D31: Full mode — background sync strategy

**Choice:** `@Scheduled` driver loads channels ordered by `last_activity DESC`, batch-copies messages (default 1000 per batch). Real-time messages arrive via pg_notify during sync — no gap. Relay starts as shallow immediately, transitions to full once all channels synced.

Sync progress: in-memory `SyncCursor` tracks per-channel last-synced ID and overall state (`SYNCING` → `READY`). Health endpoint reports `depth_status`.

Process:
1. Query channels ordered by most recent activity
2. For each channel: load messages in batches (`afterId` cursor, ascending)
3. Populate ring buffer (unbounded in full mode)
4. Sleep `full-sync-interval` between batches
5. When all channels synced, set status to READY

**Alternatives:**
- On-demand sync (cache-miss triggers channel sync) — latency spikes on first access
- Single bulk load at startup — relay unavailable until sync completes
**Rationale:** Background sync with activity ordering serves the most valuable channels first. Real-time messages are never delayed.
**Trade-offs:** History queries for not-yet-synced channels fall through to PostgreSQL during sync. Documented via health endpoint.
**Depends on:** D24 (shallow/full), D26 (ring buffer — unbounded for full), D29 (fall-through)
**Sources:** Consolidated spec §5 (sync strategy)
**Exploration:** quick
**Status:** captured

## D32: Write-frequency tracking component

**Choice:** Standalone `WriteFrequencyTracker` bean in the cluster module — a new `@ApplicationScoped` bean maintaining a `ConcurrentHashMap<UUID, ChannelWriteStats>` with per-node write counts in a sliding window. `WriteRoutingDecorator` calls `tracker.recordWrite(channelId)` after each dispatch. CDI-free POJO with `Clock` injection for testability.
**Alternatives:**
- Inside `ClusterManager` — co-locates with hash ring but violates single responsibility (already handles peer lifecycle, ring management, quorum)
- Inside `WriteRoutingDecorator` — avoids a new class but mixes routing logic with frequency tracking
**Rationale:** Separation keeps each component focused and independently testable. The tracker is a pure counter; the decorator consults it for routing decisions.
**Trade-offs:** One more class to wire, but the separation pays for itself in test clarity.
**Sources:** ClusterManager.java, WriteRoutingDecorator.java, RelayConfig.java
**Exploration:** quick
**Status:** captured

## D33: Ownership resolution strategy

**Choice:** Layered `DynamicOwnershipResolver` wrapping the existing `ConsistentHashRing`. For each channel: if the `WriteFrequencyTracker` has data and a node exceeds the 2x hysteresis threshold over the current owner, that node is the owner. Otherwise, fall back to hash ring. `ClusterManager.owner()` delegates to this resolver instead of directly to the hash ring.
**Alternatives:**
- Full replacement (no hash ring in dynamic mode) — fragile: a single write locks ownership until window expires
- Separate ownership map updated by periodic evaluator — introduces sync gap between map and tracker
**Rationale:** Hash ring is always the sensible default; dynamic ownership is an optimisation layered on top. Channels only transfer when evidence is strong (2x threshold).
**Trade-offs:** Hash ring is never fully eliminated — channels with no write history or no dominant writer always use it. This is a feature, not a limitation.
**Depends on:** D5 (consistent hashing), D11 (hash ring + DB safety net), D32 (tracker)
**Sources:** ConsistentHashRing.java, ClusterManager.owner(), consolidated spec §2 Level 4
**Exploration:** quick
**Status:** captured

## D34: Ownership claim propagation

**Choice:** Ownership claims piggybacked on existing heartbeat protocol. Add `Map<UUID, OwnershipClaim>` field to `HeartbeatResponse` — each node advertises which channels it claims to own. On receiving a heartbeat response, the local node updates its ownership map. On restart, one heartbeat round reconstructs the full cluster ownership map.
**Alternatives:**
- Separate ownership endpoint polled on heartbeat tick — cleaner separation but doubles HTTP calls per tick (N-1 heartbeats + N-1 ownership fetches)
- Event-sourced ownership log with delta sync — handles large maps efficiently but heavy machinery for hundreds of channels
**Rationale:** At conversation scale, the ownership map is small (hundreds of entries). Piggybacking on heartbeat keeps the protocol surface minimal. Payload increase is negligible.
**Trade-offs:** Ownership transfer latency bounded by heartbeat interval (3s default). Acceptable for conversation-pace workloads.
**Depends on:** D7 (heartbeat), D33 (resolver)
**Sources:** HeartbeatService.java, HeartbeatResponse.java, consolidated spec §2 Level 4
**Exploration:** quick
**Status:** captured

## D35: Sliding window implementation

**Choice:** Bucket-based sliding window. Divide the window into fixed-size buckets (e.g. 5-minute window with 10 × 30-second buckets). Each bucket holds an `AtomicLong` counter. On write, increment the current bucket. To query, sum all non-expired buckets. On tick, rotate: clear the oldest bucket and advance the pointer. O(1) per write, O(buckets) per query, bounded memory.
**Alternatives:**
- Timestamp queue per channel — exact counts but unbounded memory under high write rates, wrong data structure for a counter
- Single atomic counter with periodic decay — simplest memory but abrupt halving creates spurious ownership transfers
**Rationale:** Bucket-based windows are the standard pattern for rate counting. Bounded memory, O(1) writes, predictable decay. At conversation pace even 10 buckets per channel is negligible memory.
**Trade-offs:** Granularity limited by bucket size (30s default). Writes at bucket boundaries may be attributed to adjacent buckets. Acceptable imprecision for an optimisation heuristic.
**Depends on:** D32 (tracker)
**Sources:** Standard sliding-window rate limiter pattern (e.g. Redis sliding window, Guava RateLimiter internals)
**Exploration:** quick
**Status:** captured

## D36: Ownership evaluation cadence

**Choice:** Periodic `@Scheduled` evaluator running every 10 seconds (configurable). For each channel with write data, compares the local node's write count against the current owner's write count (from heartbeat claims). Claims channel if local writes > 2x current owner's writes. Each node evaluates independently — can only claim for itself, never assign to others.
**Alternatives:**
- Evaluate on every write — lowest latency but adds overhead to every write path; without other nodes' counts, single-node comparison is incomplete
- Evaluate on heartbeat tick — ties evaluation to heartbeat cadence, mixes heartbeat concerns with ownership logic
**Rationale:** Dedicated scheduled evaluation keeps concerns separated. 10-second default balances responsiveness with overhead. Evaluation has full access to local write counts and peer ownership data from heartbeats.
**Trade-offs:** Up to 10s delay between earning ownership and claiming it. At conversation pace this is negligible.
**Depends on:** D32 (tracker), D33 (resolver), D34 (heartbeat claims)
**Sources:** WatchdogScheduler pattern (existing @Scheduled evaluation), consolidated spec §4
**Exploration:** quick
**Status:** captured

## D37: Write count visibility for 2x comparison

**Choice:** Include write count in heartbeat ownership claims. Change payload from `Map<UUID, String>` to `Map<UUID, OwnershipClaim>` where `OwnershipClaim(long writeCount)`. Each owner advertises its write rate alongside the claim. A challenger compares: `myWrites > 2 * owner.writeCount`.
**Alternatives:**
- Absolute threshold only (no cross-node comparison) — prevents ownership transfer between active writers
- Full write-count gossip (all nodes share all channel counts) — most information but heaviest payload (N nodes × M channels)
**Rationale:** Only owners share counts, and only for channels they claim. Payload bounded by channels a node owns. Challengers have exactly the data needed for the 2x comparison.
**Trade-offs:** Write counts are slightly stale (up to one heartbeat interval). A challenger might over-claim briefly, but DB locks guarantee correctness during the overlap.
**Depends on:** D34 (heartbeat claims), D35 (sliding window), D36 (evaluator)
**Sources:** HeartbeatResponse.java, consolidated spec risk register (oscillation prevention)
**Exploration:** quick
**Status:** captured

## D38: Ownership relinquishment

**Choice:** Revert to hash ring when writes drop to zero. If the owner's write count drops to 0 within the sliding window (no writes in the full 5-minute window), ownership reverts to the hash ring assignment. The next active writer can earn it via the normal 2x claim process.
**Alternatives:**
- Revert below minimum threshold (e.g. 2 writes/window) — more responsive but adds tuning parameter interacting with hysteresis
- Never relinquish (only transfer via 2x) — stable but idle channels never return to hash ring, creating unbalanced distribution
**Rationale:** Zero writes is an unambiguous signal. The channel returns to its hash ring home, which is the stable default. If a new writer emerges, they earn it normally.
**Trade-offs:** A channel with very sporadic writes (1 write every 6 minutes) oscillates between dynamic and hash ring. The hysteresis on the claim side (2x + min-claim-writes) prevents routing instability.
**Depends on:** D33 (resolver), D35 (sliding window)
**Sources:** Consolidated spec §2 Level 4 (ownership drift), risk register (oscillation)
**Exploration:** quick
**Status:** captured

---

# Phase 8: Audit and E2E Testing Decisions (#484)

## D39: Overall structure — three-phase pipeline

**Choice:** Three sequential phases: A (code audit + fixes), B (integration tests), C (Podman e2e). Each phase validates the prior.
**Alternatives:**
- E2E first, audit as you go — gaps cause container startup failures, wastes effort debugging infra not behavior
- Monolithic single phase — no clean cut points, harder to resume across sessions
**Rationale:** Dependency chain is real: wiring gaps must be fixed before integration tests can verify composition, and composition must work before multi-node e2e tests make sense.
**Trade-offs:** More phases = more planning overhead. Mitigated by each phase being independently valuable.
**Sources:** Cluster module code read (7 gaps), cache module code read (4 gaps)
**Exploration:** quick
**Status:** captured

## D40: InternalMeshResource gating

**Choice:** `@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true")` on the class
**Alternatives:**
- Runtime guard per method — resource exists but returns 503. CDI injection of cluster beans may fail.
**Rationale:** Same pattern as `RelayProducer`. Resource doesn't exist when relay is disabled, preventing CDI resolution failures.
**Trade-offs:** Build-time only — can't toggle at runtime. Acceptable: relay is an architectural choice, not a runtime toggle.
**Depends on:** D39 (Phase A audit scope)
**Sources:** RelayProducer.java (existing @IfBuildProperty pattern)
**Exploration:** quick
**Status:** captured

## D41: HeartbeatService CDI production

**Choice:** Add to `RelayProducer` alongside `ClusterManager`, `WriteProxyClient`, etc.
**Alternatives:**
- Self-registered `@ApplicationScoped` with own `@IfBuildProperty` — less centralized
**Rationale:** Single producer, single gate. All cluster beans share the same lifecycle and activation condition.
**Trade-offs:** `RelayProducer` grows by one `@Produces` method. Trivial.
**Depends on:** D39 (Phase A audit scope)
**Sources:** RelayProducer.java (cluster CDI producer)
**Exploration:** quick
**Status:** captured

## D42: Write-frequency tracking position

**Choice:** Track only locally-executed writes. Move `tracker.recordWrite()` into the local-dispatch branch of `WriteRoutingDecorator`, exclude proxied writes.
**Alternatives:**
- Track all writes, subtract proxied — more complex, no clear benefit
**Rationale:** The node that actually handles writes gets the count. Proxied-away writes inflating the local count would cause oscillating ownership claims.
**Trade-offs:** The proxying node has no frequency data for the channel it proxied — this is correct, it shouldn't claim ownership.
**Depends on:** D32 (tracker), D36 (evaluator)
**Sources:** WriteRoutingDecorator.java (current tracking position), OwnershipEvaluator.java
**Exploration:** quick
**Status:** captured

## D43: E2E test module structure

**Choice:** New `e2e-cluster/` Maven module, profile-gated with `-Pwith-e2e-cluster`
**Alternatives:**
- Tests inside `cluster/src/test/` with profile gate — mixes unit and container tests
**Rationale:** Follows `examples/agent-communication/` pattern. Clean separation from fast unit tests. Testcontainers + PostgreSQL container are heavyweight deps.
**Trade-offs:** Additional Maven module and profile. CI must explicitly opt in.
**Depends on:** D39 (Phase C scope)
**Sources:** examples/agent-communication/ (profile-gated module pattern)
**Exploration:** quick
**Status:** captured

## D44: Testcontainers topology

**Choice:** `GenericContainer` with mesh module JAR in a JRE base image. Each container gets different env vars for `nodeId`, `peers`, shared PostgreSQL URL.
**Alternatives:**
- `DockerComposeContainer` — declarative but less programmatic control over individual node lifecycle
- Custom Quarkus DevServices extension — most Quarkus-native but heaviest to build
**Rationale:** Programmatic control over individual node lifecycle is essential for failure+recovery scenarios (stop Node C, verify detection, restart). `GenericContainer` provides this directly.
**Trade-offs:** Must build a Dockerfile and manage JAR copying. Straightforward with Quarkus uber-jar.
**Depends on:** D43 (module structure)
**Sources:** Testcontainers documentation, mesh/ module (QuarkusMain)
**Exploration:** quick
**Status:** captured

## D45: Internal endpoint security

**Choice:** Add shared-secret header authentication. Pre-shared key in `casehub.qhorus.relay.internal-secret` config, checked by a JAX-RS `@PreMatching` filter on `/internal/*` paths.
**Alternatives:**
- Document only, defer — lower effort but leaves endpoints open
**Rationale:** Defense-in-depth. Even on single-machine deployments, other processes can call these endpoints. Low implementation cost.
**Trade-offs:** One more config property. `WriteProxyClient` must send the header. Tests must set the config value.
**Depends on:** D39 (Phase A scope), D40 (InternalMeshResource gating)
**Sources:** TenancyContextFilter (existing @PreMatching filter pattern)
**Exploration:** quick
**Status:** captured

## D46: FullSyncService scheduler

**Choice:** New `CacheSyncScheduler` — `@Scheduled` driver bean in cache module, analogous to `OwnershipScheduler`. Gated by `cache.enabled + cache.mode=full`.
**Alternatives:**
- `@Observes StartupEvent` one-shot bulk load — misses channels created after startup
**Rationale:** Periodic sync handles new channels and recovers from interrupted syncs. Same pattern as `OwnershipScheduler`.
**Trade-offs:** One more `@Scheduled` bean. `@IfBuildProperty` gate keeps it invisible in non-cache deployments.
**Depends on:** D30 (cache module), D31 (full mode sync strategy)
**Sources:** OwnershipScheduler.java (existing @Scheduled pattern), FullSyncService.java
**Exploration:** quick
**Status:** captured

---

# Implicit Decisions Surfaced by Review

## D47: Quorum enforcement for write availability

**Choice:** Majority quorum on the write path. `ClusterManager.canServeWrites()` checks that a strict majority of configured peers are reachable before allowing writes. `WriteRoutingDecorator` and `ChannelManagerDecorator` call this check before every mutation. Bypassed for single-node deployments (`configuredPeers.size() <= 1`). Configurable via `casehub.qhorus.relay.quorum-enforced` (default: `true`).
**Alternatives:**
- No quorum (DB locks only) — higher availability during partitions, but both sides of a network split accept writes and create conflicting ownership claims even though DB locks serialize individual writes correctly
- Fencing tokens / epoch-based ownership — stronger guarantees but requires a consensus store (etcd, ZooKeeper) that the architecture deliberately avoids
- Lease-based ownership with time-bounded writes — avoids coordination but introduces clock-skew sensitivity
**Rationale:** Majority quorum is the simplest split-brain prevention mechanism that works with the existing heartbeat protocol (D7). It prevents a minority partition from accepting writes, which would create conflicting ownership claims.
**Trade-offs:** A 2-node cluster has zero write fault tolerance when quorum is enforced — if one node dies, the survivor cannot serve writes (`1 > 2/2` is false with integer division). Minimum recommended cluster size for write fault tolerance is 3 nodes. Operators deploying 2-node clusters should set `casehub.qhorus.relay.quorum-enforced=false` and accept that both nodes may write concurrently during a partition (DB locks guarantee correctness). The `quorumEnforced` default of `true` is safe for 3+ node clusters but surprising for 2-node deployments — this should be documented in operational guidance.
**Depends on:** D7 (heartbeat failure detection), D18 (topology ladder — Level 3+ only)
**Sources:** ClusterManager.canServeWrites(), WriteRoutingDecorator.dispatch(), ChannelManagerDecorator, RelayConfig.quorumEnforced()
**Exploration:** quick (surfaced by decision review R1-07)
**Status:** captured

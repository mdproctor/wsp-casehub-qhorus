# ⛔ SUPERSEDED — Do not use

> **This spec is superseded by [`2026-10-06-distributed-mesh-consolidated.md`](2026-10-06-distributed-mesh-consolidated.md).**
> Retained for git history only. The cluster module implementation details are
> accurate for what was built, but the architectural framing changed (D18-D24).

---

# Cluster Module Design — casehub-qhorus-cluster

**Issue:** casehubio/qhorus#475 (Phase 2)
**Date:** 2026-10-06
**Status:** Superseded
**Depends on:** Phase 1 runtime safety net (merged), overall distributed mesh design spec

## 1. Scope

This spec covers the `casehub-qhorus-cluster` module — the new Maven module
that adds multi-node clustering to qhorus. It handles hash ring management,
heartbeat failure detection, write routing via CDI decorators, and internal
node-to-node RPC.

**In scope:** ConsistentHashRing, ClusterManager, HeartbeatService,
WriteRoutingDecorator, ChannelManagerRoutingDecorator, InternalMeshClient,
InternalMeshResource, ClusterHealthResource, ClusterConfig.

**Out of scope:** REST API gaps (Phase 3), mesh service wiring (Phase 4),
ops integration (Phase 5). These phases build on this module but are
designed separately.

## 2. Module Structure

```
casehub-qhorus-cluster/
├── pom.xml
└── src/
    ├── main/java/io/casehub/qhorus/cluster/
    │   ├── ConsistentHashRing.java        — POJO, no CDI
    │   ├── NodeInfo.java                  — record(nodeId, address)
    │   ├── NodeState.java                 — enum: ALIVE, SUSPECT, DEAD
    │   ├── PeerState.java                 — record(nodeInfo, state, lastHeartbeat, missCount)
    │   ├── ClusterConfig.java             — @ConfigMapping
    │   ├── ClusterManager.java            — @ApplicationScoped
    │   ├── ClusterMembershipEvent.java    — CDI event record
    │   ├── HeartbeatService.java          — @ApplicationScoped
    │   ├── HeartbeatResponse.java         — record
    │   ├── WriteRoutingDecorator.java      — @Decorator on MessageDispatcher
    │   ├── ChannelManagerDecorator.java    — @Decorator on ChannelManager
    │   ├── WriteProxyClient.java             — @ApplicationScoped, wraps InternalMeshClient with dynamic base URL
    │   ├── InternalMeshClient.java         — @RegisterRestClient
    │   ├── InternalDispatchRequest.java   — record (serializable MessageDispatch)
    │   ├── InternalChannelRequest.java    — record (serializable ChannelCreateRequest)
    │   ├── InternalMeshResource.java      — JAX-RS @Path("/internal")
    │   ├── ClusterHealthResource.java     — JAX-RS @Path("/health/cluster")
    │   └── QuorumViolationException.java  — extends RuntimeException
    └── test/java/io/casehub/qhorus/cluster/
        ├── ConsistentHashRingTest.java
        ├── ClusterManagerTest.java
        ├── HeartbeatServiceTest.java
        ├── WriteRoutingDecoratorTest.java
        ├── ChannelManagerDecoratorTest.java
        └── InternalMeshResourceTest.java
```

**Package:** `io.casehub.qhorus.cluster`

**Maven coordinates:**
- `groupId`: `io.casehub`
- `artifactId`: `casehub-qhorus-cluster`
- `version`: `${project.version}` (0.2-SNAPSHOT)

**Dependencies:**
- `casehub-qhorus-api` — MessageDispatcher, ChannelManager, store interfaces
- `casehub-qhorus` (runtime) — for MessageService, ChannelService (decorator targets)
- `quarkus-rest-client-jackson` — for InternalMeshClient
- `quarkus-scheduler` — for HeartbeatService @Scheduled

**Not a Quarkus extension** — no deployment module. All behavior is runtime
CDI beans, activated by config gate. Follows the pattern of `connector-backend`,
`slack-channel`, and other optional modules (D14).

## 3. Activation

All cluster beans are gated behind `casehub.qhorus.cluster.enabled=true`
using `@IfBuildProperty` (D17).

When `enabled=false` or absent:
- No decorator on MessageDispatcher — direct dispatch, zero overhead
- No heartbeat scheduler
- No `/internal/*` or `/health/cluster` endpoints
- Existing behavior unchanged — all qhorus tests pass without modification

When `enabled=true`:
- `WriteRoutingDecorator` wraps `MessageDispatcher`
- `ChannelManagerDecorator` wraps `ChannelManager`
- `HeartbeatService` starts polling peers
- Internal and health endpoints become available
- `ClusterManager` computes the ring from the configured peer list

## 4. ConsistentHashRing

Pure POJO — no CDI, no dependencies beyond `java.security.MessageDigest`.

### 4.1 Hash Function

SHA-256 of the input bytes, truncated to 64 bits (first 8 bytes as `long`).
Deterministic — given the same input, every JVM produces the same hash.

```java
public final class ConsistentHashRing {

    private final NavigableMap<Long, String> ring;
    private final int virtualNodes;
    private final Set<String> members;

    public ConsistentHashRing(Set<String> members, int virtualNodes) {
        this.virtualNodes = virtualNodes;
        this.members = Set.copyOf(members);
        this.ring = buildRing(this.members, virtualNodes);
    }

    public String owner(UUID channelId) {
        long hash = hash(channelId.toString().getBytes(UTF_8));
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
        return entry != null ? entry.getValue() : ring.firstEntry().getValue();
    }

    public ConsistentHashRing withNode(String nodeId) { ... }
    public ConsistentHashRing withoutNode(String nodeId) { ... }
    public Set<String> members() { return members; }
}
```

### 4.2 Virtual Nodes

Each physical node maps to `virtualNodes` (default 128) positions on the ring.
Position key: `SHA-256(nodeId + "#" + virtualIndex)`. This distributes channels
evenly — with 3 nodes and 128 vnodes each, standard deviation of channel
assignment is ~2% of mean.

### 4.3 Immutability

Ring instances are immutable. `withNode()` and `withoutNode()` return new
instances. `ClusterManager` holds an `AtomicReference<ConsistentHashRing>`
for lock-free reads during ring transitions.

### 4.4 Ring Hash

`ringHash()` returns SHA-256 of the sorted member list — used by heartbeat
to detect ring disagreement between nodes.

## 5. ClusterManager

`@ApplicationScoped` bean. Central coordination point for clustering.

### 5.1 Responsibilities

- Owns the current `ConsistentHashRing` via `AtomicReference`
- Tracks peer state (ALIVE / SUSPECT / DEAD) per node
- Provides `owner(UUID channelId)` → `NodeInfo` for routing decisions
- Provides `isLocal(NodeInfo)` → `boolean`
- Enforces quorum: `canServeWrites()` returns false when this node can
  reach fewer than a majority of configured peers

### 5.2 Peer State Machine

```
ALIVE ──(1 missed heartbeat)──→ SUSPECT ──(2nd miss)──→ DEAD
  ↑                                │
  └────(heartbeat received)────────┘
```

State transitions fire `ClusterMembershipEvent` CDI async events for
observability (logging, metrics, alerting).

When a peer transitions to DEAD:
1. `ClusterManager` recalculates the ring excluding the dead node
2. Channels previously owned by the dead node are now owned by their
   next node on the ring (consistent hashing — only ~1/N channels move)
3. No state transfer needed — PostgreSQL holds all state

When a DEAD peer sends a heartbeat (recovery):
1. Peer transitions DEAD → ALIVE
2. Ring recalculated to include the recovered node
3. Channels rebalance back (~1/N channels return to the recovered node)

### 5.3 Quorum

`canServeWrites()` returns `true` when `reachableCount > configuredPeerCount / 2`.

In a 3-node cluster: need ≥ 2 reachable (including self).
In a 5-node cluster: need ≥ 3 reachable.

When quorum is lost, `WriteRoutingDecorator` throws `QuorumViolationException`
(mapped to HTTP 503 with `Retry-After` header). Reads are always served —
stale reads are acceptable per D4.

Single-node mode (no peers configured): quorum check is disabled.

### 5.4 Graceful Shutdown

On `@PreDestroy`:
1. ClusterManager sends `POST /internal/leave` to all ALIVE peers
2. Peers recalculate ring immediately (no heartbeat delay)
3. `MeshApp` shutdown hook drains in-flight requests (configurable timeout,
   default 30s)

### 5.5 Startup

On `@Observes StartupEvent`:
1. Validate config: `enabled=true` with empty `peers` → log WARN and
   operate in single-node mode (no routing, no heartbeat). Malformed
   peer addresses (missing port, unparseable host) → `IllegalStateException`
   at startup. Duplicate node IDs in peer list → `IllegalStateException`.
2. Parse peer list, build initial ring with all configured peers
   (optimistic — assume all alive)
3. HeartbeatService will detect dead peers on first heartbeat cycle (~3s)
4. During this window, writes to channels owned by dead peers will be
   proxied and fail with connection error — `WriteRoutingDecorator` catches
   this and returns 503

## 6. HeartbeatService

`@ApplicationScoped` with `@Scheduled(every = "{casehub.qhorus.cluster.heartbeat-interval}")`.

### 6.1 Polling

Each tick, HTTP GET to each peer's `/internal/heartbeat` endpoint:

```json
GET /internal/heartbeat
→ 200 OK
{
  "nodeId": "mesh-1",
  "timestamp": "2026-10-06T12:00:00Z",
  "ringHash": "a1b2c3...",
  "status": "UP"
}
```

### 6.2 Failure Detection

- HTTP error or timeout: increment miss count via `ClusterManager.recordMiss(nodeId)`
- HTTP 200 with matching ringHash: `ClusterManager.recordHeartbeat(nodeId)` → ALIVE
- HTTP 200 with mismatched ringHash: ALIVE but log WARN ("ring disagreement
  with node X — expected hash Y, got Z"). No action — ring agreement is
  config-driven, disagreement means a rolling update is in progress.

### 6.3 Clock Injection

Constructor accepts `Clock` parameter for deterministic time in tests —
same pattern as `PresenceService`. CDI-free tests use `Clock.fixed()`.

## 7. Write Routing

### 7.1 WriteRoutingDecorator

```java
@Decorator
@Priority(Interceptor.Priority.APPLICATION)
@IfBuildProperty(name = "casehub.qhorus.cluster.enabled",
                 stringValue = "true", enableIfMissing = false)
public class WriteRoutingDecorator implements MessageDispatcher {

    @Inject @Delegate MessageDispatcher delegate;
    @Inject ClusterManager cluster;
    @Inject WriteProxyClient proxy;

    @Override
    public DispatchResult dispatch(MessageDispatch dispatch) {
        if (!cluster.canServeWrites()) {
            throw new QuorumViolationException("minority partition");
        }
        NodeInfo owner = cluster.owner(dispatch.channelId());
        if (cluster.isLocal(owner)) {
            return delegate.dispatch(dispatch);
        }
        return proxy.dispatch(owner, dispatch);
    }
}
```

### 7.2 ChannelManagerDecorator

Same pattern for `ChannelManager`. Channel creation pre-generates a UUID
to route by (D16):

```java
@Override
public Channel create(ChannelCreateRequest request) {
    UUID channelId = UUID.randomUUID();
    NodeInfo owner = cluster.owner(channelId);
    if (cluster.isLocal(owner)) {
        return delegate.create(request.withPreAssignedId(channelId));
    }
    return proxy.createChannel(owner, request.withPreAssignedId(channelId));
}
```

For mutations that take a channelId (delete, pause, resume, config changes),
the existing channelId is the routing key.

### 7.3 Proxy Error Handling

When the proxy call to the owning node fails:
- Connection error (node down): return 503 with `Retry-After: 5`
- HTTP 503 from owner (owner in minority partition): propagate 503
- HTTP 4xx from owner (validation error): propagate the error as-is
- Timeout (configurable, default 10s): return 504 Gateway Timeout

No retry — the caller retries. This avoids proxy chains amplifying
retries and keeps the failure path predictable.

## 8. Internal RPC

### 8.1 InternalMeshClient

```java
@Path("/internal")
@RegisterRestClient(configKey = "internal-mesh")
public interface InternalMeshClient {

    @POST @Path("/dispatch")
    @Consumes(APPLICATION_JSON) @Produces(APPLICATION_JSON)
    DispatchResult dispatch(InternalDispatchRequest request);

    @POST @Path("/channel")
    @Consumes(APPLICATION_JSON) @Produces(APPLICATION_JSON)
    Channel createChannel(InternalChannelRequest request);

    @POST @Path("/channel/{id}/delete")
    @Consumes(APPLICATION_JSON)
    void deleteChannel(@PathParam("id") UUID channelId,
                       InternalDeleteRequest request);

    @POST @Path("/channel/{id}/pause")
    void pauseChannel(@PathParam("id") UUID channelId);

    @POST @Path("/channel/{id}/resume")
    void resumeChannel(@PathParam("id") UUID channelId);

    @GET @Path("/heartbeat")
    HeartbeatResponse heartbeat();

    @POST @Path("/leave")
    void leave(LeaveRequest request);
}
```

### 8.2 Dynamic Base URL

The client base URL is set per-call based on the target node's address.
`WriteProxyClient` wraps `InternalMeshClient` and sets the URL from
`NodeInfo.address()` before each call using Quarkus REST Client's
programmatic API (`RestClientBuilder.baseUrl()`).

### 8.3 InternalMeshResource

JAX-RS resource serving internal endpoints. Injects the concrete
`MessageService` and `ChannelService` classes directly — CDI decorators
only wrap injection points typed to the *interface* (`MessageDispatcher`,
`ChannelManager`), so injecting the concrete class bypasses the decorator
and prevents recursive routing loops:

```java
@Path("/internal")
@IfBuildProperty(name = "casehub.qhorus.cluster.enabled",
                 stringValue = "true", enableIfMissing = false)
public class InternalMeshResource {

    @Inject MessageService messageService;      // real impl, not decorator
    @Inject ChannelService channelService;       // real impl, not decorator

    @POST @Path("/dispatch")
    public DispatchResult dispatch(InternalDispatchRequest request) {
        return messageService.dispatch(request.toMessageDispatch());
    }

    @POST @Path("/channel")
    public Channel createChannel(InternalChannelRequest request) {
        return channelService.create(request.toChannelCreateRequest());
    }

    @GET @Path("/heartbeat")
    public HeartbeatResponse heartbeat() {
        return new HeartbeatResponse(
            clusterManager.nodeId(),
            Instant.now(),
            clusterManager.ringHash(),
            "UP");
    }

    @POST @Path("/leave")
    public void leave(LeaveRequest request) {
        clusterManager.handlePeerDeparture(request.nodeId());
    }
}
```

### 8.4 InternalDispatchRequest

Serializable record that captures all 16 `MessageDispatch` fields.
`toMessageDispatch()` reconstructs the domain object. This is the
wire format for internal dispatch — JSON via Jackson.

```java
public record InternalDispatchRequest(
    UUID channelId, String sender, String type, String content,
    String payload, String correlationId, Long inReplyTo,
    List<ArtefactRef> artefactRefs, String target,
    String subjectId, Long causedByEntryId, String actorType,
    String deadline, String telemetry, String tenancyId, String topic
) {
    public static InternalDispatchRequest from(MessageDispatch d) { ... }
    public MessageDispatch toMessageDispatch() { ... }
}
```

## 9. Health and Topology

### 9.1 ClusterHealthResource

```java
@Path("/health/cluster")
@IfBuildProperty(...)
public class ClusterHealthResource {

    @GET
    public ClusterHealthResponse health() {
        return new ClusterHealthResponse(
            clusterManager.nodeId(),
            clusterManager.status(),
            clusterManager.clusterSize(),
            clusterManager.expectedSize(),
            ringStatus(),
            partitionStatus());
    }
}
```

Response schema as defined in the overall design spec (Section 7.1).

### 9.2 Topology API

`GET /admin/topology` returns the full partition map:

```java
@Path("/admin/topology")
@IfBuildProperty(...)
public class TopologyResource {

    @GET
    public TopologyResponse topology() { ... }
}
```

Response schema as defined in the overall design spec (Section 7.2).

## 10. Configuration

```java
@ConfigMapping(prefix = "casehub.qhorus.cluster")
public interface ClusterConfig {
    @WithDefault("false")
    boolean enabled();

    Optional<List<String>> peers();

    Optional<String> nodeId();  // defaults to hostname

    @WithDefault("128")
    int virtualNodes();

    @WithDefault("3s")
    Duration heartbeatInterval();

    @WithDefault("2")
    int heartbeatMissThreshold();

    @WithDefault("30s")
    Duration drainTimeout();

    @WithDefault("true")
    boolean quorumEnforced();

    @WithDefault("10s")
    Duration proxyTimeout();
}
```

Environment variable mapping:
- `CASEHUB_QHORUS_CLUSTER_ENABLED` → `casehub.qhorus.cluster.enabled`
- `CASEHUB_QHORUS_CLUSTER_PEERS` → `casehub.qhorus.cluster.peers`
- `CASEHUB_QHORUS_CLUSTER_NODE_ID` → `casehub.qhorus.cluster.node-id`

(Quarkus maps env vars automatically via the standard `UPPER_SNAKE_CASE`
convention.)

## 11. API Surface Changes

### 11.1 ChannelCreateRequest

Gains `Optional<UUID> preAssignedId()` field via the builder API:
`ChannelCreateRequest.builder("name").preAssignedId(uuid).build()`. The
record's canonical constructor adds `preAssignedId` as a new nullable
parameter (16th position), with a backward-compatible 15-param constructor
that passes `null` (same pattern as the existing Space addition in D16's
reference). `ChannelCreateHelper.createInNewTransaction()` uses
`request.preAssignedId()` when non-null instead of generating a new UUID.
When absent (default), behavior is unchanged. This is the only change to
the existing qhorus API module (D16).

### 11.2 MessageDispatcher

No changes — the decorator implements the existing interface.

### 11.3 ChannelManager

No changes — the decorator implements the existing interface.

## 12. Testing Strategy

### 12.1 CDI-Free Unit Tests

All core components are testable without Quarkus:

- **ConsistentHashRingTest** — ring construction, deterministic lookup,
  virtual node distribution (chi-squared test for evenness), add/remove
  node migration count (~1/N channels move), ring hash determinism
- **ClusterManagerTest** — peer state machine transitions, quorum
  calculation, ring recalculation on membership change, graceful shutdown
  leave protocol. Uses `Clock.fixed()` for time control.
- **HeartbeatServiceTest** — miss counting, state transition triggering,
  ring hash mismatch detection. Mockito mocks for `InternalMeshClient`.
- **WriteRoutingDecoratorTest** — local dispatch delegation, remote proxy
  forwarding, quorum violation exception, proxy error handling (connection
  error → 503, timeout → 504)
- **ChannelManagerDecoratorTest** — pre-generated UUID routing for create,
  channelId routing for mutations, quorum enforcement

### 12.2 Integration Tests

- **Single-node mode** — existing qhorus `@QuarkusTest` tests run unchanged
  with `casehub.qhorus.cluster.enabled=false` (default)
- **InternalMeshResourceTest** — `@QuarkusTest` with cluster enabled,
  verifying dispatch/heartbeat/leave endpoints directly
- **Two-node simulation** — both nodes in the same JVM with different
  node IDs, verifying proxy routing decision (local vs remote) and
  response propagation

## 13. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Proxy latency (~1ms per hop) | Only non-owner writes pay this; hash ring minimizes non-owner traffic to agents connecting to the "wrong" node |
| Heartbeat false positive (transient network blip) | Two-miss threshold before DEAD; SUSPECT state absorbs single blips |
| Ring disagreement during rolling update | Heartbeat response includes ring hash; WARN log but no action — ops controls the rollout |
| ChannelCreateRequest API change | Optional field with backward-compatible default; existing callers unaffected |
| Recursive routing (internal endpoint → decorator → proxy → loop) | InternalMeshResource injects `MessageService` directly, not `MessageDispatcher` interface |

## References

- Phase 1 runtime safety net — 3 commits on `issue-475-distributed-mesh`
- Overall design spec: `specs/issue-475-distributed-mesh/2026-10-06-distributed-mesh-design.md`
- Decisions D12-D17: `specs/issue-475-distributed-mesh/decisions.md`
- `PostgresChannelActivityBroadcaster`: `postgres-broadcaster/src/main/java/.../PostgresChannelActivityBroadcaster.java`
- `ChannelGateway.deliverRemote()`: `runtime-core/src/main/java/.../gateway/ChannelGateway.java:334`
- `ChannelActivityBroadcaster` SPI: `api/src/main/java/.../gateway/ChannelActivityBroadcaster.java`
- `QhorusConfig` pattern: `runtime-core/src/main/java/.../config/QhorusConfig.java`
- `MeshService` (existing mesh): `mesh/src/main/java/.../mesh/MeshService.java`
- Consistent hashing: Karger et al., "Consistent Hashing and Random Trees" (1997)
- Quarkus CDI decorators: https://quarkus.io/guides/cdi-reference#decorators
- Quarkus REST Client: https://quarkus.io/guides/rest-client

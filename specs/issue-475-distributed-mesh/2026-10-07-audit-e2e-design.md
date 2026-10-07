# Distributed Mesh Audit and E2E Cluster Testing — Design Spec

**Date:** 2026-10-07
**Status:** Draft
**Issue:** #484
**Branch:** issue-475-distributed-mesh

## Problem

Phases 1-7 of the distributed mesh (#475) built the architecture incrementally as CDI-free POJOs with unit tests. A code audit surfaced 15 concrete gaps across the cluster and cache modules — unwired CDI beans, ungated resources, missing production drivers for the heartbeat protocol, a write-frequency tracking bug that risks ownership oscillation, an infinite proxy loop risk, and missing cache operations. Before the mesh is production-ready, these gaps must be fixed, composition must be verified via integration tests, and distributed behaviour must be validated via multi-node e2e tests.

## Approach

Three sequential phases, each validating the prior:

1. **Phase A — Code audit + fixes:** Systematic entry-point audit, then fix the 15 identified gaps in cluster and cache modules
2. **Phase B — Integration tests:** Single-JVM `@QuarkusTest` classes verifying cross-module CDI composition
3. **Phase C — Podman e2e tests:** Multi-node Testcontainers tests with shared PostgreSQL verifying distributed behaviour

## Phase A: Code Audit + Fixes

### A0. Systematic entry-point audit

Per #484's scope: map all paths into the dispatch pipeline (REST, MCP, A2A, WebSocket, connector, internal mesh). Verify all entry points converge through `MessageService.dispatch()` — no shortcuts that bypass ownership routing, rate limiting, or protocol enforcement. Identify and remove stubs that are now wired (#482). Consolidate duplication introduced across phases.

This is a documentation + verification pass — its output informs fixes A1–A15 and may surface additional gaps.

### Cluster Module (10 items)

#### A1. HeartbeatScheduler — production driver for failure detection

`HeartbeatService.tick()` is only called from test classes. No `@Scheduled` bean drives heartbeat polling in production. Without this, peers are never polled, `ClusterManager.recordMiss()` is never called, nodes are never transitioned to SUSPECT/DEAD, and the entire failure detection protocol is inert.

**Fix:** New `HeartbeatScheduler`:
```java
@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true",
                 enableIfMissing = false)
public class HeartbeatScheduler {
    @Inject HeartbeatService heartbeatService;

    @Scheduled(every = "${casehub.qhorus.relay.heartbeat-interval:3s}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void tick() { heartbeatService.tick(); }
}
```

This is the most critical item — Phase C scenarios C5 (failure+recovery) and C6 (quorum) depend on heartbeat-driven failure detection.

#### A2. HeartbeatService CDI production

`HeartbeatService` has no CDI producer. Its actual constructor is:
```java
public HeartbeatService(ClusterManager clusterManager,
                        Function<NodeInfo, HeartbeatResponse> heartbeatCaller)
```

**Fix:** Add `@Produces` method to `RelayProducer` that constructs the `heartbeatCaller` function from `WriteProxyClient`:
```java
@Produces @ApplicationScoped
public HeartbeatService heartbeatService(ClusterManager clusterManager,
        WriteProxyClient proxyClient) {
    return new HeartbeatService(clusterManager, proxyClient::heartbeat);
}
```

`WriteProxyClient` already has (or needs) a `heartbeat(NodeInfo)` method that calls `GET /internal/heartbeat` on the peer and deserialises the response.

#### A3. InternalMeshResource gating

`InternalMeshResource` is not gated by `@IfBuildProperty` — discovered even when relay is disabled, causing CDI resolution failures for injected cluster beans.

**Fix:** Add `@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true")` on the class. Same pattern as `RelayProducer`.

Also gate `ClusterHealthResource` (same annotation). `OwnershipScheduler` is already gated — no action needed.

#### A4. InternalMeshResource must use undecorated MessageService

`InternalMeshResource` injects `MessageDispatcher` (the CDI interface). When relay is enabled, `WriteRoutingDecorator` is the `@Alternative` — so proxied dispatches arriving at `/internal/dispatch` flow through the decorator, which may try to re-proxy if ownership shifted. This creates an infinite proxy loop.

**Fix:** `InternalMeshResource` must inject `CdiMessageService` (the concrete undecorated class), not `MessageDispatcher`. Same pattern used by `RelayProducer` to break the interface cycle. Proxied dispatches execute locally — they've already been routed.

#### A5. ChannelManagerDecorator CDI production

`RelayProducer` produces `WriteRoutingDecorator` as `MessageDispatcher` but has no corresponding producer for `ChannelManagerDecorator` as `ChannelManager`.

**Fix:** Add `@Produces` method to `RelayProducer`:
```java
@Produces @ApplicationScoped
@jakarta.annotation.Priority(100)
@jakarta.enterprise.inject.Alternative
public ChannelManager channelManager(ChannelService delegate,
        ClusterManager clusterManager, WriteProxyClient proxyClient) {
    return new ChannelManagerDecorator(delegate, clusterManager, proxyClient);
}
```

Inject `ChannelService` (concrete class) to break the interface cycle.

#### A6. Wire config mutation proxying — all 15 mutations

`ChannelManagerDecorator.routeChannelMutation()` throws `UnsupportedOperationException` for remote config mutations. The `ChannelManager` interface defines 15 mutation methods beyond `create`/`delete`.

Note: `pause` and `resume` endpoints already exist in `InternalMeshResource`, and `WriteProxyClient` already has `pauseChannel()` and `resumeChannel()` methods. These need to be wired into `routeChannelMutation()`.

**Fix:** Implement proxying for all 15 mutations via a generic `proxyClient.channelConfig(owner, channelId, operation, payload)` method. Add corresponding endpoints to `InternalMeshResource`:

Already exist (wire into routeChannelMutation):
- `POST /internal/channels/{id}/pause`
- `POST /internal/channels/{id}/resume`

New endpoints needed:
- `PUT /internal/channels/{id}/rate-limits`
- `PUT /internal/channels/{id}/allowed-writers`
- `PUT /internal/channels/{id}/admin-instances`
- `PUT /internal/channels/{id}/type-constraints`
- `PUT /internal/channels/{id}/reviewer-instances`
- `PUT /internal/channels/{id}/protocols`
- `PUT /internal/channels/{id}/protocol-participants`
- `PUT /internal/channels/{id}/enforcement-mode`
- `PUT /internal/channels/{id}/enforcement-exclusions`
- `PUT /internal/channels/{id}/routing-trust-threshold`
- `PUT /internal/channels/{id}/redistribution-capacity-threshold`
- `PUT /internal/channels/{id}/routing-capacity-threshold`
- `PUT /internal/channels/{id}/policy-overrides`

All internal-only, gated by the secret filter (A9).

#### A7. Channel creation — keep findOrCreate local

`findOrCreate()` always delegates to local — no routing logic.

**Fix:** Leave `findOrCreate()` executing locally. The DB unique constraint handles idempotency — if the channel already exists on another node, the local create gets a duplicate key and the find-path returns the existing channel. Routing `findOrCreate()` by name hash would diverge from the UUID-based routing used everywhere else in the system (create, dispatch, delete all hash the channel UUID), creating inconsistency.

This is a deliberate non-action — `findOrCreate()` is correct as-is.

#### A8. Fix write-frequency tracking position

`WriteRoutingDecorator.dispatch()` calls `tracker.recordWrite()` unconditionally after dispatch — including proxied writes. This inflates local frequency counts.

**Fix:** Track in the local-dispatch and fallback-to-local paths, but NOT in the successful proxy path. The actual code has three dispatch paths:

```java
if (clusterManager.isLocal(owner)) {
    MessageResult result = delegate.dispatch(dispatch);
    tracker.recordWrite(dispatch.channelId()); // local write — track
    return result;
} else {
    try {
        return proxyClient.dispatch(owner, dispatch);
        // proxy succeeded — do NOT track (remote node tracks its own)
    } catch (Exception e) {
        LOG.warnf("Proxy to %s failed, falling back to local: %s", owner, e.getMessage());
        MessageResult result = delegate.dispatch(dispatch);
        tracker.recordWrite(dispatch.channelId()); // fallback local — track
        return result;
    }
}
```

#### A9. Internal endpoint shared-secret auth

No authentication on `/internal/*` endpoints.

**Fix:** New `InternalSecretFilter`:
- `@Provider @PreMatching @Priority(50) @ApplicationScoped`
- `@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true")`
- Matches request paths starting with `/internal/`
- Reads expected secret from `casehub.qhorus.relay.internal-secret` config (optional — if absent, filter is a no-op for backward compatibility)
- Compares `X-Internal-Secret` request header against configured value
- Returns 401 on mismatch

`WriteProxyClient` sends `X-Internal-Secret` header on all requests when the config is set.

#### A10. ClusterManager shutdown hook

`ClusterManager.shutdown(Consumer<NodeInfo> leaveSender)` exists but no lifecycle hook calls it. Peers detect absence only through heartbeat misses, wasting `heartbeatMissThreshold * heartbeatInterval` time.

**Fix:** Add `@PreDestroy` or `@Observes ShutdownEvent` handler in `RelayProducer` (or a new `ClusterShutdownHandler` bean) that calls `clusterManager.shutdown(proxyClient::sendLeave)`. Sends leave notifications to all peers immediately, enabling instant ring removal.

### Cache Module (4 items)

#### A11. Wire findRecent() to use cache

`CachingMessageStore.findRecent()` always delegates to JPA.

**Fix:** Check channel buffer first. Note: `findRecent()` returns `List<MessageView>` but `ChannelMessageBuffer` stores `Message` objects. The buffer needs a `recentAsViews(int limit, QhorusEntityMapper mapper)` method that converts `Message → MessageView` via the existing `QhorusEntityMapper.toMessageView()`.

```java
public List<MessageView> findRecent(UUID channelId, int limit) {
    ChannelMessageBuffer buffer = buffers.getIfPresent(channelId);
    if (buffer != null && buffer.size() >= limit) {
        return buffer.recentAsViews(limit, entityMapper);
    }
    return delegate.findRecent(channelId, limit);
}
```

`CachingMessageStore` must inject `QhorusEntityMapper` for the conversion.

#### A12. Fix delete() cache invalidation

`CachingMessageStore.delete(Long id)` delegates to JPA but doesn't remove the message from the channel buffer.

**Fix:** After delegating, remove from buffer:
```java
public void delete(Long id) {
    delegate.delete(id);
    buffers.asMap().values().forEach(b -> b.remove(id));
}
```

`ChannelMessageBuffer` needs a `remove(Long messageId)` method.

#### A13. Add CacheSyncScheduler

`FullSyncService.syncBatch()` is produced but never driven by a scheduler.

**Fix:** New `CacheSyncScheduler`:
```java
@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.cache.enabled", stringValue = "true")
public class CacheSyncScheduler {
    @Inject FullSyncService fullSyncService;
    @Inject CacheConfig config;

    @Scheduled(every = "${casehub.qhorus.cache.full-sync-interval:5s}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void sync() {
        if ("full".equals(config.mode())) {
            fullSyncService.syncBatch();
        }
    }
}
```

Only active when cache is enabled. Runtime check for full mode so shallow-mode deployments pay no cost.

#### A14. Fix CacheProducer config alignment

`CacheProducer` uses `@IfBuildProperty(enableIfMissing = false)` — cache is off by default despite the config default of `enabled = true`. This is a build-time vs runtime mismatch.

**Fix:** Change `enableIfMissing = false` to `enableIfMissing = true`. The runtime config default is `enabled = true` (`CacheConfig.enabled()` has `@WithDefault("true")`), so the build-time gate must also default to enabled. Currently, if the property is absent from `application.properties`, the runtime config says "enabled" but the build-time gate says "disabled" — the producer class is excluded and no cache beans are created.

## Phase B: Integration Tests

Single-JVM `@QuarkusTest` classes verifying cross-module CDI composition. These live in the respective modules.

### B1. CDI wiring smoke test

`@QuarkusTest` with `@TestProfile` enabling relay + cache. Inject every relay-gated and cache-gated bean and assert non-null:

**Cluster beans:**
- `ClusterManager`, `HeartbeatService`, `HeartbeatScheduler`, `WriteProxyClient`
- `WriteRoutingDecorator` (as `MessageDispatcher`)
- `ChannelManagerDecorator` (as `ChannelManager`)
- `OwnershipScheduler`, `WriteFrequencyTracker`
- `InternalMeshResource`, `ClusterHealthResource`
- `InternalSecretFilter`

**Cache beans:**
- `CachingMessageStore` (as `MessageStore`)
- `CachePopulationObserver`, `FullSyncService`, `CacheSyncScheduler`

### B2. Decorator ordering test

Verify `WriteRoutingDecorator` wraps `CdiMessageService`, not itself. Dispatch a message, confirm it flows through the decorator to the real service. Verify `ChannelManagerDecorator` wraps the real `ChannelService`.

### B3. Config gate tests

Two `@TestProfile` variants:
- **All disabled:** relay=false, cache=false → verify `Instance<ClusterManager>.isResolvable() == false`, `Instance<CachingMessageStore>.isResolvable() == false`
- **Relay only:** relay=true, cache=false → verify cluster beans exist, cache beans don't

### B4. Internal secret filter test

With relay enabled and `internal-secret=test-secret`: verify `/internal/dispatch` returns 401 without header, 200 with correct `X-Internal-Secret: test-secret`.

### B5. Ownership evaluation integration

Single-node, relay enabled, `routing=dynamic`. Register a channel, dispatch several messages, trigger `OwnershipScheduler` evaluation manually, verify ownership claim appears in `ClusterManager`.

### B6. Cache composition test

Cache enabled (shallow mode). Dispatch a message via `MessageDispatcher`, verify:
- Subsequent `MessageStore.scan()` serves from cache
- `findRecent()` returns cached data
- `delete()` invalidates the cache entry (subsequent scan misses)

## Phase C: Podman E2E Tests

### C1. Module: e2e-cluster/

New Maven module, profile-gated with `-Pwith-e2e-cluster`.

```xml
<profile>
    <id>with-e2e-cluster</id>
    <modules>
        <module>e2e-cluster</module>
    </modules>
</profile>
```

Dependencies: `casehub-qhorus-cluster`, `casehub-qhorus-cache`, `casehub-qhorus-postgres-broadcaster`, `testcontainers`, `testcontainers-postgresql`, `rest-assured`, `junit-jupiter`.

The `postgres-broadcaster` dependency is required for pg_notify cross-node signaling — without it, `ChannelActivityBroadcaster` won't fire NOTIFY or LISTEN, and Scenario C3's cache coherence assertion will fail.

Tests are plain JUnit (not `@QuarkusTest`) — they manage containers externally.

### C2. Test infrastructure

**ClusterTestHarness** — utility class managing the multi-node topology:

```java
public class ClusterTestHarness implements AutoCloseable {
    private final Network network = Network.newNetwork();
    private PostgreSQLContainer<?> postgres;
    private Map<String, GenericContainer<?>> nodes;

    public void startPostgres() { ... }
    public void startNode(String nodeId, int port) { ... }
    public void stopNode(String nodeId) { ... }
    public void restartNode(String nodeId) { ... }
    public String nodeUrl(String nodeId) { ... }
    public void waitForHealthy(String nodeId, Duration timeout) { ... }
    public void waitForClusterConvergence(Duration timeout) { ... }
}
```

**Container networking:** All containers share a Testcontainers `Network`. Each node gets a network alias matching its `nodeId` (e.g. `node-a`, `node-b`, `node-c`). The `CASEHUB_QHORUS_RELAY_PEERS` env var uses these aliases with the fixed container port (e.g. `node-a:8080,node-b:8080,node-c:8080`). PostgreSQL gets alias `postgres` and nodes connect via `jdbc:postgresql://postgres:5432/qhorus`.

**Startup ordering:** PostgreSQL starts first. `waitForHealthy()` polls its JDBC connection. Nodes start with `casehub.qhorus.relay.heartbeat-miss-threshold` set high (e.g. 10) during startup to avoid premature DEAD marking of peers that haven't started yet. After all nodes are up and `waitForClusterConvergence()` confirms all peers are ALIVE, tests begin. Alternatively, start nodes sequentially with a brief wait between each.

**Environment per node:**
- `CASEHUB_QHORUS_RELAY_ENABLED=true`
- `CASEHUB_QHORUS_RELAY_NODE_ID=<alias>` (e.g. `node-a`)
- `CASEHUB_QHORUS_RELAY_PEERS=node-a:8080,node-b:8080,node-c:8080`
- `CASEHUB_QHORUS_RELAY_INTERNAL_SECRET=test-secret`
- `CASEHUB_QHORUS_RELAY_ROUTING=dynamic`
- `QUARKUS_DATASOURCE_QHORUS_JDBC_URL=jdbc:postgresql://postgres:5432/qhorus`
- `CASEHUB_QHORUS_CACHE_ENABLED=true`

**Dockerfile** in `e2e-cluster/src/test/resources/`:
```dockerfile
FROM eclipse-temurin:26-jre
COPY mesh-runner.jar /app/app.jar
ENTRYPOINT ["java", "-Xmx256m", "-jar", "/app/app.jar"]
```

Build order: `mesh/` builds uber-jar → `e2e-cluster/` copies it during test phase.

### C3. Scenario 1 — Dispatch + ownership + cache coherence

1. Start 3 nodes (A, B, C) + PostgreSQL
2. Wait for cluster convergence (all nodes see each other via heartbeat)
3. Create a channel via Node A REST API
4. Determine owner from `/health/ownership` endpoint
5. Send a message via a non-owner node → verify it proxied to owner (owner's ledger has the entry)
6. Poll the third node's REST API until the message appears (pg_notify via `postgres-broadcaster` propagation + cache population via `CachePopulationObserver`)
7. Verify all 3 nodes return identical message content

### C4. Scenario 2 — Ownership transfer

1. Start 3 nodes, create channel owned by Node A (verify via hash ring)
2. Configure short ownership window for fast transfer: `CASEHUB_QHORUS_RELAY_OWNERSHIP_WINDOW_SECONDS=30`, `CASEHUB_QHORUS_RELAY_OWNERSHIP_BUCKET_COUNT=3` (10s buckets)
3. Send >5 messages to the channel via Node B (exceeding `minClaimWrites=5`), spaced to span at least one full bucket rotation
4. Wait for ownership evaluation cycle (~10s + buffer) — writes must be visible in the tracker's window before evaluation
5. Poll Node B's `/health/ownership` → verify it now claims the channel
6. Send a message via Node C → verify it routes to Node B (check Node B ledger, not Node A)

### C5. Scenario 3 — Node failure + recovery

1. Start 3 nodes, create channel, send baseline message
2. `harness.stopNode("C")`
3. Wait for heartbeat miss threshold (2 misses × 3s interval = ~6s + buffer)
4. Poll Node A `/health/cluster` → verify Node C is DEAD
5. Dispatch a message that would hash to Node C → verify it succeeds (fallback-to-local)
6. `harness.restartNode("C")`
7. Wait for heartbeat round (~3s + buffer)
8. Poll Node A `/health/cluster` → verify Node C is ALIVE
9. Verify Node C appears in `/admin/topology`

### C6. Scenario 4 — Quorum enforcement

1. Start 3 nodes, verify all healthy
2. Stop Node B and Node C
3. Wait for Node A to detect both as DEAD
4. Send a message via Node A → verify 503 response (minority partition, `QuorumViolationException`)
5. Restart Node B → wait for convergence → verify Node A can write again (majority restored)

Simplified approach: stopping 2 of 3 nodes tests the same quorum arithmetic without network simulation. `canServeWrites()` counts `reachable` starting at 1 (self), with `configuredPeers.size()` = 3 (all peers including self). With only self reachable: `1 > 3/2` = `1 > 1` = false → writes rejected.

## Testing Considerations

- **Container memory:** JVM containers with `-Xmx256m` should fit 3 nodes + PostgreSQL on a dev machine (~1.5 GB total)
- **Startup order:** PostgreSQL must be healthy before app nodes start. Quarkus health check readiness probe gates test execution. High `heartbeat-miss-threshold` during startup prevents premature peer death marking.
- **Heartbeat timing:** Tests must account for heartbeat intervals and evaluation cycles. Use polling with timeouts rather than fixed sleeps.
- **Port allocation:** Testcontainers assigns random host ports for external access (test assertions). Nodes communicate internally via network aliases and fixed container ports.
- **Flyway migrations:** Shared PostgreSQL runs Qhorus + Ledger Flyway migrations on first node startup. Subsequent nodes skip (already migrated).
- **CI compatibility:** Profile-gated. CI can opt in with `-Pwith-e2e-cluster`. Requires Podman/Docker runtime. Estimated CI time: 2-3 minutes for all 4 scenarios.

## Non-Goals

- **Native image e2e:** JVM mode only. Native image testing deferred to a follow-up.
- **Network partition simulation:** No Toxiproxy. Quorum tested via node stop/start, not true partitioning.
- **Performance benchmarking:** Functional correctness only, no throughput or latency measurement.
- **Multi-machine federation:** Single-machine multi-container only. Cross-machine relay testing is Phase 2 scope.

## References

- `cluster/` — 28 source files, 12 test files (78 unit tests)
- `cache/` — 8 source files, 4 test files (30 unit tests)
- `RelayProducer.java` — CDI production hub for cluster beans
- `CacheProducer.java` — CDI production hub for cache beans
- `WriteRoutingDecorator.java:46-58` — three dispatch paths (local, proxy, fallback)
- `ChannelManagerDecorator.java` — channel mutation interceptor, `routeChannelMutation()` body
- `InternalMeshResource.java` — internal mesh REST endpoints (inject CdiMessageService, not MessageDispatcher)
- `HeartbeatService.java:15` — constructor: `(ClusterManager, Function<NodeInfo, HeartbeatResponse>)`
- `ClusterManager.java:163` — `shutdown(Consumer<NodeInfo>)` for graceful leave
- `CachingMessageStore.java` — cache decorator over JpaMessageStore
- `ChannelMessageBuffer.java` — stores `Message`, must convert to `MessageView` for `findRecent()`
- `FullSyncService.java` — background sync for full cache mode
- `QhorusEntityMapper.java` — `toMessageView(Message)` conversion
- ADR-0017 — shared database as multi-node prerequisite
- D39-D47 — audit and e2e decisions in `decisions.md`
- PP-20260608-07daa6 — observer test transaction discipline (relevant to Phase B)
- `examples/agent-communication/` — profile-gated module pattern (precedent for e2e-cluster/)
- `postgres-broadcaster/` — pg_notify cross-node notification (required for C3 cache coherence)
- Light review R1-01 through R1-16 — all findings incorporated

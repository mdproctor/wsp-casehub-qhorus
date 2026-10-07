# Distributed Mesh Audit and E2E Cluster Testing — Design Spec

**Date:** 2026-10-07
**Status:** Draft
**Issue:** #484
**Branch:** issue-475-distributed-mesh

## Problem

Phases 1-7 of the distributed mesh (#475) built the architecture incrementally as CDI-free POJOs with unit tests. A code audit surfaced 11 concrete gaps across the cluster and cache modules — unwired CDI beans, ungated resources, a write-frequency tracking bug that risks ownership oscillation, and missing cache operations. Before the mesh is production-ready, these gaps must be fixed, composition must be verified via integration tests, and distributed behaviour must be validated via multi-node e2e tests.

## Approach

Three sequential phases, each validating the prior:

1. **Phase A — Code audit + fixes:** Fix the 11 identified gaps in cluster and cache modules
2. **Phase B — Integration tests:** Single-JVM `@QuarkusTest` classes verifying cross-module CDI composition
3. **Phase C — Podman e2e tests:** Multi-node Testcontainers tests with shared PostgreSQL verifying distributed behaviour

## Phase A: Code Audit + Fixes

### Cluster Module (7 items)

#### A1. HeartbeatService CDI production

`HeartbeatService` has no CDI producer — it's constructed manually but never discoverable by CDI.

**Fix:** Add `@Produces` method to `RelayProducer`:
```java
@Produces @ApplicationScoped
public HeartbeatService heartbeatService(RelayConfig config,
        ClusterManager clusterManager, WriteProxyClient proxyClient) {
    return new HeartbeatService(config, clusterManager, proxyClient);
}
```

Same lifecycle and gate as other cluster beans. `InternalMeshResource` injects via CDI instead of constructor wiring.

#### A2. InternalMeshResource gating

`InternalMeshResource` is not gated by `@IfBuildProperty` — discovered even when relay is disabled, causing CDI resolution failures for injected cluster beans.

**Fix:** Add `@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true")` on the class. Same pattern as `RelayProducer`. Resource doesn't exist when relay is disabled.

Similarly gate `ClusterHealthResource` and `OwnershipScheduler` if not already gated.

#### A3. ChannelManagerDecorator CDI production

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

Inject `ChannelService` (concrete class) to break the interface cycle, same pattern as `CdiMessageService` for `MessageDispatcher`.

#### A4. Wire config mutation proxying

`ChannelManagerDecorator.routeChannelMutation()` throws `UnsupportedOperationException` for remote config mutations.

**Fix:** Implement proxying for all channel config mutations: `setRateLimits`, `setAllowedWriters`, `setAdminInstances`, `setTypeConstraints`, `pause`, `resume`. Each proxied via `WriteProxyClient` to the owner node. Add corresponding endpoints to `InternalMeshResource`:

- `PUT /internal/channels/{id}/rate-limits`
- `PUT /internal/channels/{id}/allowed-writers`
- `PUT /internal/channels/{id}/admin-instances`
- `PUT /internal/channels/{id}/type-constraints`
- `POST /internal/channels/{id}/pause`
- `POST /internal/channels/{id}/resume`

These mirror the existing `ChannelResource` REST API but are internal-only (gated by the secret filter, A7).

#### A5. Channel creation routing

`findOrCreate()` always delegates to local — no routing logic.

**Fix:** Hash the channel name to determine the target node. If non-local, proxy creation via `WriteProxyClient`. Add `POST /internal/channels/find-or-create` to `InternalMeshResource`.

Note: `findOrCreate()` is idempotent — racing requests on different nodes produce the same channel (DB unique constraints handle it). The routing is an optimisation (avoid creating on the wrong node then proxying all writes), not a correctness requirement.

#### A6. Fix write-frequency tracking position

`WriteRoutingDecorator.dispatch()` calls `tracker.recordWrite()` unconditionally after dispatch — including proxied writes. This inflates local frequency counts for channels the node doesn't own, risking oscillating ownership claims.

**Fix:** Move `tracker.recordWrite(dispatch.channelId())` into the local-dispatch branch only:

```java
if (isLocal) {
    MessageResult result = delegate.dispatch(dispatch);
    tracker.recordWrite(dispatch.channelId());
    return result;
} else {
    return proxyClient.dispatch(ownerNode, dispatch);
    // no tracker.recordWrite() — we didn't handle this write
}
```

#### A7. Internal endpoint shared-secret auth

No authentication on `/internal/*` endpoints.

**Fix:** New `InternalSecretFilter`:
- `@Provider @PreMatching @Priority(50) @ApplicationScoped`
- `@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true")`
- Matches request paths starting with `/internal/`
- Reads expected secret from `casehub.qhorus.relay.internal-secret` config (optional — if absent, filter is a no-op for backward compatibility)
- Compares `X-Internal-Secret` request header against configured value
- Returns 401 on mismatch

`WriteProxyClient` sends `X-Internal-Secret` header on all requests when the config is set.

### Cache Module (4 items)

#### A8. Wire findRecent() to use cache

`CachingMessageStore.findRecent()` always delegates to JPA, even when the channel buffer has sufficient data.

**Fix:** Check channel buffer first:
```java
public List<MessageView> findRecent(UUID channelId, int limit) {
    ChannelMessageBuffer buffer = buffers.getIfPresent(channelId);
    if (buffer != null && buffer.size() >= limit) {
        return buffer.recent(limit);
    }
    return delegate.findRecent(channelId, limit);
}
```

`ChannelMessageBuffer` needs a `recent(int limit)` method returning the N most recent entries (tail of the buffer, descending order).

#### A9. Fix delete() cache invalidation

`CachingMessageStore.delete(Long id)` delegates to JPA but doesn't remove the message from the channel buffer, leaving stale data.

**Fix:** After delegating, remove from buffer:
```java
public void delete(Long id) {
    delegate.delete(id);
    // Invalidate across all buffers — id doesn't carry channelId
    buffers.asMap().values().forEach(b -> b.remove(id));
}
```

`ChannelMessageBuffer` needs a `remove(Long messageId)` method.

#### A10. Add CacheSyncScheduler

`FullSyncService.syncBatch()` is produced but never driven by a scheduler.

**Fix:** New `CacheSyncScheduler`:
```java
@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.cache.enabled", stringValue = "true")
public class CacheSyncScheduler {
    @Inject FullSyncService fullSyncService;
    @Inject CacheConfig config;

    @Scheduled(every = "${casehub.qhorus.cache.full-sync-interval:5s}")
    void sync() {
        if ("full".equals(config.mode())) {
            fullSyncService.syncBatch();
        }
    }
}
```

Only active when cache is enabled. Runtime check for full mode so shallow-mode deployments pay no cost.

#### A11. Fix CacheProducer config alignment

`CacheProducer` uses `@IfBuildProperty(enableIfMissing = false)` — cache is off by default despite the config default of `enabled = true`. This is a build-time vs runtime mismatch.

**Fix:** Change to `enableIfMissing = true` so the build-time gate matches the runtime config default. When both say "enabled by default," the behaviour is consistent.

## Phase B: Integration Tests

Single-JVM `@QuarkusTest` classes verifying cross-module CDI composition. These live in the respective modules.

### B1. CDI wiring smoke test

`@QuarkusTest` with `@TestProfile` enabling relay + cache. Inject every bean and assert non-null:
- `ClusterManager`, `HeartbeatService`, `WriteProxyClient`
- `WriteRoutingDecorator` (as `MessageDispatcher`)
- `ChannelManagerDecorator` (as `ChannelManager`)
- `OwnershipScheduler`
- `CachingMessageStore` (as `MessageStore`)
- `CachePopulationObserver`, `FullSyncService`

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

Dependencies: `casehub-qhorus-cluster`, `casehub-qhorus-cache`, `testcontainers`, `testcontainers-postgresql`, `rest-assured`, `junit-jupiter`.

Tests are plain JUnit (not `@QuarkusTest`) — they manage containers externally.

### C2. Test infrastructure

**ClusterTestHarness** — utility class managing the multi-node topology:

```java
public class ClusterTestHarness implements AutoCloseable {
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

Each node container runs:
- JRE base image + mesh module uber-jar
- Environment: `CASEHUB_QHORUS_RELAY_ENABLED=true`, `CASEHUB_QHORUS_RELAY_NODE_ID=<id>`, `CASEHUB_QHORUS_RELAY_PEERS=<host:port,host:port>`, `CASEHUB_QHORUS_RELAY_INTERNAL_SECRET=test-secret`, `QUARKUS_DATASOURCE_QHORUS_JDBC_URL=<shared pg url>`
- Exposed HTTP port mapped to random host port

**Dockerfile** in `e2e-cluster/src/test/resources/`:
```dockerfile
FROM eclipse-temurin:26-jre
COPY mesh-runner.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Build order: `mesh/` builds uber-jar → `e2e-cluster/` copies it during test phase.

### C3. Scenario 1 — Dispatch + ownership + cache coherence

1. Start 3 nodes (A, B, C) + PostgreSQL
2. Wait for cluster convergence (all nodes see each other via heartbeat)
3. Create a channel via Node A REST API
4. Determine owner from `/health/ownership` endpoint
5. Send a message via a non-owner node → verify it proxied to owner (owner's ledger has the entry)
6. Poll the third node's REST API until the message appears (pg_notify propagation + cache population)
7. Verify all 3 nodes return identical message content

### C4. Scenario 2 — Ownership transfer

1. Start 3 nodes, create channel owned by Node A (verify via hash ring)
2. Send >5 messages to the channel via Node B (exceeding `minClaimWrites=5`)
3. Wait for ownership evaluation cycle (~10s + buffer)
4. Poll Node B's `/health/ownership` → verify it now claims the channel
5. Send a message via Node C → verify it routes to Node B (check Node B ledger, not Node A)

### C5. Scenario 3 — Node failure + recovery

1. Start 3 nodes, create channel, send baseline message
2. `harness.stopNode("C")`
3. Wait for heartbeat miss threshold (~6s + buffer)
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

Simplified approach: rather than true network partitioning (which requires Toxiproxy), stopping 2 of 3 nodes tests the same quorum arithmetic. The quorum check is `reachable > configuredPeers.size() / 2` — with 0 reachable out of 2 configured peers, the check fails.

## Testing Considerations

- **Container memory:** JVM containers with `-Xmx256m` should fit 3 nodes + PostgreSQL on a dev machine (~1.5 GB total)
- **Startup order:** PostgreSQL must be healthy before app nodes start. Quarkus health check readiness probe gates test execution.
- **Heartbeat timing:** Tests must account for heartbeat intervals and evaluation cycles. Use polling with timeouts rather than fixed sleeps.
- **Port allocation:** Testcontainers assigns random host ports. Nodes communicate via the Testcontainers internal network using fixed container ports.
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
- `WriteRoutingDecorator.java` — message dispatch interceptor
- `ChannelManagerDecorator.java` — channel mutation interceptor
- `InternalMeshResource.java` — internal mesh REST endpoints
- `HeartbeatService.java` — heartbeat protocol
- `CachingMessageStore.java` — cache decorator over JpaMessageStore
- `FullSyncService.java` — background sync for full cache mode
- ADR-0017 — shared database as multi-node prerequisite
- D39-D47 — audit and e2e decisions in `decisions.md`
- PP-20260608-07daa6 — observer test transaction discipline (relevant to Phase B)
- `examples/agent-communication/` — profile-gated module pattern (precedent for e2e-cluster/)

# Distributed Mesh Audit and E2E Cluster Testing — Design Spec

**Issue:** casehubio/qhorus#484
**Date:** 2026-10-10
**Branch:** issue-484-mesh-audit-e2e

## Summary

Comprehensive audit of the distributed mesh implementation (Phases 1–7), code improvements for robustness and clarity, and new end-to-end tests covering ownership transfer, cache coherence, and channel creation routing. Adds a native image Dockerfile variant for opt-in validation.

## Scope

Three workstreams executed sequentially (D1): audit → improve → e2e.

---

## Workstream 1 — Code Audit

### Part A: Entry-Point Trace

Map every write path into `MessageService.dispatch()` and verify convergence through the enforcement gate (rate limiting, ACL, protocol enforcement, type policy).

| Entry point | Module | Path |
|---|---|---|
| REST `POST /api/channels/{id}/messages` | runtime | ChannelResource → MessageService.dispatch() |
| MCP `send_message` | runtime | QhorusTestHelper → MessageService.dispatch() |
| A2A `message/send` (JSON-RPC) | runtime | A2AResource → MessageService.dispatch() |
| WebSocket inbound | websocket-observer | Observer only — not a write path |
| Connector inbound | connector-backend | ConnectorChannelBackend → ChannelGateway.receiveHumanMessage() → MessageService.dispatch() (×2: content + normaliser telemetry EVENT) |
| Internal mesh `POST /internal/dispatch` | cluster | InternalMeshResource → CdiMessageService.dispatch() |
| MeshService `meshSendMessage` | mesh | MeshService → MessageDispatcher.dispatch() |

Verify:
1. All write paths converge through `MessageService.dispatch()` enforcement gate
2. The cluster decorator `WriteRoutingDecorator` intercepts all paths via `ConsumerMessaging` / `MessageDispatcher` CDI alternatives (`@Alternative @Priority(100)`)
3. No path bypasses rate limiting, ACL, or protocol enforcement
4. `InternalMeshResource` dispatches through `CdiMessageService` (not `ConsumerMessaging`) to avoid proxy loops — the proxied message must execute locally on the target node

### Part B: Cross-Module Composition

Verify correct composition when cluster, cache, and postgres-broadcaster modules are all present:

| Bean | Type | Priority | Wraps |
|---|---|---|---|
| `RoutingConsumerMessaging` | `ConsumerMessaging` @Alternative | 100 | `CdiMessageService` (dispatch) + `CdiMessageService` (queries) |
| `ChannelManagerDecorator` | `ChannelManager` @Alternative | 100 | `ChannelService` |
| `CachingMessageStore` | `MessageStore` @Alternative | 100 | `JpaMessageStore` |
| `CachePopulationObserver` | `MessageObserver` CLUSTER | — | Direct cache population on remote nodes |

Check:
- Decorator ordering: `WriteRoutingDecorator` → `MessageService.dispatch()` → `CachingMessageStore.put()` → `ChannelGateway.fanOut()` → `CachePopulationObserver` (on remote nodes)
- `CachePopulationObserver` fires on remote nodes via `deliverRemote()` → `dispatchClusterObservers()`
- When cache/cluster modules are absent (not on classpath), system degrades to single-node with no errors
- `@IfBuildProperty` gates on all cluster beans prevent config mapping registration failures when relay is disabled
- `ChannelManagerDecorator.applyTo()` switch covers all `ChannelManager` mutation methods (verify against current interface)

### Audit Output

Code changes directly — find gaps, fix them, test them. No separate audit report document.

---

## Workstream 2 — Code Improvement

### 2a: Ownership Transfer Race Conditions

The current `OwnershipEvaluator.evaluate()` runs periodically (default 10s). Between ownership transfer and heartbeat propagation, messages can be proxied to the old owner.

**Mitigations:**
- In `WriteRoutingDecorator`: when a proxy to the owner fails and the channel was recently owned locally (within one heartbeat interval), fall back to local dispatch. This is partially handled by `proxyFallback=local`, but the fallback should log at INFO (not WARN — WARN is noisy for expected transient states during ownership transfer)
- Document the race window in code comments on `OwnershipEvaluator` — the window is bounded by `heartbeatInterval × 2` (one cycle for evaluation, one for propagation)
- Verify that the fallback-to-local path still records the write in `WriteFrequencyTracker` (it does — checked in existing code)

### 2b: Error Propagation

Replace generic `RuntimeException` in `WriteProxyClient` with typed exceptions:

| Exception | Trigger | Caller reaction |
|---|---|---|
| `ProxyTimeoutException` | HTTP timeout, connect timeout, `IOException`, `InterruptedException` | Fallback silently (transient) |
| `ProxyAuthException` | HTTP 401/403 from `InternalSecretFilter` | Log loudly (misconfiguration), fallback |
| `ProxyDispatchException` (exists) | HTTP 4xx (non-auth), HTTP 5xx, fail-fast mode | Propagate or fallback per config |

`WriteRoutingDecorator` catch blocks distinguish:
- `ProxyTimeoutException` → fallback, log at DEBUG (expected during node failure)
- `ProxyAuthException` → fallback, log at ERROR (misconfiguration needs attention)
- `ProxyDispatchException` → existing behavior (fallback or rethrow per `proxyFallback` config)

### 2c: Cleanup and Visibility

- Audit `public` vs `package-private` on cluster module classes — `BucketWindow`, `WriteFrequencyTracker`, `OwnershipClaim` etc. should be package-private where only used internally
- Remove stale TODO/stub code from the phased build
- Consolidate `WriteProxyClient` constructor overloads (4 constructors → primary + test-injection)
- Verify `ChannelConfigRequest.applyTo()` switch covers all current `ChannelManager` mutation methods

---

## Workstream 3 — E2E Tests

All tests extend the existing `ClusterTestHarness` pattern (D2). Each test class gets its own cluster lifecycle (`@TestInstance(PER_CLASS)`).

### 3a: OwnershipTransferE2ETest

2-node cluster. Tests:

1. **Baseline ownership:** Create channel on node-a, send messages → verify node-a owns it
2. **Write pattern shift:** Send N messages from node-b (exceeding `minClaimWrites` and `hysteresisRatio`) → wait for ownership evaluation cycle → verify node-b claims ownership
3. **Post-transfer proxy:** After transfer, send message from node-a → verify it gets proxied to node-b and is visible on both nodes
4. **Relinquish:** Stop writing from node-b → wait for evaluation → verify ownership reverts to hash-ring default

**Harness addition:** `getOwnership(nodeId, channelId)` — calls `GET /health/cluster/ownership/{channelId}` (new endpoint on `ClusterHealthResource` returning `{owner: nodeId, claimed: boolean}`). The existing `/health/cluster` endpoint reports cluster-level health; per-channel ownership needs a dedicated query.

**Infrastructure requirement:** The ownership evaluation interval must be shortened for e2e tests (e.g., 3s instead of 10s) via container env var `CASEHUB_QHORUS_RELAY_OWNERSHIP_EVALUATION_INTERVAL_SECONDS=3`.

### 3b: CacheCoherenceE2ETest

2-node cluster with cache enabled. Tests:

1. **Cross-node cache population:** Send message on node-a → verify node-b's cache stats show the channel cached (via `/health/cache` endpoint)
2. **Rapid message convergence:** Send multiple messages rapidly on node-a → verify all messages visible on node-b
3. **Cache invalidation on delete:** Delete channel on node-a → verify node-b's cache stats no longer include the channel

**Harness addition:** `getCacheStats(nodeId)` — calls existing `/health/cache` endpoint, returns parsed response.

### 3c: ChannelCreationRoutingE2ETest

2-node cluster. Tests:

1. **Basic creation:** Create channel on node-a → verify queryable from both nodes
2. **Pre-assigned ID routing:** Create channel with `preAssignedId` that hashes to node-b, issued from node-a → verify creation is proxied to node-b, channel visible on both
3. **Concurrent creation:** Create same-named channel from both nodes simultaneously → verify exactly one channel exists (idempotent findOrCreate)

**Harness addition:** `createChannelWithId(nodeId, channelName, preAssignedId)` — POST with `preAssignedId` field.

---

## Workstream 4 — Native Image

### Dockerfile.native

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.4
COPY target/*-runner /application
RUN chmod 775 /application
ENTRYPOINT ["./application", "-Xmx64m"]
```

Placed alongside the existing JVM `Dockerfile` in `e2e-cluster/src/test/resources/`.

### Build Integration

- `mesh/pom.xml` already has the `native` profile from Quarkus parent
- `ClusterTestHarness` gains a system property `mesh.container.mode` (default `jvm`, alternative `native`) that selects between Dockerfiles
- When `native`, the harness copies `mesh/target/*-runner` instead of `mesh/target/quarkus-app/`
- New Maven profile in `e2e-cluster/pom.xml`: `-Pwith-e2e-native` sets `mesh.container.mode=native`

### What This Catches

- Reflection registration gaps (Jackson serialization of `DispatchResult`, `HeartbeatResponse`, `OwnershipClaim`, `ChannelConfigRequest`)
- Missing native-image resource registrations beyond what `QhorusProcessor` handles
- Vert.x/reactive issues in native mode (postgres-broadcaster uses reactive PgPool)

### CI Consideration

Native builds take ~3-5 minutes. The profile is opt-in, not part of the default build. CI can run it nightly.

---

## Non-Goals

- Kubernetes deployment manifests or Helm charts
- Multi-region or WAN clustering
- Performance benchmarking (separate concern)
- Changes to the MCP tool layer or MeshApi

## References

- [cluster/ClusterManager.java] — core cluster state management
- [cluster/WriteRoutingDecorator.java] — dispatch routing with proxy fallback
- [cluster/WriteProxyClient.java] — cross-node HTTP client
- [cluster/RelayProducer.java] — CDI wiring for all relay beans
- [cluster/ChannelManagerDecorator.java] — channel mutation proxying
- [cluster/HeartbeatService.java] — peer health detection
- [cluster/DynamicOwnershipResolver.java] — claim-based ownership resolution
- [cluster/OwnershipEvaluator.java] — periodic claim/relinquish evaluation
- [cache/CachingMessageStore.java] — LRU message cache decorator
- [cache/CachePopulationObserver.java] — CLUSTER-scoped remote cache population
- [postgres-broadcaster/PostgresChannelActivityBroadcaster.java] — LISTEN/NOTIFY cross-node delivery
- [e2e-cluster/ClusterTestHarness.java] — Podman test infrastructure
- [e2e-cluster/DispatchRoutingE2ETest.java] — existing dispatch routing tests
- [e2e-cluster/NodeFailureE2ETest.java] — existing node failure tests
- [e2e-cluster/QuorumEnforcementE2ETest.java] — existing quorum tests
- [mesh/MeshService.java] — standalone relay application service
- [docs/protocols/casehub/startup-event-handler-exception-isolation.md] — protocol for startup recovery
- [docs/protocols/casehub/channel-initialised-event-observer-idempotency.md] — protocol for channel init

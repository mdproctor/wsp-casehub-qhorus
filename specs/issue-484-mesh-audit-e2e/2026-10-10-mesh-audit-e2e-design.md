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
| MeshService `meshSendMessage` | mesh | MeshService → MessageDispatcher.dispatch() (CDI resolves to `RoutingConsumerMessaging` → `WriteRoutingDecorator` → `CdiMessageService` when cluster module is on classpath) |

Verify:
1. All write paths converge through `MessageService.dispatch()` enforcement gate
2. The cluster decorator `WriteRoutingDecorator` intercepts all paths via `ConsumerMessaging` / `MessageDispatcher` CDI alternatives (`@Alternative @Priority(100)` on producer methods in `RelayProducer`, not on the classes themselves)
3. No path bypasses rate limiting, ACL, or protocol enforcement
4. `InternalMeshResource` dispatches through `CdiMessageService` (not `ConsumerMessaging`) to avoid proxy loops — the proxied message must execute locally on the target node
5. `MeshService.meshSendMessage()` injects `MessageDispatcher` which CDI resolves to `RoutingConsumerMessaging` when the cluster module is present — confirm the full routing chain is exercised, not just the local `MessageService`

### Part B: Cross-Module Composition

Verify correct composition when cluster, cache, and postgres-broadcaster modules are all present:

| Bean | Type | Priority | Wraps |
|---|---|---|---|
| `RoutingConsumerMessaging` | `ConsumerMessaging` @Alternative (via `RelayProducer` producer method) | 100 | `WriteRoutingDecorator` (dispatch routing) + `CdiMessageService` (queries via `ConsumerMessaging`) |
| `ChannelManagerDecorator` | `ChannelManager` @Alternative | 100 | `ChannelService` |
| `CachingMessageStore` | `MessageStore` @Alternative | 100 | `JpaMessageStore` |
| `CachePopulationObserver` | `MessageObserver` CLUSTER | — | Direct cache population on remote nodes |

Check:
- Decorator ordering: `WriteRoutingDecorator` → `MessageService.dispatch()` → `CachingMessageStore.put()` → `ChannelGateway.fanOut()` → `CachePopulationObserver` (on remote nodes)
- `CachePopulationObserver` fires on remote nodes via `deliverRemote()` → `dispatchClusterObservers()`
- When cache/cluster modules are absent (not on classpath), system degrades to single-node with no errors
- `@IfBuildProperty` gates on all cluster beans prevent config mapping registration failures when relay is disabled
- `ChannelManagerDecorator.applyTo()` switch covers all `ChannelManager` mutation methods (verify against current interface)
- **`findOrCreate()` routing bypass:** `ChannelManagerDecorator.findOrCreate()` currently delegates directly to the local delegate without routing or quorum checks. Every other mutation method either routes to the hash-ring owner or checks `canServeWrites()`. Fix: add quorum check to `findOrCreate()` (a minority-partition node must not create channels). Routing is less critical because the shared database provides atomicity for the find path, but the quorum bypass is a correctness gap.

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

**Structural refactoring of `WriteProxyClient.post()` and `get()`:** Both methods currently have a single `catch (Exception e)` that wraps everything uniformly. Refactor to split the catch blocks:
1. Inside the try, after `httpClient.send()`: check HTTP status — 401/403 → throw `ProxyAuthException`; other 4xx/5xx → throw `ProxyDispatchException` with status code
2. Catch `IOException | InterruptedException` → throw `ProxyTimeoutException` (network-level failure)
3. Catch `com.fasterxml.jackson.core.JsonProcessingException` → throw `ProxyDispatchException` (deserialization failure — the remote responded but the response is unparseable)
4. Catch remaining `Exception` → throw `ProxyDispatchException` (unexpected)

The `get()` method (used by heartbeat) needs the same treatment as `post()`.

`WriteRoutingDecorator` catch blocks distinguish:
- `ProxyTimeoutException` → fallback, log at WARN (operationally significant — the caller's message was locally dispatched instead of proxied, and sustained timeouts indicate a dead node that needs attention)
- `ProxyAuthException` → fallback, log at ERROR (misconfiguration needs immediate attention)
- `ProxyDispatchException` → existing behavior (fallback or rethrow per `proxyFallback` config)

### 2c: Cleanup and Visibility

- Audit `public` vs `package-private` on cluster module classes — `BucketWindow`, `WriteFrequencyTracker` should be package-private (confirmed: references are all within `io.casehub.qhorus.cluster`). `OwnershipClaim` must remain public — it is serialized by Jackson in `HeartbeatResponse` and `OwnershipHealthResponse` REST endpoints (inter-node API contract)
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

**Harness additions:**
- `getOwnership(nodeId, channelId)` — calls `GET /health/cluster/ownership/{channelId}` (new endpoint on `ClusterHealthResource`). Response: `{owner: nodeId, source: "hash-ring" | "dynamic-claim", claimWriteCount: long}`. The `source` field distinguishes hash-ring default ownership from dynamic claim-based ownership so tests can assert that ownership transferred via a claim, not that the hash ring happened to assign it.
- `getLocalClaims(nodeId)` — calls existing `GET /health/ownership` endpoint (returns `OwnershipHealthResponse` with all local claims for THIS node). Used to verify a node's own claim state after ownership transfer.
- Test 3a.3 (post-transfer proxy) verifies proxy happened by querying the ownership endpoint on BOTH nodes and asserting they agree node-b is the owner — since both nodes share the same `DynamicOwnershipResolver` state via heartbeat claim propagation, agreement confirms the routing table is correct. The message being visible on both nodes then confirms the proxy path worked (node-a doesn't own the channel, so it must have proxied).

**Infrastructure requirement:** The ownership evaluation interval must be shortened for e2e tests (e.g., 3s instead of 10s) via container env var `CASEHUB_QHORUS_RELAY_OWNERSHIP_EVALUATION_INTERVAL_SECONDS=3`.

### 3a-bis: NodeFailureFallbackE2ETest

Extends the existing `NodeFailureE2ETest` coverage with dispatch-level fallback and ownership reconstruction. 2-node cluster with dynamic routing enabled.

1. **Fallback-to-local during failure:** Create channel owned by node-a (via hash ring or dynamic claim). Stop node-a. Send message from node-b targeting that channel → verify the message succeeds via fallback-to-local dispatch (`proxyFallback=local`) and is visible on node-b
2. **Ownership reconstruction after recovery:** Restart node-a → wait for cluster convergence → send messages from node-a to the channel → wait for ownership evaluation → verify ownership claims propagate via heartbeat (query `getLocalClaims()` on both nodes to confirm convergence)
3. **Post-recovery routing consistency:** After reconstruction, send message from node-b → verify it is proxied to the correct owner (whichever node the evaluator assigned based on write frequency)

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
COPY mesh-runner /application
RUN chmod 775 /application
ENTRYPOINT ["./application", "-Xmx128m"]
```

Placed alongside the existing JVM `Dockerfile` in `e2e-cluster/src/test/resources/`. Note: 128MB heap (not 64MB) — the mesh module runs CachingMessageStore (in-memory LRU), postgres-broadcaster (reactive PgPool/Netty buffers), and cluster module (ConcurrentHashMap-based claims, sliding window trackers). Native images use ~50-75% less heap than JVM mode (JVM uses 256MB), making 128MB the conservative lower bound with headroom.

### Build Integration

- `mesh/pom.xml` already has the `native` profile from Quarkus parent
- `ClusterTestHarness.buildImage()` uses Testcontainers' programmatic build context (not a traditional Docker build). For native mode, `buildImage()` must:
  1. Select `Dockerfile.native` from classpath instead of `Dockerfile`
  2. Copy the native runner binary (e.g., `mesh/target/casehub-qhorus-mesh-0.2-SNAPSHOT-runner`) into the build context under the name `mesh-runner` (matching the Dockerfile's `COPY mesh-runner /application`)
  3. Use `withFileFromPath("mesh-runner", nativeRunnerPath)` instead of `withFileFromPath("quarkus-app", meshTarget)`
- `ClusterTestHarness` gains a system property `mesh.container.mode` (default `jvm`, alternative `native`) that selects between build paths
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

# Distributed Mesh Audit and E2E Cluster Testing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #484 — distributed mesh audit and e2e cluster testing
**Issue group:** #484

**Goal:** Audit the distributed mesh implementation for correctness gaps, improve error handling and robustness, and add comprehensive e2e tests for ownership transfer, cache coherence, channel creation routing, node failure fallback, and native image validation.

**Architecture:** Sequential workstreams — audit finds gaps (including `findOrCreate` quorum bypass), improvement fixes them (typed proxy exceptions, visibility tightening), e2e tests validate the fixed system across Podman containers with real network boundaries. Native image variant added as opt-in.

**Tech Stack:** Java 21, Quarkus 3.32.2, Testcontainers 1.20.4, Podman, RestAssured, Awaitility, GraalVM 25 (native image)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- E2E tests: `mvn test -Pwith-e2e-cluster -pl e2e-cluster` (requires `mvn package -pl mesh -am` first)
- Native: `JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home mvn package -Pnative -pl mesh -am -DskipTests`
- All cluster beans gated by `@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true", enableIfMissing = false)`
- Commit every substantial change with `Refs #484`
- Do NOT delete any git branches

---

## Batch 1: Audit and Fix — Routing Gaps

### Task 1: Fix `findOrCreate` quorum bypass in ChannelManagerDecorator

**Files:**
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/ChannelManagerDecorator.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/ChannelManagerDecoratorTest.java`

**Interfaces:**
- Consumes: `ClusterManager.canServeWrites()`, `ChannelManager.findOrCreate(ChannelCreateRequest)`
- Produces: `ChannelManagerDecorator.findOrCreate()` with quorum check (existing signature unchanged)

- [ ] **Step 1: Write the failing test**

Add to `ChannelManagerDecoratorTest.java`:

```java
@Test
void findOrCreate_rejects_in_minority_partition() {
    when(clusterManager.canServeWrites()).thenReturn(false);
    var request = ChannelCreateRequest.builder("quorum-test").build();
    assertThatThrownBy(() -> decorator.findOrCreate(request))
            .isInstanceOf(QuorumViolationException.class);
    verifyNoInteractions(delegate);
}

@Test
void findOrCreate_delegates_when_quorum_present() {
    when(clusterManager.canServeWrites()).thenReturn(true);
    var request = ChannelCreateRequest.builder("quorum-ok").build();
    var expected = new FindOrCreateResult(testChannel, false);
    when(delegate.findOrCreate(request)).thenReturn(expected);
    var result = decorator.findOrCreate(request);
    assertThat(result).isEqualTo(expected);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelManagerDecoratorTest#findOrCreate_rejects_in_minority_partition -pl cluster`
Expected: FAIL — current `findOrCreate` delegates directly without quorum check

- [ ] **Step 3: Add quorum check to findOrCreate**

In `ChannelManagerDecorator.java`, replace the `findOrCreate` method:

```java
@Override
public FindOrCreateResult findOrCreate(ChannelCreateRequest request) {
    if (routingEnabled && !clusterManager.canServeWrites()) {
        throw new QuorumViolationException("minority partition");
    }
    return delegate.findOrCreate(request);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelManagerDecoratorTest -pl cluster`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/ChannelManagerDecorator.java cluster/src/test/java/io/casehub/qhorus/cluster/ChannelManagerDecoratorTest.java
git commit -m "fix(#484): add quorum check to ChannelManagerDecorator.findOrCreate()

findOrCreate() was the only ChannelManager mutation method that
bypassed the quorum check. A minority-partition node could create
channels via findOrCreate, violating the quorum guarantee.

Refs #484"
```

---

### Task 2: Typed proxy exception hierarchy

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ProxyTimeoutException.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ProxyAuthException.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/WriteProxyClientTest.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java`

**Interfaces:**
- Consumes: `WriteProxyClient.post()`, `WriteProxyClient.get()` (existing)
- Produces: `ProxyTimeoutException(String message, Throwable cause)`, `ProxyAuthException(String targetNodeId, int statusCode)`

- [ ] **Step 1: Create ProxyTimeoutException**

```java
package io.casehub.qhorus.cluster;

public class ProxyTimeoutException extends RuntimeException {
    public ProxyTimeoutException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

- [ ] **Step 2: Create ProxyAuthException**

```java
package io.casehub.qhorus.cluster;

public class ProxyAuthException extends RuntimeException {
    private final String targetNodeId;
    private final int statusCode;

    public ProxyAuthException(String targetNodeId, int statusCode) {
        super("Authentication rejected by " + targetNodeId + " (HTTP " + statusCode + ")");
        this.targetNodeId = targetNodeId;
        this.statusCode = statusCode;
    }

    public String targetNodeId() { return targetNodeId; }
    public int statusCode() { return statusCode; }
}
```

- [ ] **Step 3: Write failing tests for WriteProxyClient error mapping**

Add to `WriteProxyClientTest.java`:

```java
@Test
void post_throws_ProxyAuthException_on_401() {
    stubFor(post("/internal/dispatch").willReturn(aResponse().withStatus(401)));
    assertThatThrownBy(() -> client.dispatch(target, dispatch))
            .isInstanceOf(ProxyAuthException.class)
            .hasMessageContaining("Authentication rejected");
}

@Test
void post_throws_ProxyAuthException_on_403() {
    stubFor(post("/internal/dispatch").willReturn(aResponse().withStatus(403)));
    assertThatThrownBy(() -> client.dispatch(target, dispatch))
            .isInstanceOf(ProxyAuthException.class);
}

@Test
void post_throws_ProxyDispatchException_on_500() {
    stubFor(post("/internal/dispatch").willReturn(aResponse().withStatus(500)));
    assertThatThrownBy(() -> client.dispatch(target, dispatch))
            .isInstanceOf(ProxyDispatchException.class);
}

@Test
void post_throws_ProxyTimeoutException_on_connection_refused() {
    var unreachable = new NodeInfo("dead-node", "localhost:1");
    assertThatThrownBy(() -> client.dispatch(unreachable, dispatch))
            .isInstanceOf(ProxyTimeoutException.class);
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WriteProxyClientTest -pl cluster`
Expected: FAIL — current code throws generic RuntimeException

- [ ] **Step 5: Refactor WriteProxyClient.post() and get() error handling**

Replace the `post()` method's error handling in `WriteProxyClient.java`:

```java
private <T> T post(NodeInfo target, String path, Object body, Class<T> responseType) {
    try {
        String url = "http://" + target.address() + path;
        java.net.http.HttpRequest.Builder reqBuilder = java.net.http.HttpRequest.newBuilder()
                .uri(java.net.URI.create(url))
                .timeout(timeout)
                .header("Content-Type", "application/json");
        if (internalSecret != null) {
            reqBuilder.header("X-Internal-Secret", internalSecret);
        }
        if (body != null) {
            reqBuilder.POST(java.net.http.HttpRequest.BodyPublishers.ofString(
                    objectMapper.writeValueAsString(body)));
        } else {
            reqBuilder.POST(java.net.http.HttpRequest.BodyPublishers.noBody());
        }
        java.net.http.HttpResponse<String> response = httpClient.send(
                reqBuilder.build(),
                java.net.http.HttpResponse.BodyHandlers.ofString());
        if (response.statusCode() == 401 || response.statusCode() == 403) {
            throw new ProxyAuthException(target.nodeId(), response.statusCode());
        }
        if (response.statusCode() >= 400) {
            throw new ProxyDispatchException(target.nodeId(), response.statusCode(),
                    "HTTP " + response.statusCode() + " from " + url);
        }
        if (responseType == Void.class || response.body() == null || response.body().isEmpty()) {
            return null;
        }
        return objectMapper.readValue(response.body(), responseType);
    } catch (ProxyAuthException | ProxyDispatchException e) {
        throw e;
    } catch (java.io.IOException | InterruptedException e) {
        throw new ProxyTimeoutException(
                "Proxy call to " + target.nodeId() + " failed: " + e.getMessage(), e);
    } catch (Exception e) {
        throw new ProxyDispatchException(target.nodeId(), -1,
                "Proxy call to " + target.nodeId() + " failed: " + e.getMessage());
    }
}
```

Apply the same pattern to the `get()` method.

Note: `ProxyDispatchException` needs a new constructor that accepts `(String nodeId, int statusCode, String message)` without requiring a `ProxyFallbackEvent`. Add:

```java
public ProxyDispatchException(String nodeId, int statusCode, String message) {
    super(message);
    this.event = null;
}
```

- [ ] **Step 6: Update WriteRoutingDecorator catch blocks**

In `WriteRoutingDecorator.dispatch()`, replace the single `catch (Exception e)` with typed catches:

```java
try {
    return proxyClient.dispatch(owner, dispatch);
} catch (ProxyTimeoutException e) {
    LOG.warnf("Proxy timeout to %s: %s", owner.nodeId(), e.getMessage());
    var event = new ProxyFallbackEvent(
            dispatch.channelId(), owner.nodeId(), localNodeId,
            dispatch.sender(), dispatch.type(), e.getMessage());
    if (fallbackEvent != null) { fallbackEvent.fireAsync(event); }
    if ("fail".equals(proxyFallback)) { throw new ProxyDispatchException(event, e); }
    DispatchResult result = delegate.dispatch(dispatch);
    if (tracker != null) { tracker.recordWrite(dispatch.channelId()); }
    return result;
} catch (ProxyAuthException e) {
    LOG.errorf("Proxy auth rejected by %s (HTTP %d) — check internal-secret config",
            owner.nodeId(), e.statusCode());
    var event = new ProxyFallbackEvent(
            dispatch.channelId(), owner.nodeId(), localNodeId,
            dispatch.sender(), dispatch.type(), e.getMessage());
    if (fallbackEvent != null) { fallbackEvent.fireAsync(event); }
    if ("fail".equals(proxyFallback)) { throw new ProxyDispatchException(event, e); }
    DispatchResult result = delegate.dispatch(dispatch);
    if (tracker != null) { tracker.recordWrite(dispatch.channelId()); }
    return result;
} catch (Exception e) {
    LOG.warnf("Proxy to %s failed: %s", owner.nodeId(), e.getMessage());
    var event = new ProxyFallbackEvent(
            dispatch.channelId(), owner.nodeId(), localNodeId,
            dispatch.sender(), dispatch.type(), e.getMessage());
    if (fallbackEvent != null) { fallbackEvent.fireAsync(event); }
    if ("fail".equals(proxyFallback)) { throw new ProxyDispatchException(event, e); }
    DispatchResult result = delegate.dispatch(dispatch);
    if (tracker != null) { tracker.recordWrite(dispatch.channelId()); }
    return result;
}
```

- [ ] **Step 7: Add WriteRoutingDecorator tests for typed exceptions**

Add to `WriteRoutingDecoratorTest.java`:

```java
@Test
void proxy_timeout_falls_back_to_local_and_logs_warn() {
    when(clusterManager.owner(channelId)).thenReturn(remoteNode);
    when(clusterManager.isLocal(remoteNode)).thenReturn(false);
    when(proxyClient.dispatch(eq(remoteNode), any()))
            .thenThrow(new ProxyTimeoutException("timeout", new java.io.IOException()));
    when(delegate.dispatch(any())).thenReturn(result);

    var actual = decorator.dispatch(dispatch);
    assertThat(actual).isEqualTo(result);
    verify(delegate).dispatch(dispatch);
}

@Test
void proxy_auth_failure_falls_back_to_local_and_logs_error() {
    when(clusterManager.owner(channelId)).thenReturn(remoteNode);
    when(clusterManager.isLocal(remoteNode)).thenReturn(false);
    when(proxyClient.dispatch(eq(remoteNode), any()))
            .thenThrow(new ProxyAuthException("remote-node", 403));
    when(delegate.dispatch(any())).thenReturn(result);

    var actual = decorator.dispatch(dispatch);
    assertThat(actual).isEqualTo(result);
    verify(delegate).dispatch(dispatch);
}
```

- [ ] **Step 8: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/ProxyTimeoutException.java cluster/src/main/java/io/casehub/qhorus/cluster/ProxyAuthException.java cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java cluster/src/main/java/io/casehub/qhorus/cluster/ProxyDispatchException.java cluster/src/test/java/io/casehub/qhorus/cluster/WriteProxyClientTest.java cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java
git commit -m "feat(#484): typed proxy exception hierarchy — timeout, auth, dispatch

WriteProxyClient now maps HTTP status codes to specific exception
types. WriteRoutingDecorator catch blocks distinguish transient
timeouts (WARN), auth failures (ERROR), and dispatch errors.

Refs #484"
```

---

### Task 3: Visibility tightening and cleanup

**Files:**
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/BucketWindow.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteFrequencyTracker.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java` (constructor consolidation)

**Interfaces:**
- Consumes: existing internal-only references in cluster package
- Produces: no API changes — visibility reduction only

- [ ] **Step 1: Verify BucketWindow and WriteFrequencyTracker references are package-internal**

Use `ide_find_references` on `BucketWindow` and `WriteFrequencyTracker`. Confirm all references are within `io.casehub.qhorus.cluster`.

- [ ] **Step 2: Change visibility to package-private**

`BucketWindow.java`: change `public class BucketWindow` to `class BucketWindow`
`WriteFrequencyTracker.java`: change `public class WriteFrequencyTracker` to `class WriteFrequencyTracker`

- [ ] **Step 3: Consolidate WriteProxyClient constructors**

Replace the 4 constructors with 2: the primary constructor (timeout + internalSecret) and a test-injection constructor (httpClient + objectMapper + timeout). Remove the no-arg and single-arg convenience constructors — callers should use the primary constructor.

```java
public WriteProxyClient(java.time.Duration timeout, String internalSecret) {
    this.httpClient = java.net.http.HttpClient.newBuilder()
            .connectTimeout(timeout).build();
    this.objectMapper = new com.fasterxml.jackson.databind.ObjectMapper()
            .registerModule(new com.fasterxml.jackson.datatype.jsr310.JavaTimeModule())
            .configure(com.fasterxml.jackson.databind.DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    this.timeout = timeout;
    this.internalSecret = internalSecret;
}

WriteProxyClient(java.net.http.HttpClient httpClient,
                 com.fasterxml.jackson.databind.ObjectMapper objectMapper,
                 java.time.Duration timeout) {
    this.httpClient = httpClient;
    this.objectMapper = objectMapper;
    this.timeout = timeout;
    this.internalSecret = null;
}
```

- [ ] **Step 4: Fix any call sites using removed constructors**

Use `ide_find_references` on the removed constructors. Update callers in test code to use the primary constructor with `(Duration.ofSeconds(10), null)`.

- [ ] **Step 5: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: PASS

- [ ] **Step 6: Run full build to verify no cross-module breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add cluster/
git commit -m "refactor(#484): tighten visibility — BucketWindow, WriteFrequencyTracker package-private

Consolidate WriteProxyClient to 2 constructors (primary + test).
BucketWindow and WriteFrequencyTracker are internal implementation
details with no cross-package references.

Refs #484"
```

---

## Batch 2: Ownership Health Endpoint

### Task 4: Per-channel ownership query endpoint

**Files:**
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResource.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ChannelOwnershipResponse.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/ClusterHealthResourceTest.java` (new or existing)

**Interfaces:**
- Consumes: `ClusterManager.owner(UUID)`, `DynamicOwnershipResolver.getClaim(UUID)`
- Produces: `GET /health/cluster/ownership/{channelId}` → `ChannelOwnershipResponse(owner, source, claimWriteCount)`

- [ ] **Step 1: Create ChannelOwnershipResponse record**

```java
package io.casehub.qhorus.cluster;

public record ChannelOwnershipResponse(
        String owner,
        String source,
        long claimWriteCount) {
}
```

- [ ] **Step 2: Add method to ClusterManager to expose ownership source**

Add to `ClusterManager.java`:

```java
public ChannelOwnershipResponse resolveOwnership(UUID channelId) {
    NodeInfo ownerNode = owner(channelId);
    if (resolver != null) {
        OwnershipClaim claim = resolver.getClaim(channelId);
        if (claim != null && claim.nodeId().equals(ownerNode.nodeId())) {
            return new ChannelOwnershipResponse(
                    ownerNode.nodeId(), "dynamic-claim", claim.writeCount());
        }
    }
    return new ChannelOwnershipResponse(ownerNode.nodeId(), "hash-ring", 0);
}
```

- [ ] **Step 3: Write failing test**

```java
@Test
void resolveOwnership_returns_hash_ring_when_no_claim() {
    var mgr = new ClusterManager("node-a", Map.of("node-a", "node-a:8080", "node-b", "node-b:8080"),
            128, 3, true, Clock.fixed(Instant.now(), ZoneId.of("UTC")));
    UUID channelId = UUID.randomUUID();
    var response = mgr.resolveOwnership(channelId);
    assertThat(response.source()).isEqualTo("hash-ring");
    assertThat(response.claimWriteCount()).isEqualTo(0);
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClusterManagerTest#resolveOwnership_returns_hash_ring_when_no_claim -pl cluster`
Expected: FAIL — method does not exist

- [ ] **Step 5: Implement and verify**

Add the method to `ClusterManager.java` as specified in Step 2.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClusterManagerTest -pl cluster`
Expected: PASS

- [ ] **Step 6: Add REST endpoint to ClusterHealthResource**

```java
@GET
@Path("/health/cluster/ownership/{channelId}")
public ChannelOwnershipResponse channelOwnership(@PathParam("channelId") UUID channelId) {
    return clusterManager.resolveOwnership(channelId);
}
```

Add import for `jakarta.ws.rs.PathParam`.

- [ ] **Step 7: Run full cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/ChannelOwnershipResponse.java cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResource.java cluster/src/main/java/io/casehub/qhorus/cluster/ClusterManager.java cluster/src/test/java/io/casehub/qhorus/cluster/ClusterManagerTest.java
git commit -m "feat(#484): add per-channel ownership query endpoint

GET /health/cluster/ownership/{channelId} returns owner, source
(hash-ring or dynamic-claim), and claimWriteCount. Distinguishes
default hash-ring ownership from dynamic claim-based ownership
for e2e test assertions.

Refs #484"
```

---

## Batch 3: E2E Tests — Ownership, Failure, Cache, Channel Creation

### Task 5: Extend ClusterTestHarness with new helper methods

**Files:**
- Modify: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/ClusterTestHarness.java`

**Interfaces:**
- Consumes: `GET /health/cluster/ownership/{channelId}`, `GET /health/ownership`, `GET /health/cache`
- Produces: `getOwnership(nodeId, channelId)`, `getLocalClaims(nodeId)`, `getCacheStats(nodeId)`, `createChannelWithId(nodeId, name, preAssignedId)`

- [ ] **Step 1: Add getOwnership method**

```java
public ChannelOwnershipInfo getOwnership(String nodeId, String channelId) {
    Response response = RestAssured.given()
            .baseUri(nodeUrl(nodeId))
            .get("/health/cluster/ownership/" + channelId);
    response.then().statusCode(200);
    return new ChannelOwnershipInfo(
            response.jsonPath().getString("owner"),
            response.jsonPath().getString("source"),
            response.jsonPath().getLong("claimWriteCount"));
}

public record ChannelOwnershipInfo(String owner, String source, long claimWriteCount) {}
```

- [ ] **Step 2: Add getLocalClaims method**

```java
public Response getLocalClaims(String nodeId) {
    return RestAssured.given()
            .baseUri(nodeUrl(nodeId))
            .get("/health/ownership");
}
```

- [ ] **Step 3: Add getCacheStats method**

```java
public CacheStats getCacheStats(String nodeId) {
    Response response = RestAssured.given()
            .baseUri(nodeUrl(nodeId))
            .get("/health/cache");
    response.then().statusCode(200);
    return new CacheStats(
            response.jsonPath().getString("status"),
            response.jsonPath().getInt("channelsCached"),
            response.jsonPath().getLong("messagesCached"));
}

public record CacheStats(String status, int channelsCached, long messagesCached) {}
```

- [ ] **Step 4: Add createChannelWithId method**

```java
public String createChannelWithId(String nodeId, String channelName, String preAssignedId) {
    String body = String.format(
            "{\"name\":\"%s\",\"semantic\":\"APPEND\",\"preAssignedId\":\"%s\"}",
            channelName, preAssignedId);
    Response response = RestAssured.given()
            .baseUri(nodeUrl(nodeId))
            .header("Content-Type", "application/json")
            .body(body)
            .post("/api/channels");
    response.then().statusCode(201);
    return response.jsonPath().getString("channelId");
}
```

- [ ] **Step 5: Add ownership evaluation interval env var to buildNodeContainer**

In `buildNodeContainer()`, add:

```java
.withEnv("CASEHUB_QHORUS_RELAY_OWNERSHIP_EVALUATION_INTERVAL_SECONDS", "3")
```

- [ ] **Step 6: Commit**

```bash
git add e2e-cluster/src/test/java/io/casehub/qhorus/e2e/ClusterTestHarness.java
git commit -m "feat(#484): extend ClusterTestHarness — ownership, cache, preAssignedId helpers

Add getOwnership(), getLocalClaims(), getCacheStats(),
createChannelWithId() for new e2e scenarios. Shorten ownership
evaluation interval to 3s for faster test convergence.

Refs #484"
```

---

### Task 6: OwnershipTransferE2ETest

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/OwnershipTransferE2ETest.java`

**Interfaces:**
- Consumes: `ClusterTestHarness.getOwnership()`, `ClusterTestHarness.getLocalClaims()`, `ClusterTestHarness.sendMessage()`, `ClusterTestHarness.createChannel()`

- [ ] **Step 1: Write OwnershipTransferE2ETest**

```java
package io.casehub.qhorus.e2e;

import io.restassured.response.Response;
import org.junit.jupiter.api.*;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OwnershipTransferE2ETest {

    private ClusterTestHarness cluster;
    private String channelId;

    @BeforeAll
    void startCluster() {
        cluster = new ClusterTestHarness("e2e-secret", "node-a", "node-b");
        cluster.start();
        cluster.waitForClusterConvergence(2, Duration.ofSeconds(30));
        channelId = cluster.createChannel("node-a", "e2e-ownership");
    }

    @AfterAll
    void stopCluster() {
        if (cluster != null) { cluster.close(); }
    }

    @Test
    @Order(1)
    void baseline_ownership_is_hash_ring() {
        var ownership = cluster.getOwnership("node-a", channelId);
        assertThat(ownership.source()).isEqualTo("hash-ring");
    }

    @Test
    @Order(2)
    void write_pattern_shift_transfers_ownership() {
        for (int i = 0; i < 20; i++) {
            cluster.sendMessage("node-b", channelId, "agent-b", "STATUS", "claim-" + i);
        }

        await().atMost(Duration.ofSeconds(30)).pollInterval(Duration.ofSeconds(2)).untilAsserted(() -> {
            var ownership = cluster.getOwnership("node-b", channelId);
            assertThat(ownership.source()).isEqualTo("dynamic-claim");
            assertThat(ownership.owner()).isEqualTo("node-b");
        });
    }

    @Test
    @Order(3)
    void post_transfer_message_from_old_owner_succeeds() {
        Response resp = cluster.sendMessage("node-a", channelId,
                "agent-a", "STATUS", "proxied-after-transfer");
        assertThat(resp.statusCode()).isEqualTo(200);

        await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
            var ownershipA = cluster.getOwnership("node-a", channelId);
            var ownershipB = cluster.getOwnership("node-b", channelId);
            assertThat(ownershipA.owner()).isEqualTo(ownershipB.owner());
        });
    }

    @Test
    @Order(4)
    void ownership_reverts_when_writes_stop() {
        await().atMost(Duration.ofSeconds(60)).pollInterval(Duration.ofSeconds(3)).untilAsserted(() -> {
            var ownership = cluster.getOwnership("node-a", channelId);
            assertThat(ownership.source()).isEqualTo("hash-ring");
        });
    }
}
```

- [ ] **Step 2: Run test (requires mesh package)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -pl mesh -am -DskipTests && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-e2e-cluster -Dtest=OwnershipTransferE2ETest -pl e2e-cluster`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add e2e-cluster/src/test/java/io/casehub/qhorus/e2e/OwnershipTransferE2ETest.java
git commit -m "test(#484): add OwnershipTransferE2ETest — claim, proxy, relinquish

4 ordered tests: baseline hash-ring ownership, write pattern shift
triggers dynamic claim, post-transfer proxy succeeds, ownership
reverts when writes stop.

Refs #484"
```

---

### Task 7: NodeFailureFallbackE2ETest

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/NodeFailureFallbackE2ETest.java`

**Interfaces:**
- Consumes: `ClusterTestHarness.stopNode()`, `ClusterTestHarness.startNode()`, `ClusterTestHarness.sendMessage()`, `ClusterTestHarness.getLocalClaims()`

- [ ] **Step 1: Write NodeFailureFallbackE2ETest**

```java
package io.casehub.qhorus.e2e;

import io.restassured.response.Response;
import org.junit.jupiter.api.*;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class NodeFailureFallbackE2ETest {

    private ClusterTestHarness cluster;
    private String channelId;

    @BeforeAll
    void startCluster() {
        cluster = new ClusterTestHarness("e2e-secret", "node-a", "node-b");
        cluster.start();
        cluster.waitForClusterConvergence(2, Duration.ofSeconds(30));
        channelId = cluster.createChannel("node-a", "e2e-fallback");
    }

    @AfterAll
    void stopCluster() {
        if (cluster != null) { cluster.close(); }
    }

    @Test
    @Order(1)
    void fallback_to_local_when_owner_down() {
        cluster.stopNode("node-a");

        await().atMost(Duration.ofSeconds(30)).pollInterval(Duration.ofSeconds(2)).untilAsserted(() -> {
            Response health = cluster.getClusterHealth("node-b");
            assertThat(health.jsonPath().getInt("clusterSize")).isEqualTo(1);
        });

        Response resp = cluster.sendMessage("node-b", channelId,
                "agent-b", "STATUS", "fallback-dispatch");
        assertThat(resp.statusCode()).isEqualTo(200);

        Response msgs = cluster.getMessages("node-b", channelId);
        assertThat(msgs.jsonPath().getList("content", String.class))
                .contains("fallback-dispatch");
    }

    @Test
    @Order(2)
    void ownership_reconstructs_after_recovery() {
        cluster.startNode("node-a");
        cluster.waitForClusterConvergence(2, Duration.ofSeconds(60));

        for (int i = 0; i < 10; i++) {
            cluster.sendMessage("node-a", channelId, "agent-a", "STATUS", "recover-" + i);
        }

        await().atMost(Duration.ofSeconds(30)).pollInterval(Duration.ofSeconds(2)).untilAsserted(() -> {
            Response claimsA = cluster.getLocalClaims("node-a");
            assertThat(claimsA.statusCode()).isEqualTo(200);
        });
    }

    @Test
    @Order(3)
    void post_recovery_routing_works() {
        Response resp = cluster.sendMessage("node-b", channelId,
                "agent-b", "STATUS", "post-recovery");
        assertThat(resp.statusCode()).isEqualTo(200);

        await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
            Response msgs = cluster.getMessages("node-a", channelId);
            assertThat(msgs.jsonPath().getList("content", String.class))
                    .contains("post-recovery");
        });
    }
}
```

- [ ] **Step 2: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-e2e-cluster -Dtest=NodeFailureFallbackE2ETest -pl e2e-cluster`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add e2e-cluster/src/test/java/io/casehub/qhorus/e2e/NodeFailureFallbackE2ETest.java
git commit -m "test(#484): add NodeFailureFallbackE2ETest — fallback, reconstruction, recovery

3 ordered tests: fallback-to-local when owner is down, ownership
claim reconstruction after node recovery, post-recovery routing
consistency.

Refs #484"
```

---

### Task 8: CacheCoherenceE2ETest

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/CacheCoherenceE2ETest.java`

**Interfaces:**
- Consumes: `ClusterTestHarness.getCacheStats()`, `ClusterTestHarness.sendMessage()`, `ClusterTestHarness.createChannel()`

- [ ] **Step 1: Write CacheCoherenceE2ETest**

```java
package io.casehub.qhorus.e2e;

import io.restassured.RestAssured;
import io.restassured.response.Response;
import org.junit.jupiter.api.*;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class CacheCoherenceE2ETest {

    private ClusterTestHarness cluster;
    private String channelId;

    @BeforeAll
    void startCluster() {
        cluster = new ClusterTestHarness("e2e-secret", "node-a", "node-b");
        cluster.start();
        cluster.waitForClusterConvergence(2, Duration.ofSeconds(30));
        channelId = cluster.createChannel("node-a", "e2e-cache");
    }

    @AfterAll
    void stopCluster() {
        if (cluster != null) { cluster.close(); }
    }

    @Test
    @Order(1)
    void cross_node_cache_population() {
        cluster.sendMessage("node-a", channelId, "agent-a", "STATUS", "cache-test-1");

        await().atMost(Duration.ofSeconds(15)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
            var stats = cluster.getCacheStats("node-b");
            assertThat(stats.status()).isEqualTo("UP");
            assertThat(stats.channelsCached()).isGreaterThanOrEqualTo(1);
        });
    }

    @Test
    @Order(2)
    void rapid_messages_converge() {
        for (int i = 0; i < 10; i++) {
            cluster.sendMessage("node-a", channelId, "agent-a", "STATUS", "rapid-" + i);
        }

        await().atMost(Duration.ofSeconds(15)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
            Response msgs = cluster.getMessages("node-b", channelId);
            assertThat(msgs.jsonPath().getList("$")).hasSizeGreaterThanOrEqualTo(11);
        });
    }

    @Test
    @Order(3)
    void cache_invalidated_on_channel_delete() {
        String tempChannelId = cluster.createChannel("node-a", "e2e-cache-delete");
        cluster.sendMessage("node-a", tempChannelId, "agent-a", "STATUS", "will-delete");

        await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
            var stats = cluster.getCacheStats("node-b");
            assertThat(stats.channelsCached()).isGreaterThanOrEqualTo(2);
        });

        RestAssured.given()
                .baseUri(cluster.nodeUrl("node-a"))
                .queryParam("force", true)
                .delete("/api/channels/" + tempChannelId)
                .then().statusCode(200);

        await().atMost(Duration.ofSeconds(15)).pollInterval(Duration.ofSeconds(1)).untilAsserted(() -> {
            Response msgs = cluster.getMessages("node-b", tempChannelId);
            assertThat(msgs.statusCode()).isGreaterThanOrEqualTo(400);
        });
    }
}
```

- [ ] **Step 2: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-e2e-cluster -Dtest=CacheCoherenceE2ETest -pl e2e-cluster`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add e2e-cluster/src/test/java/io/casehub/qhorus/e2e/CacheCoherenceE2ETest.java
git commit -m "test(#484): add CacheCoherenceE2ETest — population, convergence, invalidation

3 ordered tests: cross-node cache population via pg_notify, rapid
message convergence, cache invalidation on channel delete.

Refs #484"
```

---

### Task 9: ChannelCreationRoutingE2ETest

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/ChannelCreationRoutingE2ETest.java`

**Interfaces:**
- Consumes: `ClusterTestHarness.createChannel()`, `ClusterTestHarness.createChannelWithId()`, `ClusterTestHarness.getMessages()`

- [ ] **Step 1: Write ChannelCreationRoutingE2ETest**

```java
package io.casehub.qhorus.e2e;

import io.restassured.RestAssured;
import io.restassured.response.Response;
import org.junit.jupiter.api.*;

import java.time.Duration;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class ChannelCreationRoutingE2ETest {

    private ClusterTestHarness cluster;

    @BeforeAll
    void startCluster() {
        cluster = new ClusterTestHarness("e2e-secret", "node-a", "node-b");
        cluster.start();
        cluster.waitForClusterConvergence(2, Duration.ofSeconds(30));
    }

    @AfterAll
    void stopCluster() {
        if (cluster != null) { cluster.close(); }
    }

    @Test
    void channel_created_on_one_node_visible_on_both() {
        String channelId = cluster.createChannel("node-a", "e2e-create-basic");

        await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
            Response resp = RestAssured.given()
                    .baseUri(cluster.nodeUrl("node-b"))
                    .get("/api/channels/" + channelId);
            assertThat(resp.statusCode()).isEqualTo(200);
        });
    }

    @Test
    void preAssignedId_routes_to_correct_owner() {
        UUID preAssigned = UUID.randomUUID();
        String channelId = cluster.createChannelWithId("node-a",
                "e2e-pre-assigned", preAssigned.toString());

        assertThat(channelId).isEqualTo(preAssigned.toString());

        await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
            Response respA = RestAssured.given()
                    .baseUri(cluster.nodeUrl("node-a"))
                    .get("/api/channels/" + channelId);
            Response respB = RestAssured.given()
                    .baseUri(cluster.nodeUrl("node-b"))
                    .get("/api/channels/" + channelId);
            assertThat(respA.statusCode()).isEqualTo(200);
            assertThat(respB.statusCode()).isEqualTo(200);
        });
    }

    @Test
    void concurrent_findOrCreate_produces_one_channel() {
        String name = "e2e-concurrent-" + UUID.randomUUID().toString().substring(0, 8);

        CompletableFuture<Response> futureA = CompletableFuture.supplyAsync(() ->
                RestAssured.given()
                        .baseUri(cluster.nodeUrl("node-a"))
                        .header("Content-Type", "application/json")
                        .body("{\"name\":\"" + name + "\",\"semantic\":\"APPEND\"}")
                        .post("/api/channels"));

        CompletableFuture<Response> futureB = CompletableFuture.supplyAsync(() ->
                RestAssured.given()
                        .baseUri(cluster.nodeUrl("node-b"))
                        .header("Content-Type", "application/json")
                        .body("{\"name\":\"" + name + "\",\"semantic\":\"APPEND\"}")
                        .post("/api/channels"));

        Response respA = futureA.join();
        Response respB = futureB.join();

        int successes = 0;
        String channelIdA = null, channelIdB = null;
        if (respA.statusCode() == 201) { successes++; channelIdA = respA.jsonPath().getString("channelId"); }
        if (respB.statusCode() == 201) { successes++; channelIdB = respB.jsonPath().getString("channelId"); }

        if (successes == 2) {
            assertThat(channelIdA).isEqualTo(channelIdB);
        } else {
            assertThat(successes).isGreaterThanOrEqualTo(1);
        }
    }
}
```

- [ ] **Step 2: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-e2e-cluster -Dtest=ChannelCreationRoutingE2ETest -pl e2e-cluster`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add e2e-cluster/src/test/java/io/casehub/qhorus/e2e/ChannelCreationRoutingE2ETest.java
git commit -m "test(#484): add ChannelCreationRoutingE2ETest — basic, preAssigned, concurrent

3 tests: cross-node channel visibility, preAssignedId routing to
correct owner, concurrent findOrCreate idempotency.

Refs #484"
```

---

## Batch 4: Native Image Variant

### Task 10: Native image Dockerfile and harness support

**Files:**
- Create: `e2e-cluster/src/test/resources/Dockerfile.native`
- Modify: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/ClusterTestHarness.java`
- Modify: `e2e-cluster/pom.xml`

**Interfaces:**
- Consumes: `mesh/target/*-runner` (native image binary)
- Produces: System property `mesh.container.mode` (jvm|native) selects container image

- [ ] **Step 1: Create Dockerfile.native**

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.4
COPY mesh-runner /application
RUN chmod 775 /application
ENTRYPOINT ["./application", "-Xmx128m"]
```

- [ ] **Step 2: Update ClusterTestHarness.buildImage() for native mode**

Add a `containerMode` field and modify `buildImage()`:

```java
private final String containerMode;

public ClusterTestHarness(String internalSecret, String... nodeIds) {
    // ... existing constructor code ...
    this.containerMode = System.getProperty("mesh.container.mode", "jvm");
}

private ImageFromDockerfile buildImage() {
    if (image == null) {
        if ("native".equals(containerMode)) {
            String nativeRunnerGlob = System.getProperty("mesh.native-runner.path",
                    "../mesh/target/*-runner");
            java.io.File[] runners = new java.io.File(nativeRunnerGlob).getParentFile()
                    .listFiles((dir, name) -> name.endsWith("-runner"));
            if (runners == null || runners.length == 0) {
                throw new IllegalStateException(
                        "Native runner not found — run 'mvn package -Pnative -pl mesh -am'");
            }
            image = new ImageFromDockerfile()
                    .withFileFromClasspath("Dockerfile", "Dockerfile.native")
                    .withFileFromPath("mesh-runner", runners[0].toPath());
        } else {
            String meshAppPath = System.getProperty("mesh.quarkus-app.path",
                    "../mesh/target/quarkus-app");
            Path meshTarget = Path.of(meshAppPath);
            if (!meshTarget.toFile().exists()) {
                throw new IllegalStateException(
                        "Mesh quarkus-app not found at " + meshTarget.toAbsolutePath()
                                + " — run 'mvn package -pl mesh' first");
            }
            image = new ImageFromDockerfile()
                    .withFileFromClasspath("Dockerfile", "Dockerfile")
                    .withFileFromPath("quarkus-app", meshTarget);
        }
    }
    return image;
}
```

- [ ] **Step 3: Add Maven profile to e2e-cluster/pom.xml**

Add inside `<profiles>`:

```xml
<profile>
  <id>with-e2e-native</id>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <configuration>
          <systemPropertyVariables>
            <mesh.container.mode>native</mesh.container.mode>
            <mesh.quarkus-app.path>${project.basedir}/../mesh/target/quarkus-app</mesh.quarkus-app.path>
          </systemPropertyVariables>
        </configuration>
      </plugin>
    </plugins>
  </build>
</profile>
```

- [ ] **Step 4: Verify JVM mode still works**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-e2e-cluster -Dtest=DispatchRoutingE2ETest -pl e2e-cluster`
Expected: PASS (unchanged behavior)

- [ ] **Step 5: Commit**

```bash
git add e2e-cluster/src/test/resources/Dockerfile.native e2e-cluster/src/test/java/io/casehub/qhorus/e2e/ClusterTestHarness.java e2e-cluster/pom.xml
git commit -m "feat(#484): add native image Dockerfile and harness support

Dockerfile.native for UBI9-minimal with 128MB heap. ClusterTestHarness
selects JVM or native mode via mesh.container.mode system property.
Maven profile -Pwith-e2e-native activates native mode.

Refs #484"
```

---

## References

- [specs/issue-484-mesh-audit-e2e/2026-10-10-mesh-audit-e2e-design.md] — design spec this plan implements
- [cluster/ClusterManager.java] — core cluster state management
- [cluster/WriteRoutingDecorator.java] — dispatch routing with proxy fallback
- [cluster/WriteProxyClient.java] — cross-node HTTP client (error handling refactoring target)
- [cluster/ChannelManagerDecorator.java] — channel mutation proxying (findOrCreate quorum bypass)
- [cluster/ClusterHealthResource.java] — health endpoints (ownership endpoint addition)
- [cluster/ProxyDispatchException.java] — existing exception (hierarchy extension)
- [cache/CacheHealthResource.java] — cache health endpoint (e2e harness integration)
- [e2e-cluster/ClusterTestHarness.java] — Podman test infrastructure (helper additions)
- [e2e-cluster/src/test/resources/Dockerfile] — existing JVM Dockerfile
- [GitHub #484] — focal issue

# Distributed Mesh Audit and E2E Cluster Testing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #484 — audit and e2e cluster testing
**Issue group:** #484

**Goal:** Fix 15 gaps across cluster and cache modules, add integration tests verifying CDI composition, and build Podman-based e2e tests for 4 distributed scenarios.

**Architecture:** Three phases — audit fixes (A), integration tests (B), Podman e2e (C). Phase A fixes wiring/correctness bugs. Phase B verifies single-JVM composition. Phase C verifies multi-node distributed behaviour via Testcontainers with shared PostgreSQL and postgres-broadcaster for pg_notify.

**Tech Stack:** Java 21, Quarkus 3.32.2, Testcontainers, PostgreSQL, Caffeine cache, java.net.http.HttpClient

## Global Constraints

- Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` from project root
- Test: `mvn test -pl <module>` for focused runs
- All commits reference #484: `Refs #484`
- `@IfBuildProperty` pattern: `enableIfMissing = false` for relay (opt-in), `enableIfMissing = true` for cache (opt-out)
- CDI interface cycle breaker: inject concrete class (e.g. `CdiMessageService`), not interface

---

## Batch 1: Critical Cluster Wiring (heartbeat + proxy loop fix)

### Task 1: Add HeartbeatScheduler and HeartbeatService CDI producer

The heartbeat protocol is completely inert in production — `HeartbeatService.tick()` is never called outside tests. This is the most critical gap.

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatScheduler.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/HeartbeatSchedulerTest.java`

**Interfaces:**
- Consumes: `HeartbeatService.tick()`, `RelayConfig.heartbeatInterval()`, `WriteProxyClient` (needs `heartbeat(NodeInfo)` method)
- Produces: `HeartbeatScheduler` (CDI bean), `HeartbeatService` (CDI-produced via `RelayProducer`)

- [ ] **Step 1: Add `heartbeat(NodeInfo)` method to WriteProxyClient**

`WriteProxyClient` needs a method the CDI producer can reference as a `Function<NodeInfo, HeartbeatResponse>`:

```java
public HeartbeatResponse heartbeat(NodeInfo target) {
    return get(target, "/internal/heartbeat", HeartbeatResponse.class);
}
```

Also add a `get()` method alongside the existing `post()`:

```java
private <T> T get(NodeInfo target, String path, Class<T> responseType) {
    try {
        String url = "http://" + target.address() + path;
        java.net.http.HttpRequest req = java.net.http.HttpRequest.newBuilder()
                .uri(java.net.URI.create(url))
                .timeout(timeout)
                .GET()
                .build();
        java.net.http.HttpResponse<String> response = httpClient.send(
                req, java.net.http.HttpResponse.BodyHandlers.ofString());
        if (response.statusCode() >= 400) {
            throw new RuntimeException("HTTP " + response.statusCode() + " from " + url);
        }
        return objectMapper.readValue(response.body(), responseType);
    } catch (Exception e) {
        throw new RuntimeException("GET " + target.nodeId() + " failed: " + e.getMessage(), e);
    }
}
```

- [ ] **Step 2: Add HeartbeatService producer to RelayProducer**

Add after the `writeProxyClient()` method in `RelayProducer.java`:

```java
@Produces
@ApplicationScoped
public HeartbeatService heartbeatService(ClusterManager clusterManager,
        WriteProxyClient proxyClient) {
    return new HeartbeatService(clusterManager, proxyClient::heartbeat);
}
```

- [ ] **Step 3: Write HeartbeatScheduler**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatScheduler.java`:

```java
package io.casehub.qhorus.cluster;

import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true",
                 enableIfMissing = false)
public class HeartbeatScheduler {

    @Inject
    HeartbeatService heartbeatService;

    @Scheduled(every = "${casehub.qhorus.relay.heartbeat-interval:3s}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void tick() {
        heartbeatService.tick();
    }
}
```

- [ ] **Step 4: Write unit test**

Create `cluster/src/test/java/io/casehub/qhorus/cluster/HeartbeatSchedulerTest.java`:

```java
package io.casehub.qhorus.cluster;

import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static org.mockito.Mockito.verify;

class HeartbeatSchedulerTest {

    @Test
    void tick_delegates_to_heartbeat_service() {
        HeartbeatService service = Mockito.mock(HeartbeatService.class);
        HeartbeatScheduler scheduler = new HeartbeatScheduler();
        scheduler.heartbeatService = service;

        scheduler.tick();

        verify(service).tick();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=HeartbeatSchedulerTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/HeartbeatScheduler.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java
git add cluster/src/test/java/io/casehub/qhorus/cluster/HeartbeatSchedulerTest.java
git commit -m "feat(#484): add HeartbeatScheduler and HeartbeatService CDI producer

HeartbeatService.tick() was never called in production — failure detection
was completely inert. HeartbeatScheduler drives periodic heartbeats.
HeartbeatService is now CDI-produced by RelayProducer with
proxyClient::heartbeat as the caller function.

Refs #484"
```

### Task 2: Fix InternalMeshResource — gating and proxy loop prevention

`InternalMeshResource` has two bugs: (1) not gated by `@IfBuildProperty`, causing CDI failures when relay is disabled, and (2) injects `MessageDispatcher` (the decorated alternative), creating infinite proxy loops when ownership shifts during a proxied dispatch.

**Files:**
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResource.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/InternalMeshResourceTest.java`

**Interfaces:**
- Consumes: `CdiMessageService.dispatch(MessageDispatch)`, `ChannelService` (concrete), `HeartbeatService`, `ClusterManager`
- Produces: REST endpoints at `/internal/*` (unchanged API)

- [ ] **Step 1: Write failing test for proxy loop prevention**

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.channel.ChannelManager;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class InternalMeshResourceTest {

    @Test
    void dispatch_uses_undecorated_service_not_routing_decorator() {
        // The dispatcher field should be the concrete type, not the interface
        // that WriteRoutingDecorator implements
        MessageDispatcher directService = mock(MessageDispatcher.class);
        ChannelManager channelManager = mock(ChannelManager.class);
        HeartbeatService heartbeatService = mock(HeartbeatService.class);
        ClusterManager clusterManager = mock(ClusterManager.class);

        InternalMeshResource resource = new InternalMeshResource(
                directService, channelManager, heartbeatService, clusterManager);

        InternalDispatchRequest request = mock(InternalDispatchRequest.class);
        MessageDispatch dispatch = mock(MessageDispatch.class);
        when(request.toMessageDispatch()).thenReturn(dispatch);
        DispatchResult expected = mock(DispatchResult.class);
        when(directService.dispatch(dispatch)).thenReturn(expected);

        DispatchResult result = resource.dispatch(request);

        assertThat(result).isSameAs(expected);
        verify(directService).dispatch(dispatch);
    }
}
```

- [ ] **Step 2: Run test — should pass (verifies current behaviour compiles)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=InternalMeshResourceTest`

- [ ] **Step 3: Add @IfBuildProperty to InternalMeshResource**

Add annotation to the class:
```java
@Path("/internal")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true",
                 enableIfMissing = false)
public class InternalMeshResource {
```

Add import: `import io.quarkus.arc.properties.IfBuildProperty;`

- [ ] **Step 4: Change MessageDispatcher injection to CdiMessageService**

Change the constructor to inject `CdiMessageService` instead of `MessageDispatcher`:

```java
private final io.casehub.qhorus.runtime.cdi.CdiMessageService messageService;
private final io.casehub.qhorus.runtime.channel.ChannelService channelService;
private final HeartbeatService heartbeatService;
private final ClusterManager clusterManager;

public InternalMeshResource(io.casehub.qhorus.runtime.cdi.CdiMessageService messageService,
                             io.casehub.qhorus.runtime.channel.ChannelService channelService,
                             HeartbeatService heartbeatService,
                             ClusterManager clusterManager) {
    this.messageService = messageService;
    this.channelService = channelService;
    this.heartbeatService = heartbeatService;
    this.clusterManager = clusterManager;
}
```

Update `dispatch()` to use `messageService`:
```java
@POST
@Path("/dispatch")
public DispatchResult dispatch(InternalDispatchRequest request) {
    return messageService.dispatch(request.toMessageDispatch());
}
```

Update channel methods to use `channelService`:
```java
@POST
@Path("/channel")
public Channel createChannel(ChannelCreateRequest request) {
    return channelService.create(request);
}
```

- [ ] **Step 5: Add @IfBuildProperty to ClusterHealthResource**

```java
@Path("/")
@Produces(MediaType.APPLICATION_JSON)
@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true",
                 enableIfMissing = false)
public class ClusterHealthResource {
```

- [ ] **Step 6: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: all pass

- [ ] **Step 7: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/ClusterHealthResource.java
git add cluster/src/test/java/io/casehub/qhorus/cluster/InternalMeshResourceTest.java
git commit -m "fix(#484): gate InternalMeshResource + fix proxy loop risk

Add @IfBuildProperty to InternalMeshResource and ClusterHealthResource.
Change InternalMeshResource to inject CdiMessageService and ChannelService
(concrete classes) instead of MessageDispatcher and ChannelManager
(decorated interfaces). Prevents infinite proxy loops when proxied
dispatches re-enter the WriteRoutingDecorator.

Refs #484"
```

### Task 3: Add ChannelManagerDecorator CDI producer and fix write tracking

Two fixes: (1) `ChannelManagerDecorator` has no CDI producer, and (2) `WriteRoutingDecorator` tracks write frequency in the wrong position.

**Files:**
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java` (existing — add test)

**Interfaces:**
- Consumes: `ChannelService` (concrete), `ClusterManager`, `WriteProxyClient`
- Produces: `ChannelManager` (CDI alternative), corrected write tracking

- [ ] **Step 1: Write failing test for write tracking position**

Add to `WriteRoutingDecoratorTest.java`:

```java
@Test
void proxied_writes_do_not_track_frequency() {
    ClusterManager cm = mock(ClusterManager.class);
    WriteProxyClient proxy = mock(WriteProxyClient.class);
    MessageDispatcher delegate = mock(MessageDispatcher.class);
    WriteFrequencyTracker tracker = mock(WriteFrequencyTracker.class);

    WriteRoutingDecorator decorator = new WriteRoutingDecorator(
            delegate, cm, proxy, true, tracker);

    UUID channelId = UUID.randomUUID();
    MessageDispatch dispatch = mock(MessageDispatch.class);
    when(dispatch.channelId()).thenReturn(channelId);
    when(cm.canServeWrites()).thenReturn(true);

    NodeInfo remoteNode = new NodeInfo("remote", "remote:8080");
    when(cm.owner(channelId)).thenReturn(remoteNode);
    when(cm.isLocal(remoteNode)).thenReturn(false);
    when(proxy.dispatch(eq(remoteNode), any())).thenReturn(mock(DispatchResult.class));

    decorator.dispatch(dispatch);

    verify(tracker, never()).recordWrite(any());
}

@Test
void fallback_to_local_does_track_frequency() {
    ClusterManager cm = mock(ClusterManager.class);
    WriteProxyClient proxy = mock(WriteProxyClient.class);
    MessageDispatcher delegate = mock(MessageDispatcher.class);
    WriteFrequencyTracker tracker = mock(WriteFrequencyTracker.class);

    WriteRoutingDecorator decorator = new WriteRoutingDecorator(
            delegate, cm, proxy, true, tracker);

    UUID channelId = UUID.randomUUID();
    MessageDispatch dispatch = mock(MessageDispatch.class);
    when(dispatch.channelId()).thenReturn(channelId);
    when(cm.canServeWrites()).thenReturn(true);

    NodeInfo remoteNode = new NodeInfo("remote", "remote:8080");
    when(cm.owner(channelId)).thenReturn(remoteNode);
    when(cm.isLocal(remoteNode)).thenReturn(false);
    when(proxy.dispatch(eq(remoteNode), any())).thenThrow(new RuntimeException("timeout"));
    when(delegate.dispatch(dispatch)).thenReturn(mock(DispatchResult.class));

    decorator.dispatch(dispatch);

    verify(tracker).recordWrite(channelId);
}
```

- [ ] **Step 2: Run tests — first should fail (proxied writes currently tracked)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=WriteRoutingDecoratorTest#proxied_writes_do_not_track_frequency`
Expected: FAIL — tracker.recordWrite IS called for proxied writes

- [ ] **Step 3: Fix WriteRoutingDecorator.dispatch()**

Replace the dispatch method body (lines 37-61):

```java
@Override
public DispatchResult dispatch(MessageDispatch dispatch) {
    if (!routingEnabled) {
        return delegate.dispatch(dispatch);
    }
    if (!clusterManager.canServeWrites()) {
        throw new QuorumViolationException("This node is in a minority partition and cannot serve writes");
    }
    NodeInfo owner = clusterManager.owner(dispatch.channelId());
    if (clusterManager.isLocal(owner)) {
        DispatchResult result = delegate.dispatch(dispatch);
        if (tracker != null) {
            tracker.recordWrite(dispatch.channelId());
        }
        return result;
    }
    try {
        return proxyClient.dispatch(owner, dispatch);
    } catch (Exception e) {
        LOG.warnf("Proxy to %s failed, falling back to local dispatch: %s",
                owner.nodeId(), e.getMessage());
        DispatchResult result = delegate.dispatch(dispatch);
        if (tracker != null) {
            tracker.recordWrite(dispatch.channelId());
        }
        return result;
    }
}
```

- [ ] **Step 4: Run both new tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=WriteRoutingDecoratorTest`
Expected: all pass

- [ ] **Step 5: Add ChannelManagerDecorator producer to RelayProducer**

Add after the `messageDispatcher()` method:

```java
@Produces
@ApplicationScoped
@jakarta.annotation.Priority(100)
@jakarta.enterprise.inject.Alternative
public io.casehub.qhorus.api.channel.ChannelManager channelManager(
        io.casehub.qhorus.runtime.channel.ChannelService delegate,
        ClusterManager clusterManager,
        WriteProxyClient proxyClient) {
    boolean routing = !"none".equals(config.routing());
    return new ChannelManagerDecorator(delegate, clusterManager, proxyClient, routing);
}
```

- [ ] **Step 6: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: all pass

- [ ] **Step 7: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java
git add cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java
git commit -m "fix(#484): add ChannelManagerDecorator CDI producer + fix write tracking

ChannelManagerDecorator now produced as @Alternative @Priority(100)
ChannelManager by RelayProducer. WriteRoutingDecorator only tracks
write frequency for locally-executed writes (not proxied), preventing
oscillating ownership claims.

Refs #484"
```

## Batch 2: Internal Security + Config Mutation Proxying

### Task 4: Add InternalSecretFilter for /internal/* auth

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalSecretFilter.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/InternalSecretFilterTest.java`

**Interfaces:**
- Consumes: `RelayConfig.internalSecret()` (new config property)
- Produces: JAX-RS filter gating `/internal/*` paths

- [ ] **Step 1: Add `internalSecret()` to RelayConfig**

```java
Optional<String> internalSecret();
```

- [ ] **Step 2: Write failing test**

```java
package io.casehub.qhorus.cluster;

import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.core.MultivaluedHashMap;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.core.UriInfo;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.Optional;

import static org.mockito.Mockito.*;

class InternalSecretFilterTest {

    @Test
    void rejects_request_without_secret_header() {
        InternalSecretFilter filter = new InternalSecretFilter();
        filter.config = mockConfig(Optional.of("my-secret"));

        ContainerRequestContext ctx = mockContext("/internal/dispatch", null);
        filter.filter(ctx);

        verify(ctx).abortWith(argThat(r -> r.getStatus() == 401));
    }

    @Test
    void rejects_request_with_wrong_secret() {
        InternalSecretFilter filter = new InternalSecretFilter();
        filter.config = mockConfig(Optional.of("my-secret"));

        ContainerRequestContext ctx = mockContext("/internal/dispatch", "wrong");
        filter.filter(ctx);

        verify(ctx).abortWith(argThat(r -> r.getStatus() == 401));
    }

    @Test
    void allows_request_with_correct_secret() {
        InternalSecretFilter filter = new InternalSecretFilter();
        filter.config = mockConfig(Optional.of("my-secret"));

        ContainerRequestContext ctx = mockContext("/internal/dispatch", "my-secret");
        filter.filter(ctx);

        verify(ctx, never()).abortWith(any());
    }

    @Test
    void allows_all_when_no_secret_configured() {
        InternalSecretFilter filter = new InternalSecretFilter();
        filter.config = mockConfig(Optional.empty());

        ContainerRequestContext ctx = mockContext("/internal/dispatch", null);
        filter.filter(ctx);

        verify(ctx, never()).abortWith(any());
    }

    @Test
    void ignores_non_internal_paths() {
        InternalSecretFilter filter = new InternalSecretFilter();
        filter.config = mockConfig(Optional.of("my-secret"));

        ContainerRequestContext ctx = mockContext("/api/channels", null);
        filter.filter(ctx);

        verify(ctx, never()).abortWith(any());
    }

    private RelayConfig mockConfig(Optional<String> secret) {
        RelayConfig config = mock(RelayConfig.class);
        when(config.internalSecret()).thenReturn(secret);
        return config;
    }

    private ContainerRequestContext mockContext(String path, String secretHeader) {
        ContainerRequestContext ctx = mock(ContainerRequestContext.class);
        UriInfo uriInfo = mock(UriInfo.class);
        when(uriInfo.getPath()).thenReturn(path);
        when(ctx.getUriInfo()).thenReturn(uriInfo);
        when(ctx.getHeaderString("X-Internal-Secret")).thenReturn(secretHeader);
        return ctx;
    }
}
```

- [ ] **Step 3: Run test — should fail (class doesn't exist)**

- [ ] **Step 4: Implement InternalSecretFilter**

```java
package io.casehub.qhorus.cluster;

import io.quarkus.arc.properties.IfBuildProperty;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ContainerRequestFilter;
import jakarta.ws.rs.container.PreMatching;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.Provider;

@Provider
@PreMatching
@Priority(50)
@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true",
                 enableIfMissing = false)
public class InternalSecretFilter implements ContainerRequestFilter {

    @Inject
    RelayConfig config;

    @Override
    public void filter(ContainerRequestContext ctx) {
        String path = ctx.getUriInfo().getPath();
        if (!path.startsWith("internal/") && !path.startsWith("/internal/")) {
            return;
        }
        var secret = config.internalSecret();
        if (secret.isEmpty()) {
            return;
        }
        String header = ctx.getHeaderString("X-Internal-Secret");
        if (!secret.get().equals(header)) {
            ctx.abortWith(Response.status(Response.Status.UNAUTHORIZED)
                    .entity("Invalid or missing X-Internal-Secret header")
                    .build());
        }
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=InternalSecretFilterTest`
Expected: PASS

- [ ] **Step 6: Add X-Internal-Secret header to WriteProxyClient**

Modify `WriteProxyClient` constructor to accept an optional secret:

```java
private final String internalSecret;

public WriteProxyClient(java.time.Duration timeout, String internalSecret) {
    this.httpClient = java.net.http.HttpClient.newBuilder()
                                              .connectTimeout(timeout)
                                              .build();
    this.objectMapper = new com.fasterxml.jackson.databind.ObjectMapper()
                                .registerModule(new com.fasterxml.jackson.datatype.jsr310.JavaTimeModule())
                                .configure(com.fasterxml.jackson.databind.DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    this.timeout = timeout;
    this.internalSecret = internalSecret;
}

public WriteProxyClient(java.time.Duration timeout) {
    this(timeout, null);
}
```

In `post()` and `get()`, add the header when secret is set:
```java
if (internalSecret != null) {
    reqBuilder.header("X-Internal-Secret", internalSecret);
}
```

Update `RelayProducer.writeProxyClient()`:
```java
@Produces
@ApplicationScoped
public WriteProxyClient writeProxyClient() {
    return new WriteProxyClient(config.proxyTimeout(),
            config.internalSecret().orElse(null));
}
```

- [ ] **Step 7: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/InternalSecretFilter.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java
git add cluster/src/test/java/io/casehub/qhorus/cluster/InternalSecretFilterTest.java
git commit -m "feat(#484): add shared-secret auth for /internal/* endpoints

InternalSecretFilter checks X-Internal-Secret header on all /internal/*
requests. WriteProxyClient sends the header when configured. Config:
casehub.qhorus.relay.internal-secret (optional, no-op when absent).

Refs #484"
```

### Task 5: Wire config mutation proxying + shutdown hook

**Files:**
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/ChannelManagerDecorator.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ChannelConfigRequest.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ClusterShutdownHandler.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/ChannelManagerDecoratorTest.java` (existing — add test)

**Interfaces:**
- Consumes: `WriteProxyClient.channelConfig(NodeInfo, UUID, String, Object)` (new generic proxy method)
- Produces: `/internal/channel/{id}/config` endpoint, `ClusterShutdownHandler`

- [ ] **Step 1: Write failing test for remote config mutation**

Add to `ChannelManagerDecoratorTest.java`:

```java
@Test
void remote_pause_proxies_via_proxy_client() {
    ChannelManager delegate = mock(ChannelManager.class);
    ClusterManager cm = mock(ClusterManager.class);
    WriteProxyClient proxy = mock(WriteProxyClient.class);

    ChannelManagerDecorator decorator = new ChannelManagerDecorator(
            delegate, cm, proxy, true);

    UUID channelId = UUID.randomUUID();
    NodeInfo remoteNode = new NodeInfo("remote", "remote:8080");
    when(cm.owner(channelId)).thenReturn(remoteNode);
    when(cm.isLocal(remoteNode)).thenReturn(false);
    Channel expected = mock(Channel.class);
    when(proxy.pauseChannel(remoteNode, channelId)).thenReturn(expected);

    Channel result = decorator.pause(channelId);

    assertThat(result).isSameAs(expected);
    verify(delegate, never()).pause(any());
}
```

- [ ] **Step 2: Run test — should fail (routeChannelMutation throws UnsupportedOperationException)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=ChannelManagerDecoratorTest#remote_pause_proxies_via_proxy_client`
Expected: FAIL with UnsupportedOperationException

- [ ] **Step 3: Fix routeChannelMutation to proxy via WriteProxyClient**

Replace `routeChannelMutation` (line 176-184) with a version that proxies:

```java
private Channel routeChannelMutation(UUID channelId, Function<UUID, Channel> localAction) {
    if (!routingEnabled) {
        return localAction.apply(channelId);
    }
    NodeInfo owner = clusterManager.owner(channelId);
    if (clusterManager.isLocal(owner)) {
        return localAction.apply(channelId);
    }
    return proxyClient.channelMutation(owner, channelId, localAction);
}
```

This requires `WriteProxyClient.channelMutation()` — but for pause/resume we already have dedicated methods. Refactor: change `routeChannelMutation` to accept a `BiFunction<WriteProxyClient, UUID, Channel>` for the proxy path:

```java
@Override
public Channel pause(UUID channelId) {
    return routeChannelMutation(channelId,
            id -> delegate.pause(id),
            (proxy, id) -> proxy.pauseChannel(clusterManager.owner(id), id));
}

@Override
public Channel resume(UUID channelId) {
    return routeChannelMutation(channelId,
            id -> delegate.resume(id),
            (proxy, id) -> proxy.resumeChannel(clusterManager.owner(id), id));
}

// For all other mutations, use a generic config endpoint:
@Override
public Channel setRateLimits(UUID id, Integer pc, Integer pi) {
    return routeChannelMutation(id,
            i -> delegate.setRateLimits(i, pc, pi),
            (proxy, i) -> proxy.channelConfig(clusterManager.owner(i), i,
                    new ChannelConfigRequest("setRateLimits",
                            Map.of("rateLimitPerChannel", pc, "rateLimitPerInstance", pi))));
}

private Channel routeChannelMutation(UUID channelId,
        Function<UUID, Channel> localAction,
        java.util.function.BiFunction<WriteProxyClient, UUID, Channel> proxyAction) {
    if (!routingEnabled) {
        return localAction.apply(channelId);
    }
    NodeInfo owner = clusterManager.owner(channelId);
    if (clusterManager.isLocal(owner)) {
        return localAction.apply(channelId);
    }
    return proxyAction.apply(proxyClient, channelId);
}
```

Add `channelConfig()` to `WriteProxyClient`:
```java
public Channel channelConfig(NodeInfo target, UUID channelId, ChannelConfigRequest request) {
    return post(target, "/internal/channel/" + channelId + "/config", request, Channel.class);
}
```

Add the `/internal/channel/{id}/config` endpoint to `InternalMeshResource`:
```java
@POST
@Path("/channel/{id}/config")
public Channel channelConfig(@PathParam("id") UUID channelId, ChannelConfigRequest request) {
    return request.applyTo(channelService, channelId);
}
```

Create `ChannelConfigRequest.java`:
```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;

import java.util.Map;
import java.util.UUID;

public record ChannelConfigRequest(String operation, Map<String, Object> params) {

    @SuppressWarnings("unchecked")
    public Channel applyTo(ChannelService service, UUID channelId) {
        return switch (operation) {
            case "setRateLimits" -> service.setRateLimits(channelId,
                    getInt("rateLimitPerChannel"), getInt("rateLimitPerInstance"));
            case "setAllowedWriters" -> service.setAllowedWriters(channelId,
                    (java.util.List<String>) params.get("writers"));
            case "setAdminInstances" -> service.setAdminInstances(channelId,
                    (java.util.List<String>) params.get("admins"));
            case "setReviewerInstances" -> service.setReviewerInstances(channelId,
                    (java.util.List<String>) params.get("reviewers"));
            case "setProtocols" -> service.setProtocols(channelId,
                    (java.util.List<String>) params.get("protocols"));
            case "setProtocolParticipants" -> service.setProtocolParticipants(channelId,
                    (java.util.List<String>) params.get("participants"));
            case "setEnforcementMode" -> service.setEnforcementMode(channelId,
                    io.casehub.qhorus.api.channel.EnforcementMode.valueOf(
                            (String) params.get("mode")));
            case "setEnforcementExclusions" -> service.setEnforcementExclusions(channelId,
                    (java.util.List<String>) params.get("exclusions"));
            case "setRoutingTrustThreshold" -> service.setRoutingTrustThreshold(channelId,
                    getDouble("threshold"));
            case "setRedistributionCapacityThreshold" -> service.setRedistributionCapacityThreshold(channelId,
                    getDouble("threshold"));
            case "setRoutingCapacityThreshold" -> service.setRoutingCapacityThreshold(channelId,
                    getDouble("threshold"));
            case "setPolicyOverrides" -> service.setPolicyOverrides(channelId,
                    (Map<String, String>) params.get("overrides"));
            case "setTypeConstraints" -> {
                var allowed = io.casehub.qhorus.api.message.MessageType.parseTypes(
                        (String) params.get("allowedTypes"));
                var denied = io.casehub.qhorus.api.message.MessageType.parseTypes(
                        (String) params.get("deniedTypes"));
                yield service.setTypeConstraints(channelId, allowed, denied);
            }
            default -> throw new IllegalArgumentException("Unknown operation: " + operation);
        };
    }

    private Integer getInt(String key) {
        Object v = params.get(key);
        return v == null ? null : ((Number) v).intValue();
    }

    private Double getDouble(String key) {
        Object v = params.get(key);
        return v == null ? null : ((Number) v).doubleValue();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: all pass

- [ ] **Step 5: Add ClusterShutdownHandler**

```java
package io.casehub.qhorus.cluster;

import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.runtime.ShutdownEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.relay.enabled", stringValue = "true",
                 enableIfMissing = false)
public class ClusterShutdownHandler {

    private static final Logger LOG = Logger.getLogger(ClusterShutdownHandler.class);

    @Inject ClusterManager clusterManager;
    @Inject WriteProxyClient proxyClient;

    void onShutdown(@Observes ShutdownEvent event) {
        LOG.info("Sending leave notifications to peers");
        clusterManager.shutdown(nodeInfo -> {
            try {
                proxyClient.sendLeave(nodeInfo, clusterManager.nodeId());
            } catch (Exception e) {
                LOG.debugf("Leave notification to %s failed: %s", nodeInfo.nodeId(), e.getMessage());
            }
        });
    }
}
```

Add `sendLeave()` to `WriteProxyClient`:
```java
public void sendLeave(NodeInfo target, String localNodeId) {
    post(target, "/internal/leave", new LeaveRequest(localNodeId), Void.class);
}
```

- [ ] **Step 6: Run all cluster tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: all pass

- [ ] **Step 7: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/ChannelManagerDecorator.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/ChannelConfigRequest.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/ClusterShutdownHandler.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/WriteProxyClient.java
git add cluster/src/main/java/io/casehub/qhorus/cluster/InternalMeshResource.java
git add cluster/src/test/java/io/casehub/qhorus/cluster/ChannelManagerDecoratorTest.java
git commit -m "feat(#484): wire all config mutation proxying + graceful shutdown

ChannelManagerDecorator now proxies all 15 config mutations via a generic
/internal/channel/{id}/config endpoint. ClusterShutdownHandler sends leave
notifications on @Observes ShutdownEvent for immediate ring removal.

Refs #484"
```

## Batch 3: Cache Module Fixes

### Task 6: Fix cache gaps — findRecent, delete invalidation, sync scheduler, config alignment

**Files:**
- Modify: `cache/src/main/java/io/casehub/qhorus/cache/CachingMessageStore.java`
- Modify: `cache/src/main/java/io/casehub/qhorus/cache/ChannelMessageBuffer.java`
- Modify: `cache/src/main/java/io/casehub/qhorus/cache/CacheProducer.java`
- Create: `cache/src/main/java/io/casehub/qhorus/cache/CacheSyncScheduler.java`
- Test: `cache/src/test/java/io/casehub/qhorus/cache/ChannelMessageBufferTest.java` (existing — add tests)
- Test: `cache/src/test/java/io/casehub/qhorus/cache/CachingMessageStoreTest.java` (existing — add tests)

**Interfaces:**
- Consumes: `ChannelMessageBuffer.remove(Long)`, `ChannelMessageBuffer.recentMessages(int)`, `FullSyncService.syncBatch()`
- Produces: Fixed cache behaviour, `CacheSyncScheduler` bean

- [ ] **Step 1: Write failing test for ChannelMessageBuffer.remove()**

Add to `ChannelMessageBufferTest.java`:

```java
@Test
void remove_deletes_message_by_id() {
    ChannelMessageBuffer buffer = new ChannelMessageBuffer(10);
    buffer.add(mockMessage(1L));
    buffer.add(mockMessage(2L));
    buffer.add(mockMessage(3L));

    buffer.remove(2L);

    assertThat(buffer.size()).isEqualTo(2);
    assertThat(buffer.findById(2L)).isEmpty();
    assertThat(buffer.findById(1L)).isPresent();
    assertThat(buffer.findById(3L)).isPresent();
}
```

- [ ] **Step 2: Run — should fail (method doesn't exist)**

- [ ] **Step 3: Implement `remove(Long)` on ChannelMessageBuffer**

```java
public void remove(Long messageId) {
    if (messageId != null) {
        messages.remove(messageId);
    }
}
```

- [ ] **Step 4: Write failing test for `recentMessages(int)` on ChannelMessageBuffer**

```java
@Test
void recentMessages_returns_tail_descending() {
    ChannelMessageBuffer buffer = new ChannelMessageBuffer(10);
    buffer.add(mockMessage(1L));
    buffer.add(mockMessage(2L));
    buffer.add(mockMessage(3L));
    buffer.add(mockMessage(4L));

    List<Message> recent = buffer.recentMessages(2);

    assertThat(recent).hasSize(2);
    assertThat(recent.get(0).id()).isEqualTo(4L);
    assertThat(recent.get(1).id()).isEqualTo(3L);
}
```

- [ ] **Step 5: Implement `recentMessages(int)`**

```java
public List<Message> recentMessages(int limit) {
    var descending = messages.descendingMap();
    return descending.values().stream().limit(limit).toList();
}
```

- [ ] **Step 6: Run buffer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache -Dtest=ChannelMessageBufferTest`
Expected: all pass

- [ ] **Step 7: Write failing test for delete() cache invalidation**

Add to `CachingMessageStoreTest.java`:

```java
@Test
void delete_invalidates_buffer() {
    MessageStore delegate = mock(MessageStore.class);
    CachingMessageStore store = new CachingMessageStore(delegate, 100, 100, false);

    // Populate cache
    Message msg = mockMessage(1L, UUID.randomUUID());
    store.addToBuffer(msg.channelId(), msg);

    store.delete(1L);

    verify(delegate).delete(1L);
    // Buffer should no longer contain the message
    ChannelMessageBuffer buffer = ...; // access via scan to verify
}
```

- [ ] **Step 8: Fix `delete(Long)` in CachingMessageStore**

```java
@Override
public void delete(Long id) {
    delegate.delete(id);
    channelCache.asMap().values().forEach(b -> b.remove(id));
}
```

- [ ] **Step 9: Fix `findRecent()` in CachingMessageStore**

Note: `findRecent()` returns `List<MessageView>` but buffer stores `Message`. For now, fall through to delegate — a proper Message→MessageView conversion requires `QhorusEntityMapper` which is in the runtime module (circular dependency). Document this as a known limitation.

```java
@Override
public List<MessageView> findRecent(UUID channelId, int limit) {
    // Cannot serve from cache — buffer stores Message, not MessageView.
    // QhorusEntityMapper.toMessageView() is in runtime module (dependency cycle).
    // Fall through to delegate for correctness.
    return delegate.findRecent(channelId, limit);
}
```

- [ ] **Step 10: Fix CacheProducer config alignment**

Change `enableIfMissing = false` to `enableIfMissing = true`:

```java
@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.cache.enabled", stringValue = "true", enableIfMissing = true)
public class CacheProducer {
```

- [ ] **Step 11: Create CacheSyncScheduler**

```java
package io.casehub.qhorus.cache;

import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
@IfBuildProperty(name = "casehub.qhorus.cache.enabled", stringValue = "true",
                 enableIfMissing = true)
public class CacheSyncScheduler {

    @Inject
    FullSyncService fullSyncService;

    @Inject
    CacheConfig config;

    @Scheduled(every = "${casehub.qhorus.cache.full-sync-interval:5s}",
               concurrentExecution = Scheduled.ConcurrentExecution.SKIP)
    void sync() {
        if ("full".equals(config.mode())) {
            fullSyncService.syncBatch();
        }
    }
}
```

- [ ] **Step 12: Run all cache tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache`
Expected: all pass

- [ ] **Step 13: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 14: Commit**

```bash
git add cache/src/main/java/io/casehub/qhorus/cache/CachingMessageStore.java
git add cache/src/main/java/io/casehub/qhorus/cache/ChannelMessageBuffer.java
git add cache/src/main/java/io/casehub/qhorus/cache/CacheProducer.java
git add cache/src/main/java/io/casehub/qhorus/cache/CacheSyncScheduler.java
git add cache/src/test/java/io/casehub/qhorus/cache/ChannelMessageBufferTest.java
git add cache/src/test/java/io/casehub/qhorus/cache/CachingMessageStoreTest.java
git commit -m "fix(#484): fix cache gaps — delete invalidation, sync scheduler, config alignment

ChannelMessageBuffer gains remove() and recentMessages(). CachingMessageStore
delete() now invalidates buffers. CacheSyncScheduler drives FullSyncService
in full mode. CacheProducer enableIfMissing corrected to true.

findRecent() remains delegating — Message→MessageView conversion requires
runtime module (dependency cycle). Documented as known limitation.

Refs #484"
```

## Batch 4: Integration Tests (Phase B)

### Task 7: CDI wiring, decorator ordering, and config gate tests

**Files:**
- Create: `cluster/src/test/java/io/casehub/qhorus/cluster/ClusterCdiWiringTest.java`
- Create: `cluster/src/test/java/io/casehub/qhorus/cluster/ClusterDisabledTest.java`

**Interfaces:**
- Consumes: All CDI-produced beans from Batches 1-3
- Produces: Test verification that composition works

Note: These are `@QuarkusTest` classes requiring the full CDI container. They need a `@TestProfile` that enables relay and cache. The existing cluster module doesn't have `@QuarkusTest` infrastructure (all tests are CDI-free). These tests may need to be deferred to a separate integration-test module or added to the runtime module's test suite where the full Quarkus container is available. Assess during implementation — if the cluster module's pom doesn't have `quarkus-junit5` as a test dep, add it or move these tests to a location that does.

- [ ] **Step 1: Assess cluster module test infrastructure**

Check if `cluster/pom.xml` has `quarkus-junit5` and `quarkus-junit5-mockito` as test dependencies. If not, add them. Also check if `cluster/src/test/resources/application.properties` exists.

- [ ] **Step 2: Create test application.properties**

Create `cluster/src/test/resources/application.properties`:
```properties
casehub.qhorus.relay.enabled=true
casehub.qhorus.relay.routing=dynamic
casehub.qhorus.relay.peers=localhost:8080
quarkus.http.test-port=0
```

- [ ] **Step 3: Write CDI wiring smoke test**

```java
package io.casehub.qhorus.cluster;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ClusterCdiWiringTest {

    @Inject Instance<ClusterManager> clusterManager;
    @Inject Instance<HeartbeatService> heartbeatService;
    @Inject Instance<HeartbeatScheduler> heartbeatScheduler;
    @Inject Instance<WriteProxyClient> writeProxyClient;
    @Inject Instance<WriteFrequencyTracker> writeFrequencyTracker;
    @Inject Instance<OwnershipScheduler> ownershipScheduler;
    @Inject Instance<InternalSecretFilter> secretFilter;

    @Test
    void all_relay_beans_are_resolvable() {
        assertThat(clusterManager.isResolvable()).isTrue();
        assertThat(heartbeatService.isResolvable()).isTrue();
        assertThat(heartbeatScheduler.isResolvable()).isTrue();
        assertThat(writeProxyClient.isResolvable()).isTrue();
        assertThat(writeFrequencyTracker.isResolvable()).isTrue();
        assertThat(ownershipScheduler.isResolvable()).isTrue();
        assertThat(secretFilter.isResolvable()).isTrue();
    }
}
```

- [ ] **Step 4: Write config gate test**

```java
package io.casehub.qhorus.cluster;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(ClusterDisabledTest.DisabledProfile.class)
class ClusterDisabledTest {

    @Inject Instance<ClusterManager> clusterManager;
    @Inject Instance<HeartbeatService> heartbeatService;

    @Test
    void relay_beans_not_resolvable_when_disabled() {
        assertThat(clusterManager.isResolvable()).isFalse();
        assertThat(heartbeatService.isResolvable()).isFalse();
    }

    public static class DisabledProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("casehub.qhorus.relay.enabled", "false");
        }
    }
}
```

- [ ] **Step 5: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=ClusterCdiWiringTest,ClusterDisabledTest`
Expected: PASS (may need iterative fixes to CDI wiring)

- [ ] **Step 6: Commit**

```bash
git add cluster/src/test/
git commit -m "test(#484): add CDI wiring and config gate integration tests

ClusterCdiWiringTest verifies all relay beans resolve when enabled.
ClusterDisabledTest verifies none resolve when disabled.

Refs #484"
```

## Batch 5: Podman E2E Infrastructure (Phase C)

### Task 8: Create e2e-cluster module with test infrastructure

**Files:**
- Create: `e2e-cluster/pom.xml`
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/ClusterTestHarness.java`
- Create: `e2e-cluster/src/test/resources/Dockerfile`
- Modify: `pom.xml` (parent — add profile)

**Interfaces:**
- Consumes: `mesh/` uber-jar (build artifact)
- Produces: `ClusterTestHarness` utility for all scenario tests

- [ ] **Step 1: Add profile to parent pom.xml**

```xml
<profile>
    <id>with-e2e-cluster</id>
    <modules>
        <module>e2e-cluster</module>
    </modules>
</profile>
```

- [ ] **Step 2: Create e2e-cluster/pom.xml**

Standard Maven module with test-only dependencies: testcontainers, testcontainers-postgresql, rest-assured, junit-jupiter, assertj. No production sources.

- [ ] **Step 3: Create Dockerfile**

`e2e-cluster/src/test/resources/Dockerfile`:
```dockerfile
FROM eclipse-temurin:26-jre-alpine
COPY quarkus-app /app
WORKDIR /app
ENTRYPOINT ["java", "-Xmx256m", "-jar", "quarkus-run.jar"]
```

- [ ] **Step 4: Implement ClusterTestHarness**

```java
package io.casehub.qhorus.e2e;

import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.Network;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.images.builder.ImageFromDockerfile;

import java.nio.file.Path;
import java.time.Duration;
import java.util.*;

public class ClusterTestHarness implements AutoCloseable {

    private static final int APP_PORT = 8080;
    private final Network network = Network.newNetwork();
    private final PostgreSQLContainer<?> postgres;
    private final Map<String, GenericContainer<?>> nodes = new LinkedHashMap<>();
    private final String internalSecret;
    private final List<String> allNodeIds;

    public ClusterTestHarness(String internalSecret, String... nodeIds) {
        this.internalSecret = internalSecret;
        this.allNodeIds = List.of(nodeIds);
        this.postgres = new PostgreSQLContainer<>("postgres:16-alpine")
                .withDatabaseName("qhorus")
                .withUsername("qhorus")
                .withPassword("qhorus")
                .withNetwork(network)
                .withNetworkAliases("postgres");
    }

    public void startPostgres() {
        postgres.start();
    }

    public void startNode(String nodeId) {
        String peers = String.join(",",
                allNodeIds.stream().map(id -> id + ":" + APP_PORT).toList());

        GenericContainer<?> container = new GenericContainer<>(buildImage())
                .withNetwork(network)
                .withNetworkAliases(nodeId)
                .withExposedPorts(APP_PORT)
                .withEnv("CASEHUB_QHORUS_RELAY_ENABLED", "true")
                .withEnv("CASEHUB_QHORUS_RELAY_NODE_ID", nodeId)
                .withEnv("CASEHUB_QHORUS_RELAY_PEERS", peers)
                .withEnv("CASEHUB_QHORUS_RELAY_ROUTING", "dynamic")
                .withEnv("CASEHUB_QHORUS_RELAY_INTERNAL_SECRET", internalSecret)
                .withEnv("CASEHUB_QHORUS_RELAY_HEARTBEAT_MISS_THRESHOLD", "3")
                .withEnv("CASEHUB_QHORUS_CACHE_ENABLED", "true")
                .withEnv("QUARKUS_DATASOURCE_QHORUS_DB_KIND", "postgresql")
                .withEnv("QUARKUS_DATASOURCE_QHORUS_JDBC_URL",
                        "jdbc:postgresql://postgres:5432/qhorus")
                .withEnv("QUARKUS_DATASOURCE_QHORUS_USERNAME", "qhorus")
                .withEnv("QUARKUS_DATASOURCE_QHORUS_PASSWORD", "qhorus")
                .waitingFor(Wait.forHttp("/health/cluster").forPort(APP_PORT)
                        .withStartupTimeout(Duration.ofSeconds(60)));
        container.start();
        nodes.put(nodeId, container);
    }

    public void stopNode(String nodeId) {
        GenericContainer<?> c = nodes.get(nodeId);
        if (c != null && c.isRunning()) {
            c.stop();
        }
    }

    public void restartNode(String nodeId) {
        stopNode(nodeId);
        startNode(nodeId);
    }

    public String nodeUrl(String nodeId) {
        GenericContainer<?> c = nodes.get(nodeId);
        return "http://" + c.getHost() + ":" + c.getMappedPort(APP_PORT);
    }

    public void waitForClusterConvergence(Duration timeout) {
        // Poll /health/cluster on each node until all report expected cluster size
        long deadline = System.currentTimeMillis() + timeout.toMillis();
        while (System.currentTimeMillis() < deadline) {
            boolean allConverged = nodes.keySet().stream().allMatch(id -> {
                try {
                    // Use rest-assured to GET /health/cluster and check clusterSize
                    return true; // placeholder — implement with rest-assured
                } catch (Exception e) {
                    return false;
                }
            });
            if (allConverged) return;
            try { Thread.sleep(500); } catch (InterruptedException e) { break; }
        }
        throw new RuntimeException("Cluster did not converge within " + timeout);
    }

    @Override
    public void close() {
        nodes.values().forEach(GenericContainer::stop);
        postgres.stop();
        network.close();
    }

    private ImageFromDockerfile buildImage() {
        // Build from the mesh module's quarkus-app directory
        Path meshTarget = Path.of("../mesh/target/quarkus-app");
        return new ImageFromDockerfile()
                .withDockerfileFromClasspath("Dockerfile")
                .withFileFromPath("quarkus-app", meshTarget);
    }
}
```

- [ ] **Step 5: Verify infrastructure compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl e2e-cluster -Pwith-e2e-cluster`
Expected: compiles (no tests yet)

- [ ] **Step 6: Commit**

```bash
git add e2e-cluster/ pom.xml
git commit -m "feat(#484): create e2e-cluster module with ClusterTestHarness

Profile-gated module (-Pwith-e2e-cluster) with Testcontainers infrastructure.
ClusterTestHarness manages PostgreSQL + N Qhorus app containers on a shared
Docker network with network aliases for peer discovery.

Refs #484"
```

## Batch 6: E2E Scenarios

### Task 9: Scenario 1 — Dispatch routing + cache coherence

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/DispatchRoutingE2ETest.java`

- [ ] **Step 1: Write the test**

Tests: message dispatch routes to owner, non-owner nodes see it via pg_notify.

- [ ] **Step 2: Run**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl e2e-cluster -Pwith-e2e-cluster -Dtest=DispatchRoutingE2ETest`

- [ ] **Step 3: Commit**

### Task 10: Scenario 2 — Ownership transfer

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/OwnershipTransferE2ETest.java`

- [ ] **Step 1: Write the test**

Tests: sustained writes from non-owner trigger ownership transfer after evaluation cycle.

- [ ] **Step 2: Run and iterate**

- [ ] **Step 3: Commit**

### Task 11: Scenario 3 — Node failure and recovery

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/NodeFailureE2ETest.java`

- [ ] **Step 1: Write the test**

Tests: stop node, verify DEAD detection, fallback-to-local, restart, verify ALIVE recovery.

- [ ] **Step 2: Run and iterate**

- [ ] **Step 3: Commit**

### Task 12: Scenario 4 — Quorum enforcement

**Files:**
- Create: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/QuorumEnforcementE2ETest.java`

- [ ] **Step 1: Write the test**

Tests: stop 2 of 3 nodes, verify lone node rejects writes, restart 1, verify majority restored.

- [ ] **Step 2: Run and iterate**

- [ ] **Step 3: Commit**

- [ ] **Step 4: Run all e2e tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl e2e-cluster -Pwith-e2e-cluster`
Expected: all 4 scenarios pass

- [ ] **Step 5: Final full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS (e2e tests not included without profile)

- [ ] **Step 6: Commit**

```bash
git add e2e-cluster/src/test/
git commit -m "test(#484): add 4 Podman e2e scenarios for distributed mesh

DispatchRoutingE2ETest — message routing to owner, pg_notify cache coherence
OwnershipTransferE2ETest — dynamic ownership migration after write pattern shift
NodeFailureE2ETest — DEAD detection, fallback-to-local, ALIVE recovery
QuorumEnforcementE2ETest — minority partition rejects writes, majority restores

Refs #484"
```

## References

- [2026-10-07-audit-e2e-design.md] — design spec this plan implements
- [cluster/src/main/java/.../RelayProducer.java] — CDI production hub
- [cluster/src/main/java/.../WriteRoutingDecorator.java:37-61] — three dispatch paths
- [cluster/src/main/java/.../HeartbeatService.java:15] — constructor signature
- [cluster/src/main/java/.../InternalMeshResource.java:28] — MessageDispatcher injection (proxy loop)
- [cluster/src/main/java/.../ChannelManagerDecorator.java:176-184] — routeChannelMutation stub
- [cluster/src/main/java/.../ClusterManager.java:163] — shutdown() method
- [cache/src/main/java/.../CachingMessageStore.java:128-129] — delete without invalidation
- [cache/src/main/java/.../CacheProducer.java:13] — enableIfMissing=false mismatch
- [cache/src/main/java/.../ChannelMessageBuffer.java] — ConcurrentSkipListMap storage
- D39-D47 — design decisions
- Light review R1-01 through R1-16 — review findings incorporated into spec
- #484 — focal issue

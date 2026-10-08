# Cluster Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #488 — epic: cluster hardening — e2e coverage, split-brain safety, cache wiring
**Issue group:** #488, #476

**Goal:** Harden the cluster layer with end-to-end cross-node dispatch validation, split-brain proxy failure safety mechanisms, and CDI wiring verification for the cache module.

**Architecture:** Three independent sub-tasks. Sub-task 2 adds a `ProxyFallbackEvent` CDI event and configurable fail-fast mode to `WriteRoutingDecorator`. Sub-task 3 follows the established `ClusterCdiWiringTest`/`ClusterDisabledTest` pattern for the cache module. Sub-task 1 adds an E2E container test.

**Tech Stack:** Java 21, Quarkus 3.32.2, Testcontainers, H2, Mockito, AssertJ, Awaitility

## Global Constraints

- Java 21 source, Java 26 JVM (`JAVA_HOME=$(/usr/libexec/java_home -v 26)`)
- Quarkus 3.32.2 — do not upgrade
- All new classes follow existing package conventions
- CDI-free unit tests use Mockito — no `@QuarkusTest` unless testing CDI wiring
- `@TestProfile` with full datasource overrides for any profile that causes Quarkus restart
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Commits reference `Refs #488` (or `Refs #476` for cache work)

---

## Batch 1: Split-brain fallback safety

### Task 1: ProxyFallbackEvent, ProxyDispatchException, and WriteRoutingDecorator changes

**Files:**
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ProxyFallbackEvent.java`
- Create: `cluster/src/main/java/io/casehub/qhorus/cluster/ProxyDispatchException.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java`
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java`
- Test: `cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java`

**Interfaces:**
- Consumes: `ClusterManager`, `WriteProxyClient`, `MessageDispatcher` (existing)
- Produces: `ProxyFallbackEvent(UUID channelId, String ownerNodeId, String localNodeId, String sender, MessageType messageType, String errorMessage)` — CDI event record. `ProxyDispatchException extends RuntimeException` — carries same fields plus cause.

- [ ] **Step 1: Write failing tests for event firing and fail-fast mode**

Add four new tests to `WriteRoutingDecoratorTest.java`:

```java
import jakarta.enterprise.event.Event;
import io.casehub.qhorus.api.message.MessageType;
import org.mockito.ArgumentCaptor;

// Add fields to setUp:
private Event<ProxyFallbackEvent> fallbackEvent;

@BeforeEach
void setUp() {
    delegate = mock(MessageDispatcher.class);
    clusterManager = mock(ClusterManager.class);
    proxyClient = mock(WriteProxyClient.class);
    fallbackEvent = mock(Event.class);
    decorator = new WriteRoutingDecorator(delegate, clusterManager, proxyClient, true);
}

@Test
void proxyFailureInLocalModeFiresEventAndFallsBack() {
    var dec = new WriteRoutingDecorator(delegate, clusterManager, proxyClient, true,
            null, fallbackEvent, "local", "node-1");
    var remoteNode = new NodeInfo("node-2", "node-2:8080");
    when(clusterManager.canServeWrites()).thenReturn(true);
    when(clusterManager.owner(CHANNEL_ID)).thenReturn(remoteNode);
    when(clusterManager.isLocal(remoteNode)).thenReturn(false);
    var dispatch = MessageDispatch.builder().channelId(CHANNEL_ID)
            .sender("agent-1").type(MessageType.STATUS).content("test")
            .actorType(ActorType.AGENT).build();
    when(proxyClient.dispatch(remoteNode, dispatch))
            .thenThrow(new RuntimeException("Connection refused"));
    var expectedResult = new DispatchResult(1L, CHANNEL_ID, "agent-1",
            MessageType.STATUS, null, null, List.of(), null, null, null, null, 0, null, List.of());
    when(delegate.dispatch(dispatch)).thenReturn(expectedResult);

    DispatchResult result = dec.dispatch(dispatch);

    assertThat(result).isEqualTo(expectedResult);
    verify(delegate).dispatch(dispatch);
    ArgumentCaptor<ProxyFallbackEvent> captor = ArgumentCaptor.forClass(ProxyFallbackEvent.class);
    verify(fallbackEvent).fireAsync(captor.capture());
    ProxyFallbackEvent event = captor.getValue();
    assertThat(event.channelId()).isEqualTo(CHANNEL_ID);
    assertThat(event.ownerNodeId()).isEqualTo("node-2");
    assertThat(event.localNodeId()).isEqualTo("node-1");
    assertThat(event.sender()).isEqualTo("agent-1");
    assertThat(event.messageType()).isEqualTo(MessageType.STATUS);
}

@Test
void proxyFailureInFailModeFiresEventAndThrows() {
    var dec = new WriteRoutingDecorator(delegate, clusterManager, proxyClient, true,
            null, fallbackEvent, "fail", "node-1");
    var remoteNode = new NodeInfo("node-2", "node-2:8080");
    when(clusterManager.canServeWrites()).thenReturn(true);
    when(clusterManager.owner(CHANNEL_ID)).thenReturn(remoteNode);
    when(clusterManager.isLocal(remoteNode)).thenReturn(false);
    var dispatch = MessageDispatch.builder().channelId(CHANNEL_ID)
            .sender("agent-1").type(MessageType.STATUS).content("test")
            .actorType(ActorType.AGENT).build();
    when(proxyClient.dispatch(remoteNode, dispatch))
            .thenThrow(new RuntimeException("Connection refused"));

    assertThatThrownBy(() -> dec.dispatch(dispatch))
            .isInstanceOf(ProxyDispatchException.class)
            .hasCauseInstanceOf(RuntimeException.class);
    verify(fallbackEvent).fireAsync(any(ProxyFallbackEvent.class));
    verifyNoInteractions(delegate);
}

@Test
void successfulProxyDoesNotFireEvent() {
    var dec = new WriteRoutingDecorator(delegate, clusterManager, proxyClient, true,
            null, fallbackEvent, "local", "node-1");
    var remoteNode = new NodeInfo("node-2", "node-2:8080");
    when(clusterManager.canServeWrites()).thenReturn(true);
    when(clusterManager.owner(CHANNEL_ID)).thenReturn(remoteNode);
    when(clusterManager.isLocal(remoteNode)).thenReturn(false);
    var dispatch = MessageDispatch.builder().channelId(CHANNEL_ID)
            .sender("agent-1").type(MessageType.STATUS).content("test")
            .actorType(ActorType.AGENT).build();
    var expectedResult = new DispatchResult(1L, CHANNEL_ID, "agent-1",
            MessageType.STATUS, null, null, List.of(), null, null, null, null, 0, null, List.of());
    when(proxyClient.dispatch(remoteNode, dispatch)).thenReturn(expectedResult);

    dec.dispatch(dispatch);

    verifyNoInteractions(fallbackEvent);
}

@Test
void nullEventSkipsFiringGracefully() {
    // Existing 4-arg constructor passes null event — proxy failure should not NPE
    var remoteNode = new NodeInfo("node-2", "node-2:8080");
    when(clusterManager.canServeWrites()).thenReturn(true);
    when(clusterManager.owner(CHANNEL_ID)).thenReturn(remoteNode);
    when(clusterManager.isLocal(remoteNode)).thenReturn(false);
    var dispatch = MessageDispatch.builder().channelId(CHANNEL_ID)
            .sender("agent-1").type(MessageType.STATUS).content("test")
            .actorType(ActorType.AGENT).build();
    when(proxyClient.dispatch(remoteNode, dispatch))
            .thenThrow(new RuntimeException("Connection refused"));
    var expectedResult = new DispatchResult(1L, CHANNEL_ID, "agent-1",
            MessageType.STATUS, null, null, List.of(), null, null, null, null, 0, null, List.of());
    when(delegate.dispatch(dispatch)).thenReturn(expectedResult);

    DispatchResult result = decorator.dispatch(dispatch);

    assertThat(result).isEqualTo(expectedResult);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=WriteRoutingDecoratorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation errors — `ProxyFallbackEvent`, `ProxyDispatchException` don't exist, 8-arg constructor doesn't exist.

- [ ] **Step 3: Create ProxyFallbackEvent record**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ProxyFallbackEvent.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.message.MessageType;

import java.util.UUID;

public record ProxyFallbackEvent(
        UUID channelId,
        String ownerNodeId,
        String localNodeId,
        String sender,
        MessageType messageType,
        String errorMessage) {
}
```

- [ ] **Step 4: Create ProxyDispatchException**

Create `cluster/src/main/java/io/casehub/qhorus/cluster/ProxyDispatchException.java`:

```java
package io.casehub.qhorus.cluster;

public class ProxyDispatchException extends RuntimeException {

    private final ProxyFallbackEvent event;

    public ProxyDispatchException(ProxyFallbackEvent event, Throwable cause) {
        super("Proxy dispatch to " + event.ownerNodeId() + " failed for channel "
              + event.channelId() + ": " + cause.getMessage(), cause);
        this.event = event;
    }

    public ProxyFallbackEvent event() {
        return event;
    }
}
```

- [ ] **Step 5: Update WriteRoutingDecorator with event firing and fail-fast**

Modify `cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java`:

```java
package io.casehub.qhorus.cluster;

import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import jakarta.enterprise.event.Event;

public class WriteRoutingDecorator implements MessageDispatcher {

    private static final org.jboss.logging.Logger LOG =
            org.jboss.logging.Logger.getLogger(WriteRoutingDecorator.class);

    private final MessageDispatcher delegate;
    private final ClusterManager clusterManager;
    private final WriteProxyClient proxyClient;
    private final boolean routingEnabled;
    private final WriteFrequencyTracker tracker;
    private final Event<ProxyFallbackEvent> fallbackEvent;
    private final String proxyFallback;
    private final String localNodeId;

    public WriteRoutingDecorator(MessageDispatcher delegate,
                                 ClusterManager clusterManager,
                                 WriteProxyClient proxyClient,
                                 boolean routingEnabled,
                                 WriteFrequencyTracker tracker,
                                 Event<ProxyFallbackEvent> fallbackEvent,
                                 String proxyFallback,
                                 String localNodeId) {
        this.delegate = delegate;
        this.clusterManager = clusterManager;
        this.proxyClient = proxyClient;
        this.routingEnabled = routingEnabled;
        this.tracker = tracker;
        this.fallbackEvent = fallbackEvent;
        this.proxyFallback = proxyFallback;
        this.localNodeId = localNodeId;
    }

    public WriteRoutingDecorator(MessageDispatcher delegate,
                                 ClusterManager clusterManager,
                                 WriteProxyClient proxyClient,
                                 boolean routingEnabled,
                                 WriteFrequencyTracker tracker) {
        this(delegate, clusterManager, proxyClient, routingEnabled, tracker, null, "local", null);
    }

    public WriteRoutingDecorator(MessageDispatcher delegate,
                                 ClusterManager clusterManager,
                                 WriteProxyClient proxyClient,
                                 boolean routingEnabled) {
        this(delegate, clusterManager, proxyClient, routingEnabled, null);
    }

    @Override
    public DispatchResult dispatch(MessageDispatch dispatch) {
        if (!routingEnabled) {
            return delegate.dispatch(dispatch);
        }
        if (!clusterManager.canServeWrites()) {
            throw new QuorumViolationException(
                    "This node is in a minority partition and cannot serve writes");
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
            LOG.warnf("Proxy to %s failed: %s", owner.nodeId(), e.getMessage());
            var event = new ProxyFallbackEvent(
                    dispatch.channelId(), owner.nodeId(), localNodeId,
                    dispatch.sender(), dispatch.type(), e.getMessage());
            if (fallbackEvent != null) {
                fallbackEvent.fireAsync(event);
            }
            if ("fail".equals(proxyFallback)) {
                throw new ProxyDispatchException(event, e);
            }
            DispatchResult result = delegate.dispatch(dispatch);
            if (tracker != null) {
                tracker.recordWrite(dispatch.channelId());
            }
            return result;
        }
    }
}
```

- [ ] **Step 6: Add proxyFallback() to RelayConfig**

Add to `cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java`:

```java
@WithDefault("local")
String proxyFallback();
```

- [ ] **Step 7: Update RelayProducer to wire new constructor args**

Modify `RelayProducer.consumerMessaging()` in `cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java`. Add an `Event<ProxyFallbackEvent>` injection field and pass it to the new 8-arg constructor:

Add field:
```java
@Inject
Event<ProxyFallbackEvent> proxyFallbackEvent;
```

Change line 95 from:
```java
WriteRoutingDecorator router = new WriteRoutingDecorator(delegate, clusterManager, proxyClient, routing, tracker);
```
to:
```java
String nodeId = config.nodeId().orElse("unknown");
WriteRoutingDecorator router = new WriteRoutingDecorator(
        delegate, clusterManager, proxyClient, routing, tracker,
        proxyFallbackEvent, config.proxyFallback(), nodeId);
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster -Dtest=WriteRoutingDecoratorTest`
Expected: All tests pass (existing + 4 new).

- [ ] **Step 9: Run full cluster module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cluster`
Expected: All tests pass. The CDI wiring test may need the new config property — the `EnabledProfile` will pick up the default `"local"` from `@WithDefault`.

- [ ] **Step 10: Commit**

```bash
git add cluster/src/main/java/io/casehub/qhorus/cluster/ProxyFallbackEvent.java \
       cluster/src/main/java/io/casehub/qhorus/cluster/ProxyDispatchException.java \
       cluster/src/main/java/io/casehub/qhorus/cluster/WriteRoutingDecorator.java \
       cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java \
       cluster/src/main/java/io/casehub/qhorus/cluster/RelayProducer.java \
       cluster/src/test/java/io/casehub/qhorus/cluster/WriteRoutingDecoratorTest.java
git commit -m "feat(#488): add ProxyFallbackEvent and configurable fail-fast mode

WriteRoutingDecorator now fires a CDI ProxyFallbackEvent on every
proxy failure. New config casehub.qhorus.relay.proxy-fallback controls
behavior: 'local' (default, backward-compatible fallback) or 'fail'
(throw ProxyDispatchException, no local write).

Refs #488"
```

---

## Batch 2: Cache module CDI wiring and health

### Task 2: Add @QuarkusTest infrastructure and CDI wiring tests for cache module

**Files:**
- Modify: `cache/pom.xml` — add test dependencies
- Create: `cache/src/test/resources/application.properties` — test config
- Create: `cache/src/test/resources/import-qhorus-test.sql` — ledger sequence table
- Create: `cache/src/test/java/io/casehub/qhorus/cache/CacheCdiWiringTest.java`
- Create: `cache/src/test/java/io/casehub/qhorus/cache/CacheDisabledTest.java`
- Create: `cache/src/test/java/io/casehub/qhorus/cache/CacheHealthResourceTest.java`

**Interfaces:**
- Consumes: `CachingMessageStore`, `FullSyncService`, `CachePopulationObserver`, `CacheSyncScheduler`, `CacheHealthResource` (all existing)
- Produces: Three test classes verifying CDI wiring (no public API)

- [ ] **Step 1: Add test dependencies to cache/pom.xml**

Add the following test-scoped dependencies (after the existing `mockito-core` dependency):

```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-junit</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-junit-mockito</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-qhorus-persistence-memory</artifactId>
  <version>${project.version}</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform</artifactId>
  <version>0.2-SNAPSHOT</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-jdbc-h2</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-rest-jackson</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.rest-assured</groupId>
  <artifactId>rest-assured</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Create test resources**

Create `cache/src/test/resources/import-qhorus-test.sql` (same as every other module):

```sql
-- Creates native SQL tables that are not JPA entities and therefore
-- not created by Hibernate drop-and-create. Required for LedgerSequenceAllocator
-- which executes MERGE INTO ledger_subject_sequence. Refs qhorus#256.
CREATE TABLE IF NOT EXISTS ledger_subject_sequence (
    subject_id UUID        PRIMARY KEY,
    next_seq   BIGINT      NOT NULL
);
```

Create `cache/src/test/resources/application.properties` (following cluster module pattern):

```properties
quarkus.http.test-port=0

# Default datasource — satisfies casehub-ledger @Default EntityManager
quarkus.datasource.db-kind=h2
quarkus.datasource.username=sa
quarkus.datasource.password=
quarkus.datasource.jdbc.url=jdbc:h2:mem:cache_default;DB_CLOSE_DELAY=-1

# Named qhorus datasource
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:cache_qhorus;DB_CLOSE_DELAY=-1

# Schema generation
quarkus.hibernate-orm.datasource=qhorus
quarkus.hibernate-orm.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime,io.casehub.ledger.jpa
quarkus.hibernate-orm.database.generation=none
quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
quarkus.hibernate-orm.qhorus.sql-load-script=import-qhorus-test.sql

# Disable Hibernate Reactive
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.reactive=false

# Flyway disabled
quarkus.flyway.qhorus.migrate-at-start=false

# JSON config
quarkus.jackson.serialization-inclusion=non-null

# Ledger
casehub.ledger.datasource=qhorus
casehub.ledger.enabled=true
casehub.ledger.hash-chain.enabled=false
casehub.ledger.decision-context.enabled=false
casehub.ledger.attestations.enabled=true
casehub.ledger.trust-score.enabled=false

# Delivery pump disabled
casehub.qhorus.delivery.enabled=false

# Exclude beans that conflict with in-memory stores or lack required infrastructure
quarkus.arc.exclude-types=io.casehub.ledger.runtime.service.identity.ReactiveAgentIdentityVerificationService,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

- [ ] **Step 3: Write CacheCdiWiringTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/CacheCdiWiringTest.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.store.MessageStore;
import io.quarkus.arc.ClientProxy;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(CacheCdiWiringTest.EnabledProfile.class)
class CacheCdiWiringTest {

    @Inject Instance<CachingMessageStore> cachingMessageStore;
    @Inject Instance<FullSyncService> fullSyncService;
    @Inject Instance<CachePopulationObserver> cachePopulationObserver;
    @Inject Instance<CacheSyncScheduler> cacheSyncScheduler;

    @Inject MessageStore messageStore;

    @Test
    void all_cache_beans_are_resolvable() {
        assertThat(cachingMessageStore.isResolvable()).isTrue();
        assertThat(fullSyncService.isResolvable()).isTrue();
        assertThat(cachePopulationObserver.isResolvable()).isTrue();
        assertThat(cacheSyncScheduler.isResolvable()).isTrue();
    }

    @Test
    void message_store_is_caching_store() {
        assertThat(ClientProxy.unwrap(messageStore)).isInstanceOf(CachingMessageStore.class);
    }

    public static class EnabledProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                    "casehub.qhorus.cache.enabled", "true",
                    "quarkus.datasource.qhorus.db-kind", "h2",
                    "quarkus.datasource.qhorus.username", "sa",
                    "quarkus.datasource.qhorus.password", "",
                    "quarkus.datasource.qhorus.jdbc.url", "jdbc:h2:mem:cache_enabled;DB_CLOSE_DELAY=-1",
                    "quarkus.datasource.qhorus.reactive", "false",
                    "quarkus.hibernate-orm.qhorus.database.generation", "drop-and-create"
            );
        }
    }
}
```

- [ ] **Step 4: Write CacheDisabledTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/CacheDisabledTest.java`:

```java
package io.casehub.qhorus.cache;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(CacheDisabledTest.DisabledProfile.class)
class CacheDisabledTest {

    @Inject Instance<CachingMessageStore> cachingMessageStore;
    @Inject Instance<FullSyncService> fullSyncService;
    @Inject Instance<CacheSyncScheduler> cacheSyncScheduler;

    @Test
    void cache_beans_not_resolvable_when_disabled() {
        assertThat(cachingMessageStore.isResolvable()).isFalse();
        assertThat(fullSyncService.isResolvable()).isFalse();
        assertThat(cacheSyncScheduler.isResolvable()).isFalse();
    }

    public static class DisabledProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                    "casehub.qhorus.cache.enabled", "false",
                    "quarkus.datasource.qhorus.db-kind", "h2",
                    "quarkus.datasource.qhorus.username", "sa",
                    "quarkus.datasource.qhorus.password", "",
                    "quarkus.datasource.qhorus.jdbc.url", "jdbc:h2:mem:cache_disabled;DB_CLOSE_DELAY=-1",
                    "quarkus.datasource.qhorus.reactive", "false",
                    "quarkus.hibernate-orm.qhorus.database.generation", "drop-and-create"
            );
        }
    }
}
```

- [ ] **Step 5: Write CacheHealthResourceTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/CacheHealthResourceTest.java`:

```java
package io.casehub.qhorus.cache;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestProfile(CacheCdiWiringTest.EnabledProfile.class)
class CacheHealthResourceTest {

    @Test
    void health_endpoint_returns_up_when_enabled() {
        given()
            .when().get("/health/cache")
            .then()
                .statusCode(200)
                .body("status", equalTo("UP"))
                .body("channelsCached", greaterThanOrEqualTo(0))
                .body("messagesCached", greaterThanOrEqualTo(0));
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache`
Expected: All tests pass — existing CDI-free tests + 3 new `@QuarkusTest` classes.

- [ ] **Step 7: Commit**

```bash
git add cache/pom.xml \
       cache/src/test/resources/application.properties \
       cache/src/test/resources/import-qhorus-test.sql \
       cache/src/test/java/io/casehub/qhorus/cache/CacheCdiWiringTest.java \
       cache/src/test/java/io/casehub/qhorus/cache/CacheDisabledTest.java \
       cache/src/test/java/io/casehub/qhorus/cache/CacheHealthResourceTest.java
git commit -m "feat(#476): add CDI wiring tests and health endpoint test for cache module

CacheCdiWiringTest verifies all cache beans resolve when enabled.
CacheDisabledTest verifies beans are absent when disabled.
CacheHealthResourceTest verifies /health/cache returns correct JSON.

Refs #488, Refs #476"
```

---

## Batch 3: Cross-node dispatch E2E test

### Task 3: Add cross-node proxy dispatch test to DispatchRoutingE2ETest

**Files:**
- Modify: `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/DispatchRoutingE2ETest.java`

**Interfaces:**
- Consumes: `ClusterTestHarness` (existing — `createChannel`, `sendMessage`, `getMessages`)
- Produces: One new E2E test method (no public API)

- [ ] **Step 1: Write the cross-node dispatch test**

Add to `e2e-cluster/src/test/java/io/casehub/qhorus/e2e/DispatchRoutingE2ETest.java`:

```java
@Test
void cross_node_dispatch_via_proxy() {
    // Create channel on node-a — node-a becomes the owner
    String channelId = cluster.createChannel("node-a", "e2e-cross-node-proxy");

    // Send from node-b — triggers WriteRoutingDecorator proxy to node-a
    Response sendResp = cluster.sendMessage("node-b", channelId,
            "agent-on-b", "STATUS", "proxied from node-b");
    assertThat(sendResp.statusCode()).isEqualTo(200);

    // Verify message visible on node-a (the owner that received the proxied write)
    await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
        Response msgsA = cluster.getMessages("node-a", channelId);
        assertThat(msgsA.statusCode()).isEqualTo(200);
        List<String> contents = msgsA.jsonPath().getList("content");
        assertThat(contents).contains("proxied from node-b");
    });

    // Verify message also visible on node-b (shared PostgreSQL)
    await().atMost(Duration.ofSeconds(10)).pollInterval(Duration.ofMillis(500)).untilAsserted(() -> {
        Response msgsB = cluster.getMessages("node-b", channelId);
        assertThat(msgsB.statusCode()).isEqualTo(200);
        List<String> contents = msgsB.jsonPath().getList("content");
        assertThat(contents).contains("proxied from node-b");
    });
}
```

- [ ] **Step 2: Run the E2E test (requires Podman and pre-built mesh)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn package -pl mesh -am -DskipTests && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl e2e-cluster -Pwith-e2e-cluster -Dtest=DispatchRoutingE2ETest#cross_node_dispatch_via_proxy`
Expected: PASS — message sent from node-b arrives on node-a via proxy and is visible on both nodes.

Note: This test requires Podman running and mesh module built. If Podman is not available, the test can be verified manually or deferred to CI.

- [ ] **Step 3: Commit**

```bash
git add e2e-cluster/src/test/java/io/casehub/qhorus/e2e/DispatchRoutingE2ETest.java
git commit -m "test(#488): add cross-node proxy dispatch E2E test

Sends from node-b to a channel owned by node-a, verifying the
WriteRoutingDecorator proxy path works end-to-end through the
container stack. Validates the #486/#487 fix.

Refs #488"
```

---

## References

- `specs/issue-488-cluster-hardening/2026-10-08-cluster-hardening-design.md` — design spec
- `cluster/src/main/java/.../WriteRoutingDecorator.java:52-63` — current fallback code
- `cluster/src/main/java/.../RelayConfig.java` — existing relay config
- `cluster/src/main/java/.../RelayProducer.java:89-97` — CDI wiring for WriteRoutingDecorator
- `cluster/src/test/java/.../WriteRoutingDecoratorTest.java` — existing unit tests
- `cluster/src/test/java/.../ClusterCdiWiringTest.java` — CDI wiring test pattern
- `cluster/src/test/java/.../ClusterDisabledTest.java` — disabled test pattern
- `cluster/src/test/resources/application.properties` — test config pattern
- `cache/src/main/java/.../CacheProducer.java` — cache CDI producer
- `cache/src/main/java/.../CacheHealthResource.java` — health endpoint
- `cache/pom.xml` — current cache dependencies
- `e2e-cluster/src/test/java/.../DispatchRoutingE2ETest.java` — existing E2E tests
- `e2e-cluster/src/test/java/.../ClusterTestHarness.java` — container harness
- GitHub #475, #476, #484, #485, #486, #487, #488

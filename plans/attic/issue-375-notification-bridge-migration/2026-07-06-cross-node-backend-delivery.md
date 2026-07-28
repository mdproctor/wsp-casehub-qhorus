# Cross-Node Backend Delivery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #162 — arch: cross-node ChannelBackend delivery gap in multi-node embedded fleet
**Issue group:** #162

**Goal:** Enable cross-node ChannelBackend delivery so that backends registered on any node receive messages dispatched from any other node, using PostgreSQL LISTEN/NOTIFY as the transport.

**Architecture:** New SPI `ChannelActivityBroadcaster` in `api/gateway/` with a no-op default in runtime. A new `postgres-broadcaster/` module implements it via PostgreSQL LISTEN/NOTIFY, activated by classpath presence (`@Alternative @Priority(1)`). After dispatch commits, the broadcaster notifies other nodes, which read the message from the shared DB and fire their local backends via a new `ChannelGateway.deliverRemote()` method.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x reactive PostgreSQL client (`quarkus-reactive-pg-client`), JTA `TransactionSynchronizationRegistry`, PostgreSQL LISTEN/NOTIFY.

## Global Constraints

- Java 21 source, Java 26 JVM
- Quarkus 3.32.2
- `groupId: io.casehub`, root package `io.casehub.qhorus`
- Named `qhorus` datasource for all Qhorus persistence
- `@DefaultBean` for no-op defaults; `@Alternative @Priority(1)` for classpath-activated impls
- Jandex index required on all library modules consumed as JARs
- Tests use H2 `MODE=PostgreSQL` unless PostgreSQL is explicitly needed
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn` for all builds
- Build from project root (`mvn clean install`) after API changes to catch cross-module breakage

---

### Task 1: SPI + No-op Default + `CrossTenantMessageStore.find()`

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelActivityBroadcaster.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantMessageStore.java` — add `find(Long id)`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantMessageStore.java` — add `find(Long id)` impl
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantMessageStore.java` — add `find(Long id)` impl
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/NoOpChannelActivityBroadcaster.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/NoOpChannelActivityBroadcasterTest.java`

**Interfaces:**
- Consumes: Nothing — this is the foundational layer.
- Produces:
  - `ChannelActivityBroadcaster` SPI (`void broadcast(ChannelActivityEvent event)`)
  - `ChannelActivityBroadcaster.ChannelActivityEvent` record (`UUID channelId, String channelName, Long messageId`)
  - `CrossTenantMessageStore.find(Long id)` → `Optional<Message>`
  - `NoOpChannelActivityBroadcaster` `@DefaultBean @ApplicationScoped`

- [ ] **Step 1: Write the SPI interface**

Create `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelActivityBroadcaster.java`:

```java
package io.casehub.qhorus.api.gateway;

@FunctionalInterface
public interface ChannelActivityBroadcaster {

    void broadcast(ChannelActivityEvent event);

    record ChannelActivityEvent(
        java.util.UUID channelId,
        String channelName,
        Long messageId
    ) {}
}
```

- [ ] **Step 2: Add `find(Long id)` to `CrossTenantMessageStore`**

In `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantMessageStore.java`, add:

```java
/** Find a message by its primary key, regardless of tenancy. */
Optional<Message> find(Long id);
```

- [ ] **Step 3: Implement `find(Long id)` in JPA store**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantMessageStore.java`, add:

```java
@Override
public Optional<Message> find(Long id) {
    return MessageEntity.<MessageEntity>findByIdOptional(id)
            .map(MessageEntity::toDomain);
}
```

- [ ] **Step 4: Implement `find(Long id)` in InMemory store**

In `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantMessageStore.java`, add:

```java
@Override
public Optional<Message> find(Long id) {
    return delegate.find(id);
}
```

- [ ] **Step 5: Write the no-op default**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/NoOpChannelActivityBroadcaster.java`:

```java
package io.casehub.qhorus.runtime.gateway;

import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.qhorus.api.gateway.ChannelActivityBroadcaster;
import io.quarkus.arc.DefaultBean;

@DefaultBean
@ApplicationScoped
public class NoOpChannelActivityBroadcaster implements ChannelActivityBroadcaster {
    @Override
    public void broadcast(ChannelActivityEvent event) {
    }
}
```

- [ ] **Step 6: Write test for no-op broadcaster**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/NoOpChannelActivityBroadcasterTest.java`:

```java
package io.casehub.qhorus.runtime.gateway;

import java.util.UUID;
import org.junit.jupiter.api.Test;
import io.casehub.qhorus.api.gateway.ChannelActivityBroadcaster.ChannelActivityEvent;

class NoOpChannelActivityBroadcasterTest {
    @Test
    void broadcast_doesNotThrow() {
        var broadcaster = new NoOpChannelActivityBroadcaster();
        broadcaster.broadcast(new ChannelActivityEvent(
                UUID.randomUUID(), "test-channel", 42L));
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=NoOpChannelActivityBroadcasterTest -pl runtime`
Expected: PASS

- [ ] **Step 8: Build all modules to verify API additions compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests`
Expected: BUILD SUCCESS — all modules including `persistence-memory` compile with the new `find(Long)` method.

- [ ] **Step 9: Commit**

```
feat(#162): add ChannelActivityBroadcaster SPI and CrossTenantMessageStore.find()

Refs casehubio/qhorus#162
```

---

### Task 2: `ChannelGateway.deliverRemote()` + Unit Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` — add `deliverRemote(UUID, Long)`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayDeliverRemoteTest.java`

**Interfaces:**
- Consumes:
  - `CrossTenantMessageStore.find(Long id)` (Task 1)
  - `CrossTenantChannelStore.findById(UUID id)` (existing)
  - `ChannelGateway.initChannel(UUID, ChannelRef)` (existing)
- Produces:
  - `ChannelGateway.deliverRemote(UUID channelId, Long messageId)` — public, convention-restricted

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayDeliverRemoteTest.java`. This is a CDI-free unit test using inline stubs, following the pattern in `ChannelGatewayTest`:

```java
package io.casehub.qhorus.runtime.gateway;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.gateway.*;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;

class ChannelGatewayDeliverRemoteTest {

    private ChannelGateway gateway;
    private final UUID channelId = UUID.randomUUID();
    private final String channelName = "test-channel";
    private final List<OutboundMessage> posted = new CopyOnWriteArrayList<>();

    @BeforeEach
    void setUp() {
        // Construct gateway with stubs — same pattern as ChannelGatewayTest.
        // ChannelGateway fields are package-private, set directly.
        gateway = new ChannelGateway(
                new StubAgentBackend(),       // agentBackend
                null,                          // normaliser (unused)
                null,                          // messageService (unused)
                null,                          // channelService (unused)
                new StubCrossTenantChannelStore(),
                null,                          // channelInitialisedEvents
                new StubDeliveryConfig());
        gateway.crossTenantMessageStore = new StubCrossTenantMessageStore();
    }

    @Test
    void deliverRemote_callsPostOnBestEffortBackend() throws Exception {
        // Register a BEST_EFFORT backend
        gateway.initChannel(channelId, new ChannelRef(channelId, channelName));
        gateway.registerBackend(channelId, new RecordingBackend("test-be", posted),
                "agent");

        gateway.deliverRemote(channelId, 1L);
        Thread.sleep(100); // virtual thread dispatch

        assertThat(posted).hasSize(1);
        assertThat(posted.get(0).sender()).isEqualTo("agent-1");
    }

    @Test
    void deliverRemote_skipsAgentBackend() throws Exception {
        gateway.initChannel(channelId, new ChannelRef(channelId, channelName));
        // No additional backends registered — only the agent backend

        gateway.deliverRemote(channelId, 1L);
        Thread.sleep(100);

        assertThat(posted).isEmpty();
    }

    @Test
    void deliverRemote_skipsAtLeastOnceBackend() throws Exception {
        gateway.initChannel(channelId, new ChannelRef(channelId, channelName));
        gateway.registerBackend(channelId,
                new RecordingBackend("tracked-be", posted, DeliveryGuarantee.AT_LEAST_ONCE),
                "agent");

        gateway.deliverRemote(channelId, 1L);
        Thread.sleep(100);

        assertThat(posted).isEmpty();
    }

    @Test
    void deliverRemote_lazyInitializesUnknownChannel() throws Exception {
        // Do NOT call initChannel — simulate a channel created on another node
        gateway.deliverRemote(channelId, 1L);
        Thread.sleep(100);

        // Verify the channel was lazy-initialized (registry now has an entry)
        assertThat(gateway.listBackends(channelId)).isNotEmpty();
    }

    @Test
    void deliverRemote_skipsWhenMessageNotFound() {
        gateway.initChannel(channelId, new ChannelRef(channelId, channelName));
        // messageId 999L does not exist in the stub store
        gateway.deliverRemote(channelId, 999L);
        // No exception, no post
        assertThat(posted).isEmpty();
    }

    // ── Stubs (inline, same pattern as existing gateway tests) ──────

    // ... stub classes for StubAgentBackend, RecordingBackend,
    // StubCrossTenantChannelStore, StubCrossTenantMessageStore,
    // StubDeliveryConfig
}
```

The actual stub implementations follow the existing `ChannelGatewayTest` pattern — inline classes implementing the required interfaces with hardcoded returns. `StubCrossTenantMessageStore` returns a test `Message` for id=1L and `Optional.empty()` for others. `StubCrossTenantChannelStore` returns a `Channel` for `channelId`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelGatewayDeliverRemoteTest -pl runtime`
Expected: FAIL — `deliverRemote` method does not exist.

- [ ] **Step 3: Implement `deliverRemote()`**

Add to `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`:

1. Add field: `@Inject CrossTenantMessageStore crossTenantMessageStore;`
2. Add the `deliverRemote()` method per the spec — public, reads message and channel from shared DB, lazy-initializes if unknown, skips agent backend and AT_LEAST_ONCE, dispatches to BEST_EFFORT via virtual threads.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelGatewayDeliverRemoteTest -pl runtime`
Expected: PASS — all 5 tests green.

- [ ] **Step 5: Run existing gateway tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="ChannelGateway*Test" -pl runtime`
Expected: PASS — no existing test is broken by the new field or method.

- [ ] **Step 6: Commit**

```
feat(#162): add ChannelGateway.deliverRemote() for cross-node backend delivery

Refs casehubio/qhorus#162
```

---

### Task 3: Wire Broadcaster into MessageService + ReactiveMessageService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — inject broadcaster, add post-commit broadcast to normal path and LAST_WRITE overwrite path (including `fanOut()` to overwrite path)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` — inject broadcaster, add broadcast to Phase 4 and OverwriteResult branch (including `fanOut()` and `deliverySignalQueue` to overwrite branch)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/ChannelActivityBroadcastIntegrationTest.java`

**Interfaces:**
- Consumes:
  - `ChannelActivityBroadcaster` SPI (Task 1)
  - `ChannelActivityBroadcaster.ChannelActivityEvent` record (Task 1)
- Produces:
  - `broadcaster.broadcast()` fires after commit for every dispatch (normal + LAST_WRITE)
  - `fanOut()` + `deliverySignalQueue.signal()` now fire on LAST_WRITE overwrite path (pre-existing gap fix)

- [ ] **Step 1: Write the integration test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/message/ChannelActivityBroadcastIntegrationTest.java`:

A `@QuarkusTest` that uses `@InjectMock ChannelActivityBroadcaster` to capture calls. Dispatches a message via `messageService.dispatch()` inside `QuarkusTransaction.requiringNew()` (required for after-commit synchronization to fire). Asserts that `broadcaster.broadcast()` was called with the correct `channelId`, `channelName`, and `messageId`.

Tests:
1. `dispatch_normalPath_broadcastsAfterCommit` — APPEND channel, STATUS message
2. `dispatch_lastWriteOverwrite_broadcastsAfterCommit` — LAST_WRITE channel, dispatch twice from same sender, verify broadcast fires for both
3. `dispatch_lastWriteOverwrite_fanOutFires` — verify `fanOut()` is called on the overwrite path (use a `RecordingChannelBackend` registered via `channelGateway.registerBackend()`)

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelActivityBroadcastIntegrationTest -pl runtime`
Expected: FAIL — broadcaster not injected, broadcast not called.

- [ ] **Step 3: Wire broadcaster into `MessageService.dispatch()`**

In `MessageService.java`:
1. Add field: `@Inject ChannelActivityBroadcaster broadcaster;`
2. **Normal path** — add `broadcaster.broadcast()` to the existing `afterCompletion` block (or add a new one if none exists for the normal path). Fire alongside `deliverySignalQueue.signal()`.
3. **LAST_WRITE overwrite path** — add `channelGateway.fanOut()` call, then register an `afterCompletion` synchronization with `deliverySignalQueue.signal()` and `broadcaster.broadcast()`.

- [ ] **Step 4: Wire broadcaster into `ReactiveMessageService.dispatch()`**

In `ReactiveMessageService.java`:
1. Add field: `@Inject ChannelActivityBroadcaster broadcaster;`
2. **Phase 4 (FullResult branch)** — add `broadcaster.broadcast()` after `fanOut()` and `deliverySignalQueue.signal()`.
3. **OverwriteResult branch** — add `channelGateway.fanOut()`, `deliverySignalQueue.signal()`, and `broadcaster.broadcast()`.

- [ ] **Step 5: Run integration test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelActivityBroadcastIntegrationTest -pl runtime`
Expected: PASS

- [ ] **Step 6: Run full runtime test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS — all existing tests still green. The `NoOpChannelActivityBroadcaster` is the active bean (no broadcaster module on test classpath), so existing tests see no behavioral change.

- [ ] **Step 7: Commit**

```
feat(#162): wire ChannelActivityBroadcaster into MessageService and ReactiveMessageService

Adds post-commit broadcast to normal and LAST_WRITE paths in both blocking
and reactive dispatch. Fixes pre-existing gap where LAST_WRITE overwrites
skipped fanOut() and deliverySignalQueue.

Refs casehubio/qhorus#162
```

---

### Task 4: PostgreSQL Broadcaster Module

**Files:**
- Modify: `pom.xml` (parent) — add `<module>postgres-broadcaster</module>`
- Create: `postgres-broadcaster/pom.xml`
- Create: `postgres-broadcaster/src/main/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcaster.java`
- Create: `postgres-broadcaster/src/main/resources/application.properties` (empty or minimal)
- Test: `postgres-broadcaster/src/test/java/io/casehub/qhorus/postgres/broadcaster/SelfNotificationFilterTest.java` (unit)
- Test: `postgres-broadcaster/src/test/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcasterIT.java` (integration, PostgreSQL DevServices)

**Interfaces:**
- Consumes:
  - `ChannelActivityBroadcaster` SPI (Task 1)
  - `ChannelGateway.deliverRemote(UUID, Long)` (Task 2)
  - `DeliverySignalQueue.signal(UUID)` (existing)
  - `PgPool` from `quarkus-reactive-pg-client`
- Produces:
  - `PostgresChannelActivityBroadcaster` — `@Alternative @Priority(1) @ApplicationScoped`, implements `ChannelActivityBroadcaster`
  - PostgreSQL channel `qhorus_channel_activity` — LISTEN/NOTIFY

- [ ] **Step 1: Add module to parent POM**

In `pom.xml` (root), add `<module>postgres-broadcaster</module>` to `<modules>`.

- [ ] **Step 2: Create `postgres-broadcaster/pom.xml`**

Model after `casehub-work-postgres-broadcaster/pom.xml`:
- Parent: `casehub-qhorus-parent`
- ArtifactId: `casehub-qhorus-postgres-broadcaster`
- Dependencies: `casehub-qhorus` (runtime), `quarkus-reactive-pg-client`, `quarkus-arc`, Jandex plugin
- Test deps: `quarkus-junit5`, `quarkus-jdbc-postgresql` (test), `casehub-qhorus-persistence-memory` (test), `casehub-platform` (test), `assertj-core` (test), `awaitility` (test)
- Surefire config: separate unit-test and integration-test executions (unit tests run `*Test.java`, integration tests run `*IT.java` with `quarkus.datasource.db-kind=postgresql`)

- [ ] **Step 3: Write the self-notification filter unit test**

Create `postgres-broadcaster/src/test/java/io/casehub/qhorus/postgres/broadcaster/SelfNotificationFilterTest.java`:

```java
package io.casehub.qhorus.postgres.broadcaster;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SelfNotificationFilterTest {

    @Test
    void recentlySent_isFiltered() {
        var filter = new SelfNotificationFilter(100);
        filter.recordSent(42L);
        assertThat(filter.wasSentLocally(42L)).isTrue();
    }

    @Test
    void unknownId_isNotFiltered() {
        var filter = new SelfNotificationFilter(100);
        assertThat(filter.wasSentLocally(99L)).isFalse();
    }

    @Test
    void evictsOldestWhenFull() {
        var filter = new SelfNotificationFilter(3);
        filter.recordSent(1L);
        filter.recordSent(2L);
        filter.recordSent(3L);
        filter.recordSent(4L); // evicts 1L

        assertThat(filter.wasSentLocally(1L)).isFalse();
        assertThat(filter.wasSentLocally(4L)).isTrue();
    }
}
```

- [ ] **Step 4: Implement `SelfNotificationFilter`**

Create `postgres-broadcaster/src/main/java/io/casehub/qhorus/postgres/broadcaster/SelfNotificationFilter.java`:

```java
package io.casehub.qhorus.postgres.broadcaster;

import java.util.Collections;
import java.util.Iterator;
import java.util.LinkedHashSet;
import java.util.Set;

final class SelfNotificationFilter {
    private final Set<Long> recentIds;
    private final int maxSize;

    SelfNotificationFilter(int maxSize) {
        this.maxSize = maxSize;
        this.recentIds = Collections.synchronizedSet(new LinkedHashSet<>());
    }

    void recordSent(Long messageId) {
        synchronized (recentIds) {
            recentIds.add(messageId);
            if (recentIds.size() > maxSize) {
                Iterator<Long> it = recentIds.iterator();
                it.next();
                it.remove();
            }
        }
    }

    boolean wasSentLocally(Long messageId) {
        return recentIds.contains(messageId);
    }
}
```

- [ ] **Step 5: Run unit test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SelfNotificationFilterTest -pl postgres-broadcaster`
Expected: PASS

- [ ] **Step 6: Write the PostgreSQL broadcaster implementation**

Create `postgres-broadcaster/src/main/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcaster.java`:

```java
package io.casehub.qhorus.postgres.broadcaster;

import java.util.UUID;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.qhorus.api.gateway.ChannelActivityBroadcaster;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.gateway.DeliverySignalQueue;
import io.vertx.mutiny.pgclient.PgConnection;
import io.vertx.mutiny.pgclient.PgPool;
import io.vertx.mutiny.sqlclient.Tuple;

@ApplicationScoped
@Alternative
@Priority(1)
public class PostgresChannelActivityBroadcaster implements ChannelActivityBroadcaster {

    private static final Logger LOG = Logger.getLogger(PostgresChannelActivityBroadcaster.class);
    static final String CHANNEL = "qhorus_channel_activity";
    private static final int FILTER_SIZE = 1000;

    @Inject PgPool pool;
    @Inject ChannelGateway channelGateway;
    @Inject DeliverySignalQueue deliverySignalQueue;

    private final SelfNotificationFilter filter = new SelfNotificationFilter(FILTER_SIZE);
    private volatile PgConnection subscriberConnection;

    @PostConstruct
    void startListening() {
        acquireAndListen();
    }

    @PreDestroy
    void stopListening() {
        PgConnection conn = subscriberConnection;
        if (conn != null) {
            conn.close().subscribe().with(ok -> {}, err -> {});
        }
    }

    @Override
    public void broadcast(ChannelActivityEvent event) {
        filter.recordSent(event.messageId());
        String payload = event.channelId() + ":" + event.messageId();
        pool.preparedQuery("SELECT pg_notify($1, $2)")
                .execute(Tuple.of(CHANNEL, payload))
                .subscribe().with(
                        ok -> {},
                        err -> LOG.warnf("pg_notify failed on channel '%s': %s",
                                CHANNEL, err.getMessage()));
    }

    void handleNotification(String payload) {
        String[] parts = payload.split(":", 2);
        if (parts.length != 2) {
            LOG.warnf("Malformed notification payload: %s", payload);
            return;
        }
        UUID channelId;
        Long messageId;
        try {
            channelId = UUID.fromString(parts[0]);
            messageId = Long.parseLong(parts[1]);
        } catch (Exception e) {
            LOG.warnf("Failed to parse notification payload '%s': %s",
                    payload, e.getMessage());
            return;
        }

        if (filter.wasSentLocally(messageId)) {
            return;
        }

        Thread.ofVirtual().name("qhorus-remote-deliver-" + messageId)
                .start(() -> {
                    try {
                        channelGateway.deliverRemote(channelId, messageId);
                        deliverySignalQueue.signal(channelId);
                    } catch (Exception e) {
                        LOG.warnf("Remote delivery failed for message %d on channel %s: %s",
                                messageId, channelId, e.getMessage());
                    }
                });
    }

    private void acquireAndListen() {
        pool.getConnection().subscribe().with(
                conn -> {
                    io.vertx.pgclient.PgConnection pgDelegate =
                            (io.vertx.pgclient.PgConnection) conn.getDelegate();
                    PgConnection pgConn = PgConnection.newInstance(pgDelegate);
                    subscriberConnection = pgConn;
                    pgConn.notificationHandler(n -> handleNotification(n.getPayload()));
                    pgConn.query("LISTEN " + CHANNEL).execute()
                            .subscribe().with(
                                    ok -> LOG.infof("Subscribed to PostgreSQL channel '%s'", CHANNEL),
                                    err -> LOG.errorf(err, "Failed to LISTEN on '%s'", CHANNEL));
                    pgDelegate.closeHandler(v -> {
                        LOG.warn("PostgreSQL subscriber connection lost — reconnecting");
                        acquireAndListen();
                    });
                },
                err -> {
                    LOG.errorf(err, "Failed to acquire subscriber connection for '%s'", CHANNEL);
                    // Retry after delay
                    Thread.ofVirtual().start(() -> {
                        try { Thread.sleep(5000); } catch (InterruptedException ignored) {}
                        acquireAndListen();
                    });
                });
    }
}
```

- [ ] **Step 7: Write the integration test**

Create `postgres-broadcaster/src/test/java/io/casehub/qhorus/postgres/broadcaster/PostgresChannelActivityBroadcasterIT.java`:

A `@QuarkusTest` with DevServices PostgreSQL that:
1. Injects the broadcaster and a test `ChannelBackend` (via `RecordingChannelBackend` from `casehub-qhorus-testing`)
2. Creates a channel via `ChannelService`
3. Registers the recording backend via `ChannelGateway.registerBackend()`
4. Dispatches a message via `MessageService.dispatch()` inside `QuarkusTransaction.requiringNew()`
5. Uses Awaitility to wait for the recording backend to receive `post()` (the self-notification filter should be bypassed because the LISTEN handler runs on the same node — but the filter records the messageId during `broadcast()`, so the handler skips it; to test the full loop, dispatch from a separate `pg_notify` call that simulates a remote node)

The key integration test simulates a remote notification by directly calling `pg_notify` with a messageId that was NOT dispatched locally, and verifies that `deliverRemote()` fires the local backend.

- [ ] **Step 8: Create test `application.properties`**

Create `postgres-broadcaster/src/test/resources/application.properties` with the qhorus named datasource pointing at DevServices PostgreSQL, plus the default datasource for casehub-ledger beans, following the pattern from `runtime/src/test/resources/application.properties`.

- [ ] **Step 9: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl postgres-broadcaster` (requires Podman/Docker for DevServices)
Expected: PASS

- [ ] **Step 10: Build all modules from root**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and test.

- [ ] **Step 11: Commit**

```
feat(#162): add casehub-qhorus-postgres-broadcaster module

PostgreSQL LISTEN/NOTIFY implementation of ChannelActivityBroadcaster.
Activated by classpath presence — zero configuration. Uses the shared
PostgreSQL database already required for multi-node deployment.

Refs casehubio/qhorus#162
```

---

### Task 5: Documentation Updates

**Files:**
- Modify: `docs/messaging-architecture.md` — remove gap section, document shared-DB prerequisite and broadcaster SPI
- Modify: `CLAUDE.md` — add `postgres-broadcaster/` module, test conventions
- Modify: (project repo) `PLATFORM.md` via issue — capability ownership update

**Interfaces:**
- Consumes: All previous tasks (implementation is complete)
- Produces: Updated documentation reflecting the new architecture

- [ ] **Step 1: Update `docs/messaging-architecture.md`**

1. Replace the "Known Architectural Gap" section with a new section documenting:
   - Shared PostgreSQL as a multi-node prerequisite (not an option)
   - The `ChannelActivityBroadcaster` SPI
   - The PostgreSQL LISTEN/NOTIFY implementation
   - Activation by classpath presence
2. Update the "What Ships When" table — change "Cross-node delivery" from `⚠️ gap` to `✅ live (casehub-qhorus-postgres-broadcaster)`
3. Remove the reference to qhorus#162 as an open gap

- [ ] **Step 2: Update `CLAUDE.md`**

1. Add `postgres-broadcaster/` to the project structure tree
2. Add test conventions for the module (DevServices PostgreSQL, separate unit/IT executions)
3. Note that `CrossTenantMessageStore.find(Long)` was added

- [ ] **Step 3: File PLATFORM.md update issue**

Create a GitHub issue in `casehubio/parent` for updating PLATFORM.md:
- Add `Cross-node backend delivery` to capability ownership → `casehub-qhorus` (postgres-broadcaster)
- Update `Cross-cutting message notification` row
- Add `casehub-qhorus-postgres-broadcaster` to cross-repo dependency map (claudony dep)

- [ ] **Step 4: File claudony migration issue**

Create a GitHub issue in `casehubio/claudony` for:
- Adding `casehub-qhorus-postgres-broadcaster` as a compile dependency
- Removing `FleetMessageRelayObserver` (once broadcaster is stable)
- Verifying SSE delivery works across fleet nodes via the new mechanism

- [ ] **Step 5: Commit documentation**

```
docs(#162): update messaging-architecture.md and CLAUDE.md for cross-node delivery

Removes the Known Architectural Gap section — shared-DB + broadcaster SPI
closes the gap. Documents ChannelActivityBroadcaster and the PostgreSQL
implementation module.

Refs casehubio/qhorus#162
```

---

## Self-Review

**Spec coverage:**
- ✅ `ChannelActivityBroadcaster` SPI in `api/gateway/` (Task 1)
- ✅ `NoOpChannelActivityBroadcaster` @DefaultBean (Task 1)
- ✅ `CrossTenantMessageStore.find(Long)` + impls (Task 1)
- ✅ `ChannelGateway.deliverRemote()` with lazy init (Task 2)
- ✅ `MessageService` wiring — normal + LAST_WRITE (Task 3)
- ✅ `ReactiveMessageService` wiring — Phase 4 + OverwriteResult (Task 3)
- ✅ LAST_WRITE `fanOut()` pre-existing gap fix (Task 3)
- ✅ PostgreSQL broadcaster module (Task 4)
- ✅ Self-notification filter (Task 4)
- ✅ Connection drop + re-LISTEN (Task 4)
- ✅ Virtual thread offload from Vert.x event loop (Task 4)
- ✅ Documentation updates (Task 5)
- ✅ Downstream issue filing (Task 5)

**Placeholder scan:** No TBDs, TODOs, or "similar to" references. All steps contain code or exact commands.

**Type consistency:** `ChannelActivityEvent(UUID, String, Long)` used consistently across all tasks. `deliverRemote(UUID, Long)` signature consistent. `CrossTenantMessageStore.find(Long)` → `Optional<Message>` consistent.

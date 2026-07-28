# Delivery Guarantee Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add cursor-based delivery tracking for AT_LEAST_ONCE channel backends, with an event-driven delivery pump and scheduled reconciler.

**Architecture:** The message store is the durable outbox. Each AT_LEAST_ONCE backend gets a per-channel delivery cursor (last delivered message ID). An event-driven pump delivers pending messages; a scheduled reconciler catches JVM restart gaps. BEST_EFFORT backends keep current fire-and-forget behavior with zero overhead.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Panache (named `qhorus` datasource), Flyway, ManagedExecutor, TransactionSynchronizationRegistry

**Spec:** `specs/issue-132-delivery-guarantee-backends/2026-06-29-delivery-guarantee-design.md` (adversarial-reviewed, 24 issues resolved)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl runtime`
- Flyway next domain migration: **V25** (V23/V24 occupied by slack-channel)
- All commits reference `Refs #132`
- Named `qhorus` datasource for all JPA entities
- `import-qhorus-test.sql` required for test modules with `casehub.ledger.enabled=true`
- `@IfBuildProperty` for reactive beans; `casehub.qhorus.reactive.enabled` is BUILD_TIME only — never in `application.properties`
- After API changes in `api/`, run `mvn install` from project root (not just `mvn test -pl runtime`)
- Use IntelliJ MCP for all rename/move/find-references operations

---

### Task 1: SPI Layer — DeliveryGuarantee enum and ChannelBackend default method

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/DeliveryGuarantee.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelBackend.java`
- Test: `api/src/test/java/io/casehub/qhorus/api/gateway/DeliveryGuaranteeTest.java`

**Interfaces:**
- Produces: `DeliveryGuarantee.BEST_EFFORT`, `DeliveryGuarantee.AT_LEAST_ONCE`; `ChannelBackend.deliveryGuarantee()` default method returning `BEST_EFFORT`

- [ ] **Step 1: Write test for DeliveryGuarantee enum values**

```java
package io.casehub.qhorus.api.gateway;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class DeliveryGuaranteeTest {

    @Test
    void enumHasTwoValues() {
        assertThat(DeliveryGuarantee.values())
                .containsExactly(DeliveryGuarantee.BEST_EFFORT, DeliveryGuarantee.AT_LEAST_ONCE);
    }

    @Test
    void channelBackendDefaultIsBestEffort() {
        ChannelBackend stub = new ChannelBackend() {
            @Override public String backendId() { return "stub"; }
            @Override public io.casehub.platform.api.identity.ActorType actorType() {
                return io.casehub.platform.api.identity.ActorType.AGENT;
            }
            @Override public void open(ChannelRef channel, java.util.Map<String, String> metadata) {}
            @Override public void post(ChannelRef channel, OutboundMessage message) {}
            @Override public void close(ChannelRef channel) {}
        };
        assertThat(stub.deliveryGuarantee()).isEqualTo(DeliveryGuarantee.BEST_EFFORT);
    }
}
```

- [ ] **Step 2: Run test — verify it fails** (DeliveryGuarantee does not exist yet)

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryGuaranteeTest -pl api -Dno-format`
Expected: compilation failure

- [ ] **Step 3: Create DeliveryGuarantee enum**

```java
package io.casehub.qhorus.api.gateway;

public enum DeliveryGuarantee {
    BEST_EFFORT,
    AT_LEAST_ONCE
}
```

- [ ] **Step 4: Add default method to ChannelBackend**

Add to `ChannelBackend.java` after the `close()` method:

```java
default DeliveryGuarantee deliveryGuarantee() {
    return DeliveryGuarantee.BEST_EFFORT;
}
```

- [ ] **Step 5: Run test — verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryGuaranteeTest -pl api -Dno-format`
Expected: PASS

- [ ] **Step 6: Run full build to verify no downstream breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -Dno-format`
Expected: BUILD SUCCESS (default method is backward compatible)

- [ ] **Step 7: Commit**

```
feat(#132): add DeliveryGuarantee enum and ChannelBackend.deliveryGuarantee() default

Refs #132
```

---

### Task 2: Data Model + Persistence — DeliveryCursor entity, Flyway V25, store interface, JPA and InMemory implementations, contract tests

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryCursor.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V25__delivery_cursor.sql`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/DeliveryCursorStore.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaDeliveryCursorStore.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/DeliveryCursorPanacheRepo.java`
- Create: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryDeliveryCursorStore.java`
- Create: `testing/src/test/java/io/casehub/qhorus/testing/contract/DeliveryCursorStoreContractTest.java`
- Create: `testing/src/test/java/io/casehub/qhorus/testing/contract/InMemoryDeliveryCursorStoreTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/migration/FlywayMigrationSchemaTest.java` (add V25 assertion)

**Interfaces:**
- Consumes: `DeliveryCursor` (this task defines it)
- Produces: `DeliveryCursorStore` interface with `save()`, `findByChannelAndBackend()`, `findByChannel()`, `findAll()`, `deleteByChannel()`; `InMemoryDeliveryCursorStore` for test consumers

- [ ] **Step 1: Write contract test (abstract base)**

```java
package io.casehub.qhorus.testing.contract;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.gateway.DeliveryCursor;
import io.casehub.qhorus.runtime.store.DeliveryCursorStore;

abstract class DeliveryCursorStoreContractTest {

    protected abstract DeliveryCursorStore store();

    @BeforeEach
    void clearStore() {
        // Subclasses may override for cleanup
    }

    @Test
    void save_newCursor_assignsId() {
        DeliveryCursor cursor = cursor(UUID.randomUUID(), "backend-1", 100L);
        DeliveryCursor saved = store().save(cursor);
        assertThat(saved.id).isNotNull();
    }

    @Test
    void findByChannelAndBackend_exists_returnsIt() {
        UUID channelId = UUID.randomUUID();
        store().save(cursor(channelId, "slack", 50L));
        assertThat(store().findByChannelAndBackend(channelId, "slack"))
                .isPresent()
                .hasValueSatisfying(c -> {
                    assertThat(c.channelId).isEqualTo(channelId);
                    assertThat(c.backendId).isEqualTo("slack");
                    assertThat(c.lastDeliveredId).isEqualTo(50L);
                });
    }

    @Test
    void findByChannelAndBackend_absent_returnsEmpty() {
        assertThat(store().findByChannelAndBackend(UUID.randomUUID(), "nonexistent"))
                .isEmpty();
    }

    @Test
    void findByChannel_returnsAllForChannel() {
        UUID ch = UUID.randomUUID();
        store().save(cursor(ch, "slack", 10L));
        store().save(cursor(ch, "connector", 20L));
        store().save(cursor(UUID.randomUUID(), "other", 30L));
        assertThat(store().findByChannel(ch)).hasSize(2);
    }

    @Test
    void findAll_returnsEverything() {
        store().save(cursor(UUID.randomUUID(), "a", 1L));
        store().save(cursor(UUID.randomUUID(), "b", 2L));
        assertThat(store().findAll()).hasSizeGreaterThanOrEqualTo(2);
    }

    @Test
    void deleteByChannel_removesOnlyThatChannel() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        store().save(cursor(ch1, "slack", 10L));
        store().save(cursor(ch2, "slack", 20L));
        store().deleteByChannel(ch1);
        assertThat(store().findByChannel(ch1)).isEmpty();
        assertThat(store().findByChannel(ch2)).hasSize(1);
    }

    @Test
    void save_existingCursor_updatesLastDeliveredId() {
        UUID ch = UUID.randomUUID();
        DeliveryCursor saved = store().save(cursor(ch, "slack", 10L));
        saved.lastDeliveredId = 50L;
        saved.updatedAt = Instant.now();
        store().save(saved);
        assertThat(store().findByChannelAndBackend(ch, "slack"))
                .hasValueSatisfying(c -> assertThat(c.lastDeliveredId).isEqualTo(50L));
    }

    static DeliveryCursor cursor(UUID channelId, String backendId, Long lastDeliveredId) {
        DeliveryCursor c = new DeliveryCursor();
        c.channelId = channelId;
        c.backendId = backendId;
        c.lastDeliveredId = lastDeliveredId;
        c.createdAt = Instant.now();
        c.updatedAt = Instant.now();
        return c;
    }
}
```

- [ ] **Step 2: Create DeliveryCursor entity**

```java
package io.casehub.qhorus.runtime.gateway;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity
@Table(name = "delivery_cursor",
       uniqueConstraints = @UniqueConstraint(
           name = "uq_delivery_cursor_channel_backend",
           columnNames = {"channel_id", "backend_id"}))
public class DeliveryCursor extends PanacheEntityBase {

    @Id
    @GeneratedValue
    public UUID id;

    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(name = "backend_id", nullable = false)
    public String backendId;

    @Column(name = "last_delivered_id")
    public Long lastDeliveredId;

    @Column(name = "updated_at")
    public Instant updatedAt;

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @PrePersist
    void onPersist() {
        if (createdAt == null) createdAt = Instant.now();
    }
}
```

- [ ] **Step 3: Create DeliveryCursorStore interface**

```java
package io.casehub.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.casehub.qhorus.runtime.gateway.DeliveryCursor;

public interface DeliveryCursorStore {

    DeliveryCursor save(DeliveryCursor cursor);

    Optional<DeliveryCursor> findByChannelAndBackend(UUID channelId, String backendId);

    List<DeliveryCursor> findByChannel(UUID channelId);

    List<DeliveryCursor> findAll();

    void deleteByChannel(UUID channelId);
}
```

- [ ] **Step 4: Create JPA implementation**

`DeliveryCursorPanacheRepo.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import io.quarkus.hibernate.orm.panache.PanacheRepositoryBase;
import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.qhorus.runtime.gateway.DeliveryCursor;
import java.util.UUID;

@ApplicationScoped
public class DeliveryCursorPanacheRepo implements PanacheRepositoryBase<DeliveryCursor, UUID> {}
```

`JpaDeliveryCursorStore.java`:
```java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.qhorus.runtime.gateway.DeliveryCursor;
import io.casehub.qhorus.runtime.store.DeliveryCursorStore;

@ApplicationScoped
public class JpaDeliveryCursorStore implements DeliveryCursorStore {

    @Inject
    DeliveryCursorPanacheRepo repo;

    @Override
    @Transactional
    public DeliveryCursor save(DeliveryCursor c) {
        if (c.id == null) {
            repo.persist(c);
        } else {
            c = repo.getEntityManager().merge(c);
        }
        return c;
    }

    @Override
    public Optional<DeliveryCursor> findByChannelAndBackend(UUID channelId, String backendId) {
        return repo.find("channelId = ?1 AND backendId = ?2", channelId, backendId)
                .firstResultOptional();
    }

    @Override
    public List<DeliveryCursor> findByChannel(UUID channelId) {
        return repo.list("channelId", channelId);
    }

    @Override
    public List<DeliveryCursor> findAll() {
        return repo.listAll();
    }

    @Override
    @Transactional
    public void deleteByChannel(UUID channelId) {
        repo.delete("channelId", channelId);
    }
}
```

- [ ] **Step 5: Create InMemoryDeliveryCursorStore**

```java
package io.casehub.qhorus.testing;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.runtime.gateway.DeliveryCursor;
import io.casehub.qhorus.runtime.store.DeliveryCursorStore;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryDeliveryCursorStore implements DeliveryCursorStore {

    private final Map<UUID, DeliveryCursor> byId = new LinkedHashMap<>();

    @Override
    public DeliveryCursor save(DeliveryCursor c) {
        if (c.id == null) {
            c.id = UUID.randomUUID();
        }
        if (c.createdAt == null) {
            c.createdAt = Instant.now();
        }
        byId.put(c.id, c);
        return c;
    }

    @Override
    public Optional<DeliveryCursor> findByChannelAndBackend(UUID channelId, String backendId) {
        return byId.values().stream()
                .filter(c -> channelId.equals(c.channelId) && backendId.equals(c.backendId))
                .findFirst();
    }

    @Override
    public List<DeliveryCursor> findByChannel(UUID channelId) {
        return byId.values().stream()
                .filter(c -> channelId.equals(c.channelId))
                .toList();
    }

    @Override
    public List<DeliveryCursor> findAll() {
        return List.copyOf(byId.values());
    }

    @Override
    public void deleteByChannel(UUID channelId) {
        byId.values().removeIf(c -> channelId.equals(c.channelId));
    }

    public void clear() {
        byId.clear();
    }
}
```

- [ ] **Step 6: Write concrete contract test runner**

```java
package io.casehub.qhorus.testing.contract;

import io.casehub.qhorus.runtime.store.DeliveryCursorStore;
import io.casehub.qhorus.testing.InMemoryDeliveryCursorStore;
import org.junit.jupiter.api.BeforeEach;

class InMemoryDeliveryCursorStoreTest extends DeliveryCursorStoreContractTest {

    private final InMemoryDeliveryCursorStore store = new InMemoryDeliveryCursorStore();

    @Override
    protected DeliveryCursorStore store() { return store; }

    @Override
    @BeforeEach
    void clearStore() { store.clear(); }
}
```

- [ ] **Step 7: Create Flyway V25 migration**

`runtime/src/main/resources/db/qhorus/migration/V25__delivery_cursor.sql`:
```sql
CREATE TABLE delivery_cursor (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id        UUID         NOT NULL,
    backend_id        VARCHAR(255) NOT NULL,
    last_delivered_id BIGINT,
    updated_at        TIMESTAMP,
    created_at        TIMESTAMP    NOT NULL DEFAULT now(),
    CONSTRAINT uq_delivery_cursor_channel_backend UNIQUE (channel_id, backend_id),
    CONSTRAINT fk_delivery_cursor_channel FOREIGN KEY (channel_id) REFERENCES channel(id) ON DELETE CASCADE
);
```

- [ ] **Step 8: Add DeliveryCursor to hibernate packages**

The `DeliveryCursor` entity is in `io.casehub.qhorus.runtime.gateway` — verify this package is covered by the qhorus PU's `packages` config. If not, add it. Check `runtime/src/main/resources/application.properties` for `quarkus.hibernate-orm.qhorus.packages`.

- [ ] **Step 9: Run contract tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryDeliveryCursorStoreTest -pl testing -Dno-format`
Expected: PASS

- [ ] **Step 10: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```
feat(#132): add DeliveryCursor entity, V25 migration, store interface with JPA and InMemory impls

Refs #132
```

---

### Task 3: Infrastructure — DeliverySignalQueue, DeliveryConfig, RecordingChannelBackend enhancement

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliverySignalQueue.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/config/DeliveryConfig.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/gateway/RecordingChannelBackend.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/DeliverySignalQueueTest.java`

**Interfaces:**
- Consumes: `DeliveryGuarantee` (Task 1)
- Produces: `DeliverySignalQueue.signal(UUID)`, `DeliverySignalQueue.poll(long, TimeUnit)`, `DeliverySignalQueue.drainTo(Collection)`; `DeliveryConfig` with `enabled()`, `batchSize()`, `maxConsecutiveFailures()`, `reconciliationInterval()`; `RecordingChannelBackend(String, ActorType, DeliveryGuarantee)` 3-arg constructor

- [ ] **Step 1: Write DeliverySignalQueue test**

```java
package io.casehub.qhorus.runtime.gateway;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.Test;

class DeliverySignalQueueTest {

    @Test
    void signal_wakesPollThread() throws InterruptedException {
        DeliverySignalQueue queue = new DeliverySignalQueue();
        UUID channelId = UUID.randomUUID();
        queue.signal(channelId);
        UUID polled = queue.poll(1, TimeUnit.SECONDS);
        assertThat(polled).isEqualTo(channelId);
    }

    @Test
    void drainTo_collectsMultipleSignals() throws InterruptedException {
        DeliverySignalQueue queue = new DeliverySignalQueue();
        UUID a = UUID.randomUUID();
        UUID b = UUID.randomUUID();
        queue.signal(a);
        queue.signal(b);
        UUID first = queue.poll(1, TimeUnit.SECONDS);
        List<UUID> rest = new ArrayList<>();
        queue.drainTo(rest);
        assertThat(first).isEqualTo(a);
        assertThat(rest).containsExactly(b);
    }

    @Test
    void poll_timeout_returnsNull() throws InterruptedException {
        DeliverySignalQueue queue = new DeliverySignalQueue();
        UUID polled = queue.poll(50, TimeUnit.MILLISECONDS);
        assertThat(polled).isNull();
    }
}
```

- [ ] **Step 2: Create DeliverySignalQueue**

```java
package io.casehub.qhorus.runtime.gateway;

import java.util.Collection;
import java.util.UUID;
import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.TimeUnit;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class DeliverySignalQueue {

    private final LinkedBlockingDeque<UUID> queue = new LinkedBlockingDeque<>();

    public void signal(UUID channelId) {
        queue.offer(channelId);
    }

    public UUID poll(long timeout, TimeUnit unit) throws InterruptedException {
        return queue.poll(timeout, unit);
    }

    public int drainTo(Collection<? super UUID> c) {
        return queue.drainTo(c);
    }
}
```

- [ ] **Step 3: Create DeliveryConfig**

```java
package io.casehub.qhorus.runtime.config;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.qhorus.delivery")
public interface DeliveryConfig {

    @WithDefault("true")
    boolean enabled();

    @WithDefault("100")
    int batchSize();

    @WithDefault("10")
    int maxConsecutiveFailures();

    @WithDefault("30s")
    String reconciliationInterval();
}
```

- [ ] **Step 4: Enhance RecordingChannelBackend**

Add a `deliveryGuarantee` field and 3-arg constructor. The existing 2-arg constructor defaults to `BEST_EFFORT`:

```java
// Add import
import io.casehub.qhorus.api.gateway.DeliveryGuarantee;

// Add field
private final DeliveryGuarantee deliveryGuarantee;

// Replace constructor
public RecordingChannelBackend(String backendId, ActorType actorType) {
    this(backendId, actorType, DeliveryGuarantee.BEST_EFFORT);
}

public RecordingChannelBackend(String backendId, ActorType actorType, DeliveryGuarantee deliveryGuarantee) {
    this.backendId = backendId;
    this.actorType = actorType;
    this.deliveryGuarantee = deliveryGuarantee;
}

// Add override
@Override
public DeliveryGuarantee deliveryGuarantee() { return deliveryGuarantee; }
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliverySignalQueueTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat(#132): add DeliverySignalQueue, DeliveryConfig, RecordingChannelBackend enhancement

Refs #132
```

---

### Task 4: DeliveryService — the pump core with unit tests

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryService.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/DeliveryServiceTest.java`

**Interfaces:**
- Consumes: `DeliveryCursorStore` (Task 2), `DeliverySignalQueue` (Task 3), `DeliveryConfig` (Task 3), `CrossTenantMessageStore.scan()`, `CrossTenantMessageStore.findLastMessage()`, `CrossTenantChannelStore.findById()`, `ChannelGateway.trackedEntries()` (Task 5 — stub in tests), `ManagedExecutor`
- Produces: `DeliveryService.signal(UUID)` (delegates to queue), `DeliveryService.processChannel(UUID)`, `DeliveryService.reconcileAll()`, `DeliveryService.isUnhealthy(String)`

This is the largest task. The unit tests use CDI-free wiring with InMemory stores and a mock gateway. The pump thread is NOT started in unit tests — tests call `processChannel()` and `deliverPending()` directly.

- [ ] **Step 1: Write unit tests for DeliveryService**

Write `DeliveryServiceTest.java` with these tests (CDI-free — construct DeliveryService directly, inject InMemory stores and mock ChannelGateway):

Test cases:
- `deliverBatch_pendingMessages_deliversInOrderAndAdvancesCursor` — messages 101,102,103 delivered, cursor at 103
- `deliverBatch_noMessages_returnsEmpty` — cursor at head, BatchResult.EMPTY
- `deliverBatch_postFailure_stopsAndPreservesOrder` — 102 fails, cursor stays at 101
- `deliverBatch_consecutiveFailures_marksUnhealthy` — N failures → isUnhealthy returns true
- `deliverBatch_successAfterFailures_resetsHealth` — delivery succeeds → isUnhealthy returns false
- `deliverBatch_noCursor_initializesAtHead` — first call creates cursor at message head
- `deliverBatch_channelDeleted_returnsFailed` — channel not found → BatchResult.FAILED
- `processChannel_unhealthyBackend_skipped` — unhealthy backend not processed
- `processChannel_multipleBackends_processIndependently` — each backend gets its own delivery

Each test constructs the service with InMemory stores, a mock ChannelGateway that returns a controllable `trackedEntries()`, and a `RecordingChannelBackend`.

- [ ] **Step 2: Implement DeliveryService**

Key implementation points from the spec:
- `@PostConstruct` starts pump thread via `ManagedExecutor.execute()`
- `@PreDestroy` sets `volatile running = false`
- `pumpLoop()` — poll/drain signal queue, process each channel with top-level try-catch
- `processChannel()` — iterate `trackedEntries()`, skip unhealthy, spawn per-backend task via `managedExecutor.execute()` guarded by `activeDeliveries`
- `deliverPending()` — loop calling `deliverBatch()` until EMPTY or FAILED
- `deliverBatch()` — `@Transactional`, loads cursor (lazy init), queries `messageStore.scan(MessageQuery.poll(...))`, delivers in order, cursor per-batch, health tracking
- `toOutbound(Message)` — constructs OutboundMessage with `ActorTypeResolver.resolve(sender)`
- `@Scheduled reconcileAll()` — scans all cursors, joins with registry, calls `processChannel()`
- Thread creation failure cleanup on `activeDeliveries`

The `deliverBatch()` method must be on a separate CDI bean (or use `self` injection pattern) for `@Transactional` to work via proxy — self-invocation bypasses CDI interceptors. Create a package-private `DeliveryBatchExecutor` bean with the `@Transactional deliverBatch()` method.

- [ ] **Step 3: Run unit tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryServiceTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 4: Commit**

```
feat(#132): add DeliveryService — event-driven pump with scheduled reconciler

Refs #132
```

---

### Task 5: Gateway + Dispatch Integration — fanOut changes, trackedEntries, post-commit signaling, backend overrides

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java`
- Modify: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java` (extend existing)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/FanOutDeliveryGuaranteeTest.java`

**Interfaces:**
- Consumes: `DeliveryGuarantee` (Task 1), `DeliverySignalQueue` (Task 3), `DeliveryConfig` (Task 3), `DeliveryService` (Task 4)
- Produces: `ChannelGateway.fanOut()` returns `boolean hasTracked`; `ChannelGateway.trackedEntries(UUID)`; post-commit signal in `MessageService.dispatch()`

- [ ] **Step 1: Write fanOut delivery guarantee tests**

```java
package io.casehub.qhorus.runtime.gateway;

// Tests for fanOut behavior with mixed delivery guarantees
class FanOutDeliveryGuaranteeTest {

    // CDI-free — construct ChannelGateway with mocks

    @Test
    void fanOut_bestEffortBackend_deliveredDirectly()
    // Backend with BEST_EFFORT receives post() call

    @Test
    void fanOut_atLeastOnceBackend_skippedWhenEnabled()
    // Backend with AT_LEAST_ONCE NOT called when delivery enabled

    @Test
    void fanOut_atLeastOnceBackend_deliveredWhenDisabled()
    // Backend with AT_LEAST_ONCE IS called when delivery disabled (safe fallback)

    @Test
    void fanOut_mixedBackends_correctRouting()
    // BEST_EFFORT delivered, AT_LEAST_ONCE skipped, returns hasTracked=true

    @Test
    void fanOut_noTrackedBackends_returnsFalse()
    // Only BEST_EFFORT → returns false

    @Test
    void trackedEntries_filtersCorrectly()
    // Returns only AT_LEAST_ONCE backends, snapshots list
}
```

- [ ] **Step 2: Modify ChannelGateway**

Changes:
1. Inject `DeliveryConfig` (for `enabled` check)
2. Add package-private `trackedEntries(UUID)` method with `List.copyOf()` snapshot
3. Change `fanOut()` return type from `void` to `boolean`
4. In `fanOut()`: check `deliveryConfig.enabled() && backend.deliveryGuarantee() == AT_LEAST_ONCE` → skip + set `hasTracked = true`; when `!deliveryConfig.enabled()` → fire-and-forget for ALL backends (safe fallback)
5. Return `hasTracked`

- [ ] **Step 3: Modify MessageService.dispatch()**

After `channelGateway.fanOut()`:
1. Capture `boolean hasTracked` return value
2. If `hasTracked`, register post-commit synchronization via existing `tsr` field:
```java
if (hasTracked) {
    final UUID signalChannelId = ch.id;
    tsr.registerInterposedSynchronization(new Synchronization() {
        @Override public void beforeCompletion() {}
        @Override public void afterCompletion(int status) {
            if (status == STATUS_COMMITTED) {
                deliverySignalQueue.signal(signalChannelId);
            }
        }
    });
}
```
3. Inject `DeliverySignalQueue` into MessageService

- [ ] **Step 4: Modify ReactiveMessageService**

Apply the same pattern for the reactive path. The reactive `dispatch()` returns `Uni<DispatchResult>`. The signal should fire after the reactive transaction commits. Use `Uni.invoke()` after `Panache.withTransaction()` returns.

- [ ] **Step 5: Override deliveryGuarantee in SlackChannelBackend**

Add to `SlackChannelBackend.java`:
```java
@Override
public DeliveryGuarantee deliveryGuarantee() {
    return DeliveryGuarantee.AT_LEAST_ONCE;
}
```
Add import for `DeliveryGuarantee`.

- [ ] **Step 6: Override deliveryGuarantee in ConnectorChannelBackend**

Add to `ConnectorChannelBackend.java`:
```java
@Override
public DeliveryGuarantee deliveryGuarantee() {
    return DeliveryGuarantee.AT_LEAST_ONCE;
}
```
Add import for `DeliveryGuarantee`.

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FanOutDeliveryGuaranteeTest -pl runtime -Dno-format`
Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelGatewayTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: BUILD SUCCESS (verify slack-channel and connector-backend modules compile)

- [ ] **Step 9: Commit**

```
feat(#132): integrate delivery pump — fanOut routing, post-commit signaling, backend overrides

SlackChannelBackend and ConnectorChannelBackend declare AT_LEAST_ONCE.
fanOut() skips tracked backends when delivery enabled; falls back to
fire-and-forget when disabled. Post-commit signal via TSR ensures pump
sees committed messages.

Refs #132
```

---

### Task 6: Integration Tests — end-to-end pump delivery verification

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/DeliveryServiceIntegrationTest.java`

**Interfaces:**
- Consumes: All prior tasks

Integration tests using `@QuarkusTest` with `RecordingChannelBackend(AT_LEAST_ONCE)` registered on a channel. Verify end-to-end: dispatch → post-commit signal → pump delivers → cursor advances.

- [ ] **Step 1: Write integration tests**

Test cases:
- `dispatch_toTrackedBackend_deliveredByPump` — dispatch message, Awaitility wait, verify RecordingChannelBackend received it
- `dispatch_cursorAdvancedAfterDelivery` — verify cursor store has correct lastDeliveredId
- `dispatch_backendFailsThenRecovers_reconcilerCatchesUp` — first post throws, wait for reconciler cycle, verify delivery on retry
- `dispatch_channelDeleted_cursorsCleanedByCascade` — create channel + cursor, delete channel, verify cursor gone
- `dispatch_multipleMessages_deliveredInOrder` — 3 messages dispatched, verify order matches message IDs

Key patterns:
- Use `QuarkusTransaction.requiringNew().run(() -> ...)` for entity setup (committed before dispatch)
- Use `QuarkusTransaction.requiringNew().run(() -> messageService.dispatch(...))` for dispatch (transaction commits, post-commit signal fires)
- Use Awaitility with 5-second timeout to wait for async pump delivery
- Register `RecordingChannelBackend` via `gateway.registerBackend()` in `@BeforeEach`
- Use unique channel names per test to avoid RateLimiter cross-test interference

- [ ] **Step 2: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryServiceIntegrationTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: BUILD SUCCESS — all modules, all tests

- [ ] **Step 4: Commit**

```
feat(#132): integration tests — end-to-end delivery pump verification

Refs #132
```

---

### Task 7: Documentation and CLAUDE.md sync

**Files:**
- Modify: `CLAUDE.md` — add DeliveryService conventions, testing notes, config properties

**Interfaces:**
- Consumes: All prior tasks

- [ ] **Step 1: Update CLAUDE.md**

Add to the testing conventions section:
- `DeliveryService` uses `ManagedExecutor` — tests can mock it for synchronous execution
- `DeliveryConfig` defaults: enabled=true, batch-size=100, max-consecutive-failures=10, reconciliation-interval=30s
- `DeliveryCursor` entity is in `io.casehub.qhorus.runtime.gateway` — included in qhorus PU packages
- Flyway V25 = delivery_cursor table
- `RecordingChannelBackend` 3-arg constructor for AT_LEAST_ONCE testing
- Integration tests for the pump must use `QuarkusTransaction.requiringNew()` for dispatch (post-commit signal fires after commit)
- `fanOut()` returns `boolean hasTracked` — callers that previously called `fanOut()` as void must be updated if they exist in consumer repos

Add to the project structure section:
- `runtime/gateway/DeliveryService.java` — event-driven delivery pump for AT_LEAST_ONCE backends
- `runtime/gateway/DeliverySignalQueue.java` — signal queue mediator (breaks Gateway↔Service cycle)
- `runtime/gateway/DeliveryCursor.java` — per-backend-per-channel delivery cursor entity
- `runtime/config/DeliveryConfig.java` — `casehub.qhorus.delivery.*` config

- [ ] **Step 2: Commit**

```
docs(#132): update CLAUDE.md with delivery guarantee conventions

Refs #132
```

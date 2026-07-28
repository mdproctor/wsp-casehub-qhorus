# Notification Bridge → Subscription Engine Migration

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #375 — refactor: migrate notification bridge to platform SubscribableEvent + subscription engine
**Issue group:** #375

**Goal:** Replace the notification bridge's direct `NotificationStore.store()` calls with event dispatch into the platform subscription engine, enabling user-configurable subscriptions, suppression, and multi-channel delivery.

**Architecture:** The bridge becomes a thin event adapter. `NotificationBridgeObserver` and `CommitmentEventNotifier` create `QhorusObligationEvent` POJOs (implementing `SubscribableEvent`) and push them into the platform's notification DataSource via `DataSourceRegistry.resolveSource()`. The subscription engine's alpha network matches events against subscriptions. `NotificationDispatcher` handles target resolution, suppression, template rendering, and delivery. A `QhorusSubscriptionBootstrap` registers default SYSTEM-scope subscriptions at startup for out-of-the-box behavior parity.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api (SubscribableEvent, DataSourceRegistry, SubscriptionStore), CDI events, Mockito (CDI-free unit tests)

## Global Constraints

- Pre-release platform — breaking changes are acceptable
- `notification-bridge/` has no JPA entities, no Flyway, no `@QuarkusTest` — tests are CDI-free with Mockito
- Module depends on `casehub-qhorus-api` and `casehub-platform-api` only
- `casehub-platform-api` already provides `DataSourceRegistry`, `SubscriptionStore`, `SubscribableEvent`, `SubscriptionInput`, etc.
- IntelliJ MCP required for all source file operations
- Workspace: `/Users/mdproctor/claude/casehub/qhorus` + `/Users/mdproctor/claude/casehub/platform` (already open)

---

### Task 1: QhorusObligationEvent — the event POJO

**Files:**
- Create: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusObligationEvent.java`
- Test: `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/QhorusObligationEventTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.subscription.SubscribableEvent` (platform SPI)
- Produces: `QhorusObligationEvent` record — used by Tasks 2, 3, and 4

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.qhorus.notification.bridge;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class QhorusObligationEventTest {

    private static final UUID CHANNEL_ID = UUID.randomUUID();

    @ParameterizedTest
    @EnumSource(QhorusObligationEvent.Kind.class)
    void type_returns_prefixed_kind(QhorusObligationEvent.Kind kind) {
        var event = new QhorusObligationEvent(
                kind, "tenant-1", "obligor-1", "requester-1",
                CHANNEL_ID, "test-channel", "sender-1",
                UUID.randomUUID().toString(), "content");

        assertThat(event.type()).isEqualTo("io.casehub.qhorus.obligation." + kind.name().toLowerCase());
    }

    @Test
    void tenancyId_returned_correctly() {
        var event = new QhorusObligationEvent(
                QhorusObligationEvent.Kind.ASSIGNED, "my-tenant", "o", "r",
                CHANNEL_ID, "ch", "s", "corr", null);

        assertThat(event.tenancyId()).isEqualTo("my-tenant");
    }

    @Test
    void content_is_nullable() {
        var event = new QhorusObligationEvent(
                QhorusObligationEvent.Kind.FULFILLED, "t", "o", "r",
                CHANNEL_ID, "ch", "s", "corr", null);

        assertThat(event.content()).isNull();
    }

    @Test
    void null_kind_rejected() {
        assertThatThrownBy(() -> new QhorusObligationEvent(
                null, "t", "o", "r", CHANNEL_ID, "ch", "s", "corr", null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void null_tenancyId_rejected() {
        assertThatThrownBy(() -> new QhorusObligationEvent(
                QhorusObligationEvent.Kind.ASSIGNED, null, "o", "r",
                CHANNEL_ID, "ch", "s", "corr", null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void implements_subscribable_event() {
        var event = new QhorusObligationEvent(
                QhorusObligationEvent.Kind.FAILED, "t", "o", "r",
                CHANNEL_ID, "ch", "s", "corr", "body");

        assertThat(event).isInstanceOf(io.casehub.platform.api.subscription.SubscribableEvent.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusObligationEventTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — `QhorusObligationEvent` does not exist

- [ ] **Step 3: Write minimal implementation**

Use `ide_create_file` to create:

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.subscription.SubscribableEvent;

import java.util.Objects;
import java.util.UUID;

public record QhorusObligationEvent(
        Kind kind,
        String tenancyId,
        String obligor,
        String requester,
        UUID channelId,
        String channelName,
        String senderId,
        String correlationId,
        String content
) implements SubscribableEvent {

    public enum Kind {
        ASSIGNED, FULFILLED, FAILED, DECLINED, EXPIRED
    }

    private static final String TYPE_PREFIX = "io.casehub.qhorus.obligation.";

    public QhorusObligationEvent {
        Objects.requireNonNull(kind, "kind");
        Objects.requireNonNull(tenancyId, "tenancyId");
    }

    @Override
    public String type() {
        return TYPE_PREFIX + kind.name().toLowerCase();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusObligationEventTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusObligationEvent.java notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/QhorusObligationEventTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#375): add QhorusObligationEvent — SubscribableEvent POJO for obligation lifecycle"
```

---

### Task 2: Migrate NotificationBridgeObserver to fire events into the DataSource

**Files:**
- Modify: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/NotificationBridgeObserver.java`
- Modify: `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/NotificationBridgeObserverTest.java`

**Interfaces:**
- Consumes: `QhorusObligationEvent` (Task 1), `DataSourceRegistry` (platform), `DataSource.add()` (platform)
- Produces: `NotificationBridgeObserver` — constructor takes `(CommitmentStore, DataSourceRegistry)` instead of `(CommitmentStore, NotificationStore)`

- [ ] **Step 1: Rewrite the test to verify events pushed to DataSource**

Replace the entire test class. The mock target changes from `NotificationStore` to `DataSource<Object>`. `DataSourceRegistry.resolveSource()` returns the mocked DataSource. Assertions capture `QhorusObligationEvent` instead of `NotificationInput`.

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.CommitmentStore;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;
import static io.casehub.platform.api.tenancy.TenancyConstants.PLATFORM_TENANT_ID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class NotificationBridgeObserverTest {

    private CommitmentStore commitmentStore;
    @SuppressWarnings("unchecked")
    private final DataSource<Object> dataSource = mock(DataSource.class);
    private NotificationBridgeObserver observer;

    private static final UUID CHANNEL_ID = UUID.randomUUID();
    private static final String CHANNEL_NAME = "test-channel";
    private static final String TENANCY_ID = "tenant-1";
    private static final String CORRELATION_ID = UUID.randomUUID().toString();
    private static final String REQUESTER = "agent-requester";
    private static final String OBLIGOR = "agent-obligor";

    @BeforeEach
    void setUp() {
        commitmentStore = mock(CommitmentStore.class);
        var registry = mock(DataSourceRegistry.class);
        when(registry.resolveSource(NOTIFICATION_DATASOURCE_PATH, PLATFORM_TENANT_ID))
                .thenReturn(Optional.of(dataSource));
        observer = new NotificationBridgeObserver(commitmentStore, registry);
    }

    @Test
    void command_with_obligor_fires_assigned_event() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, OBLIGOR)));

        observer.onMessage(event(MessageType.COMMAND, REQUESTER, "Do this task"));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.kind()).isEqualTo(QhorusObligationEvent.Kind.ASSIGNED);
        assertThat(fired.tenancyId()).isEqualTo(TENANCY_ID);
        assertThat(fired.obligor()).isEqualTo(OBLIGOR);
        assertThat(fired.requester()).isEqualTo(REQUESTER);
        assertThat(fired.channelId()).isEqualTo(CHANNEL_ID);
        assertThat(fired.channelName()).isEqualTo(CHANNEL_NAME);
        assertThat(fired.senderId()).isEqualTo(REQUESTER);
        assertThat(fired.correlationId()).isEqualTo(CORRELATION_ID);
        assertThat(fired.content()).isEqualTo("Do this task");
    }

    @Test
    void command_without_commitment_skips() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.empty());

        observer.onMessage(event(MessageType.COMMAND, REQUESTER, "Do this"));

        verify(dataSource, never()).add(any());
    }

    @Test
    void command_with_blank_obligor_skips() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, "")));

        observer.onMessage(event(MessageType.COMMAND, REQUESTER, "Do this"));

        verify(dataSource, never()).add(any());
    }

    @Test
    void done_fires_fulfilled_event_for_requester() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, OBLIGOR)));

        observer.onMessage(event(MessageType.DONE, OBLIGOR, "Task complete"));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.kind()).isEqualTo(QhorusObligationEvent.Kind.FULFILLED);
        assertThat(fired.requester()).isEqualTo(REQUESTER);
        assertThat(fired.obligor()).isEqualTo(OBLIGOR);
    }

    @Test
    void failure_fires_failed_event() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, OBLIGOR)));

        observer.onMessage(event(MessageType.FAILURE, OBLIGOR, "Could not complete"));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.kind()).isEqualTo(QhorusObligationEvent.Kind.FAILED);
    }

    @Test
    void done_from_requester_to_self_skips() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, OBLIGOR)));

        observer.onMessage(event(MessageType.DONE, REQUESTER, "Self-resolved"));

        verify(dataSource, never()).add(any());
    }

    @Test
    void status_message_skips() {
        observer.onMessage(event(MessageType.STATUS, OBLIGOR, "Progress update"));

        verify(commitmentStore, never()).findByCorrelationId(any());
        verify(dataSource, never()).add(any());
    }

    @Test
    void null_correlationId_skips_all_processing() {
        var evt = new MessageReceivedEvent(
                1L, CHANNEL_NAME, CHANNEL_ID, TENANCY_ID,
                MessageType.COMMAND, REQUESTER, null,
                Instant.now(), "Do this", null);

        observer.onMessage(evt);

        verify(commitmentStore, never()).findByCorrelationId(any());
        verify(dataSource, never()).add(any());
    }

    @Test
    void scope_is_local() {
        assertThat(observer.scope()).isEqualTo(MessageObserver.Scope.LOCAL);
    }

    @Test
    void datasource_add_failure_is_non_fatal() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, OBLIGOR)));
        doThrow(new RuntimeException("DS down")).when(dataSource).add(any());

        observer.onMessage(event(MessageType.COMMAND, REQUESTER, "Do this"));
        // no exception propagated
    }

    @Test
    void long_content_is_truncated() {
        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, OBLIGOR)));

        observer.onMessage(event(MessageType.COMMAND, REQUESTER, "a".repeat(300)));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.content()).hasSize(200);
    }

    @Test
    void datasource_not_registered_skips_silently() {
        var emptyRegistry = mock(DataSourceRegistry.class);
        when(emptyRegistry.resolveSource(any(), any())).thenReturn(Optional.empty());
        var obs = new NotificationBridgeObserver(commitmentStore, emptyRegistry);

        when(commitmentStore.findByCorrelationId(CORRELATION_ID))
                .thenReturn(Optional.of(commitment(REQUESTER, OBLIGOR)));

        obs.onMessage(event(MessageType.COMMAND, REQUESTER, "Do this"));
        // no exception, no add
    }

    private MessageReceivedEvent event(MessageType type, String sender, String content) {
        return new MessageReceivedEvent(
                1L, CHANNEL_NAME, CHANNEL_ID, TENANCY_ID,
                type, sender, CORRELATION_ID,
                Instant.now(), content, null);
    }

    private Commitment commitment(String requester, String obligor) {
        return Commitment.builder()
                .id(UUID.randomUUID())
                .correlationId(CORRELATION_ID)
                .channelId(CHANNEL_ID)
                .messageType(MessageType.COMMAND)
                .requester(requester)
                .obligor(obligor)
                .state(CommitmentState.OPEN)
                .tenancyId(TENANCY_ID)
                .createdAt(Instant.now())
                .build();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=NotificationBridgeObserverTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — constructor signature mismatch (still takes `NotificationStore`)

- [ ] **Step 3: Rewrite the implementation**

Replace `NotificationBridgeObserver.java` entirely via `ide_edit_member` (class-level replacement):

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.store.CommitmentStore;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import java.util.Optional;

import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;
import static io.casehub.platform.api.tenancy.TenancyConstants.PLATFORM_TENANT_ID;

@ApplicationScoped
public class NotificationBridgeObserver implements MessageObserver {

    private static final Logger LOG = Logger.getLogger(NotificationBridgeObserver.class);
    private static final int MAX_CONTENT_LENGTH = 200;

    private final CommitmentStore commitmentStore;
    private final DataSourceRegistry dataSourceRegistry;

    @Inject
    public NotificationBridgeObserver(CommitmentStore commitmentStore,
                                      DataSourceRegistry dataSourceRegistry) {
        this.commitmentStore = commitmentStore;
        this.dataSourceRegistry = dataSourceRegistry;
    }

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (event.correlationId() == null) {
            return;
        }
        switch (event.messageType()) {
            case COMMAND -> fireAssigned(event);
            case DONE    -> fireResolved(event, QhorusObligationEvent.Kind.FULFILLED);
            case FAILURE -> fireResolved(event, QhorusObligationEvent.Kind.FAILED);
            default -> { }
        }
    }

    @Override
    public Scope scope() {
        return Scope.LOCAL;
    }

    private void fireAssigned(MessageReceivedEvent event) {
        Optional<Commitment> commitment = commitmentStore.findByCorrelationId(event.correlationId());
        if (commitment.isEmpty()) {
            LOG.debugf("No commitment for correlationId=%s — skipping COMMAND notification", event.correlationId());
            return;
        }
        String obligor = commitment.get().obligor();
        if (obligor == null || obligor.isBlank()) {
            return;
        }
        fire(new QhorusObligationEvent(
                QhorusObligationEvent.Kind.ASSIGNED,
                event.tenancyId(),
                obligor,
                commitment.get().requester(),
                event.channelId(),
                event.channelName(),
                event.senderId(),
                event.correlationId(),
                truncate(event.content(), MAX_CONTENT_LENGTH)));
    }

    private void fireResolved(MessageReceivedEvent event, QhorusObligationEvent.Kind kind) {
        Optional<Commitment> commitment = commitmentStore.findByCorrelationId(event.correlationId());
        if (commitment.isEmpty()) {
            return;
        }
        String requester = commitment.get().requester();
        if (requester == null || requester.isBlank()) {
            return;
        }
        if (requester.equals(event.senderId())) {
            return;
        }
        fire(new QhorusObligationEvent(
                kind,
                event.tenancyId(),
                commitment.get().obligor(),
                requester,
                event.channelId(),
                event.channelName(),
                event.senderId(),
                event.correlationId(),
                truncate(event.content(), MAX_CONTENT_LENGTH)));
    }

    private void fire(QhorusObligationEvent event) {
        try {
            Optional<DataSource<?>> ds = dataSourceRegistry.resolveSource(
                    NOTIFICATION_DATASOURCE_PATH, PLATFORM_TENANT_ID);
            if (ds.isEmpty()) {
                LOG.warnf("Notification DataSource not available — dropping %s event", event.kind());
                return;
            }
            @SuppressWarnings("unchecked")
            DataSource<Object> source = (DataSource<Object>) ds.get();
            source.add(event);
        } catch (Exception e) {
            LOG.warnf("Failed to fire obligation event %s: %s", event.kind(), e.getMessage());
        }
    }

    static String truncate(String s, int max) {
        if (s == null) return null;
        return s.length() <= max ? s : s.substring(0, max - 1) + "…";
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=NotificationBridgeObserverTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS — all 11 tests green

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/NotificationBridgeObserver.java notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/NotificationBridgeObserverTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#375): NotificationBridgeObserver fires SubscribableEvent into DataSource"
```

---

### Task 3: Migrate CommitmentEventNotifier to fire events into the DataSource

**Files:**
- Modify: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/CommitmentEventNotifier.java`
- Modify: `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/CommitmentEventNotifierTest.java`

**Interfaces:**
- Consumes: `QhorusObligationEvent` (Task 1), `DataSourceRegistry` (platform)
- Produces: `CommitmentEventNotifier` — constructor takes `(CommitmentStore, DataSourceRegistry)` instead of `(CommitmentStore, NotificationStore)`

- [ ] **Step 1: Rewrite the test**

Replace the entire test class. Same pattern as Task 2 — mock `DataSource<Object>`, capture `QhorusObligationEvent`.

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.CommitmentDeclinedEvent;
import io.casehub.qhorus.api.message.CommitmentExpiredEvent;
import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.CommitmentStore;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;
import static io.casehub.platform.api.tenancy.TenancyConstants.PLATFORM_TENANT_ID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class CommitmentEventNotifierTest {

    private CommitmentStore commitmentStore;
    @SuppressWarnings("unchecked")
    private final DataSource<Object> dataSource = mock(DataSource.class);
    private CommitmentEventNotifier notifier;

    private static final UUID COMMITMENT_ID = UUID.randomUUID();
    private static final UUID CHANNEL_ID = UUID.randomUUID();
    private static final String CORRELATION_ID = UUID.randomUUID().toString();
    private static final String TENANCY_ID = "tenant-1";
    private static final String REQUESTER = "agent-requester";
    private static final String OBLIGOR = "agent-obligor";

    @BeforeEach
    void setUp() {
        commitmentStore = mock(CommitmentStore.class);
        var registry = mock(DataSourceRegistry.class);
        when(registry.resolveSource(NOTIFICATION_DATASOURCE_PATH, PLATFORM_TENANT_ID))
                .thenReturn(Optional.of(dataSource));
        notifier = new CommitmentEventNotifier(commitmentStore, registry);

        when(commitmentStore.findById(COMMITMENT_ID))
                .thenReturn(Optional.of(commitment()));
    }

    @Test
    void declined_fires_declined_event_for_requester() {
        notifier.onDeclined(new CommitmentDeclinedEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, OBLIGOR, REQUESTER));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.kind()).isEqualTo(QhorusObligationEvent.Kind.DECLINED);
        assertThat(fired.tenancyId()).isEqualTo(TENANCY_ID);
        assertThat(fired.requester()).isEqualTo(REQUESTER);
        assertThat(fired.obligor()).isEqualTo(OBLIGOR);
        assertThat(fired.channelId()).isEqualTo(CHANNEL_ID);
        assertThat(fired.correlationId()).isEqualTo(CORRELATION_ID);
    }

    @Test
    void declined_with_null_requester_skips() {
        notifier.onDeclined(new CommitmentDeclinedEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, OBLIGOR, null));

        verify(dataSource, never()).add(any());
    }

    @Test
    void declined_with_blank_requester_skips() {
        notifier.onDeclined(new CommitmentDeclinedEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, OBLIGOR, ""));

        verify(dataSource, never()).add(any());
    }

    @Test
    void expired_fires_expired_event() {
        notifier.onExpired(new CommitmentExpiredEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, OBLIGOR, REQUESTER,
                Instant.now().minusSeconds(60)));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.kind()).isEqualTo(QhorusObligationEvent.Kind.EXPIRED);
        assertThat(fired.requester()).isEqualTo(REQUESTER);
        assertThat(fired.obligor()).isEqualTo(OBLIGOR);
    }

    @Test
    void expired_with_null_obligor_preserves_null() {
        notifier.onExpired(new CommitmentExpiredEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, null, REQUESTER,
                Instant.now()));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.obligor()).isNull();
    }

    @Test
    void expired_with_null_requester_skips() {
        notifier.onExpired(new CommitmentExpiredEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, OBLIGOR, null,
                Instant.now()));

        verify(dataSource, never()).add(any());
    }

    @Test
    void declined_uses_default_tenancy_when_commitment_not_found() {
        when(commitmentStore.findById(COMMITMENT_ID)).thenReturn(Optional.empty());

        notifier.onDeclined(new CommitmentDeclinedEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, OBLIGOR, REQUESTER));

        var captor = ArgumentCaptor.forClass(Object.class);
        verify(dataSource).add(captor.capture());
        var fired = (QhorusObligationEvent) captor.getValue();

        assertThat(fired.tenancyId()).isEqualTo("DEFAULT");
    }

    @Test
    void datasource_failure_is_non_fatal() {
        doThrow(new RuntimeException("DS down")).when(dataSource).add(any());

        notifier.onDeclined(new CommitmentDeclinedEvent(
                COMMITMENT_ID, CORRELATION_ID, CHANNEL_ID, OBLIGOR, REQUESTER));
        // no exception propagated
    }

    private Commitment commitment() {
        return Commitment.builder()
                .id(COMMITMENT_ID)
                .correlationId(CORRELATION_ID)
                .channelId(CHANNEL_ID)
                .messageType(MessageType.COMMAND)
                .requester(REQUESTER)
                .obligor(OBLIGOR)
                .state(CommitmentState.DECLINED)
                .tenancyId(TENANCY_ID)
                .createdAt(Instant.now())
                .build();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentEventNotifierTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — constructor signature mismatch

- [ ] **Step 3: Rewrite the implementation**

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.CommitmentDeclinedEvent;
import io.casehub.qhorus.api.message.CommitmentExpiredEvent;
import io.casehub.qhorus.api.store.CommitmentStore;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import java.util.Optional;

import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;
import static io.casehub.platform.api.tenancy.TenancyConstants.PLATFORM_TENANT_ID;

@ApplicationScoped
public class CommitmentEventNotifier {

    private static final Logger LOG = Logger.getLogger(CommitmentEventNotifier.class);

    private final CommitmentStore commitmentStore;
    private final DataSourceRegistry dataSourceRegistry;

    @Inject
    public CommitmentEventNotifier(CommitmentStore commitmentStore,
                                    DataSourceRegistry dataSourceRegistry) {
        this.commitmentStore = commitmentStore;
        this.dataSourceRegistry = dataSourceRegistry;
    }

    void onDeclined(@ObservesAsync CommitmentDeclinedEvent event) {
        String requester = event.requester();
        if (requester == null || requester.isBlank()) {
            return;
        }
        Optional<Commitment> commitment = commitmentStore.findById(event.commitmentId());
        String tenancyId = commitment.map(Commitment::tenancyId).orElse("DEFAULT");

        fire(new QhorusObligationEvent(
                QhorusObligationEvent.Kind.DECLINED,
                tenancyId,
                event.obligor(),
                requester,
                event.channelId(),
                null,
                event.obligor(),
                event.correlationId(),
                null));
    }

    void onExpired(@ObservesAsync CommitmentExpiredEvent event) {
        String requester = event.requester();
        if (requester == null || requester.isBlank()) {
            return;
        }
        Optional<Commitment> commitment = commitmentStore.findById(event.commitmentId());
        String tenancyId = commitment.map(Commitment::tenancyId).orElse("DEFAULT");

        fire(new QhorusObligationEvent(
                QhorusObligationEvent.Kind.EXPIRED,
                tenancyId,
                event.obligor(),
                requester,
                event.channelId(),
                null,
                event.obligor() != null ? event.obligor() : "unassigned",
                event.correlationId(),
                null));
    }

    private void fire(QhorusObligationEvent event) {
        try {
            Optional<DataSource<?>> ds = dataSourceRegistry.resolveSource(
                    NOTIFICATION_DATASOURCE_PATH, PLATFORM_TENANT_ID);
            if (ds.isEmpty()) {
                LOG.warnf("Notification DataSource not available — dropping %s event", event.kind());
                return;
            }
            @SuppressWarnings("unchecked")
            DataSource<Object> source = (DataSource<Object>) ds.get();
            source.add(event);
        } catch (Exception e) {
            LOG.warnf("Failed to fire obligation event %s: %s", event.kind(), e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentEventNotifierTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS — all 8 tests green

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/CommitmentEventNotifier.java notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/CommitmentEventNotifierTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#375): CommitmentEventNotifier fires SubscribableEvent into DataSource"
```

---

### Task 4: Delete NotificationCategories and remove NotificationStore dependency

**Files:**
- Delete: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/NotificationCategories.java`
- Modify: `notification-bridge/pom.xml` (no file changes needed — `casehub-platform-api` already provides the new types; but verify `NotificationStore` is no longer imported anywhere)

**Interfaces:**
- Consumes: nothing
- Produces: clean module with no dead code

- [ ] **Step 1: Verify no remaining references to NotificationCategories**

Use `ide_find_references` on `NotificationCategories` to confirm no remaining usages after Tasks 2 and 3.

- [ ] **Step 2: Verify no remaining references to NotificationStore**

Use `ide_search_text` for `NotificationStore` in `notification-bridge/` — should find zero matches.

- [ ] **Step 3: Delete NotificationCategories.java**

Use `ide_refactor_safe_delete` on `NotificationCategories`.

- [ ] **Step 4: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS — all tests green, no compilation errors

- [ ] **Step 5: Verify full project compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS — no module depends on `NotificationCategories`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A notification-bridge/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#375): delete NotificationCategories — replaced by QhorusObligationEvent.type()"
```

---

### Task 5: QhorusSubscriptionBootstrap — register default subscriptions at startup

**Files:**
- Create: `notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusSubscriptionBootstrap.java`
- Test: `notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/QhorusSubscriptionBootstrapTest.java`

**Interfaces:**
- Consumes: `SubscriptionStore` (platform), `SubscriptionInput` (platform), `NotificationTemplate` (platform), `NotificationTarget` (platform)
- Produces: `QhorusSubscriptionBootstrap` — `@ApplicationScoped`, registers 5 default SYSTEM subscriptions at startup

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.subscription.*;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.List;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class QhorusSubscriptionBootstrapTest {

    private SubscriptionStore subscriptionStore;
    private QhorusSubscriptionBootstrap bootstrap;

    @BeforeEach
    void setUp() {
        subscriptionStore = mock(SubscriptionStore.class);
        when(subscriptionStore.findAllEnabled()).thenReturn(Stream.empty());
        bootstrap = new QhorusSubscriptionBootstrap(subscriptionStore);
    }

    @Test
    void registers_five_default_subscriptions_when_none_exist() {
        bootstrap.onStartup(null);

        verify(subscriptionStore, times(5)).store(any(SubscriptionInput.class));
    }

    @Test
    void skips_registration_when_subscriptions_already_exist() {
        var existing = new Subscription(
                "id-1", "system:qhorus", "PLATFORM",
                "qhorus.obligation.assigned",
                "io.casehub.qhorus.obligation.assigned",
                List.of(), List.of(), false,
                new NotificationTemplate("t", null, NotificationSeverity.INFO,
                        "qhorus.obligation.assigned", null, "channel", "channelId", "senderId"),
                true, SubscriptionScope.SYSTEM, Instant.now(), Instant.now());

        when(subscriptionStore.findAllEnabled()).thenReturn(Stream.of(existing));
        bootstrap.onStartup(null);

        // only 4 created — assigned already exists
        verify(subscriptionStore, times(4)).store(any(SubscriptionInput.class));
    }

    @Test
    void assigned_subscription_targets_obligor_field() {
        bootstrap.onStartup(null);

        var captor = ArgumentCaptor.forClass(SubscriptionInput.class);
        verify(subscriptionStore, times(5)).store(captor.capture());

        var assigned = captor.getAllValues().stream()
                .filter(i -> i.eventType().endsWith(".assigned"))
                .findFirst().orElseThrow();

        assertThat(assigned.targets()).hasSize(1);
        assertThat(assigned.targets().get(0).type()).isEqualTo(TargetType.EVENT_FIELD);
        assertThat(assigned.targets().get(0).id()).isEqualTo("obligor");
        assertThat(assigned.scope()).isEqualTo(SubscriptionScope.SYSTEM);
    }

    @Test
    void fulfilled_subscription_targets_requester_field() {
        bootstrap.onStartup(null);

        var captor = ArgumentCaptor.forClass(SubscriptionInput.class);
        verify(subscriptionStore, times(5)).store(captor.capture());

        var fulfilled = captor.getAllValues().stream()
                .filter(i -> i.eventType().endsWith(".fulfilled"))
                .findFirst().orElseThrow();

        assertThat(fulfilled.targets().get(0).id()).isEqualTo("requester");
        assertThat(fulfilled.template().severity()).isEqualTo(NotificationSeverity.INFO);
    }

    @Test
    void failed_subscription_has_warning_severity() {
        bootstrap.onStartup(null);

        var captor = ArgumentCaptor.forClass(SubscriptionInput.class);
        verify(subscriptionStore, times(5)).store(captor.capture());

        var failed = captor.getAllValues().stream()
                .filter(i -> i.eventType().endsWith(".failed"))
                .findFirst().orElseThrow();

        assertThat(failed.template().severity()).isEqualTo(NotificationSeverity.WARNING);
    }

    @Test
    void store_failure_is_non_fatal() {
        doThrow(new RuntimeException("DB down")).when(subscriptionStore).store(any());

        bootstrap.onStartup(null);
        // no exception propagated
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusSubscriptionBootstrapTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — `QhorusSubscriptionBootstrap` does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.qhorus.notification.bridge;

import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.subscription.*;

import io.quarkus.runtime.StartupEvent;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

import static io.casehub.platform.api.tenancy.TenancyConstants.PLATFORM_TENANT_ID;

@ApplicationScoped
public class QhorusSubscriptionBootstrap {

    private static final Logger LOG = Logger.getLogger(QhorusSubscriptionBootstrap.class);
    private static final String OWNER_ID = "system:qhorus";
    private static final String TYPE_PREFIX = "io.casehub.qhorus.obligation.";

    private final SubscriptionStore subscriptionStore;

    @Inject
    public QhorusSubscriptionBootstrap(SubscriptionStore subscriptionStore) {
        this.subscriptionStore = subscriptionStore;
    }

    void onStartup(@Observes StartupEvent event) {
        Set<String> existing = subscriptionStore.findAllEnabled()
                .filter(s -> s.eventType().startsWith(TYPE_PREFIX))
                .map(Subscription::eventType)
                .collect(Collectors.toSet());

        register(existing, "assigned", "obligor",
                "Obligation assigned in #{channelName}", NotificationSeverity.INFO);
        register(existing, "fulfilled", "requester",
                "Request completed in #{channelName}", NotificationSeverity.INFO);
        register(existing, "failed", "requester",
                "Request failed in #{channelName}", NotificationSeverity.WARNING);
        register(existing, "declined", "requester",
                "Request declined in #{channelName}", NotificationSeverity.WARNING);
        register(existing, "expired", "requester",
                "Request expired in #{channelName}", NotificationSeverity.URGENT);

        LOG.infof("Qhorus subscription bootstrap complete — %d event types active",
                existing.size() + 5 - existing.size());
    }

    private void register(Set<String> existing, String kind, String targetField,
                          String titlePattern, NotificationSeverity severity) {
        String eventType = TYPE_PREFIX + kind;
        if (existing.contains(eventType)) {
            LOG.debugf("Subscription for %s already exists — skipping", eventType);
            return;
        }
        try {
            subscriptionStore.store(new SubscriptionInput(
                    OWNER_ID,
                    PLATFORM_TENANT_ID,
                    "qhorus.obligation." + kind,
                    eventType,
                    List.of(),
                    List.of(new NotificationTarget(TargetType.EVENT_FIELD, targetField)),
                    false,
                    new NotificationTemplate(
                            titlePattern,
                            "{content}",
                            severity,
                            "qhorus.obligation." + kind,
                            null,
                            "channel",
                            "channelId",
                            "senderId"),
                    true,
                    SubscriptionScope.SYSTEM));
            LOG.infof("Registered default subscription for %s", eventType);
        } catch (Exception e) {
            LOG.warnf("Failed to register default subscription for %s: %s", eventType, e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusSubscriptionBootstrapTest -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notification-bridge -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS — all tests across the module green

- [ ] **Step 6: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS — all modules compile and tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add notification-bridge/src/main/java/io/casehub/qhorus/notification/bridge/QhorusSubscriptionBootstrap.java notification-bridge/src/test/java/io/casehub/qhorus/notification/bridge/QhorusSubscriptionBootstrapTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#375): QhorusSubscriptionBootstrap — register default obligation subscriptions at startup"
```

---

### Task 6: Update CLAUDE.md notification-bridge documentation

**Files:**
- Modify: `CLAUDE.md` — update the `notification-bridge/` section

**Interfaces:**
- Consumes: nothing
- Produces: accurate documentation

- [ ] **Step 1: Update the CLAUDE.md notification-bridge section**

Replace the current `notification-bridge/` documentation block with the new architecture:
- `QhorusObligationEvent` — `SubscribableEvent` POJO with Kind enum
- `NotificationBridgeObserver` — fires events into platform notification DataSource (not `NotificationStore`)
- `CommitmentEventNotifier` — CDI observer, fires events into DataSource
- `QhorusSubscriptionBootstrap` — registers 5 default SYSTEM subscriptions at startup
- Note that `NotificationCategories` was deleted (replaced by `QhorusObligationEvent.type()`)
- Note the dependency shift: `NotificationStore` → `DataSourceRegistry` + `SubscriptionStore`

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#375): update CLAUDE.md — notification bridge → subscription engine migration"
```

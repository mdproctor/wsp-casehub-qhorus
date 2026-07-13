# CLUSTER-Scoped MessageObserver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #163 — CLUSTER-scoped MessageObserver implementations — Kafka, WebSocket, Webhook
**Issue group:** #163

**Goal:** Implement CLUSTER scope dispatch semantics for MessageObserver, then build three transport modules (Kafka, WebSocket, Webhook) as optional observer implementations.

**Architecture:** Give `Scope.CLUSTER` real dispatch semantics — LOCAL fires on the dispatching node only, CLUSTER fires on all nodes. Enhance `deliverRemote()` to fire CLUSTER observers via a new `MessageService.dispatchClusterObservers()` method. Fix the LAST_WRITE overwrite path to fire observers (currently excluded, creating node-asymmetric behavior). Build three optional Maven modules: `kafka-observer`, `websocket-observer`, `webhook-observer`, each activated by classpath presence.

**Tech Stack:** Java 21, Quarkus 3.32.2, SmallRye Reactive Messaging (Kafka), Quarkus WebSockets Next, Vert.x WebClient, CloudEvents

**Spec:** `docs/specs/2026-07-13-cluster-scoped-observer-design.md`

## Global Constraints

- Java 21 source, Java 26 JVM
- Quarkus 3.32.2, quarkus-mcp-server 1.11.1
- Parent groupId: `io.casehub`, version: `0.2-SNAPSHOT`
- New modules follow `postgres-broadcaster/` pattern: separate optional Maven module, jandex plugin, activated by classpath presence
- Wire format: CloudEvents for all transports
- No Flyway migrations (all three modules are stateless)
- `MessageObserverDispatcher` stays package-private — external access via `MessageService`
- Tests must be CI-safe: no external infrastructure, mock transports

---

### Task 1: CloudEventMapper extraction

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/CloudEventMapper.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapter.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/CloudEventMapperTest.java`

**Interfaces:**
- Consumes: `MessageReceivedEvent` (api/gateway), `ObjectMapper` (Jackson)
- Produces: `CloudEventMapper.toCloudEvent(MessageReceivedEvent, ObjectMapper) → CloudEvent` — used by QhorusCloudEventAdapter, KafkaMessageObserver, WebSocketMessageObserver, WebhookMessageObserver

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.qhorus.runtime.gateway;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;
import io.cloudevents.CloudEvent;

class CloudEventMapperTest {

    private final ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());

    @Test
    void mapsMessageReceivedEventToCloudEvent() {
        UUID channelId = UUID.randomUUID();
        Instant now = Instant.now();
        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "tenant-1",
                MessageType.STATUS, "agent-1", "corr-1", now, "hello", "general");

        CloudEvent ce = CloudEventMapper.toCloudEvent(event, mapper);

        assertThat(ce.getType()).isEqualTo("io.casehub.qhorus.message.status");
        assertThat(ce.getSource().toString()).isEqualTo("/casehub-qhorus/channel/" + channelId);
        assertThat(ce.getSubject()).isEqualTo("channel/" + channelId);
        assertThat(ce.getTime()).isNotNull();
        assertThat(ce.getDataContentType()).isEqualTo("application/json");
        assertThat(ce.getData()).isNotNull();
        assertThat(ce.getExtension("tenancyid")).isEqualTo("tenant-1");
    }

    @Test
    void omitsTenancyExtensionWhenNull() {
        UUID channelId = UUID.randomUUID();
        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, null,
                MessageType.QUERY, "agent-1", null, Instant.now(), "q", null);

        CloudEvent ce = CloudEventMapper.toCloudEvent(event, mapper);

        assertThat(ce.getExtension("tenancyid")).isNull();
    }

    @Test
    void handlesEventTypeWithNullContent() {
        UUID channelId = UUID.randomUUID();
        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.EVENT, "agent-1", null, Instant.now(), null, null);

        CloudEvent ce = CloudEventMapper.toCloudEvent(event, mapper);

        assertThat(ce.getType()).isEqualTo("io.casehub.qhorus.message.event");
        assertThat(ce.getData()).isNotNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CloudEventMapperTest -pl runtime -Dno-format`
Expected: FAIL — `CloudEventMapper` class not found

- [ ] **Step 3: Implement CloudEventMapper**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/CloudEventMapper.java`:

```java
package io.casehub.qhorus.runtime.gateway;

import java.net.URI;
import java.time.ZoneOffset;
import java.util.Locale;
import java.util.UUID;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import org.jboss.logging.Logger;

import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;

public final class CloudEventMapper {

    private static final Logger LOG = Logger.getLogger(CloudEventMapper.class);

    private CloudEventMapper() {}

    public static CloudEvent toCloudEvent(MessageReceivedEvent event, ObjectMapper objectMapper) {
        String type = "io.casehub.qhorus.message." + event.messageType().name().toLowerCase(Locale.ROOT);
        URI source = URI.create("/casehub-qhorus/channel/" + event.channelId());

        byte[] data;
        try {
            data = objectMapper.writeValueAsBytes(event);
        } catch (JsonProcessingException e) {
            LOG.warnf(e, "Failed to serialise MessageReceivedEvent for CloudEvent — channel=%s type=%s",
                    event.channelId(), event.messageType());
            data = new byte[0];
        }

        CloudEventBuilder builder = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType(type)
                .withSource(source)
                .withSubject("channel/" + event.channelId())
                .withTime(event.occurredAt().atOffset(ZoneOffset.UTC))
                .withDataContentType("application/json")
                .withData(data);

        if (event.tenancyId() != null) {
            builder = builder.withExtension("tenancyid", event.tenancyId());
        }

        return builder.build();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CloudEventMapperTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 5: Refactor QhorusCloudEventAdapter to use CloudEventMapper**

Use `ide_replace_member` on `QhorusCloudEventAdapter.toCloudEvent` to delegate:

```java
private CloudEvent toCloudEvent(MessageReceivedEvent event) {
    return CloudEventMapper.toCloudEvent(event, objectMapper);
}
```

- [ ] **Step 6: Run full runtime tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/gateway/CloudEventMapper.java runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapter.java runtime/src/test/java/io/casehub/qhorus/runtime/gateway/CloudEventMapperTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor: extract CloudEventMapper from QhorusCloudEventAdapter

Shared utility for MessageReceivedEvent → CloudEvent mapping. Used by
the adapter and all three transport observer modules.

Refs #163"
```

---

### Task 2: CLUSTER scope dispatch + LAST_WRITE observer fix

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (~line 213 LAST_WRITE path + new `dispatchClusterObservers` method)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` (~line 356 LAST_WRITE path)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` (~line 394 `deliverRemote`)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcherTest.java` (existing, add scope tests)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/LastWriteObserverDispatchTest.java` (new)

**Interfaces:**
- Consumes: `MessageObserver.Scope`, `Instance<MessageObserver>`, `Message`
- Produces: `MessageService.dispatchClusterObservers(String channelName, UUID channelId, String tenancyId, Message message)` — called by ChannelGateway.deliverRemote()

- [ ] **Step 1: Write scope filtering tests in MessageObserverDispatcherTest**

Add tests to the existing test file. Use `ide_insert_member`:

```java
@Test
void dispatchClusterOnly_skipsLocalObserver() {
    var received = new java.util.ArrayList<MessageReceivedEvent>();
    MessageObserver local = event -> received.add(event);
    MessageObserver cluster = new MessageObserver() {
        @Override public void onMessage(MessageReceivedEvent event) { received.add(event); }
        @Override public Scope scope() { return Scope.CLUSTER; }
    };

    Message msg = buildTestMessage();
    List<Instance.Handle<MessageObserver>> handles = List.of(
            mockHandle(local), mockHandle(cluster));

    MessageObserverDispatcher.dispatchClusterOnly(
            "ch", UUID.randomUUID(), "t1", msg, handles);

    assertThat(received).hasSize(1);
}

@Test
void dispatchClusterOnly_appliesChannelFilter() {
    var received = new java.util.ArrayList<MessageReceivedEvent>();
    MessageObserver cluster = new MessageObserver() {
        @Override public void onMessage(MessageReceivedEvent event) { received.add(event); }
        @Override public Scope scope() { return Scope.CLUSTER; }
        @Override public java.util.Set<String> channels() { return java.util.Set.of("other-channel"); }
    };

    Message msg = buildTestMessage();
    List<Instance.Handle<MessageObserver>> handles = List.of(mockHandle(cluster));

    MessageObserverDispatcher.dispatchClusterOnly(
            "my-channel", UUID.randomUUID(), "t1", msg, handles);

    assertThat(received).isEmpty();
}
```

Note: `buildTestMessage()` and `mockHandle()` are helpers — check the existing test file for the patterns used. If not present, create them following the existing test style.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageObserverDispatcherTest -pl runtime -Dno-format`
Expected: FAIL — `dispatchClusterOnly` method not found

- [ ] **Step 3: Implement dispatchClusterOnly in MessageObserverDispatcher**

Use `ide_insert_member` to add to `MessageObserverDispatcher`:

```java
static void dispatchClusterOnly(final String channelName, final UUID channelId,
        final String tenancyId,
        final Message message,
        final Iterable<? extends Instance.Handle<MessageObserver>> handles) {
    final String content = message.messageType() == MessageType.EVENT
            ? null : message.content();
    final Instant occurredAt = message.createdAt() != null
            ? message.createdAt() : Instant.now();
    final MessageReceivedEvent event = new MessageReceivedEvent(
            channelName, channelId, tenancyId,
            message.messageType(), message.sender(),
            message.correlationId(), occurredAt, content,
            message.topic());

    final List<Instance.Handle<MessageObserver>> active = new ArrayList<>();
    for (final Instance.Handle<MessageObserver> handle : handles) {
        try {
            MessageObserver observer = handle.get();
            if (observer.scope() != MessageObserver.Scope.CLUSTER) {
                handle.close();
                continue;
            }
            final java.util.Set<String> filter = observer.channels();
            if (!filter.isEmpty() && !filter.contains(channelName)) {
                handle.close();
                continue;
            }
            active.add(handle);
        } catch (Exception e) {
            LOG.warnf("MessageObserver handle.get() failed for channel '%s': %s",
                    channelName, e.getMessage());
            handle.close();
        }
    }

    if (active.isEmpty()) {
        return;
    }

    dispatchToHandles(channelName, message.messageType(), event, active);
}
```

- [ ] **Step 4: Run scope filtering tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageObserverDispatcherTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 5: Add dispatchClusterObservers to MessageService**

Use `ide_insert_member` to add a public method to `MessageService`:

```java
public void dispatchClusterObservers(String channelName, UUID channelId,
                                      String tenancyId, Message message) {
    MessageObserverDispatcher.dispatchClusterOnly(
            channelName, channelId, tenancyId, message, observers.handles());
}
```

- [ ] **Step 6: Write LAST_WRITE observer dispatch test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/message/LastWriteObserverDispatchTest.java`:

```java
package io.casehub.qhorus.runtime.message;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.narayana.jta.QuarkusTransaction;

@QuarkusTest
class LastWriteObserverDispatchTest {

    @Inject
    MessageService messageService;

    @Inject
    io.casehub.qhorus.api.channel.ChannelManager channelManager;

    @Inject
    RecordingLastWriteObserver observer;

    @ApplicationScoped
    static class RecordingLastWriteObserver implements MessageObserver {
        final List<MessageReceivedEvent> received = new ArrayList<>();
        @Override public void onMessage(MessageReceivedEvent event) {
            received.add(event);
        }
    }

    @Test
    void lastWriteOverwriteFiresObserver() {
        observer.received.clear();
        String channelName = "lw-obs-" + UUID.randomUUID().toString().substring(0, 8);

        // Create LAST_WRITE channel and dispatch twice from the same sender
        QuarkusTransaction.requiringNew().run(() -> {
            var req = io.casehub.qhorus.runtime.channel.ChannelCreateRequest.builder(channelName)
                    .semantic(ChannelSemantic.LAST_WRITE).build();
            channelManager.create(req);
        });

        QuarkusTransaction.requiringNew().run(() -> {
            var ch = channelManager.findByName(channelName).orElseThrow();
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(ch.id()).sender("agent-1")
                    .type(MessageType.STATUS).content("first").build());
        });

        QuarkusTransaction.requiringNew().run(() -> {
            var ch = channelManager.findByName(channelName).orElseThrow();
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(ch.id()).sender("agent-1")
                    .type(MessageType.STATUS).content("overwrite").build());
        });

        // Both dispatches should fire observers — including the overwrite
        assertThat(observer.received).hasSizeGreaterThanOrEqualTo(2);
        assertThat(observer.received.stream()
                .anyMatch(e -> "overwrite".equals(e.content()))).isTrue();
    }
}
```

- [ ] **Step 7: Run test to verify it fails (LAST_WRITE overwrite currently skips observers)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LastWriteObserverDispatchTest -pl runtime -Dno-format`
Expected: FAIL — the overwrite dispatch does not fire observers

- [ ] **Step 8: Fix LAST_WRITE overwrite path in MessageService**

In `MessageService.java` (~line 213-262), the LAST_WRITE overwrite block calls `channelGateway.fanOut()` and `broadcaster.broadcast()` but skips `MessageObserverDispatcher.dispatch()`. Add observer dispatch after the `messageStore.put(updated)` call and before `channelGateway.fanOut()`:

```java
MessageObserverDispatcher.dispatch(
        ch.name(), ch.id(), ch.tenancyId(),
        saved, observers.handles(), tsr);
```

Remove the comment "LAST_WRITE overwrite path: fanOut + broadcast fire, but MessageObserverDispatcher is intentionally excluded — an overwrite is a content update, not a new message event."

- [ ] **Step 9: Fix LAST_WRITE overwrite path in ReactiveMessageService**

In `ReactiveMessageService.java` (~line 356-390), the `OverwriteResult` pattern-match block calls `channelGateway.fanOut()` and `broadcaster.broadcast()` but skips observer dispatch. Add observer dispatch after the rate limit recording and before `channelGateway.fanOut()`:

```java
final Message syntheticMsg = Message.builder()
        .id(or.result().messageId())
        .channelId(dispatch.channelId())
        .sender(dispatch.sender())
        .messageType(dispatch.type())
        .actorType(dispatch.actorType())
        .content(dispatch.content())
        .correlationId(dispatch.correlationId())
        .inReplyTo(dispatch.inReplyTo())
        .artefactRefs(dispatch.artefactRefs())
        .target(dispatch.target())
        .topic(dispatch.topic())
        .createdAt(Instant.now())
        .tenancyId(dispatch.tenancyId())
        .build();
MessageObserverDispatcher.dispatch(
        ch.name(), dispatch.channelId(),
        syntheticMsg.tenancyId(),
        syntheticMsg, observers.handles(), null);
```

Remove the corresponding exclusion comment.

- [ ] **Step 10: Enhance ChannelGateway.deliverRemote()**

In `ChannelGateway.deliverRemote()` (~line 394), after the backend delivery loop (after the `for (BackendEntry entry ...)` block), add:

```java
messageService.dispatchClusterObservers(ch.name(), channelId, msg.tenancyId(), msg);
```

`messageService` is already injected into `ChannelGateway` (field at line 50).

- [ ] **Step 11: Run LAST_WRITE observer test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LastWriteObserverDispatchTest -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 12: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format`
Expected: PASS

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcherTest.java runtime/src/test/java/io/casehub/qhorus/runtime/message/LastWriteObserverDispatchTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat: CLUSTER scope dispatch + LAST_WRITE observer fix (#163)

- dispatchClusterOnly() fires only CLUSTER-scoped observers
- MessageService.dispatchClusterObservers() for ChannelGateway access
- deliverRemote() now fires CLUSTER observers on remote nodes
- LAST_WRITE overwrite path fires observers (was excluded, caused
  node-asymmetric behavior)

Refs #163"
```

---

### Task 3: Kafka Observer Module

**Files:**
- Create: `kafka-observer/pom.xml`
- Create: `kafka-observer/src/main/java/io/casehub/qhorus/kafka/KafkaObserverConfig.java`
- Create: `kafka-observer/src/main/java/io/casehub/qhorus/kafka/KafkaMessageObserver.java`
- Modify: `pom.xml` (add `<module>kafka-observer</module>`)
- Test: `kafka-observer/src/test/java/io/casehub/qhorus/kafka/KafkaMessageObserverTest.java`

**Interfaces:**
- Consumes: `MessageObserver` (api), `CloudEventMapper` (runtime), `MessageReceivedEvent` (api)
- Produces: `KafkaMessageObserver` — `@ApplicationScoped` MessageObserver, scope LOCAL, publishes CloudEvents to Kafka topic

- [ ] **Step 1: Create kafka-observer module pom.xml**

Create `kafka-observer/pom.xml` following the `postgres-broadcaster/pom.xml` pattern:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-qhorus-kafka-observer</artifactId>
  <name>CaseHub Qhorus - Kafka Observer</name>
  <description>Publishes MessageReceivedEvents to a Kafka topic as CloudEvents.
Activated by classpath presence — add this dependency to receive all channel
messages on a configurable Kafka topic.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-messaging-kafka</artifactId>
    </dependency>
    <dependency>
      <groupId>io.cloudevents</groupId>
      <artifactId>cloudevents-kafka</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

Use `ide_replace_text_in_file` to add `<module>kafka-observer</module>` after `<module>postgres-broadcaster</module>` in the parent pom.xml.

- [ ] **Step 3: Write the failing test**

Create `kafka-observer/src/test/java/io/casehub/qhorus/kafka/KafkaMessageObserverTest.java`:

```java
package io.casehub.qhorus.kafka;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import org.eclipse.microprofile.reactive.messaging.Message;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;
import io.cloudevents.CloudEvent;

class KafkaMessageObserverTest {

    private KafkaMessageObserver observer;
    private final List<Message<CloudEvent>> sent = new ArrayList<>();

    @BeforeEach
    void setUp() {
        ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());
        observer = new KafkaMessageObserver(mapper, sent::add, Set.of());
    }

    @Test
    void scopeIsLocal() {
        assertThat(observer.scope()).isEqualTo(MessageObserver.Scope.LOCAL);
    }

    @Test
    void publishesCloudEventToKafka() {
        UUID channelId = UUID.randomUUID();
        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.STATUS, "agent-1", "corr-1",
                Instant.now(), "hello", "general");

        observer.onMessage(event);

        assertThat(sent).hasSize(1);
        Message<CloudEvent> msg = sent.get(0);
        CloudEvent ce = msg.getPayload();
        assertThat(ce.getType()).isEqualTo("io.casehub.qhorus.message.status");

        // Verify Kafka key is the channelId
        org.apache.kafka.common.header.Headers headers =
                msg.getMetadata(io.smallrye.reactive.messaging.kafka.api.OutgoingKafkaRecordMetadata.class)
                        .map(m -> m.getHeaders()).orElse(null);
        // Key verification via metadata
        String key = msg.getMetadata(io.smallrye.reactive.messaging.kafka.api.OutgoingKafkaRecordMetadata.class)
                .map(m -> m.getKey()).orElse(null);
        assertThat(key).isEqualTo(channelId.toString());
    }

    @Test
    void channelFilterReturnsConfiguredSet() {
        ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());
        KafkaMessageObserver filtered = new KafkaMessageObserver(
                mapper, sent::add, Set.of("channel-a", "channel-b"));

        assertThat(filtered.channels()).containsExactlyInAnyOrder("channel-a", "channel-b");
    }

    @Test
    void emptyChannelFilterMeansAll() {
        assertThat(observer.channels()).isEmpty();
    }
}
```

Note: The exact test assertions for Kafka metadata may need adjustment based on how SmallRye exposes `OutgoingKafkaRecordMetadata`. The test uses a `Consumer<Message<CloudEvent>>` constructor for testability — the production path uses `@Channel Emitter`.

- [ ] **Step 4: Implement KafkaObserverConfig**

Create `kafka-observer/src/main/java/io/casehub/qhorus/kafka/KafkaObserverConfig.java`:

```java
package io.casehub.qhorus.kafka;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

import java.util.Optional;
import java.util.Set;

@ConfigMapping(prefix = "casehub.qhorus.kafka")
public interface KafkaObserverConfig {
    @WithDefault("qhorus-messages")
    String topic();

    Optional<Set<String>> channels();
}
```

- [ ] **Step 5: Implement KafkaMessageObserver**

Create `kafka-observer/src/main/java/io/casehub/qhorus/kafka/KafkaMessageObserver.java`:

```java
package io.casehub.qhorus.kafka;

import java.util.Set;
import java.util.function.Consumer;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;
import org.eclipse.microprofile.reactive.messaging.Message;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.runtime.gateway.CloudEventMapper;
import io.cloudevents.CloudEvent;
import io.smallrye.reactive.messaging.kafka.api.OutgoingKafkaRecordMetadata;

@ApplicationScoped
public class KafkaMessageObserver implements MessageObserver {

    private final ObjectMapper objectMapper;
    private final Consumer<Message<CloudEvent>> sender;
    private final Set<String> channelFilter;

    @Inject
    public KafkaMessageObserver(ObjectMapper objectMapper,
                                 @Channel("qhorus-messages") Emitter<CloudEvent> emitter,
                                 KafkaObserverConfig config) {
        this.objectMapper = objectMapper;
        this.sender = msg -> emitter.send(msg);
        this.channelFilter = config.channels().orElse(Set.of());
    }

    /** Test constructor — accepts a mock sender. */
    KafkaMessageObserver(ObjectMapper objectMapper,
                          Consumer<Message<CloudEvent>> sender,
                          Set<String> channelFilter) {
        this.objectMapper = objectMapper;
        this.sender = sender;
        this.channelFilter = channelFilter;
    }

    @Override
    public void onMessage(MessageReceivedEvent event) {
        CloudEvent ce = CloudEventMapper.toCloudEvent(event, objectMapper);
        OutgoingKafkaRecordMetadata<String> metadata = OutgoingKafkaRecordMetadata.<String>builder()
                .withKey(event.channelId().toString())
                .build();
        Message<CloudEvent> msg = Message.of(ce).addMetadata(metadata);
        sender.accept(msg);
    }

    @Override
    public Scope scope() {
        return Scope.LOCAL;
    }

    @Override
    public Set<String> channels() {
        return channelFilter;
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl kafka-observer -Dno-format`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add kafka-observer/ pom.xml
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat: Kafka observer module (#163)

KafkaMessageObserver publishes MessageReceivedEvents as CloudEvents to
a configurable Kafka topic. Scope LOCAL — fires once on the dispatching
node. Partitioned by channelId.

Refs #163"
```

---

### Task 4: WebSocket Observer Module

**Files:**
- Create: `websocket-observer/pom.xml`
- Create: `websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketObserverConfig.java`
- Create: `websocket-observer/src/main/java/io/casehub/qhorus/websocket/ChannelWebSocketEndpoint.java`
- Create: `websocket-observer/src/main/java/io/casehub/qhorus/websocket/WebSocketMessageObserver.java`
- Modify: `pom.xml` (add `<module>websocket-observer</module>`)
- Test: `websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketMessageObserverTest.java`

**Interfaces:**
- Consumes: `MessageObserver` (api), `CloudEventMapper` (runtime), `MessageReceivedEvent` (api)
- Produces: `WebSocketMessageObserver` — `@ApplicationScoped`, scope CLUSTER, pushes CloudEvent JSON to connected WebSocket clients. `ChannelWebSocketEndpoint` — WebSocket `@WebSocket` path-based subscription endpoint.

- [ ] **Step 1: Create websocket-observer module pom.xml**

Follow the same pattern as kafka-observer. Dependencies: `casehub-qhorus`, `quarkus-websockets-next`, `cloudevents-core`, `quarkus-arc`. Test deps: `quarkus-junit5`, `assertj-core`, `quarkus-websockets-next` (test client support).

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>websocket-observer</module>` after `<module>kafka-observer</module>`.

- [ ] **Step 3: Write the failing unit test**

Create `websocket-observer/src/test/java/io/casehub/qhorus/websocket/WebSocketMessageObserverTest.java`:

```java
package io.casehub.qhorus.websocket;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;

class WebSocketMessageObserverTest {

    private WebSocketMessageObserver observer;
    private final ConcurrentHashMap<UUID, Set<MockConnection>> registry = new ConcurrentHashMap<>();
    private final List<String> sentMessages = new ArrayList<>();

    static class MockConnection {
        final List<String> sent = new ArrayList<>();
        void sendText(String text) { sent.add(text); }
    }

    @BeforeEach
    void setUp() {
        ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());
        observer = new WebSocketMessageObserver(mapper, registry);
    }

    @Test
    void scopeIsCluster() {
        assertThat(observer.scope()).isEqualTo(MessageObserver.Scope.CLUSTER);
    }

    @Test
    void pushesToSubscribedConnections() {
        UUID channelId = UUID.randomUUID();
        MockConnection conn = new MockConnection();
        registry.put(channelId, Set.of(conn));

        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.STATUS, "agent-1", null,
                Instant.now(), "hello", null);

        observer.onMessage(event);

        assertThat(conn.sent).hasSize(1);
        assertThat(conn.sent.get(0)).contains("io.casehub.qhorus.message.status");
    }

    @Test
    void skipsChannelsWithNoSubscribers() {
        UUID channelId = UUID.randomUUID();
        // No registry entry for this channel

        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.STATUS, "agent-1", null,
                Instant.now(), "hello", null);

        observer.onMessage(event);
        // No exception, no crash — silent skip
    }
}
```

Note: The exact mock connection type depends on the Quarkus WebSockets Next API (`WebSocketConnection`). The test constructor accepts an injected registry so the unit test can bypass CDI. Adjust the mock type to match the real `WebSocketConnection` interface during implementation.

- [ ] **Step 4: Implement WebSocketObserverConfig, ChannelWebSocketEndpoint, and WebSocketMessageObserver**

`WebSocketObserverConfig`:
```java
@ConfigMapping(prefix = "casehub.qhorus.websocket")
public interface WebSocketObserverConfig {
    @WithDefault("/qhorus/ws")
    String path();
}
```

`ChannelWebSocketEndpoint` — uses `@WebSocket(path = "/qhorus/ws/channels/{channelId}")`. `@OnOpen` adds the connection to a shared `ConcurrentHashMap<UUID, Set<WebSocketConnection>>` registry. `@OnClose` removes it. The registry is injected from `WebSocketMessageObserver` (or a shared `@ApplicationScoped` registry bean).

`WebSocketMessageObserver` — `@ApplicationScoped`, scope CLUSTER. Holds the registry. `onMessage()` looks up connections by `event.channelId()`, maps event → CloudEvent JSON via `CloudEventMapper`, sends to each connection.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl websocket-observer -Dno-format`
Expected: PASS

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add websocket-observer/ pom.xml
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat: WebSocket observer module (#163)

WebSocketMessageObserver pushes CloudEvent JSON to connected WebSocket
clients. Scope CLUSTER — fires on all nodes so clients on any node
receive events from any dispatching node. Path-based subscription via
channelId.

Refs #163"
```

---

### Task 5: Webhook Observer Module

**Files:**
- Create: `webhook-observer/pom.xml`
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookObserverConfig.java`
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistration.java`
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistry.java`
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookRegistryResource.java`
- Create: `webhook-observer/src/main/java/io/casehub/qhorus/webhook/WebhookMessageObserver.java`
- Modify: `pom.xml` (add `<module>webhook-observer</module>`)
- Test: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookRegistryTest.java`
- Test: `webhook-observer/src/test/java/io/casehub/qhorus/webhook/WebhookMessageObserverTest.java`

**Interfaces:**
- Consumes: `MessageObserver` (api), `CloudEventMapper` (runtime), `MessageReceivedEvent` (api), Vert.x `WebClient`
- Produces: `WebhookMessageObserver` — `@ApplicationScoped`, scope CLUSTER, POSTs CloudEvent JSON to registered URLs. `WebhookRegistry` — in-memory registration store. `WebhookRegistryResource` — JAX-RS REST API for CRUD.

- [ ] **Step 1: Create webhook-observer module pom.xml**

Dependencies: `casehub-qhorus`, `cloudevents-core`, `quarkus-vertx` (for WebClient), `quarkus-rest` (for JAX-RS resource), `quarkus-arc`. Test deps: `quarkus-junit5`, `assertj-core`, `quarkus-rest-client` (for testing the REST endpoint), or WireMock.

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>webhook-observer</module>` after `<module>websocket-observer</module>`.

- [ ] **Step 3: Write WebhookRegistry tests**

```java
package io.casehub.qhorus.webhook;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class WebhookRegistryTest {

    private WebhookRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new WebhookRegistry();
    }

    @Test
    void registerAndLookupByChannel() {
        UUID channelId = UUID.randomUUID();
        WebhookRegistration reg = registry.register(channelId, "https://example.com/hook", null, Map.of());

        assertThat(reg.id()).isNotNull();
        assertThat(registry.findByChannelId(channelId)).hasSize(1);
        assertThat(registry.findByChannelId(channelId).iterator().next().url()).isEqualTo("https://example.com/hook");
    }

    @Test
    void globalWebhookReceivedForAnyChannel() {
        registry.register(null, "https://example.com/global", null, Map.of());

        UUID anyChannel = UUID.randomUUID();
        var hooks = registry.findForChannel(anyChannel);

        assertThat(hooks).hasSize(1);
    }

    @Test
    void channelSpecificPlusGlobal() {
        UUID channelId = UUID.randomUUID();
        registry.register(channelId, "https://example.com/specific", null, Map.of());
        registry.register(null, "https://example.com/global", null, Map.of());

        var hooks = registry.findForChannel(channelId);

        assertThat(hooks).hasSize(2);
    }

    @Test
    void deregisterRemoves() {
        UUID channelId = UUID.randomUUID();
        WebhookRegistration reg = registry.register(channelId, "https://example.com/hook", null, Map.of());

        registry.deregister(reg.id());

        assertThat(registry.findByChannelId(channelId)).isEmpty();
    }
}
```

- [ ] **Step 4: Implement WebhookRegistration, WebhookRegistry, WebhookObserverConfig**

`WebhookRegistration` — record with `id` (UUID), `channelId` (UUID, nullable for global), `url`, `secret` (nullable), `headers` (Map).

`WebhookRegistry` — `@ApplicationScoped`. Two maps: `ConcurrentHashMap<UUID, Set<WebhookRegistration>>` for channel-specific, `Set<WebhookRegistration>` (ConcurrentHashMap.newKeySet) for global. `findForChannel(UUID)` returns union.

`WebhookObserverConfig`:
```java
@ConfigMapping(prefix = "casehub.qhorus.webhook")
public interface WebhookObserverConfig {
    @WithDefault("/qhorus/webhooks")
    String path();

    @WithDefault("5000")
    int timeoutMs();
}
```

- [ ] **Step 5: Run registry tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WebhookRegistryTest -pl webhook-observer -Dno-format`
Expected: PASS

- [ ] **Step 6: Write WebhookMessageObserver test**

```java
package io.casehub.qhorus.webhook;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;

class WebhookMessageObserverTest {

    record PostRecord(String url, String body, Map<String, String> headers) {}

    private WebhookRegistry registry;
    private WebhookMessageObserver observer;
    private final List<PostRecord> posts = new ArrayList<>();

    @BeforeEach
    void setUp() {
        ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());
        registry = new WebhookRegistry();
        observer = new WebhookMessageObserver(mapper, registry, (url, body, secret, headers) -> {
            Map<String, String> allHeaders = new java.util.HashMap<>(headers);
            if (secret != null) {
                allHeaders.put("X-Qhorus-Signature", "hmac-placeholder");
            }
            posts.add(new PostRecord(url, body, allHeaders));
        });
    }

    @Test
    void scopeIsCluster() {
        assertThat(observer.scope()).isEqualTo(MessageObserver.Scope.CLUSTER);
    }

    @Test
    void postsToRegisteredWebhook() {
        UUID channelId = UUID.randomUUID();
        registry.register(channelId, "https://example.com/hook", null, Map.of());

        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.STATUS, "agent-1", null,
                Instant.now(), "hello", null);

        observer.onMessage(event);

        assertThat(posts).hasSize(1);
        assertThat(posts.get(0).url()).isEqualTo("https://example.com/hook");
        assertThat(posts.get(0).body()).contains("io.casehub.qhorus.message.status");
    }

    @Test
    void postsToGlobalWebhookForAnyChannel() {
        registry.register(null, "https://example.com/global", null, Map.of());

        UUID channelId = UUID.randomUUID();
        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.QUERY, "agent-1", null,
                Instant.now(), "q", null);

        observer.onMessage(event);

        assertThat(posts).hasSize(1);
    }

    @Test
    void includesSignatureHeaderWhenSecretPresent() {
        UUID channelId = UUID.randomUUID();
        registry.register(channelId, "https://example.com/hook", "my-secret", Map.of());

        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.STATUS, "agent-1", null,
                Instant.now(), "hello", null);

        observer.onMessage(event);

        assertThat(posts.get(0).headers()).containsKey("X-Qhorus-Signature");
    }

    @Test
    void noPostWhenNoRegistrations() {
        UUID channelId = UUID.randomUUID();
        MessageReceivedEvent event = new MessageReceivedEvent(
                "test-channel", channelId, "t1",
                MessageType.STATUS, "agent-1", null,
                Instant.now(), "hello", null);

        observer.onMessage(event);

        assertThat(posts).isEmpty();
    }
}
```

- [ ] **Step 7: Implement WebhookMessageObserver**

`@ApplicationScoped`, scope CLUSTER. Injects `WebhookRegistry` and `ObjectMapper`. `onMessage()`: lookup registrations via `registry.findForChannel(event.channelId())`, map to CloudEvent JSON, POST to each URL. Test constructor accepts a `WebhookPoster` functional interface for mock injection. Production constructor uses Vert.x `WebClient`.

HMAC-SHA256 signature: `javax.crypto.Mac` with `HmacSHA256` algorithm, hex-encoded. Header: `X-Qhorus-Signature`.

- [ ] **Step 8: Implement WebhookRegistryResource**

JAX-RS REST resource: `POST /qhorus/webhooks`, `DELETE /qhorus/webhooks/{id}`, `GET /qhorus/webhooks`, `GET /qhorus/webhooks?channelId={uuid}`. Injects `WebhookRegistry`. Standard CRUD delegation.

- [ ] **Step 9: Run all webhook tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webhook-observer -Dno-format`
Expected: PASS

- [ ] **Step 10: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add webhook-observer/ pom.xml
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat: Webhook observer module (#163)

WebhookMessageObserver POSTs CloudEvent JSON to registered callback URLs.
Scope CLUSTER. In-memory registry with REST API for dynamic registration.
HMAC-SHA256 signature when secret is configured.

Refs #163"
```

---

### Task 6: Documentation update + full integration verification

**Files:**
- Modify: `docs/messaging-architecture.md` (update "What Ships When" table and Transport Scope section)
- Modify: `CLAUDE.md` (add conventions for the three new modules)

**Interfaces:**
- Consumes: all prior tasks
- Produces: updated documentation, passing full build

- [ ] **Step 1: Update messaging-architecture.md**

Update the "What Ships When" table: change `KafkaMessageBus`, `WebSocketMessageBus`, webhook impl from `⬜ future` to `🔧 qhorus#163`. Update the "Transport Scope" section to document that CLUSTER scope is now implemented — LOCAL fires on dispatching node only, CLUSTER fires on all nodes.

- [ ] **Step 2: Update CLAUDE.md**

Add entries for the three new modules in the Project Structure section:
```
├── kafka-observer/                      — Optional Kafka observer (MessageReceivedEvent → Kafka topic)
├── websocket-observer/                  — Optional WebSocket observer (real-time push to browser clients)
├── webhook-observer/                    — Optional Webhook observer (HTTP POST callbacks)
```

Add testing conventions for the new modules.

- [ ] **Step 3: Run full build from root**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
Expected: All modules PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add docs/messaging-architecture.md CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs: update messaging-architecture and CLAUDE.md for observer modules (#163)

CLUSTER scope now implemented. Three transport modules documented:
kafka-observer, websocket-observer, webhook-observer.

Refs #163"
```

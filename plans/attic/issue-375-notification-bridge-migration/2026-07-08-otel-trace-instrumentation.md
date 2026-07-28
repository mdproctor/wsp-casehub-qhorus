# OTel Trace Instrumentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #197 — OTel trace context propagation and span emission
**Issue group:** #197

**Goal:** Add OpenTelemetry span instrumentation to Qhorus so that
dispatch, commitment transitions, fan-out, delivery, and ledger writes
are visible in distributed traces.

**Architecture:** Explicit `Tracer` injection into 9 service classes (4
blocking + 4 reactive + ChannelGateway). Each service checks a
`QhorusTracingConfig` gate before creating spans. When no OTel SDK is on
the consumer's classpath, `Tracer` is no-op — zero overhead. Cross-request
correlation uses span links via existing `MessageLedgerEntry.traceId`.

**Tech Stack:** `io.opentelemetry:opentelemetry-api` (optional),
`io.opentelemetry:opentelemetry-sdk-testing` (test), Quarkus 3.32.2,
SmallRye Config `@ConfigMapping`

## Global Constraints

- `opentelemetry-api` must be `<optional>true</optional>` — never forced on consumers
- No `quarkus-opentelemetry` dependency in Qhorus itself
- No Flyway migrations — `traceId` already exists on `LedgerEntry`
- No changes to `api/`, `deployment/`, `connector-backend/`, `slack-channel/`, `postgres-broadcaster/`, `persistence-memory/`, `testing/`
- Reactive span lifecycle: `onFailure` before `onTermination` — errors recorded before span ends
- All config defaults to `true` — tracing is on by default when OTel SDK is present
- Version managed by Quarkus BOM — no explicit `<version>` tag

---

### Task 1: Dependencies and Configuration

**Files:**
- Modify: `runtime/pom.xml` — add `opentelemetry-api` optional + `opentelemetry-sdk-testing` test
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusTracingConfig.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/config/QhorusTracingConfigTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `QhorusTracingConfig` — `@ConfigMapping(prefix = "casehub.qhorus.tracing")` with methods `enabled()`, `dispatch()`, `commitments()`, `fanOut()`, `ledgerWrite()`, `delivery()` — all `@WithDefault("true") boolean`

- [ ] **Step 1: Add Maven dependencies to `runtime/pom.xml`**

In `<dependencies>`, add:

```xml
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-api</artifactId>
  <optional>true</optional>
</dependency>
```

In the test-scope dependencies section, add:

```xml
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-sdk-testing</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Create `QhorusTracingConfig.java`**

```java
package io.casehub.qhorus.runtime.config;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.qhorus.tracing")
public interface QhorusTracingConfig {

    @WithDefault("true")
    boolean enabled();

    @WithDefault("true")
    boolean dispatch();

    @WithDefault("true")
    boolean commitments();

    @WithDefault("true")
    boolean fanOut();

    @WithDefault("true")
    boolean ledgerWrite();

    @WithDefault("true")
    boolean delivery();
}
```

- [ ] **Step 3: Write config test**

```java
package io.casehub.qhorus.runtime.config;

import static org.assertj.core.api.Assertions.*;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class QhorusTracingConfigTest {

    @Inject QhorusTracingConfig config;

    @Test
    void defaults_are_all_true() {
        assertThat(config.enabled()).isTrue();
        assertThat(config.dispatch()).isTrue();
        assertThat(config.commitments()).isTrue();
        assertThat(config.fanOut()).isTrue();
        assertThat(config.ledgerWrite()).isTrue();
        assertThat(config.delivery()).isTrue();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=QhorusTracingConfigTest`
Expected: PASS

- [ ] **Step 5: Compile the full project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests`
Expected: BUILD SUCCESS — verifies `opentelemetry-api` resolves from the Quarkus BOM

- [ ] **Step 6: Commit**

```
feat(#197): add opentelemetry-api dependency and QhorusTracingConfig

Refs #197
```

---

### Task 2: Dispatch Span — `MessageService.dispatch()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — inject `Tracer` + `QhorusTracingConfig`, wrap dispatch body in span
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/DispatchTracingTest.java`

**Interfaces:**
- Consumes: `QhorusTracingConfig` (Task 1)
- Produces: `qhorus.dispatch` span with attributes `qhorus.channel.id`, `qhorus.channel.name`, `qhorus.channel.semantic`, `qhorus.message.type`, `qhorus.message.sender`, `qhorus.message.correlation_id`, `qhorus.message.target`, `qhorus.actor.type`, `qhorus.tenancy.id`; span events `qhorus.enforcement.acl`, `qhorus.enforcement.rate_limit`, `qhorus.enforcement.trust`, `qhorus.enforcement.type_policy`, `qhorus.observer.dispatch`

- [ ] **Step 1: Write the failing test**

Create `DispatchTracingTest.java`:

```java
package io.casehub.qhorus.runtime.message;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;

@QuarkusTest
class DispatchTracingTest {

    @Inject MessageService messageService;
    @Inject ChannelStore channelStore;

    private InMemorySpanExporter exporter;

    @BeforeEach
    void setUp() {
        exporter = InMemorySpanExporter.create();
    }

    @Test @TestTransaction
    void dispatch_creates_span_with_channel_and_message_attributes() {
        UUID channelId = createChannel("tracing-test-" + UUID.randomUUID());

        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId).sender("agent-1").type(MessageType.STATUS)
                .content("hello").actorType(ActorType.AGENT).build());

        List<SpanData> spans = exporter.getFinishedSpanItems().stream()
                .filter(s -> s.getName().equals("qhorus.dispatch"))
                .toList();

        assertThat(spans).hasSize(1);
        SpanData span = spans.get(0);
        assertThat(span.getKind()).isEqualTo(SpanKind.INTERNAL);
        assertThat(span.getAttributes().get(
                io.opentelemetry.api.common.AttributeKey.stringKey("qhorus.channel.id")))
                .isEqualTo(channelId.toString());
        assertThat(span.getAttributes().get(
                io.opentelemetry.api.common.AttributeKey.stringKey("qhorus.message.type")))
                .isEqualTo("STATUS");
        assertThat(span.getAttributes().get(
                io.opentelemetry.api.common.AttributeKey.stringKey("qhorus.message.sender")))
                .isEqualTo("agent-1");
    }

    private UUID createChannel(String name) {
        var ch = channelStore.put(ChannelCreateRequest.builder(name).build());
        return ch.id();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DispatchTracingTest`
Expected: FAIL — no `qhorus.dispatch` span created yet

- [ ] **Step 3: Implement dispatch span in `MessageService.dispatch()`**

Add field injections to `MessageService`:

```java
@Inject io.opentelemetry.api.trace.Tracer tracer;
@Inject QhorusTracingConfig tracingConfig;
```

At the top of `dispatch()`, after the paused check (line ~125), add span creation. Wrap the remaining body in try/finally. Set attributes after channel resolution. Add span events after each enforcement check.

The key structure:

```java
Span span = null;
if (tracingConfig.enabled() && tracingConfig.dispatch()) {
    span = tracer.spanBuilder("qhorus.dispatch")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();
}
try (Scope scope = span != null ? span.makeCurrent() : null) {
    // ... existing dispatch body ...

    // After channel resolution, set attributes:
    if (span != null) {
        span.setAttribute("qhorus.channel.id", ch.id().toString());
        span.setAttribute("qhorus.channel.name", ch.name());
        span.setAttribute("qhorus.channel.semantic", ch.semantic().name());
        span.setAttribute("qhorus.message.type", dispatch.type().name());
        span.setAttribute("qhorus.message.sender", dispatch.sender());
        if (dispatch.correlationId() != null) {
            span.setAttribute("qhorus.message.correlation_id", dispatch.correlationId());
        }
        if (dispatch.target() != null) {
            span.setAttribute("qhorus.message.target", dispatch.target());
        }
        span.setAttribute("qhorus.actor.type", dispatch.actorType().name());
        span.setAttribute("qhorus.tenancy.id", effectiveTenancyId);
    }

    // After each enforcement check, add span event:
    if (span != null) {
        span.addEvent("qhorus.enforcement.acl");
    }
    // ... similarly for rate_limit, trust, type_policy, observer.dispatch

    return result;
} catch (Exception e) {
    if (span != null) {
        span.setStatus(StatusCode.ERROR);
        span.recordException(e);
    }
    throw e;
} finally {
    if (span != null) {
        span.end();
    }
}
```

The span starts after the paused check. The LAST_WRITE early-return path also ends the span (the `finally` block handles this). The `qhorus.channel.semantic` attribute distinguishes LAST_WRITE from normal dispatch.

Import: `io.opentelemetry.api.trace.Span`, `io.opentelemetry.api.trace.SpanKind`, `io.opentelemetry.api.trace.StatusCode`, `io.opentelemetry.context.Scope`

- [ ] **Step 4: Wire test to capture spans**

The `@QuarkusTest` needs the OTel SDK wired for test. The `opentelemetry-sdk-testing` dependency provides `InMemorySpanExporter`. To use it in a `@QuarkusTest`, the `SdkTracerProvider` must be configured.

Approach: create a test-scoped CDI producer that provides a `SdkTracerProvider` with `InMemorySpanExporter`. Alternatively, since `MessageService` injects `Tracer` directly, the test can use `@InjectMock Tracer` or configure the exporter via Quarkus OTel test properties.

Simpler approach for unit-level validation: create a CDI-free test (`DispatchTracingUnitTest`) that sets `messageService.tracer` and `messageService.tracingConfig` via package-private field access (same pattern as existing CDI-free tests in the codebase). This avoids needing the full Quarkus OTel extension in test scope.

```java
package io.casehub.qhorus.runtime.message;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.persistence.memory.InMemoryChannelStore;
import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore;

class DispatchTracingUnitTest {

    private InMemorySpanExporter exporter;
    private MessageService service;

    @BeforeEach
    void setUp() {
        exporter = InMemorySpanExporter.create();
        SdkTracerProvider provider = SdkTracerProvider.builder()
                .addSpanProcessor(SimpleSpanProcessor.create(exporter))
                .build();

        service = new MessageService();
        service.tracer = provider.get("qhorus-test");
        service.tracingConfig = new QhorusTracingConfig() {
            public boolean enabled() { return true; }
            public boolean dispatch() { return true; }
            public boolean commitments() { return true; }
            public boolean fanOut() { return true; }
            public boolean ledgerWrite() { return true; }
            public boolean delivery() { return true; }
        };

        // Wire InMemory stores and stubs for the remaining dependencies
        // (channelStore, messageStore, etc.) — same pattern as existing
        // CDI-free tests in the codebase.
    }

    @Test
    void dispatch_creates_span_with_attributes() {
        // Arrange: create channel via InMemoryChannelStore
        // Act: service.dispatch(...)
        // Assert: exporter.getFinishedSpanItems() contains "qhorus.dispatch"
        // with expected attributes
    }

    @Test
    void dispatch_with_tracing_disabled_creates_no_spans() {
        service.tracingConfig = new QhorusTracingConfig() {
            public boolean enabled() { return false; }
            public boolean dispatch() { return true; }
            public boolean commitments() { return true; }
            public boolean fanOut() { return true; }
            public boolean ledgerWrite() { return true; }
            public boolean delivery() { return true; }
        };

        // Act: service.dispatch(...)
        // Assert: exporter.getFinishedSpanItems() is empty
    }

    @Test
    void dispatch_records_error_on_exception() {
        // Arrange: dispatch to non-existent channel
        // Act + Assert: span has ERROR status and recorded exception
    }
}
```

Note: the CDI-free test requires wiring InMemory stores. The full wiring is
verbose — follow the pattern from existing tests like `DeliveryServiceTest`
which wire `MessageService` fields directly. The test file will be ~150-200
lines including setup.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DispatchTracingUnitTest`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#197): add dispatch span to MessageService

Refs #197
```

---

### Task 3: Commitment Transition Spans — `CommitmentService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java` — inject `Tracer` + `QhorusTracingConfig`, add spans to all 8 transition methods
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/CommitmentTracingTest.java`

**Interfaces:**
- Consumes: `QhorusTracingConfig` (Task 1)
- Produces: `qhorus.commitment.open`, `.acknowledge`, `.fulfill`, `.decline`, `.fail`, `.delegate`, `.expire_overdue`, `.extend_deadline` spans with attributes `qhorus.commitment.id`, `qhorus.commitment.correlation_id`, `qhorus.commitment.from_state`, `qhorus.commitment.to_state`, `qhorus.commitment.obligor`, `qhorus.channel.id`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.qhorus.runtime.message;

import static org.assertj.core.api.Assertions.*;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore;

class CommitmentTracingTest {

    private InMemorySpanExporter exporter;
    private CommitmentService service;
    private InMemoryCommitmentStore store;

    @BeforeEach
    void setUp() {
        exporter = InMemorySpanExporter.create();
        SdkTracerProvider provider = SdkTracerProvider.builder()
                .addSpanProcessor(SimpleSpanProcessor.create(exporter))
                .build();

        store = new InMemoryCommitmentStore();
        service = new CommitmentService();
        service.store = store;
        service.tracer = provider.get("qhorus-test");
        service.tracingConfig = new QhorusTracingConfig() {
            public boolean enabled() { return true; }
            public boolean dispatch() { return true; }
            public boolean commitments() { return true; }
            public boolean fanOut() { return true; }
            public boolean ledgerWrite() { return true; }
            public boolean delivery() { return true; }
        };
    }

    @Test
    void open_creates_commitment_span() {
        UUID channelId = UUID.randomUUID();
        UUID commitmentId = UUID.randomUUID();

        service.open(commitmentId, "corr-1", channelId,
                MessageType.COMMAND, "requester", "obligor", null);

        List<SpanData> spans = exporter.getFinishedSpanItems().stream()
                .filter(s -> s.getName().equals("qhorus.commitment.open"))
                .toList();

        assertThat(spans).hasSize(1);
        SpanData span = spans.get(0);
        assertThat(span.getAttributes().get(
                AttributeKey.stringKey("qhorus.commitment.id")))
                .isEqualTo(commitmentId.toString());
        assertThat(span.getAttributes().get(
                AttributeKey.stringKey("qhorus.commitment.to_state")))
                .isEqualTo("OPEN");
    }

    @Test
    void fulfill_creates_span_with_state_transition() {
        UUID channelId = UUID.randomUUID();
        UUID commitmentId = UUID.randomUUID();
        service.open(commitmentId, "corr-2", channelId,
                MessageType.COMMAND, "req", "obl", null);
        exporter.reset();

        service.fulfill("corr-2");

        List<SpanData> spans = exporter.getFinishedSpanItems().stream()
                .filter(s -> s.getName().equals("qhorus.commitment.fulfill"))
                .toList();

        assertThat(spans).hasSize(1);
        assertThat(spans.get(0).getAttributes().get(
                AttributeKey.stringKey("qhorus.commitment.from_state")))
                .isEqualTo("OPEN");
        assertThat(spans.get(0).getAttributes().get(
                AttributeKey.stringKey("qhorus.commitment.to_state")))
                .isEqualTo("FULFILLED");
    }

    @Test
    void no_span_when_commitments_tracing_disabled() {
        service.tracingConfig = new QhorusTracingConfig() {
            public boolean enabled() { return true; }
            public boolean dispatch() { return true; }
            public boolean commitments() { return false; }
            public boolean fanOut() { return true; }
            public boolean ledgerWrite() { return true; }
            public boolean delivery() { return true; }
        };

        service.open(UUID.randomUUID(), "corr-3", UUID.randomUUID(),
                MessageType.COMMAND, "req", "obl", null);

        assertThat(exporter.getFinishedSpanItems()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CommitmentTracingTest`
Expected: FAIL — no spans created

- [ ] **Step 3: Implement commitment spans**

Add to `CommitmentService`:

```java
@Inject io.opentelemetry.api.trace.Tracer tracer;
@Inject QhorusTracingConfig tracingConfig;
```

For each transition method (open, acknowledge, fulfill, decline, fail, delegate, extendDeadline), wrap in span:

```java
public Commitment open(UUID commitmentId, String correlationId, UUID channelId,
        MessageType type, String requester, String obligor, Instant expiresAt) {
    Span span = null;
    if (tracingConfig.enabled() && tracingConfig.commitments()) {
        span = tracer.spanBuilder("qhorus.commitment.open")
                .setSpanKind(SpanKind.INTERNAL)
                .startSpan();
        span.setAttribute("qhorus.commitment.id", commitmentId.toString());
        span.setAttribute("qhorus.commitment.correlation_id", correlationId);
        span.setAttribute("qhorus.commitment.to_state", "OPEN");
        span.setAttribute("qhorus.commitment.obligor", obligor != null ? obligor : "");
        span.setAttribute("qhorus.channel.id", channelId.toString());
    }
    try {
        // existing body
        return commitment;
    } catch (Exception e) {
        if (span != null) { span.setStatus(StatusCode.ERROR); span.recordException(e); }
        throw e;
    } finally {
        if (span != null) span.end();
    }
}
```

For `expireOverdue()` — root span with `qhorus.commitment.expired_count` attribute and per-commitment span events:

```java
public int expireOverdue() {
    Span span = null;
    if (tracingConfig.enabled() && tracingConfig.commitments()) {
        span = tracer.spanBuilder("qhorus.commitment.expire_overdue")
                .setSpanKind(SpanKind.INTERNAL)
                .setNoParent()
                .startSpan();
    }
    try {
        // existing body — for each expired commitment, add span event:
        // if (span != null) span.addEvent("qhorus.commitment.expired",
        //     Attributes.of(AttributeKey.stringKey("commitment_id"), c.id.toString(), ...));
        if (span != null) {
            span.setAttribute("qhorus.commitment.expired_count", count);
        }
        return count;
    } catch (Exception e) {
        if (span != null) { span.setStatus(StatusCode.ERROR); span.recordException(e); }
        throw e;
    } finally {
        if (span != null) span.end();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CommitmentTracingTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#197): add commitment transition spans to CommitmentService

Refs #197
```

---

### Task 4: Fan-out and Delivery Spans — `ChannelGateway` + `DeliveryService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` — inject `Tracer` + `QhorusTracingConfig`, add spans to `fanOut()` and `deliverRemote()`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryService.java` — inject `Tracer` + `QhorusTracingConfig`, add span to `deliverPending()`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/FanOutTracingTest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/DeliveryTracingTest.java`

**Interfaces:**
- Consumes: `QhorusTracingConfig` (Task 1)
- Produces: `qhorus.fanout` span with per-backend child spans `qhorus.fanout.backend`; `qhorus.delivery.remote` span; `qhorus.delivery.pump` span

- [ ] **Step 1: Write failing fan-out test**

```java
package io.casehub.qhorus.runtime.gateway;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageType;

class FanOutTracingTest {

    private InMemorySpanExporter exporter;
    private ChannelGateway gateway;

    @BeforeEach
    void setUp() {
        exporter = InMemorySpanExporter.create();
        SdkTracerProvider provider = SdkTracerProvider.builder()
                .addSpanProcessor(SimpleSpanProcessor.create(exporter))
                .build();

        // Wire gateway with InMemory dependencies — same pattern as
        // existing ChannelGatewayTest. Set tracer and tracingConfig
        // on the gateway instance.
    }

    @Test
    void fanOut_creates_span_with_backend_count() {
        // Arrange: register a BEST_EFFORT backend
        // Act: gateway.fanOut(channelId, "test-channel", outboundMessage)
        // Assert: "qhorus.fanout" span exists with backend_count attribute
    }

    @Test
    void fanOut_creates_child_span_per_backend() {
        // Arrange: register two BEST_EFFORT backends
        // Act: gateway.fanOut(...)
        // Assert: two "qhorus.fanout.backend" child spans with backend_id attributes
        // Note: use Awaitility to wait for virtual threads to complete
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FanOutTracingTest`
Expected: FAIL

- [ ] **Step 3: Implement fan-out span in `ChannelGateway.fanOut()`**

Add field injections to `ChannelGateway`:

```java
@Inject io.opentelemetry.api.trace.Tracer tracer;
@Inject QhorusTracingConfig tracingConfig;
```

Wrap `fanOut()`:

```java
public boolean fanOut(UUID channelId, String channelName, OutboundMessage message) {
    Span span = null;
    if (tracingConfig.enabled() && tracingConfig.fanOut()) {
        span = tracer.spanBuilder("qhorus.fanout")
                .setSpanKind(SpanKind.INTERNAL)
                .startSpan();
        span.setAttribute("qhorus.channel.id", channelId.toString());
    }
    try {
        // existing body, but for each backend.post() virtual thread:
        final Span parentSpan = span;
        final io.opentelemetry.context.Context otelContext =
                io.opentelemetry.context.Context.current();
        Thread.ofVirtual().start(otelContext.wrap(() -> {
            Span childSpan = null;
            if (parentSpan != null) {
                childSpan = tracer.spanBuilder("qhorus.fanout.backend")
                        .setSpanKind(SpanKind.INTERNAL)
                        .startSpan();
                childSpan.setAttribute("qhorus.fanout.backend_id", backend.backendId());
                childSpan.setAttribute("qhorus.fanout.delivery_guarantee",
                        backend.deliveryGuarantee().name());
            }
            try {
                backend.post(ref, message);
            } catch (Exception ex) {
                if (childSpan != null) {
                    childSpan.setStatus(StatusCode.ERROR);
                    childSpan.recordException(ex);
                }
                LOG.errorf(ex, ...);
            } finally {
                if (childSpan != null) childSpan.end();
            }
        }));

        if (span != null) {
            span.setAttribute("qhorus.fanout.backend_count", backendCount);
            span.setAttribute("qhorus.fanout.has_tracked", hasTracked);
        }
        return hasTracked;
    } catch (Exception e) {
        if (span != null) { span.setStatus(StatusCode.ERROR); span.recordException(e); }
        throw e;
    } finally {
        if (span != null) span.end();
    }
}
```

- [ ] **Step 4: Implement delivery spans in `deliverRemote()` and `DeliveryService.deliverPending()`**

Same pattern — root spans (no parent) since these execute outside dispatch context.

`deliverRemote()` — `qhorus.delivery.remote` span with `channel_id`, `message_id`, `backend_count`.

`deliverPending()` — `qhorus.delivery.pump` span with `channel_id`, `backend_id`, `cursor_position`, `batch_size`.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="FanOutTracingTest,DeliveryTracingTest"`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#197): add fan-out and delivery spans to ChannelGateway and DeliveryService

Refs #197
```

---

### Task 5: Ledger Write Span + Cross-Request Links — `LedgerWriteService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java` — inject `Tracer` + `QhorusTracingConfig`, add span to `record()`, add cross-request span links for terminal messages
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/LedgerWriteTracingTest.java`

**Interfaces:**
- Consumes: `QhorusTracingConfig` (Task 1), `MessageLedgerEntryRepository.findEarliestWithSubjectByCorrelationId()` (existing)
- Produces: `qhorus.ledger.write` span with `qhorus.ledger.entry_type`, `qhorus.ledger.channel_id`, `qhorus.ledger.message_id`, `qhorus.ledger.has_attestation`; span links to original COMMAND trace for terminal messages

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.qhorus.runtime.ledger;

import static org.assertj.core.api.Assertions.*;

import java.util.List;
import java.util.UUID;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class LedgerWriteTracingTest {

    private InMemorySpanExporter exporter;

    @BeforeEach
    void setUp() {
        exporter = InMemorySpanExporter.create();
    }

    @Test
    void record_creates_ledger_write_span() {
        // Arrange: wire LedgerWriteService with InMemory stubs
        // Act: service.record(dispatch, messageId, commitmentId, occurredAt)
        // Assert: "qhorus.ledger.write" span with entry_type, channel_id, message_id
    }

    @Test
    void terminal_message_adds_span_link_to_original_command_trace() {
        // Arrange: create a COMMAND ledger entry with traceId set
        // Act: record a DONE message with same correlationId
        // Assert: the DONE span has a link whose traceId matches the COMMAND entry's traceId
    }

    @Test
    void terminal_message_with_null_traceId_skips_link() {
        // Arrange: create a COMMAND entry with traceId = null
        // Act: record a DONE message
        // Assert: span has no links (link silently skipped)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerWriteTracingTest`
Expected: FAIL

- [ ] **Step 3: Implement ledger write span + cross-request links**

Add to `LedgerWriteService`:

```java
@Inject io.opentelemetry.api.trace.Tracer tracer;
@Inject QhorusTracingConfig tracingConfig;
```

In `record()`:

```java
public LedgerWriteOutcome record(MessageDispatch dispatch, Long messageId,
        UUID commitmentId, Instant occurredAt) {
    Span span = null;
    if (tracingConfig.enabled() && tracingConfig.ledgerWrite()) {
        SpanBuilder builder = tracer.spanBuilder("qhorus.ledger.write")
                .setSpanKind(SpanKind.INTERNAL);

        // Cross-request span link for terminal messages
        if (isTerminalType(dispatch.type()) && dispatch.correlationId() != null) {
            String tenancyId = dispatch.tenancyId();
            MessageLedgerEntry original = messageRepo
                    .findEarliestWithSubjectByCorrelationId(
                            dispatch.correlationId(), tenancyId)
                    .orElse(null);
            if (original != null && original.traceId != null) {
                SpanContext linkedContext = SpanContext.createFromRemoteParent(
                        original.traceId, "0000000000000000",
                        TraceFlags.getDefault(), TraceState.getDefault());
                builder.addLink(linkedContext);
            }
        }

        span = builder.startSpan();
        span.setAttribute("qhorus.ledger.entry_type", dispatch.type().name());
        span.setAttribute("qhorus.ledger.channel_id", dispatch.channelId().toString());
        span.setAttribute("qhorus.ledger.message_id", messageId.toString());
    }
    try {
        // existing body
        if (span != null) {
            span.setAttribute("qhorus.ledger.has_attestation", hasAttestation);
        }
        return outcome;
    } catch (Exception e) {
        if (span != null) { span.setStatus(StatusCode.ERROR); span.recordException(e); }
        throw e;
    } finally {
        if (span != null) span.end();
    }
}

private static boolean isTerminalType(MessageType type) {
    return type == MessageType.DONE || type == MessageType.FAILURE
            || type == MessageType.DECLINE;
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerWriteTracingTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#197): add ledger write span with cross-request links

Refs #197
```

---

### Task 6: Reactive Parity — `ReactiveMessageService`, `ReactiveCommitmentService`, `ReactiveLedgerWriteService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` — inject `Tracer` + `QhorusTracingConfig`, add dispatch span with Mutiny lifecycle pattern
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveCommitmentService.java` — add commitment transition spans
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` — add ledger write span + cross-request links

**Interfaces:**
- Consumes: `QhorusTracingConfig` (Task 1)
- Produces: same span names and attributes as blocking counterparts (Tasks 2, 3, 5) but with reactive span lifecycle pattern

- [ ] **Step 1: Write reactive dispatch tracing test**

Since reactive tests require PostgreSQL DevServices (all `@Disabled`), write CDI-free unit tests that verify the Mutiny operator ordering:

```java
package io.casehub.qhorus.runtime.message;

import static org.assertj.core.api.Assertions.*;

import io.opentelemetry.sdk.testing.exporter.InMemorySpanExporter;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;

import org.junit.jupiter.api.Test;

class ReactiveDispatchTracingTest {

    @Test
    void onFailure_before_onTermination_records_error_before_span_end() {
        // Verify the Mutiny operator ordering pattern:
        // onFailure fires first (records error), then onTermination (ends span)
        // This is a pattern test, not a full dispatch test.
    }
}
```

- [ ] **Step 2: Implement reactive dispatch span in `ReactiveMessageService.doDispatch()`**

Add field injections:

```java
@Inject io.opentelemetry.api.trace.Tracer tracer;
@Inject QhorusTracingConfig tracingConfig;
```

In `doDispatch()`, use the Mutiny lifecycle pattern from the spec:

```java
private Uni<DispatchResult> doDispatch(MessageDispatch dispatch) {
    Span span = null;
    Scope scope = null;
    if (tracingConfig.enabled() && tracingConfig.dispatch()) {
        span = tracer.spanBuilder("qhorus.dispatch")
                .setSpanKind(SpanKind.INTERNAL)
                .startSpan();
        scope = span.makeCurrent();
    }
    final Span finalSpan = span;
    final Scope finalScope = scope;

    return doDispatchInternal(dispatch)
            .invoke(result -> {
                if (finalSpan != null) {
                    // Set attributes from result
                }
            })
            .onFailure().invoke(t -> {
                if (finalSpan != null) {
                    finalSpan.setStatus(StatusCode.ERROR);
                    finalSpan.recordException(t);
                }
            })
            .onTermination().invoke(() -> {
                if (finalScope != null) finalScope.close();
                if (finalSpan != null) finalSpan.end();
            });
}
```

The key: `onFailure` is chained BEFORE `onTermination` so errors are recorded before the span ends.

- [ ] **Step 3: Implement reactive commitment and ledger write spans**

Same pattern in `ReactiveCommitmentService` (each transition method) and `ReactiveLedgerWriteService.record()`. Cross-request span links in the reactive ledger write use the existing `messageRepo.findEarliestWithSubjectByCorrelationId()` which returns `Uni` — chain the link construction in a `flatMap` before the span builder.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ReactiveDispatchTracingTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#197): add reactive span parity for dispatch, commitment, and ledger write

Refs #197
```

---

### Task 7: Full Build + Integration Verification

**Files:**
- Modify: `examples/type-system/pom.xml` — add `opentelemetry-sdk-testing` test-scope dependency
- No new test files in type-system (optional — add if time permits)

**Interfaces:**
- Consumes: all prior tasks
- Produces: clean full build

- [ ] **Step 1: Add test dependency to `examples/type-system/pom.xml`**

```xml
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-sdk-testing</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 3: Verify no regressions in existing tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All existing tests pass. No new test failures.

- [ ] **Step 4: Verify examples compile with new API**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -Pwith-llm-examples -f examples/agent-communication/pom.xml`
Expected: COMPILE SUCCESS

- [ ] **Step 5: Commit**

```
feat(#197): add opentelemetry-sdk-testing to type-system examples

Closes #197
```

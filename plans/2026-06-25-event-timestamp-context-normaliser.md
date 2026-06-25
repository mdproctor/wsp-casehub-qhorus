# Event Timestamp, Attestation Context, Per-Connector Normaliser — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix CloudEvent timestamp skew (#294), add capabilityTag to CommitmentContext for trust-gated attestation (#307), and enable per-connector InboundNormaliser dispatch (#216).

**Architecture:** Three sequential changes on one branch. Each touches distinct API records, runtime services, and modules — no file overlap. All three are API-breaking changes to records in `qhorus-api`; the break forces every consumer to be explicit.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5 + Mockito (CDI-free unit tests), AssertJ

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` from project root
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
- After API changes in `api/`, run full `mvn install` — child modules won't see changes from `mvn test -pl runtime`
- Use `mvn` not `./mvnw`
- Every commit references an issue: `Refs #N` (ongoing) or `Closes #N` (done)
- Use IntelliJ MCP for all semantic code navigation — never bash grep/find for classes
- TDD: write failing test → verify failure → implement → verify pass → commit

## Spec

`specs/issue-294-event-timestamp-context-normaliser/2026-06-25-event-timestamp-context-normaliser-design.md` (workspace)

---

### Task 1: #294 — Add `occurredAt` to MessageReceivedEvent and fix CloudEvent adapter

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/MessageReceivedEvent.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java:69-74`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java:304-314`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapter.java:84`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/QhorusCloudEventAdapterTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcherTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/message/QualifiedMessageObserverTest.java`

**Interfaces:**
- Produces: `MessageReceivedEvent(String channelName, UUID channelId, String tenancyId, MessageType messageType, String senderId, String correlationId, Instant occurredAt, String content)` — new 8-arg canonical constructor with `occurredAt` between `correlationId` and `content`

- [ ] **Step 1: Write failing test — adapter uses event timestamp, not now()**

Add a test to `QhorusCloudEventAdapterTest.java`:

```java
@Test
void onMessageReceived_usesOccurredAtForCloudEventTime() {
    UUID channelId = UUID.randomUUID();
    Instant fixedTime = Instant.parse("2026-01-15T10:30:00Z");
    MessageReceivedEvent event = new MessageReceivedEvent(
            "ch", channelId, "t1", MessageType.COMMAND,
            "agent:alice", UUID.randomUUID().toString(), fixedTime, "go");

    adapter.onMessageReceived(event);

    CloudEvent ce = captureCloudEvent();
    assertThat(ce.getTime()).isNotNull();
    assertThat(ce.getTime().toInstant()).isEqualTo(fixedTime);
}
```

This will not compile — `MessageReceivedEvent` doesn't have the `occurredAt` field yet.

- [ ] **Step 2: Add `occurredAt` to MessageReceivedEvent**

Replace the record definition in `api/src/main/java/io/casehub/qhorus/api/gateway/MessageReceivedEvent.java`:

```java
import java.time.Instant;
import java.util.Objects;
import java.util.UUID;

import io.casehub.qhorus.api.message.MessageType;

public record MessageReceivedEvent(
        String channelName,
        UUID channelId,
        String tenancyId,
        MessageType messageType,
        String senderId,
        String correlationId,
        Instant occurredAt,
        String content) {

    public MessageReceivedEvent {
        Objects.requireNonNull(occurredAt, "occurredAt");
        if (messageType == MessageType.EVENT && content != null) {
            throw new IllegalArgumentException(
                    "EVENT messages must have null content — Builder.build() enforces this at call-site");
        }
    }
}
```

- [ ] **Step 3: Fix all existing MessageReceivedEvent constructors in this repo**

Every existing `new MessageReceivedEvent(...)` call needs `Instant.now()` inserted as the 7th argument (after `correlationId`, before `content`). Files:

**MessageObserverDispatcher.java** (line 71 — production fix site): Replace:
```java
final MessageReceivedEvent event = new MessageReceivedEvent(
        channelName, channelId, tenancyId,
        message.messageType, message.sender,
        message.correlationId, content);
```
with:
```java
final Instant occurredAt = message.createdAt != null
        ? message.createdAt : Instant.now();
final MessageReceivedEvent event = new MessageReceivedEvent(
        channelName, channelId, tenancyId,
        message.messageType, message.sender,
        message.correlationId, occurredAt, content);
```

**ReactiveMessageService.java** (line 304-314 — synthetic Message): Add after line 313 (`syntheticMsg.target = dispatch.target();`):
```java
syntheticMsg.createdAt = ctx.occurredAt();
```

**QhorusCloudEventAdapterTest.java** — all 7 existing test methods construct `MessageReceivedEvent`. Insert `Instant.now()` as the 7th arg in each. Example for `onMessageReceived_command_firesCloudEventWithCorrectType`:
```java
MessageReceivedEvent event = new MessageReceivedEvent(
        "my-channel", channelId, "tenant-1", MessageType.COMMAND,
        "agent:alice", UUID.randomUUID().toString(), Instant.now(), "hello");
```

Apply the same pattern to all 7 test methods (lines 44, 57, 70, 83, 96, 109, 122, 135).

**MessageObserverDispatcherTest.java** — all constructor sites. Insert `Instant.now()` as the 7th arg. The two `assertThrows` sites at lines 373 and 380 also need the arg.

**QualifiedMessageObserverTest.java** — does not directly construct `MessageReceivedEvent` (uses `List<MessageReceivedEvent>` receivers). No changes needed.

- [ ] **Step 4: Fix the CloudEvent adapter**

In `QhorusCloudEventAdapter.java`, replace line 84:
```java
.withTime(OffsetDateTime.now(ZoneOffset.UTC))
```
with:
```java
.withTime(event.occurredAt().atOffset(ZoneOffset.UTC))
```

Remove the `OffsetDateTime` import if no longer used.

- [ ] **Step 5: Run tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests pass, including the new `onMessageReceived_usesOccurredAtForCloudEventTime`.

- [ ] **Step 6: Compile-check downstream modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`

This catches compile errors in `connector-backend/`, `slack-channel/`, and `examples/` modules that may construct `MessageReceivedEvent`. Fix any that surface — same pattern (insert `Instant.now()` as 7th arg).

- [ ] **Step 7: Commit**

```
git add -A
git commit -m "fix(#294): use message persist time in CloudEvent adapter, not now()

Add Instant occurredAt to MessageReceivedEvent (requireNonNull in compact
constructor). Populate from Message.createdAt in MessageObserverDispatcher.
Fix reactive path: syntheticMsg.createdAt = ctx.occurredAt().
QhorusCloudEventAdapter uses event.occurredAt() instead of OffsetDateTime.now().

Refs #294"
```

---

### Task 2: #307 — Add `capabilityTag` to CommitmentContext and remove 2-arg overload

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/spi/CommitmentContext.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/spi/CommitmentAttestationPolicy.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java:196-224`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java:170-197`
- Modify: `runtime/src/test/java/io/casehub/qhorus/ledger/CommitmentAttestationPolicyTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/audit/EvidentialCheckerObligationTest.java`
- Modify: `docs/protocols/casehub/commitment-attestation-policy-null-context.md`

**Interfaces:**
- Produces: `CommitmentContext(String correlationId, UUID channelId, String channelName, UUID commitmentId, String capabilityTag)` — new 5-arg canonical constructor
- Produces: `CommitmentAttestationPolicy` interface — 3-arg `attestationFor` only (2-arg overload removed)

- [ ] **Step 1: Write failing test — policy receives capabilityTag in context**

Add a test to `CommitmentAttestationPolicyTest.java`:

```java
@Test
void attestationFor_receivesCapabilityTag_inContext() {
    CommitmentAttestationPolicy p = (type, actorId, ctx) -> {
        assertNotNull(ctx);
        assertEquals("medical-review", ctx.capabilityTag());
        return Optional.of(new AttestationOutcome(
                AttestationVerdict.SOUND, 0.9, actorId, ActorType.AGENT));
    };
    var ctx = new CommitmentContext("corr-1", UUID.randomUUID(), "ch", UUID.randomUUID(), "medical-review");
    var result = p.attestationFor(MessageType.DONE, "agent-a", ctx);
    assertTrue(result.isPresent());
    assertEquals("medical-review", ctx.capabilityTag());
}
```

This will not compile — `CommitmentContext` doesn't have the `capabilityTag` field yet.

- [ ] **Step 2: Add `capabilityTag` to CommitmentContext**

Replace the record definition in `api/src/main/java/io/casehub/qhorus/api/spi/CommitmentContext.java`:

```java
public record CommitmentContext(
        String correlationId,
        UUID channelId,
        String channelName,
        UUID commitmentId,
        String capabilityTag
) {}
```

Update the Javadoc to document `capabilityTag`: extracted from the COMMAND content's `"capability"` JSON field; null or `CapabilityTag.GLOBAL` when not available.

- [ ] **Step 3: Remove the 2-arg overload from CommitmentAttestationPolicy**

In `api/src/main/java/io/casehub/qhorus/api/spi/CommitmentAttestationPolicy.java`, delete the entire default method:

```java
default Optional<AttestationOutcome> attestationFor(MessageType terminalType,
        String resolvedActorId) {
    return attestationFor(terminalType, resolvedActorId, null);
}
```

Remove its Javadoc.

- [ ] **Step 4: Fix all CommitmentContext constructors and attestationFor callers**

**LedgerWriteService.java** (lines 196-224): Restructure `writeAttestation` — extract capabilityTag before constructing CommitmentContext, set attestation.capabilityTag from context:

```java
private void writeAttestation(final UUID subjectId, final MessageLedgerEntry commandEntry,
        final MessageType terminalType, final String resolvedActorId, final String tenancyId,
        final CommitmentContext context) {
    attestationPolicy.attestationFor(terminalType, resolvedActorId, context).ifPresent(outcome -> {
        try {
            final LedgerAttestation attestation = new LedgerAttestation();
            attestation.ledgerEntryId = commandEntry.id;
            attestation.subjectId = subjectId;
            attestation.attestorId = outcome.attestorId();
            attestation.attestorType = outcome.attestorType();
            attestation.verdict = outcome.verdict();
            attestation.confidence = outcome.confidence();
            attestation.capabilityTag = context.capabilityTag();
            ledger.saveAttestation(attestation, tenancyId);
            LOG.debugf("LedgerAttestation %s written for COMMAND entry %s (correlationId='%s', capability='%s')",
                    attestation.verdict, commandEntry.id, commandEntry.correlationId, attestation.capabilityTag);
        } catch (final Exception e) {
            LOG.warnf("Could not write attestation for entry %s — trust signal lost but pipeline unaffected",
                    commandEntry.id);
        }
    });
}
```

And at the call site (around line 200), extract capabilityTag and pass it in CommitmentContext:

```java
final String capabilityTag = extractCapabilityTag(priorMsg.content);
final CommitmentContext ctx = new CommitmentContext(
        priorMsg.correlationId, priorMsg.channelId, null, commitmentId, capabilityTag);
writeAttestation(resolvedSubjectId, priorMsg, dispatch.type(), resolvedActorId,
        tenancyId, ctx);
```

**ReactiveLedgerWriteService.java** (lines 183-197): Same restructuring. At line 183:

```java
final String capabilityTag = extractCapabilityTag(prior.content);
final CommitmentContext ctx = new CommitmentContext(
        prior.correlationId, prior.channelId, null, commitmentId, capabilityTag);
```

And at line 197, change:
```java
attestation.capabilityTag = extractCapabilityTag(prior.content);
```
to:
```java
attestation.capabilityTag = ctx.capabilityTag();
```

**EvidentialCheckerObligationTest.java** (line 23): Add 5th arg:
```java
private final CommitmentContext ctx = new CommitmentContext(
        "corr-1", UUID.randomUUID(), "test-channel", UUID.randomUUID(), null);
```

**CommitmentAttestationPolicyTest.java**: Migrate all 13 test sites from 2-arg to 3-arg form. Delete `twoArgDefault_delegatesToThreeArg_withNullContext` test. Example:

```java
// Before:
var result = policyWithDefaults().attestationFor(MessageType.DONE, "agent-a");
// After:
var result = policyWithDefaults().attestationFor(MessageType.DONE, "agent-a", null);
```

For `lambda_threeArgForm_compiles` (line 31), change:
```java
Optional<AttestationOutcome> result = p.attestationFor(MessageType.DONE, "agent-a");
```
to:
```java
Optional<AttestationOutcome> result = p.attestationFor(MessageType.DONE, "agent-a", null);
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`

Expected: All tests pass. Full install catches any downstream `CommitmentContext` or `attestationFor` callers.

- [ ] **Step 6: Update protocol PP-20260623-77adf0**

In `docs/protocols/casehub/commitment-attestation-policy-null-context.md`:
- Remove reference to the 2-arg overload (it no longer exists)
- Add `capabilityTag` to the field list: "nullable — implementations should treat null capabilityTag as global scope (CapabilityTag.GLOBAL)"
- Keep the null-context defensive guidance (3-arg context parameter is still nullable)

- [ ] **Step 7: Commit**

```
git add -A
git commit -m "feat(#307): add capabilityTag to CommitmentContext, remove 2-arg overload

Add String capabilityTag as 5th field in CommitmentContext. Extract from
COMMAND content BEFORE attestationFor() call — single extraction, single
source of truth. LedgerAttestation.capabilityTag set from ctx.capabilityTag().

Remove 2-arg attestationFor default method (zero production callers).
13 test sites migrated to 3-arg form with null context.

Update protocol PP-20260623-77adf0.

Refs #307"
```

---

### Task 3: #216 — Per-connector InboundNormaliser via `normaliserFor(UUID channelId)`

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/HumanParticipatingChannelBackend.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java:120-121`
- Modify: `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java:84-85`
- Create: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorNormaliser.java`
- Modify: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java:249,271,321`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayIntegrationTest.java:64`
- Create: `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorNormaliserDispatchTest.java`

**Interfaces:**
- Produces: `HumanParticipatingChannelBackend.normaliserFor(UUID channelId)` — replaces `normaliser()`
- Produces: `ConnectorNormaliser extends InboundNormaliser` with `String connectorId()` — new SPI in connector-backend

- [ ] **Step 1: Write failing test — ConnectorChannelBackend dispatches normaliser by connector type**

Create `connector-backend/src/test/java/io/casehub/qhorus/connector/backend/ConnectorNormaliserDispatchTest.java`:

```java
package io.casehub.qhorus.connector.backend;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.connectors.InboundConnectorIds;
import io.casehub.qhorus.api.gateway.*;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelConnectorBinding;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.connectors.ConnectorService;
import io.casehub.qhorus.runtime.store.ChannelBindingStore;
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;

class ConnectorNormaliserDispatchTest {

    private ConnectorChannelBackend backend;
    private ChannelBindingStore bindingStore;

    @BeforeEach
    void setUp() {
        ChannelGateway gateway = mock(ChannelGateway.class);
        ChannelService channelService = mock(ChannelService.class);
        bindingStore = mock(ChannelBindingStore.class);
        ConnectorService connectorService = mock(ConnectorService.class);
        AutoChannelPolicy autoChannelPolicy = mock(AutoChannelPolicy.class);
        backend = new ConnectorChannelBackend(gateway, channelService, bindingStore,
                connectorService, new SimpleMeterRegistry(), autoChannelPolicy);
    }

    @Test
    void normaliserFor_returnsNull_whenNoCacheEntry() {
        assertThat(backend.normaliserFor(UUID.randomUUID())).isNull();
    }

    @Test
    void normaliserFor_returnsNull_whenNoConnectorNormaliserRegistered() {
        UUID channelId = UUID.randomUUID();
        ChannelConnectorBinding b = new ChannelConnectorBinding();
        b.channelId = channelId;
        b.inboundConnectorId = InboundConnectorIds.TWILIO_SMS;
        b.externalKey = "+1111";
        b.outboundConnectorId = "twilio-sms";
        b.outboundDestination = "+9999";
        when(bindingStore.findByChannelId(channelId)).thenReturn(Optional.of(b));
        backend.onChannelInitialised(new ChannelInitialisedEvent(channelId, "sms-channel", false));

        assertThat(backend.normaliserFor(channelId)).isNull();
    }
}
```

This will not compile — `ConnectorChannelBackend.normaliserFor(UUID)` doesn't exist yet.

- [ ] **Step 2: Change HumanParticipatingChannelBackend SPI**

In `api/src/main/java/io/casehub/qhorus/api/gateway/HumanParticipatingChannelBackend.java`, replace:
```java
default InboundNormaliser normaliser() { return null; }
```
with:
```java
default InboundNormaliser normaliserFor(UUID channelId) { return null; }
```

Add import: `import java.util.UUID;`

Update the Javadoc to document the parameter and the UUID-vs-ChannelRef rationale.

- [ ] **Step 3: Fix ChannelGateway.registerBackend()**

In `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`, line 120-121, replace:
```java
InboundNormaliser backendNormaliser = (backend instanceof HumanParticipatingChannelBackend hb)
        ? hb.normaliser() : null;
```
with:
```java
InboundNormaliser backendNormaliser = (backend instanceof HumanParticipatingChannelBackend hb)
        ? hb.normaliserFor(channelId) : null;
```

- [ ] **Step 4: Fix SlackChannelBackend**

In `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java`, line 84-85, replace:
```java
@Override
public InboundNormaliser normaliser() {
    return slackInboundNormaliser;
}
```
with:
```java
@Override
public InboundNormaliser normaliserFor(UUID channelId) {
    return slackInboundNormaliser;
}
```

- [ ] **Step 5: Fix test inline backends**

**ChannelGatewayTest.java line 249**: Replace `@Override public InboundNormaliser normaliser()` with `@Override public InboundNormaliser normaliserFor(UUID channelId)`.

**ChannelGatewayTest.java line 321**: Same rename.

**ChannelGatewayTest.java line 271**: Update comment from `normaliser()` to `normaliserFor(UUID)`.

**ChannelGatewayIntegrationTest.java line 64**: Same rename as above.

- [ ] **Step 6: Create ConnectorNormaliser interface**

Create `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorNormaliser.java`:

```java
package io.casehub.qhorus.connector.backend;

import io.casehub.qhorus.api.gateway.InboundNormaliser;

/**
 * A normaliser scoped to a specific inbound connector.
 *
 * <p>Implementations are {@code @ApplicationScoped} CDI beans discovered by
 * {@link ConnectorChannelBackend} at startup and dispatched based on the
 * channel's connector binding.
 *
 * <p>Keyed on connector ID (not connector type) to match the existing
 * {@link ConnectorKeyStrategy} pattern and the binding's
 * {@code inboundConnectorId} field.
 */
public interface ConnectorNormaliser extends InboundNormaliser {

    /**
     * The connector ID this normaliser handles (e.g. {@code "email-inbound"}).
     * Must match a value from {@link io.casehub.connectors.InboundConnectorIds}.
     */
    String connectorId();
}
```

- [ ] **Step 7: Add ConnectorNormaliser dispatch to ConnectorChannelBackend**

Add to `ConnectorChannelBackend.java`:

Import:
```java
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
```

New fields:
```java
@Inject @Any
Instance<ConnectorNormaliser> connectorNormalisers;

private Map<String, ConnectorNormaliser> normalisersByConnectorId = Map.of();
```

New `@PostConstruct` method:
```java
@PostConstruct
void buildNormaliserRegistry() {
    final Map<String, ConnectorNormaliser> map = new HashMap<>();
    for (final ConnectorNormaliser cn : connectorNormalisers) {
        final String id = cn.connectorId();
        if (id == null || id.isBlank()) {
            throw new IllegalStateException(
                    cn.getClass().getName() + ".connectorId() returned null or blank");
        }
        if (map.put(id, cn) != null) {
            throw new IllegalStateException(
                    "Duplicate ConnectorNormaliser for connectorId '" + id + "' — "
                    + "each connector must have at most one normaliser");
        }
    }
    normalisersByConnectorId = Collections.unmodifiableMap(map);
}
```

Override the new SPI method:
```java
@Override
public InboundNormaliser normaliserFor(UUID channelId) {
    CacheEntry entry = cache.get(channelId);
    if (entry == null) {
        return null;
    }
    return normalisersByConnectorId.get(entry.inboundConnectorId());
}
```

- [ ] **Step 8: Add correlationId passthrough in route()**

In `ConnectorChannelBackend.route()`, replace:
```java
private void route(Channel channel, InboundMessage msg) {
    gateway.receiveHumanMessage(
            new ChannelRef(channel.id, channel.name),
            new InboundHumanMessage(
                    msg.externalSenderId(),
                    msg.content(),
                    msg.receivedAt(),
                    msg.metadata(),
                    null,
                    null));
}
```
with:
```java
private void route(Channel channel, InboundMessage msg) {
    String correlationId = msg.metadata() != null
            ? msg.metadata().get("correlation-id") : null;
    gateway.receiveHumanMessage(
            new ChannelRef(channel.id, channel.name),
            new InboundHumanMessage(
                    msg.externalSenderId(),
                    msg.content(),
                    msg.receivedAt(),
                    msg.metadata(),
                    correlationId,
                    null));
}
```

- [ ] **Step 9: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`

Expected: All tests pass across all modules.

- [ ] **Step 10: Commit**

```
git add -A
git commit -m "feat(#216): per-connector InboundNormaliser via normaliserFor(UUID channelId)

Rename HumanParticipatingChannelBackend.normaliser() to normaliserFor(UUID)
so backends can return different normalisers per channel.

Add ConnectorNormaliser SPI in connector-backend — CDI beans declare affinity
via connectorId(). ConnectorChannelBackend builds dispatch map at @PostConstruct
with duplicate detection (ProjectionRegistry pattern).

Pass through correlation-id from InboundMessage metadata in route().

Refs #216"
```

---

## Post-implementation

After all three tasks pass:

- [ ] Run full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- [ ] Run `mvn test-compile -Pwith-llm-examples -f examples/agent-communication/pom.xml` to catch any stale API sites in profile-gated modules
- [ ] Verify downstream consumer compile: check casehub-engine and claudony `MessageReceivedEvent` sites are noted for cross-repo follow-up

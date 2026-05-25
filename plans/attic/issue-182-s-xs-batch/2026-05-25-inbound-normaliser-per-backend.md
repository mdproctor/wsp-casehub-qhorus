# Per-Backend InboundNormaliser + InboundHumanMessage.inReplyTo

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `DefaultInboundNormaliser` to support per-backend normalisation and `inReplyTo` pass-through, enabling human reply-type messages (RESPONSE, DONE, DECLINE, FAILURE) to correctly fulfil commitments.

**Architecture:** Add `Long inReplyTo` to `InboundHumanMessage`. Add `default InboundNormaliser normaliser()` to `HumanParticipatingChannelBackend`. Update `BackendEntry` to carry the normaliser; `ChannelGateway.receiveHumanMessage()` uses the backend's normaliser when non-null, falls back to the injected `DefaultInboundNormaliser`. `DefaultInboundNormaliser` gains metadata key `message-type` for type override and passes `inReplyTo` through.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, Mockito, AssertJ. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format`. After api changes: `mvn install -pl api -Dno-format` before running runtime tests.

**Spec:** `specs/2026-05-25-inbound-normaliser-per-backend-design.md`
**Issues:** Closes #158, closes #159 (inReplyTo part)

---

## File Map

| File | Action | What changes |
|------|--------|--------------|
| `api/src/main/java/io/casehub/qhorus/api/gateway/InboundHumanMessage.java` | Modify | Add `Long inReplyTo` as 6th record component |
| `api/src/main/java/io/casehub/qhorus/api/gateway/HumanParticipatingChannelBackend.java` | Modify | Add `default InboundNormaliser normaliser() { return null; }` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java` | Modify | Add `message-type` metadata key + `inReplyTo` pass-through |
| `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` | Modify | `BackendEntry` gets `normaliser`; `registerBackend` extracts it; `receiveHumanMessage` uses effective normaliser |
| `runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java` | Modify | New tests for metadata-type and inReplyTo; rename `normalise_alwaysReturnsQuery`; update all call sites |
| `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java` | Modify | New normaliser-selection tests; update all `InboundHumanMessage` call sites |
| `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayIntegrationTest.java` | Create | `@QuarkusTest` — RESPONSE through `receiveHumanMessage` fulfils Commitment |

---

## Task 1: Extend InboundHumanMessage + fix all call sites

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/InboundHumanMessage.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java` (6 call sites)
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java` (2 call sites)

- [ ] **Step 1: Write failing test (compilation fail = test fail)**

Add this test to `DefaultInboundNormaliserTest` after the existing tests:

```java
@Test
void normalise_passes_inReplyTo_from_InboundHumanMessage() {
    var raw = new InboundHumanMessage("user-42", "done", Instant.now(), Map.of(), "corr-1", 99L);
    assertThat(normaliser.normalise(channel, raw).inReplyTo()).isEqualTo(99L);
}
```

- [ ] **Step 2: Run to confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=DefaultInboundNormaliserTest 2>&1 | grep "ERROR\|cannot find"
```

Expected: `cannot find symbol` — `InboundHumanMessage` has no `inReplyTo`.

- [ ] **Step 3: Add inReplyTo to InboundHumanMessage**

Replace `api/src/main/java/io/casehub/qhorus/api/gateway/InboundHumanMessage.java` with:

```java
package io.casehub.qhorus.api.gateway;

import java.time.Instant;
import java.util.Map;

public record InboundHumanMessage(
        String externalSenderId,
        String content,
        Instant receivedAt,
        Map<String, String> metadata,
        String correlationId,
        Long inReplyTo) {}
```

- [ ] **Step 4: Install the updated API module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -Dno-format -DskipTests
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Fix the 6 call sites in DefaultInboundNormaliserTest**

Every `new InboundHumanMessage(...)` in this file has 5 args. Add `null` as the 6th arg to all of them. The file is at `runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java`.

Change every occurrence of this pattern (6 total):
```java
new InboundHumanMessage("...", "...", Instant.now(), Map.of(), ...)
```
to include `null` as the final arg:
```java
new InboundHumanMessage("...", "...", Instant.now(), Map.of(), ..., null)
```

Exact replacements:
```java
// Line 24 — normalise_alwaysReturnsQuery
var raw = new InboundHumanMessage("user-42", "Please analyse this", Instant.now(), Map.of(), null, null);

// Line 30 — normalise_preservesContent
var raw = new InboundHumanMessage("user-42", "Hello agent!", Instant.now(), Map.of(), null, null);

// Line 36 — normalise_senderIdPrefixedWithHuman
var raw = new InboundHumanMessage("+447911123456", "stop", Instant.now(), Map.of(), null, null);

// Line 42 — normalise_emptyContent_stillReturnsQuery
var raw = new InboundHumanMessage("user-1", "", Instant.now(), Map.of(), null, null);

// Line 50 — normalise_withCorrelationId_passesThrough
var raw = new InboundHumanMessage("user-42", "approved", Instant.now(), Map.of(), "corr-99", null);

// Line 56 — normalise_nullCorrelationId_propagatesNull
var raw = new InboundHumanMessage("user-42", "hello", Instant.now(), Map.of(), null, null);

// Line 62 — normalise_remainingNullableFields_areNull
var raw = new InboundHumanMessage("user-42", "hello", Instant.now(), Map.of(), null, null);
```

- [ ] **Step 6: Fix the 2 call sites in ChannelGatewayTest**

File: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java`

```java
// receiveHumanMessage_callsMessageServiceDispatchWithHumanSender (~line 203)
InboundHumanMessage raw = new InboundHumanMessage(
        "user-42", "Can you stop?", Instant.now(), Map.of(), null, null);

// receiveHumanMessage_withCorrelationId_passesCorrelationIdToMessageService (~line 220)
InboundHumanMessage raw = new InboundHumanMessage(
        "user-42", "approved", Instant.now(), Map.of(), "corr-abc", null);
```

- [ ] **Step 7: Run tests — new test still fails (normaliser not wired yet), others compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=DefaultInboundNormaliserTest
```

Expected: `normalise_passes_inReplyTo_from_InboundHumanMessage` FAILS (returns null), all others PASS.

- [ ] **Step 8: Commit the compilation fix only**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  api/src/main/java/io/casehub/qhorus/api/gateway/InboundHumanMessage.java \
  runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#158): add inReplyTo to InboundHumanMessage; update call sites

Refs #158, Refs #159.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Add normaliser() to HumanParticipatingChannelBackend

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/HumanParticipatingChannelBackend.java`

- [ ] **Step 1: Add the default method**

Replace the file with:

```java
package io.casehub.qhorus.api.gateway;

/**
 * At most one per channel. Full speech act inbound via {@link InboundNormaliser}.
 * {@code actorType()} must return {@code ActorType.HUMAN}.
 * Call {@code gateway.receiveHumanMessage()} when inbound arrives.
 *
 * <p>{@code post()} must catch all exceptions internally — failure is non-fatal;
 * the gateway logs and continues.
 *
 * <p>Override {@link #normaliser()} to provide channel-specific type inference.
 * Return {@code null} (the default) to use the system {@link DefaultInboundNormaliser}.
 */
public interface HumanParticipatingChannelBackend extends ChannelBackend {

    /**
     * Returns the {@link InboundNormaliser} for messages received from this backend,
     * or {@code null} to use the system default normaliser.
     *
     * <p>The normaliser converts raw prose input (an {@link InboundHumanMessage}) into
     * a typed {@link NormalisedMessage}. Backends that know the message type and
     * reply context (e.g. a UI with explicit Reply / New Message controls) should
     * override this and return a normaliser that reads from the message's fields
     * rather than inferring from metadata.
     */
    default InboundNormaliser normaliser() { return null; }
}
```

- [ ] **Step 2: Re-install the API module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -Dno-format -DskipTests
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3: Run runtime tests to confirm nothing broke**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=DefaultInboundNormaliserTest,ChannelGatewayTest
```

Expected: Same pass/fail as Task 1 Step 7 (no new failures).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  api/src/main/java/io/casehub/qhorus/api/gateway/HumanParticipatingChannelBackend.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#158): add normaliser() default to HumanParticipatingChannelBackend

Refs #158.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: TDD — DefaultInboundNormaliser (metadata type + inReplyTo pass-through)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java`

- [ ] **Step 1: Write failing tests**

Add these tests to `DefaultInboundNormaliserTest`. The `normalise_passes_inReplyTo_from_InboundHumanMessage` test from Task 1 is already there.

Add these two new tests after it:

```java
@Test
void normalise_uses_message_type_from_metadata() {
    var raw = new InboundHumanMessage("user-42", "ok done",
            Instant.now(), Map.of("message-type", "RESPONSE"), "corr-1", null);
    assertThat(normaliser.normalise(channel, raw).type()).isEqualTo(MessageType.RESPONSE);
}

@Test
void normalise_ignores_invalid_message_type_key_falls_back_to_QUERY() {
    var raw = new InboundHumanMessage("user-42", "ok",
            Instant.now(), Map.of("message-type", "NOT_A_TYPE"), null, null);
    assertThat(normaliser.normalise(channel, raw).type()).isEqualTo(MessageType.QUERY);
}
```

Also rename the existing `normalise_alwaysReturnsQuery` to `normalise_returns_QUERY_when_no_metadata_key` to reflect the new semantics:

```java
@Test
void normalise_returns_QUERY_when_no_metadata_key() {
    var raw = new InboundHumanMessage("user-42", "Please analyse this", Instant.now(), Map.of(), null, null);
    assertEquals(MessageType.QUERY, normaliser.normalise(channel, raw).type());
}
```

Also update `normalise_remainingNullableFields_areNull` — `inReplyTo` now passes through (null input → null output), so the assertion remains correct but rename it for clarity:

```java
@Test
void normalise_null_inReplyTo_propagates_null() {
    var raw = new InboundHumanMessage("user-42", "hello", Instant.now(), Map.of(), null, null);
    NormalisedMessage result = normaliser.normalise(channel, raw);
    assertNull(result.inReplyTo());
    assertNull(result.artefactRefs());
    assertNull(result.target());
}
```

- [ ] **Step 2: Run to confirm 3 tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=DefaultInboundNormaliserTest
```

Expected: 3 FAIL (`normalise_uses_message_type_from_metadata`, `normalise_ignores_invalid_message_type_key_falls_back_to_QUERY`, `normalise_passes_inReplyTo_from_InboundHumanMessage`), rest PASS.

- [ ] **Step 3: Implement DefaultInboundNormaliser**

Replace `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java` with:

```java
package io.casehub.qhorus.runtime.gateway;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkus.arc.DefaultBean;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.api.gateway.InboundNormaliser;
import io.casehub.qhorus.api.gateway.NormalisedMessage;
import io.casehub.qhorus.api.message.MessageType;

@DefaultBean
@ApplicationScoped
public class DefaultInboundNormaliser implements InboundNormaliser {

    @Override
    public NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw) {
        return new NormalisedMessage(
                parseType(raw.metadata().get("message-type")),
                raw.content(),
                "human:" + raw.externalSenderId(),
                raw.correlationId(),
                raw.inReplyTo(),
                null,
                null);
    }

    private static MessageType parseType(String value) {
        if (value == null || value.isBlank()) return MessageType.QUERY;
        try {
            return MessageType.valueOf(value.toUpperCase());
        } catch (IllegalArgumentException e) {
            return MessageType.QUERY;
        }
    }
}
```

- [ ] **Step 4: Run tests to confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=DefaultInboundNormaliserTest
```

Expected: All 9 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java \
  runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#158): DefaultInboundNormaliser — message-type metadata key + inReplyTo pass-through

Reads metadata[\"message-type\"] for type override (falls back to QUERY on absent/invalid).
Passes InboundHumanMessage.inReplyTo() through to NormalisedMessage.inReplyTo().

Refs #158.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: TDD — ChannelGateway per-backend normaliser selection

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`

- [ ] **Step 1: Write failing tests**

Add these imports to `ChannelGatewayTest.java` if not already present:

```java
import static org.assertj.core.api.Assertions.assertThat;
import org.mockito.ArgumentCaptor;
import io.casehub.qhorus.api.gateway.HumanParticipatingChannelBackend;
import io.casehub.qhorus.api.gateway.InboundNormaliser;
import io.casehub.qhorus.api.gateway.NormalisedMessage;
```

Add these two tests in the `// ── Inbound` section:

```java
@Test
void receiveHumanMessage_uses_backend_normaliser_when_provided() {
    InboundNormaliser customNormaliser = (ch, raw) -> new NormalisedMessage(
            MessageType.RESPONSE, raw.content(),
            "human:" + raw.externalSenderId(),
            raw.correlationId(), raw.inReplyTo(), null, null);

    HumanParticipatingChannelBackend customBackend = new HumanParticipatingChannelBackend() {
        @Override public String backendId()  { return "custom-backend"; }
        @Override public ActorType actorType() { return ActorType.HUMAN; }
        @Override public void open(ChannelRef ch, Map<String, String> m) {}
        @Override public void post(ChannelRef ch, OutboundMessage msg) {}
        @Override public void close(ChannelRef ch) {}
        @Override public InboundNormaliser normaliser() { return customNormaliser; }
    };
    gateway.registerBackend(channelId, customBackend, "human_participating");

    InboundHumanMessage raw = new InboundHumanMessage(
            "user-1", "task complete", Instant.now(), Map.of(), "corr-42", 99L);
    gateway.receiveHumanMessage(channelRef, raw);

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.RESPONSE);
    assertThat(captor.getValue().inReplyTo()).isEqualTo(99L);
}

@Test
void receiveHumanMessage_falls_back_to_system_default_when_backend_normaliser_is_null() {
    HumanParticipatingChannelBackend nullNormaliserBackend = new HumanParticipatingChannelBackend() {
        @Override public String backendId()  { return "null-normaliser-backend"; }
        @Override public ActorType actorType() { return ActorType.HUMAN; }
        @Override public void open(ChannelRef ch, Map<String, String> m) {}
        @Override public void post(ChannelRef ch, OutboundMessage msg) {}
        @Override public void close(ChannelRef ch) {}
        // normaliser() returns null (default) — system default should be used
    };
    gateway.registerBackend(channelId, nullNormaliserBackend, "human_participating");

    InboundHumanMessage raw = new InboundHumanMessage(
            "user-1", "hello", Instant.now(), Map.of(), null, null);
    gateway.receiveHumanMessage(channelRef, raw);

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.QUERY); // DefaultInboundNormaliser
}
```

- [ ] **Step 2: Run to confirm 2 tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=ChannelGatewayTest
```

Expected: `receiveHumanMessage_uses_backend_normaliser_when_provided` and `receiveHumanMessage_falls_back_to_system_default_when_backend_normaliser_is_null` FAIL (gateway ignores the backend's normaliser). All other tests PASS.

- [ ] **Step 3: Update BackendEntry and ChannelGateway**

In `ChannelGateway.java`, make these four changes:

**Change 1 — BackendEntry (line 196):**
```java
record BackendEntry(ChannelBackend backend, String backendType, InboundNormaliser normaliser) {}
```

**Change 2 — initChannel (line 64):**
```java
entries.add(new BackendEntry(agentBackend, "agent", null));
```

**Change 3 — registerBackend (lines 101-118), replace the entire method:**
```java
public void registerBackend(UUID channelId, ChannelBackend backend, String backendType) {
    List<BackendEntry> entries = registry.computeIfAbsent(channelId,
            id -> Collections.synchronizedList(new ArrayList<>()));
    InboundNormaliser normaliser = (backend instanceof HumanParticipatingChannelBackend hb)
            ? hb.normaliser() : null;
    if ("human_participating".equals(backendType)) {
        synchronized (entries) {
            entries.stream()
                    .filter(e -> "human_participating".equals(e.backendType()))
                    .findFirst()
                    .ifPresent(existing -> {
                        throw new DuplicateParticipatingBackendException(
                                channelId.toString(), existing.backend().backendId());
                    });
            entries.add(new BackendEntry(backend, backendType, normaliser));
        }
    } else {
        entries.add(new BackendEntry(backend, backendType, normaliser));
    }
}
```

**Change 4 — receiveHumanMessage (line 165), replace the method:**
```java
/** Inbound from HumanParticipatingChannelBackend. */
public void receiveHumanMessage(ChannelRef channel, InboundHumanMessage raw) {
    InboundNormaliser effective = registry.getOrDefault(channel.id(), List.of()).stream()
            .filter(e -> "human_participating".equals(e.backendType()))
            .map(BackendEntry::normaliser)
            .filter(Objects::nonNull)
            .findFirst()
            .orElse(this.normaliser);
    NormalisedMessage n = effective.normalise(channel, raw);
    // Uses canonical constructor to bypass builder protocol validation —
    // inbound human messages may carry reply types (DONE/RESPONSE/etc.) with inReplyTo
    // synthesised by the normaliser from human context.
    messageService.dispatch(new MessageDispatch(
            channel.id(),
            n.senderInstanceId(),
            n.type(),
            n.content(),
            n.correlationId(),
            n.inReplyTo(),
            n.artefactRefs(),
            n.target(),
            null, // subjectId
            null, // causedByEntryId
            ActorType.HUMAN,
            null)); // deadline
}
```

Also add `import io.casehub.qhorus.api.gateway.HumanParticipatingChannelBackend;` to the imports at the top of `ChannelGateway.java` if not already present via the wildcard `import io.casehub.qhorus.api.gateway.*;`.

- [ ] **Step 4: Run ChannelGatewayTest — all should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=ChannelGatewayTest
```

Expected: All tests PASS including the 2 new ones.

- [ ] **Step 5: Run full runtime suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format
```

Expected: All tests PASS, 0 failures.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java \
  runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#158): per-backend normaliser in ChannelGateway

BackendEntry gains InboundNormaliser field. registerBackend() extracts it from
HumanParticipatingChannelBackend.normaliser() at registration time. receiveHumanMessage()
picks the backend's normaliser when non-null; falls back to injected DefaultInboundNormaliser.

Refs #158.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Integration test — RESPONSE fulfils Commitment

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayIntegrationTest.java`

This test verifies the full end-to-end scenario that was previously impossible: a human responding to a COMMAND via `receiveHumanMessage` correctly fulfils the Commitment.

- [ ] **Step 1: Create the test file**

```java
package io.casehub.qhorus.runtime.gateway;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.HumanParticipatingChannelBackend;
import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.api.gateway.InboundNormaliser;
import io.casehub.qhorus.api.gateway.NormalisedMessage;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.message.CommitmentService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.store.ChannelStore;

@QuarkusTest
class ChannelGatewayIntegrationTest {

    @Inject ChannelGateway channelGateway;
    @Inject MessageService messageService;
    @Inject CommitmentService commitmentService;
    @Inject ChannelStore channelStore;

    /**
     * Verifies the end-to-end scenario enabled by per-backend normalisation:
     * a human RESPONSE (via receiveHumanMessage with a custom normaliser carrying inReplyTo)
     * correctly fulfils the Commitment opened by the preceding COMMAND.
     */
    @Test @TestTransaction
    void receiveHumanMessage_RESPONSE_with_inReplyTo_fulfils_commitment() {
        String channelName = "normaliser-int-test-" + UUID.randomUUID();
        UUID channelId = createChannel(channelName);
        channelGateway.initChannel(channelId, new ChannelRef(channelId, channelName));

        String corrId = "corr-int-" + UUID.randomUUID();
        DispatchResult cmd = messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId).sender("orchestrator").type(MessageType.COMMAND)
                .content("analyse data").correlationId(corrId).actorType(ActorType.SYSTEM).build());

        final long cmdMsgId = cmd.messageId();

        HumanParticipatingChannelBackend humanBackend = new HumanParticipatingChannelBackend() {
            @Override public String backendId()    { return "test-human-backend"; }
            @Override public ActorType actorType() { return ActorType.HUMAN; }
            @Override public void open(ChannelRef ch, Map<String, String> m) {}
            @Override public void post(ChannelRef ch, OutboundMessage msg)   {}
            @Override public void close(ChannelRef ch) {}
            @Override public InboundNormaliser normaliser() {
                return (ch, raw) -> new NormalisedMessage(
                        MessageType.RESPONSE, raw.content(),
                        "human:" + raw.externalSenderId(),
                        raw.correlationId(), cmdMsgId, null, null);
            }
        };
        channelGateway.registerBackend(channelId, humanBackend, "human_participating");

        InboundHumanMessage response = new InboundHumanMessage(
                "user-42", "Analysis complete.", Instant.now(), Map.of(), corrId, cmdMsgId);
        channelGateway.receiveHumanMessage(new ChannelRef(channelId, channelName), response);

        assertThat(commitmentService.findByCorrelationId(corrId))
                .isPresent()
                .hasValueSatisfying(c ->
                        assertThat(c.state).isEqualTo(CommitmentState.FULFILLED));
    }

    private UUID createChannel(String name) {
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = name;
        ch.semantic = ChannelSemantic.APPEND;
        channelStore.put(ch);
        return ch.id;
    }
}
```

- [ ] **Step 2: Run the integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format \
  -Dtest=ChannelGatewayIntegrationTest
```

Expected: 1 test PASS.

- [ ] **Step 3: Run full runtime suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format
```

Expected: All tests PASS, 0 failures.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#158): integration test — RESPONSE via receiveHumanMessage fulfils Commitment

End-to-end scenario previously impossible without per-backend normalisation.

Refs #158.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: Full build + close issues

- [ ] **Step 1: Full install from project root**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -Dno-format \
  -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: All modules BUILD SUCCESS.

- [ ] **Step 2: Run code review**

Invoke `superpowers:requesting-code-review` before closing issues.

- [ ] **Step 3: Close issues**

```bash
gh issue close 158 --repo casehubio/qhorus \
  --comment "Resolved. Per-backend InboundNormaliser via HumanParticipatingChannelBackend.normaliser(). DefaultInboundNormaliser gains metadata key message-type + inReplyTo pass-through. Integration test: RESPONSE through receiveHumanMessage correctly fulfils Commitment."

gh issue close 159 --repo casehubio/qhorus \
  --comment "Partially resolved — InboundHumanMessage.inReplyTo added. artefactRefs and target remain out of scope (do not add speculatively per issue guidance)."
```

- [ ] **Step 4: Run implementation-doc-sync**

Invoke `implementation-doc-sync` to capture any design changes not yet in docs.

- [ ] **Step 5: Invoke java-git-commit for final commit (if any staged changes remain)**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus status --short
```

If clean — done. If staged — commit with appropriate message referencing #158.

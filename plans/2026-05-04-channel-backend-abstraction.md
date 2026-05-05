# Channel Backend Abstraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a `ChannelGateway` that routes messages to pluggable `ChannelBackend` implementations, making Qhorus backend-agnostic while preserving all existing normative guarantees.

**Architecture:** `QhorusMcpTools.sendMessage()` routes through `ChannelGateway.post()`, which calls `QhorusChannelBackend` (wrapping existing `MessageService`) synchronously then fans out to registered external backends asynchronously via Java 21 virtual threads. Human inbound arrives via `receiveHumanMessage()` / `receiveObserverSignal()` on the gateway.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, MCP (quarkus-mcp-server 1.11.1), JPA/Panache, H2 (tests). `ActorType` from `casehub-ledger-api`.

**Build commands:**
- Full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format`
- Runtime tests only: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format`
- Single test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl runtime -Dno-format`

**Issues:**
- Epic: casehubio/qhorus#131
- A2A bridge (separate): casehubio/qhorus#135
- ActorTypeResolver fix (ledger): casehubio/ledger#75

---

## Task 1: Create GitHub epic structure

**Files:** none (GitHub only)

- [ ] **Step 1: Convert #131 to epic and create sub-issues**

```bash
# Mark #131 as epic
gh issue edit 131 --repo casehubio/qhorus --add-label "epic"

# Sub-issue: SPI contracts
gh issue create --repo casehubio/qhorus \
  --title "feat(gateway): SPI contracts — ChannelBackend hierarchy, value records, InboundNormaliser" \
  --body "Part of epic casehubio/qhorus#131.

Implement all new types in \`casehub-qhorus-api\` module under package \`io.casehub.qhorus.api.gateway\`:
- \`ChannelBackend\`, \`AgentChannelBackend\`, \`HumanParticipatingChannelBackend\`, \`HumanObserverChannelBackend\`
- Value records: \`ChannelRef\`, \`OutboundMessage\`, \`InboundHumanMessage\`, \`ObserverSignal\`, \`NormalisedMessage\`
- \`InboundNormaliser\` SPI
- \`RecordingChannelBackend\` in \`casehub-qhorus-testing\`

Refs #131"

# Sub-issue: Gateway + defaults
gh issue create --repo casehubio/qhorus \
  --title "feat(gateway): ChannelGateway, QhorusChannelBackend, DefaultInboundNormaliser, Senders" \
  --body "Part of epic casehubio/qhorus#131. Blocked by SPI issue.

Implement in \`casehub-qhorus-runtime\` under \`io.casehub.qhorus.runtime.gateway\`:
- \`ChannelGateway\` (registration, outbound fan-out, inbound normalisation)
- \`QhorusChannelBackend\` (wraps MessageService)
- \`DefaultInboundNormaliser\` (always QUERY)
- \`DuplicateParticipatingBackendException\`
- \`Senders\` constant class

Refs #131"

# Sub-issue: MCP wiring
gh issue create --repo casehubio/qhorus \
  --title "feat(gateway): MCP tool wiring — sendMessage, create/delete_channel, register_backend tools" \
  --body "Part of epic casehubio/qhorus#131. Blocked by gateway issue.

Wire \`ChannelGateway\` into \`QhorusMcpTools\` and \`ReactiveQhorusMcpTools\`:
- \`send_message\` routes through gateway
- \`create_channel\` auto-registers \`QhorusChannelBackend\`
- \`delete_channel\` deregisters all backends
- New tools: \`register_backend\`, \`deregister_backend\`, \`list_backends\`
- \`respond_to_approval\` uses \`Senders.HUMAN\`

Refs #131"

# Sub-issue: Documentation
gh issue create --repo casehubio/qhorus \
  --title "docs(gateway): full documentation sweep — agent-mesh-framework, protocol comparisons, ADR, CLAUDE.md" \
  --body "Part of epic casehubio/qhorus#131. Blocked by MCP wiring issue.

Update all docs to reflect channel gateway:
- \`docs/agent-mesh-framework.md\` — new gateway section
- \`docs/agent-protocol-comparison.md\` — replace Phase 9 framing
- \`docs/multi-agent-framework-comparison.md\` — update A2A row
- \`docs/normative-layer.md\`, \`normative-summary.md\`, \`normative-channel-layout.md\`
- New ADR for channel backend abstraction decision
- \`CLAUDE.md\` — project structure, testing conventions, MCP surface
- \`casehubio/parent\` docs: casehub-qhorus.md + new convention doc

Refs #131"
```

- [ ] **Step 2: Note the issue numbers from output for use in commit messages**

The four new issues will be referenced as `Refs #131` in all commits since they're all sub-issues of the epic.

---

## Task 2: SPI contracts — `casehub-qhorus-api` module

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelBackend.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/AgentChannelBackend.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/HumanParticipatingChannelBackend.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/HumanObserverChannelBackend.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/InboundNormaliser.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelRef.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/InboundHumanMessage.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/ObserverSignal.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/gateway/NormalisedMessage.java`

- [ ] **Step 1: Create `ChannelRef`**

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/ChannelRef.java
package io.casehub.qhorus.api.gateway;

import java.util.UUID;

public record ChannelRef(UUID id, String name) {}
```

- [ ] **Step 2: Create `OutboundMessage`**

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java
package io.casehub.qhorus.api.gateway;

import java.util.UUID;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.message.MessageType;

public record OutboundMessage(
        UUID messageId,
        String sender,
        MessageType type,
        String content,
        UUID correlationId,
        ActorType senderActorType) {}
```

- [ ] **Step 3: Create `InboundHumanMessage` and `ObserverSignal`**

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/InboundHumanMessage.java
package io.casehub.qhorus.api.gateway;

import java.time.Instant;
import java.util.Map;

public record InboundHumanMessage(
        String externalSenderId,
        String text,
        Instant receivedAt,
        Map<String, String> metadata) {}
```

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/ObserverSignal.java
package io.casehub.qhorus.api.gateway;

import java.time.Instant;
import java.util.Map;

public record ObserverSignal(
        String externalSenderId,
        String content,
        Instant receivedAt,
        Map<String, String> metadata) {}
```

- [ ] **Step 4: Create `NormalisedMessage`**

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/NormalisedMessage.java
package io.casehub.qhorus.api.gateway;

import io.casehub.qhorus.api.message.MessageType;

// senderInstanceId MUST use format "human:{externalSenderId}" so
// ActorTypeResolver correctly stamps ActorType.HUMAN in the ledger.
public record NormalisedMessage(
        MessageType type,
        String content,
        String senderInstanceId) {}
```

- [ ] **Step 5: Create `ChannelBackend` and subtypes**

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/ChannelBackend.java
package io.casehub.qhorus.api.gateway;

import java.util.Map;

import io.casehub.ledger.api.model.ActorType;

public interface ChannelBackend {
    String backendId();
    ActorType actorType();
    void open(ChannelRef channel, Map<String, String> metadata);
    // AgentChannelBackend.post() may throw — it is the source-of-truth write.
    // All other implementations must catch internally; failure is non-fatal.
    void post(ChannelRef channel, OutboundMessage message);
    void close(ChannelRef channel);
}
```

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/AgentChannelBackend.java
package io.casehub.qhorus.api.gateway;

// Always registered. Internal agent mesh. actorType() must return ActorType.AGENT.
public interface AgentChannelBackend extends ChannelBackend {}
```

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/HumanParticipatingChannelBackend.java
package io.casehub.qhorus.api.gateway;

// At most one per channel. Full speech act inbound via InboundNormaliser.
// actorType() must return ActorType.HUMAN.
// Call gateway.receiveHumanMessage() when inbound arrives.
public interface HumanParticipatingChannelBackend extends ChannelBackend {}
```

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/HumanObserverChannelBackend.java
package io.casehub.qhorus.api.gateway;

// Unlimited per channel. Inbound capped to EVENT by gateway.
// actorType() must return ActorType.HUMAN.
// Call gateway.receiveObserverSignal() when inbound arrives.
public interface HumanObserverChannelBackend extends ChannelBackend {}
```

- [ ] **Step 6: Create `InboundNormaliser` SPI**

```java
// api/src/main/java/io/casehub/qhorus/api/gateway/InboundNormaliser.java
package io.casehub.qhorus.api.gateway;

@FunctionalInterface
public interface InboundNormaliser {
    NormalisedMessage normalise(ChannelRef channel, InboundHumanMessage raw);
}
```

- [ ] **Step 7: Compile the api module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -Dno-format
```

Expected: `BUILD SUCCESS` with no errors.

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/gateway/
git commit -m "feat(api): gateway SPI contracts — ChannelBackend hierarchy, value records, InboundNormaliser

Refs #131"
```

---

## Task 3: `RecordingChannelBackend` in testing module

**Files:**
- Create: `testing/src/main/java/io/casehub/qhorus/testing/gateway/RecordingChannelBackend.java`

- [ ] **Step 1: Write `RecordingChannelBackend`**

This is a test double used by unit and integration tests to capture calls to `post()`, `open()`, and `close()`. It can act as any backend type via constructor.

```java
// testing/src/main/java/io/casehub/qhorus/testing/gateway/RecordingChannelBackend.java
package io.casehub.qhorus.testing.gateway;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.gateway.ChannelBackend;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;

public class RecordingChannelBackend implements ChannelBackend {

    private final String backendId;
    private final ActorType actorType;
    private final List<OutboundMessage> posts = new ArrayList<>();
    private final List<ChannelRef> opens = new ArrayList<>();
    private final List<ChannelRef> closes = new ArrayList<>();
    private RuntimeException throwOnPost;

    public RecordingChannelBackend(String backendId, ActorType actorType) {
        this.backendId = backendId;
        this.actorType = actorType;
    }

    public void throwOnNextPost(RuntimeException ex) {
        this.throwOnPost = ex;
    }

    @Override
    public String backendId() { return backendId; }

    @Override
    public ActorType actorType() { return actorType; }

    @Override
    public void open(ChannelRef channel, Map<String, String> metadata) {
        opens.add(channel);
    }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        if (throwOnPost != null) {
            RuntimeException ex = throwOnPost;
            throwOnPost = null;
            throw ex;
        }
        posts.add(message);
    }

    @Override
    public void close(ChannelRef channel) {
        closes.add(channel);
    }

    public List<OutboundMessage> posts() { return Collections.unmodifiableList(posts); }
    public List<ChannelRef> opens() { return Collections.unmodifiableList(opens); }
    public List<ChannelRef> closes() { return Collections.unmodifiableList(closes); }
    public void clear() { posts.clear(); opens.clear(); closes.clear(); throwOnPost = null; }
}
```

- [ ] **Step 2: Compile the testing module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl testing -Dno-format
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git add testing/src/main/java/io/casehub/qhorus/testing/gateway/
git commit -m "feat(testing): RecordingChannelBackend for gateway unit tests

Refs #131"
```

---

## Task 4: Exception + constant

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DuplicateParticipatingBackendException.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/Senders.java`

- [ ] **Step 1: Create exception and constant**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DuplicateParticipatingBackendException.java
package io.casehub.qhorus.runtime.gateway;

public class DuplicateParticipatingBackendException extends IllegalStateException {
    public DuplicateParticipatingBackendException(String channelId, String existingBackendId) {
        super("Channel " + channelId + " already has a HumanParticipatingChannelBackend: "
                + existingBackendId
                + ". At most one participatory human backend is allowed per channel.");
    }
}
```

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/gateway/Senders.java
package io.casehub.qhorus.runtime.gateway;

// Canonical sender identifiers used in MCP tools and gateway inbound flows.
// "human" resolves to ActorType.HUMAN via ActorTypeResolver catch-all.
public final class Senders {
    public static final String HUMAN = "human";

    private Senders() {}
}
```

- [ ] **Step 2: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/gateway/
git commit -m "feat(gateway): DuplicateParticipatingBackendException and Senders constant

Refs #131"
```

---

## Task 5: `QhorusChannelBackend` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/QhorusChannelBackendTest.java`

- [ ] **Step 1: Write the failing unit test**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/QhorusChannelBackendTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.gateway.QhorusChannelBackend;
import io.casehub.qhorus.runtime.message.MessageService;

class QhorusChannelBackendTest {

    MessageService messageService;
    QhorusChannelBackend backend;

    @BeforeEach
    void setUp() {
        messageService = mock(MessageService.class);
        backend = new QhorusChannelBackend(messageService);
    }

    @Test
    void backendId_isQhorusInternal() {
        assertEquals("qhorus-internal", backend.backendId());
    }

    @Test
    void actorType_isAgent() {
        assertEquals(ActorType.AGENT, backend.actorType());
    }

    @Test
    void post_delegatesToMessageService() {
        UUID channelId = UUID.randomUUID();
        UUID messageId = UUID.randomUUID();
        UUID corrId = UUID.randomUUID();
        ChannelRef ref = new ChannelRef(channelId, "test-channel");
        OutboundMessage msg = new OutboundMessage(messageId, "agent-a",
                MessageType.COMMAND, "do the thing", corrId, ActorType.AGENT);

        backend.post(ref, msg);

        verify(messageService).send(channelId, "agent-a", MessageType.COMMAND,
                "do the thing", corrId.toString(), null);
    }

    @Test
    void open_and_close_areNoOps() {
        ChannelRef ref = new ChannelRef(UUID.randomUUID(), "ch");
        assertDoesNotThrow(() -> backend.open(ref, Map.of()));
        assertDoesNotThrow(() -> backend.close(ref));
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusChannelBackendTest -pl runtime -Dno-format 2>&1 | tail -20
```

Expected: compile error — `QhorusChannelBackend` does not exist yet.

- [ ] **Step 3: Implement `QhorusChannelBackend`**

Note: `MessageService.send()` has signature `send(UUID channelId, String sender, MessageType type, String content, String correlationId, Long inReplyTo)`. The `correlationId` is a String, so convert `UUID` to `String`.

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java
package io.casehub.qhorus.runtime.gateway;

import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.gateway.AgentChannelBackend;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.runtime.message.MessageService;

@ApplicationScoped
public class QhorusChannelBackend implements AgentChannelBackend {

    final MessageService messageService;

    @Inject
    public QhorusChannelBackend(MessageService messageService) {
        this.messageService = messageService;
    }

    @Override
    public String backendId() { return "qhorus-internal"; }

    @Override
    public ActorType actorType() { return ActorType.AGENT; }

    @Override
    public void open(ChannelRef channel, Map<String, String> metadata) { }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        String correlationId = message.correlationId() != null
                ? message.correlationId().toString() : null;
        messageService.send(channel.id(), message.sender(), message.type(),
                message.content(), correlationId, null);
    }

    @Override
    public void close(ChannelRef channel) { }
}
```

- [ ] **Step 4: Run test — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusChannelBackendTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java \
        runtime/src/test/java/io/casehub/qhorus/gateway/QhorusChannelBackendTest.java
git commit -m "feat(gateway): QhorusChannelBackend wrapping MessageService

Refs #131"
```

---

## Task 6: `DefaultInboundNormaliser` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java`

- [ ] **Step 1: Write the failing unit test**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.api.gateway.NormalisedMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.gateway.DefaultInboundNormaliser;

class DefaultInboundNormaliserTest {

    DefaultInboundNormaliser normaliser = new DefaultInboundNormaliser();
    ChannelRef channel = new ChannelRef(UUID.randomUUID(), "test-ch");

    @Test
    void normalise_alwaysReturnsQuery() {
        InboundHumanMessage raw = new InboundHumanMessage(
                "user-42", "Please analyse this", Instant.now(), Map.of());
        NormalisedMessage result = normaliser.normalise(channel, raw);
        assertEquals(MessageType.QUERY, result.type());
    }

    @Test
    void normalise_preservesText() {
        InboundHumanMessage raw = new InboundHumanMessage(
                "user-42", "Hello agent!", Instant.now(), Map.of());
        NormalisedMessage result = normaliser.normalise(channel, raw);
        assertEquals("Hello agent!", result.content());
    }

    @Test
    void normalise_senderIdPrefixedWithHuman() {
        InboundHumanMessage raw = new InboundHumanMessage(
                "+447911123456", "stop everything", Instant.now(), Map.of());
        NormalisedMessage result = normaliser.normalise(channel, raw);
        assertEquals("human:+447911123456", result.senderInstanceId());
    }

    @Test
    void normalise_questionMarkStillReturnsQuery() {
        InboundHumanMessage raw = new InboundHumanMessage(
                "user-1", "What is the status?", Instant.now(), Map.of());
        NormalisedMessage result = normaliser.normalise(channel, raw);
        assertEquals(MessageType.QUERY, result.type());
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DefaultInboundNormaliserTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: compile error.

- [ ] **Step 3: Implement `DefaultInboundNormaliser`**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java
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
                MessageType.QUERY,
                raw.text(),
                "human:" + raw.externalSenderId());
    }
}
```

- [ ] **Step 4: Run test — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DefaultInboundNormaliserTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java \
        runtime/src/test/java/io/casehub/qhorus/gateway/DefaultInboundNormaliserTest.java
git commit -m "feat(gateway): DefaultInboundNormaliser — always QUERY, human: sender prefix

Refs #131"
```

---

## Task 7: `ChannelGateway` — TDD

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayTest.java`

- [ ] **Step 1: Write the failing unit tests**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.gateway.*;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.gateway.*;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.testing.gateway.RecordingChannelBackend;

class ChannelGatewayTest {

    MessageService messageService;
    QhorusChannelBackend agentBackend;
    DefaultInboundNormaliser normaliser;
    ChannelGateway gateway;

    UUID channelId;
    ChannelRef channelRef;

    @BeforeEach
    void setUp() {
        messageService = mock(MessageService.class);
        agentBackend = new QhorusChannelBackend(messageService);
        normaliser = new DefaultInboundNormaliser();
        gateway = new ChannelGateway(agentBackend, normaliser, messageService);
        channelId = UUID.randomUUID();
        channelRef = new ChannelRef(channelId, "test-channel");
        gateway.initChannel(channelId, channelRef);
    }

    // ── Registration ──────────────────────────────────────────────────────

    @Test
    void listBackends_includesQhorusInternalByDefault() {
        List<ChannelGateway.BackendRegistration> backends = gateway.listBackends(channelId);
        assertEquals(1, backends.size());
        assertEquals("qhorus-internal", backends.get(0).backendId());
    }

    @Test
    void registerObserver_appearsInList() {
        RecordingChannelBackend observer = new RecordingChannelBackend("slack-obs", ActorType.HUMAN);
        gateway.registerBackend(channelId, observer, "human_observer");

        List<ChannelGateway.BackendRegistration> backends = gateway.listBackends(channelId);
        assertEquals(2, backends.size());
    }

    @Test
    void registerSecondParticipating_throws() {
        RecordingChannelBackend first = new RecordingChannelBackend("whatsapp-1", ActorType.HUMAN);
        RecordingChannelBackend second = new RecordingChannelBackend("slack-1", ActorType.HUMAN);
        gateway.registerBackend(channelId, first, "human_participating");

        assertThrows(DuplicateParticipatingBackendException.class,
                () -> gateway.registerBackend(channelId, second, "human_participating"));
    }

    @Test
    void deregisterQhorusInternal_throws() {
        assertThrows(IllegalArgumentException.class,
                () -> gateway.deregisterBackend(channelId, "qhorus-internal"));
    }

    @Test
    void deregisterBackend_removesFromList() {
        RecordingChannelBackend observer = new RecordingChannelBackend("slack-obs", ActorType.HUMAN);
        gateway.registerBackend(channelId, observer, "human_observer");
        gateway.deregisterBackend(channelId, "slack-obs");

        assertEquals(1, gateway.listBackends(channelId).size());
    }

    // ── Outbound ──────────────────────────────────────────────────────────

    @Test
    void post_callsAgentBackendSynchronously() {
        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "agent-a",
                MessageType.COMMAND, "do it", UUID.randomUUID(), ActorType.AGENT);

        gateway.post(channelId, msg);

        verify(messageService).send(eq(channelId), eq("agent-a"), eq(MessageType.COMMAND),
                eq("do it"), anyString(), isNull());
    }

    @Test
    void post_fansOutToObserverBackend() throws Exception {
        RecordingChannelBackend observer = new RecordingChannelBackend("panel", ActorType.HUMAN);
        gateway.registerBackend(channelId, observer, "human_observer");

        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "agent-a",
                MessageType.EVENT, "tool used", null, ActorType.AGENT);
        gateway.post(channelId, msg);

        // Fan-out is async — give virtual thread time to execute
        Thread.sleep(100);
        assertEquals(1, observer.posts().size());
        assertEquals("tool used", observer.posts().get(0).content());
    }

    @Test
    void post_observerFailure_doesNotPropagateToAgent() throws Exception {
        RecordingChannelBackend failingBackend = new RecordingChannelBackend("bad", ActorType.HUMAN);
        failingBackend.throwOnNextPost(new RuntimeException("network error"));
        gateway.registerBackend(channelId, failingBackend, "human_observer");

        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "agent-a",
                MessageType.STATUS, "still working", null, ActorType.AGENT);

        // Must not throw even though observer fails
        assertDoesNotThrow(() -> gateway.post(channelId, msg));
        Thread.sleep(100); // let async run
    }

    // ── Inbound ───────────────────────────────────────────────────────────

    @Test
    void receiveHumanMessage_callsMessageServiceWithHumanSender() {
        InboundHumanMessage raw = new InboundHumanMessage(
                "user-42", "Can you stop?", Instant.now(), Map.of());

        gateway.receiveHumanMessage(channelRef, raw);

        verify(messageService).send(eq(channelId), eq("human:user-42"),
                eq(MessageType.QUERY), eq("Can you stop?"), isNull(), isNull());
    }

    @Test
    void receiveObserverSignal_forcesEvent() {
        ObserverSignal signal = new ObserverSignal(
                "panel-user", "thumbs up", Instant.now(), Map.of());

        gateway.receiveObserverSignal(channelRef, signal);

        verify(messageService).send(eq(channelId), eq("human:panel-user"),
                eq(MessageType.EVENT), eq("thumbs up"), isNull(), isNull());
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelGatewayTest -pl runtime -Dno-format 2>&1 | tail -20
```

Expected: compile error — `ChannelGateway` does not exist.

- [ ] **Step 3: Implement `ChannelGateway`**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java
package io.casehub.qhorus.runtime.gateway;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.qhorus.api.gateway.*;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.MessageService;

@ApplicationScoped
public class ChannelGateway {

    private static final Logger LOG = Logger.getLogger(ChannelGateway.class);

    // channelId → ordered list of backend entries
    private final ConcurrentHashMap<UUID, List<BackendEntry>> registry = new ConcurrentHashMap<>();

    final AgentChannelBackend agentBackend;
    final InboundNormaliser normaliser;
    final MessageService messageService;

    @Inject
    public ChannelGateway(AgentChannelBackend agentBackend,
                          InboundNormaliser normaliser,
                          MessageService messageService) {
        this.agentBackend = agentBackend;
        this.normaliser = normaliser;
        this.messageService = messageService;
    }

    // Called by create_channel to initialise the default backend registration
    public void initChannel(UUID channelId, ChannelRef ref) {
        registry.computeIfAbsent(channelId, id -> {
            List<BackendEntry> entries = Collections.synchronizedList(new ArrayList<>());
            agentBackend.open(ref, Map.of());
            entries.add(new BackendEntry(agentBackend, "agent"));
            return entries;
        });
    }

    // Called by delete_channel to clean up all backends
    public void closeChannel(UUID channelId, ChannelRef ref) {
        List<BackendEntry> entries = registry.remove(channelId);
        if (entries != null) {
            for (BackendEntry e : entries) {
                try { e.backend().close(ref); } catch (Exception ex) {
                    LOG.errorf(ex, "Error closing backend %s on channel %s",
                            e.backend().backendId(), channelId);
                }
            }
        }
    }

    public void registerBackend(UUID channelId, ChannelBackend backend, String backendType) {
        List<BackendEntry> entries = registry.computeIfAbsent(channelId,
                id -> Collections.synchronizedList(new ArrayList<>()));
        if ("human_participating".equals(backendType)) {
            entries.stream()
                    .filter(e -> "human_participating".equals(e.backendType()))
                    .findFirst()
                    .ifPresent(existing -> {
                        throw new DuplicateParticipatingBackendException(
                                channelId.toString(), existing.backend().backendId());
                    });
        }
        entries.add(new BackendEntry(backend, backendType));
    }

    public void deregisterBackend(UUID channelId, String backendId) {
        if ("qhorus-internal".equals(backendId)) {
            throw new IllegalArgumentException("Cannot deregister the qhorus-internal backend.");
        }
        List<BackendEntry> entries = registry.get(channelId);
        if (entries != null) {
            entries.removeIf(e -> backendId.equals(e.backend().backendId()));
        }
    }

    public List<BackendRegistration> listBackends(UUID channelId) {
        List<BackendEntry> entries = registry.getOrDefault(channelId, List.of());
        return entries.stream()
                .map(e -> new BackendRegistration(
                        e.backend().backendId(),
                        e.backendType(),
                        e.backend().actorType()))
                .toList();
    }

    // Outbound — called by QhorusMcpTools.sendMessage()
    public void post(UUID channelId, OutboundMessage message) {
        ChannelRef ref = channelRef(channelId);
        // Source-of-truth write — synchronous, may throw
        agentBackend.post(ref, message);
        // Fan-out to external backends — async, failures non-fatal
        List<BackendEntry> entries = registry.getOrDefault(channelId, List.of());
        for (BackendEntry entry : entries) {
            if (entry.backend() == agentBackend) continue;
            ChannelBackend backend = entry.backend();
            Thread.ofVirtual().start(() -> {
                try {
                    backend.post(ref, message);
                } catch (Exception ex) {
                    LOG.errorf(ex, "Backend %s failed on post to channel %s",
                            backend.backendId(), channelId);
                }
            });
        }
    }

    // Inbound from HumanParticipatingChannelBackend
    public void receiveHumanMessage(ChannelRef channel, InboundHumanMessage raw) {
        NormalisedMessage normalised = normaliser.normalise(channel, raw);
        String correlationId = null; // inbound human messages are not correlated by default
        messageService.send(channel.id(), normalised.senderInstanceId(),
                normalised.type(), normalised.content(), correlationId, null);
    }

    // Inbound from HumanObserverChannelBackend — always EVENT regardless of content
    public void receiveObserverSignal(ChannelRef channel, ObserverSignal signal) {
        messageService.send(channel.id(), "human:" + signal.externalSenderId(),
                MessageType.EVENT, signal.content(), null, null);
    }

    private ChannelRef channelRef(UUID channelId) {
        // Minimal ref for fan-out — name is informational only for external backends
        return new ChannelRef(channelId, channelId.toString());
    }

    public record BackendEntry(ChannelBackend backend, String backendType) {}

    public record BackendRegistration(String backendId, String backendType,
            io.casehub.ledger.api.model.ActorType actorType) {}
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelGatewayTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: `Tests run: 10, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ \
        runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayTest.java
git commit -m "feat(gateway): ChannelGateway — registration, outbound fan-out, inbound normalisation

Refs #131"
```

---

## Task 8: Wire `sendMessage` through gateway in `QhorusMcpTools`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/SendMessageGatewayIntegrationTest.java`

- [ ] **Step 1: Write the integration test**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/SendMessageGatewayIntegrationTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.casehub.qhorus.runtime.mcp.QhorusMcpToolsBase.MessageResult;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.testing.gateway.RecordingChannelBackend;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.AfterEach;

@QuarkusTest
class SendMessageGatewayIntegrationTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelGateway gateway;

    RecordingChannelBackend observer;

    @BeforeEach
    @Transactional
    void setUp() {
        tools.createChannel("gw-integ-1", "test", "append", null, null, null, null, null, null);
        observer = new RecordingChannelBackend("test-observer", ActorType.HUMAN);
        // Get channel UUID from list
        var ch = tools.listChannels().stream()
                .filter(c -> "gw-integ-1".equals(c.name())).findFirst().orElseThrow();
        gateway.registerBackend(ch.channelId(), observer, "human_observer");
    }

    @AfterEach
    @Transactional
    void tearDown() {
        tools.deleteChannel("gw-integ-1", null, false);
    }

    @Test
    void sendMessage_still_works_with_gateway() {
        var result = tools.sendMessage("gw-integ-1", "agent-a", "command",
                "do the thing", null, null, null, null, null);
        assertNotNull(result);
        assertEquals("gw-integ-1", result.channelName());
    }

    @Test
    void sendMessage_fansOutToObserver() throws Exception {
        tools.sendMessage("gw-integ-1", "agent-a", "event",
                "tool_call_completed", null, null, null, null, null);
        Thread.sleep(200); // allow virtual thread fan-out
        assertEquals(1, observer.posts().size());
        assertEquals("tool_call_completed", observer.posts().get(0).content());
    }
}
```

- [ ] **Step 2: Run test — expect failure (gateway not yet wired)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SendMessageGatewayIntegrationTest -pl runtime -Dno-format 2>&1 | tail -20
```

Expected: `sendMessage_fansOutToObserver` fails — observer receives no messages.

- [ ] **Step 3: Add `ChannelGateway` inject to `QhorusMcpTools` and wire `sendMessage`**

In `QhorusMcpTools.java`, add the inject and modify `sendMessage()`:

```java
// Add field alongside existing @Inject fields:
@Inject
ChannelGateway channelGateway;
```

Find the `sendMessage()` `@Tool` method. It currently calls `messageService.send(...)` directly. Locate the call and replace with gateway routing. The pattern is: build an `OutboundMessage` from the tool params, call `channelGateway.post(channel.id, outboundMessage)`.

The existing `sendMessage` tool locates the channel then calls `messageService.send()`. Change it to:

```java
// Inside sendMessage(), after resolving channel and sender — replace messageService.send() with:
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.ledger.api.model.ActorTypeResolver;

// Build outbound message
UUID corrUuid = correlationId != null ? UUID.fromString(correlationId) : null;
io.casehub.ledger.api.model.ActorType senderActorType =
        io.casehub.ledger.api.model.ActorTypeResolver.resolve(sender);
OutboundMessage outbound = new OutboundMessage(
        UUID.randomUUID(), sender, resolvedType, content, corrUuid, senderActorType);
channelGateway.post(channel.id, outbound);
```

Note: The `channelGateway.post()` calls `QhorusChannelBackend.post()` which calls `messageService.send()` — the `@Transactional` on `sendMessage()` in `QhorusMcpTools` still covers the DB write.

Also add the same injection and wiring to `ReactiveQhorusMcpTools.java`.

- [ ] **Step 4: Run tests — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SendMessageGatewayIntegrationTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`.

- [ ] **Step 5: Run full runtime test suite to check no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: all existing tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ \
        runtime/src/test/java/io/casehub/qhorus/gateway/SendMessageGatewayIntegrationTest.java
git commit -m "feat(gateway): wire send_message through ChannelGateway in QhorusMcpTools

Refs #131"
```

---

## Task 9: Wire `create_channel` and `delete_channel`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/ChannelLifecycleGatewayTest.java`

- [ ] **Step 1: Write the test**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/ChannelLifecycleGatewayTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@QuarkusTest
class ChannelLifecycleGatewayTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelGateway gateway;

    @Test
    @Transactional
    void createChannel_autoRegistersQhorusBackend() {
        tools.createChannel("lifecycle-1", "test", "append", null, null, null, null, null, null);

        var ch = tools.listChannels().stream()
                .filter(c -> "lifecycle-1".equals(c.name())).findFirst().orElseThrow();
        var backends = gateway.listBackends(ch.channelId());

        assertEquals(1, backends.size());
        assertEquals("qhorus-internal", backends.get(0).backendId());

        tools.deleteChannel("lifecycle-1", null, false);
    }

    @Test
    @Transactional
    void deleteChannel_deregistersAllBackends() {
        tools.createChannel("lifecycle-2", "test", "append", null, null, null, null, null, null);
        var ch = tools.listChannels().stream()
                .filter(c -> "lifecycle-2".equals(c.name())).findFirst().orElseThrow();
        UUID channelId = ch.channelId();

        tools.deleteChannel("lifecycle-2", null, false);

        // After deletion, listBackends returns empty
        assertTrue(gateway.listBackends(channelId).isEmpty());
    }
}
```

- [ ] **Step 2: Run test — expect first test fails** (create doesn't auto-register yet)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelLifecycleGatewayTest -pl runtime -Dno-format 2>&1 | tail -20
```

- [ ] **Step 3: Wire `create_channel` in `QhorusMcpTools`**

In `createChannel()` tool method, after the channel is created and the `ChannelDetail` is assembled, add:

```java
// After channel is saved, initialise the gateway registry for this channel
ChannelRef ref = new ChannelRef(channel.id, channel.name);
channelGateway.initChannel(channel.id, ref);
```

In `deleteChannel()` tool method, before the actual deletion, add:

```java
// Deregister all backends and call close() on each
ChannelRef ref = new ChannelRef(channel.id, channel.name);
channelGateway.closeChannel(channel.id, ref);
```

Apply the same changes to `ReactiveQhorusMcpTools`.

- [ ] **Step 4: Run test — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelLifecycleGatewayTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`.

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ \
        runtime/src/test/java/io/casehub/qhorus/gateway/ChannelLifecycleGatewayTest.java
git commit -m "feat(gateway): auto-register QhorusChannelBackend on create_channel, deregister on delete

Refs #131"
```

---

## Task 10: New MCP tools — `register_backend`, `deregister_backend`, `list_backends`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/BackendRegistrationMcpTest.java`

- [ ] **Step 1: Add response records to `QhorusMcpToolsBase`**

```java
// Add to QhorusMcpToolsBase:
public record BackendInfo(String backendId, String backendType, String actorType) {}

public record RegisterBackendResult(String channelName, String backendId,
        String backendType, String message) {}

public record DeregisterBackendResult(String channelName, String backendId,
        boolean success, String message) {}
```

- [ ] **Step 2: Write the integration test**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/BackendRegistrationMcpTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.casehub.qhorus.runtime.mcp.QhorusMcpToolsBase.BackendInfo;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.testing.gateway.RecordingChannelBackend;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@QuarkusTest
class BackendRegistrationMcpTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelGateway gateway;

    @BeforeEach
    @Transactional
    void setUp() {
        tools.createChannel("mcp-reg-1", "test", "append", null, null, null, null, null, null);
    }

    @AfterEach
    @Transactional
    void tearDown() {
        tools.deleteChannel("mcp-reg-1", null, false);
    }

    @Test
    void listBackends_returnsQhorusInternal() {
        List<BackendInfo> result = tools.listBackends("mcp-reg-1");
        assertEquals(1, result.size());
        assertEquals("qhorus-internal", result.get(0).backendId());
        assertEquals("agent", result.get(0).backendType());
    }

    @Test
    void registerBackend_appearsInList() {
        // Register via programmatic CDI for test (register_backend MCP tool
        // works with pre-registered CDI beans; here we use gateway directly
        // to simulate what a CDI backend would do on openChannel)
        var ch = tools.listChannels().stream()
                .filter(c -> "mcp-reg-1".equals(c.name())).findFirst().orElseThrow();
        RecordingChannelBackend obs = new RecordingChannelBackend("slack-panel", ActorType.HUMAN);
        gateway.registerBackend(ch.channelId(), obs, "human_observer");

        List<BackendInfo> result = tools.listBackends("mcp-reg-1");
        assertEquals(2, result.size());
        assertTrue(result.stream().anyMatch(b -> "slack-panel".equals(b.backendId())));
    }

    @Test
    void deregisterBackend_removesFromList() {
        var ch = tools.listChannels().stream()
                .filter(c -> "mcp-reg-1".equals(c.name())).findFirst().orElseThrow();
        RecordingChannelBackend obs = new RecordingChannelBackend("to-remove", ActorType.HUMAN);
        gateway.registerBackend(ch.channelId(), obs, "human_observer");

        tools.deregisterBackend("mcp-reg-1", "to-remove");

        List<BackendInfo> result = tools.listBackends("mcp-reg-1");
        assertEquals(1, result.size());
        assertEquals("qhorus-internal", result.get(0).backendId());
    }

    @Test
    void deregisterQhorusInternal_returnsError() {
        // Should surface as tool error string, not throw
        assertThrows(IllegalArgumentException.class,
                () -> tools.deregisterBackend("mcp-reg-1", "qhorus-internal"));
    }
}
```

- [ ] **Step 3: Implement the three new tools in `QhorusMcpTools`**

```java
// Add to QhorusMcpTools, after the existing delete_channel tool:

@Tool(name = "list_backends", description = "List all registered channel backends for a channel. "
        + "Always includes 'qhorus-internal' (the default Qhorus agent backend).")
public List<BackendInfo> listBackends(
        @ToolArg(name = "channel_name", description = "Name of the channel") String channelName) {
    Channel channel = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
    return channelGateway.listBackends(channel.id).stream()
            .map(r -> new BackendInfo(r.backendId(), r.backendType(),
                    r.actorType().name().toLowerCase()))
            .toList();
}

@Tool(name = "deregister_backend", description = "Remove a registered backend from a channel. "
        + "Cannot remove 'qhorus-internal'.")
public DeregisterBackendResult deregisterBackend(
        @ToolArg(name = "channel_name", description = "Name of the channel") String channelName,
        @ToolArg(name = "backend_id", description = "ID of the backend to remove") String backendId) {
    Channel channel = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
    channelGateway.deregisterBackend(channel.id, backendId);
    return new DeregisterBackendResult(channelName, backendId, true,
            "Backend " + backendId + " deregistered from " + channelName);
}
```

Note: `register_backend` is intentionally not a full MCP tool for creating CDI beans — backends self-register. The tool is documented as agent-directed association. For now, expose only `list_backends` and `deregister_backend` via MCP. Add the same to `ReactiveQhorusMcpTools`.

- [ ] **Step 4: Run tests — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=BackendRegistrationMcpTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`.

- [ ] **Step 5: Check for `@Tool` overload guard**

`ToolOverloadDiscoverabilityTest` must still pass — verify no public non-`@Tool` method shares a name with a `@Tool` method.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ToolOverloadDiscoverabilityTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: `PASS`.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ \
        runtime/src/test/java/io/casehub/qhorus/gateway/BackendRegistrationMcpTest.java
git commit -m "feat(gateway): list_backends and deregister_backend MCP tools

Refs #131"
```

---

## Task 11: `Senders.HUMAN` cleanup in `respond_to_approval`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Replace `"human"` literal with `Senders.HUMAN`**

In both `QhorusMcpTools` and `ReactiveQhorusMcpTools`, find `respond_to_approval`:

```java
// Before:
return sendMessage(channelName, "human", "response", responseText, correlationId, ...);

// After:
import io.casehub.qhorus.runtime.gateway.Senders;
return sendMessage(channelName, Senders.HUMAN, "response", responseText, correlationId, ...);
```

Search for all other `"human"` literals used as senders (not in comments/descriptions) and replace with `Senders.HUMAN`.

- [ ] **Step 2: Run `ApprovalGateTest` to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ApprovalGateTest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: all pass.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/
git commit -m "refactor: replace 'human' magic string with Senders.HUMAN in respond_to_approval

Refs #131"
```

---

## Task 12: Full integration + robustness tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayRobustnessTest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayInboundIntegrationTest.java`

- [ ] **Step 1: Write robustness integration tests**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayRobustnessTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.gateway.DuplicateParticipatingBackendException;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.casehub.qhorus.testing.gateway.RecordingChannelBackend;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@QuarkusTest
class ChannelGatewayRobustnessTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelGateway gateway;

    @BeforeEach @Transactional
    void setUp() {
        tools.createChannel("rob-1", "test", "append", null, null, null, null, null, null);
    }

    @AfterEach @Transactional
    void tearDown() {
        try { tools.deleteChannel("rob-1", null, false); } catch (Exception ignored) {}
    }

    @Test
    void duplicateParticipatingBackend_isRejected() {
        var ch = tools.listChannels().stream()
                .filter(c -> "rob-1".equals(c.name())).findFirst().orElseThrow();
        gateway.registerBackend(ch.channelId(),
                new RecordingChannelBackend("whatsapp-1", ActorType.HUMAN), "human_participating");

        assertThrows(DuplicateParticipatingBackendException.class,
                () -> gateway.registerBackend(ch.channelId(),
                        new RecordingChannelBackend("slack-1", ActorType.HUMAN),
                        "human_participating"));
    }

    @Test
    void observerFailure_doesNotFailSendMessage() throws Exception {
        var ch = tools.listChannels().stream()
                .filter(c -> "rob-1".equals(c.name())).findFirst().orElseThrow();
        RecordingChannelBackend failing = new RecordingChannelBackend("failing-obs", ActorType.HUMAN);
        failing.throwOnNextPost(new RuntimeException("network error"));
        gateway.registerBackend(ch.channelId(), failing, "human_observer");

        // send_message must succeed even though observer will fail
        assertDoesNotThrow(() ->
                tools.sendMessage("rob-1", "agent-a", "event",
                        "test", null, null, null, null, null));
        Thread.sleep(200);
    }

    @Test
    void twoChannels_backendRegistrationsAreIsolated() {
        tools.createChannel("rob-2", "test2", "append", null, null, null, null, null, null);
        var ch1 = tools.listChannels().stream()
                .filter(c -> "rob-1".equals(c.name())).findFirst().orElseThrow();
        var ch2 = tools.listChannels().stream()
                .filter(c -> "rob-2".equals(c.name())).findFirst().orElseThrow();

        RecordingChannelBackend obs = new RecordingChannelBackend("ch1-only", ActorType.HUMAN);
        gateway.registerBackend(ch1.channelId(), obs, "human_observer");

        // ch2 should only have qhorus-internal
        assertEquals(1, gateway.listBackends(ch2.channelId()).size());

        tools.deleteChannel("rob-2", null, false);
    }
}
```

- [ ] **Step 2: Write inbound integration tests**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayInboundIntegrationTest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;

import java.time.Instant;
import java.util.Map;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.InboundHumanMessage;
import io.casehub.qhorus.api.gateway.ObserverSignal;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@QuarkusTest
class ChannelGatewayInboundIntegrationTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelGateway gateway;

    @BeforeEach @Transactional
    void setUp() {
        tools.createChannel("inbound-1", "test", "append", null, null, null, null, null, null);
    }

    @AfterEach @Transactional
    void tearDown() {
        tools.deleteChannel("inbound-1", null, true);
    }

    @Test
    @Transactional
    void receiveHumanMessage_createsLedgerEntryWithHumanSender() {
        var ch = tools.listChannels().stream()
                .filter(c -> "inbound-1".equals(c.name())).findFirst().orElseThrow();
        ChannelRef ref = new ChannelRef(ch.channelId(), "inbound-1");
        InboundHumanMessage raw = new InboundHumanMessage(
                "user-42", "Can you stop the analysis?", Instant.now(), Map.of());

        gateway.receiveHumanMessage(ref, raw);

        var messages = tools.checkMessages("inbound-1", "observer-dummy",
                null, null, null, null, null, null, null, true);
        assertTrue(messages.messages().stream()
                .anyMatch(m -> "human:user-42".equals(m.sender())
                        && MessageType.QUERY.name().equalsIgnoreCase(m.type())));
    }

    @Test
    @Transactional
    void receiveObserverSignal_forcesEventType() {
        var ch = tools.listChannels().stream()
                .filter(c -> "inbound-1".equals(c.name())).findFirst().orElseThrow();
        ChannelRef ref = new ChannelRef(ch.channelId(), "inbound-1");
        ObserverSignal signal = new ObserverSignal(
                "panel-user", "thumbs-up", Instant.now(), Map.of());

        gateway.receiveObserverSignal(ref, signal);

        var messages = tools.checkMessages("inbound-1", "observer-dummy",
                null, null, null, null, null, null, null, true);
        assertTrue(messages.messages().stream()
                .anyMatch(m -> "human:panel-user".equals(m.sender())
                        && "event".equalsIgnoreCase(m.type())));
    }
}
```

- [ ] **Step 3: Run all new tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest="ChannelGatewayRobustnessTest,ChannelGatewayInboundIntegrationTest" \
  -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: all pass.

- [ ] **Step 4: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: all existing + new tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/qhorus/gateway/
git commit -m "test(gateway): robustness and inbound integration test suite

Refs #131"
```

---

## Task 13: E2E tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayE2ETest.java`

- [ ] **Step 1: Write E2E tests**

```java
// runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayE2ETest.java
package io.casehub.qhorus.gateway;

import static org.junit.jupiter.api.Assertions.*;

import java.time.Instant;
import java.util.Map;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.gateway.*;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.casehub.qhorus.testing.gateway.RecordingChannelBackend;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@QuarkusTest
class ChannelGatewayE2ETest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelGateway gateway;

    @BeforeEach @Transactional
    void setUp() {
        tools.createChannel("e2e-gw-1", "E2E gateway test", "append",
                null, null, null, null, null, null);
    }

    @AfterEach @Transactional
    void tearDown() {
        tools.deleteChannel("e2e-gw-1", null, true);
    }

    @Test
    void agentPosts_observerReceives_fullFanOut() throws Exception {
        var ch = tools.listChannels().stream()
                .filter(c -> "e2e-gw-1".equals(c.name())).findFirst().orElseThrow();
        RecordingChannelBackend observer = new RecordingChannelBackend("panel", ActorType.HUMAN);
        gateway.registerBackend(ch.channelId(), observer, "human_observer");

        // Agent posts via MCP tool
        tools.sendMessage("e2e-gw-1", "agent-a", "event",
                "analysis_complete", null, null, null, null, null);

        Thread.sleep(300); // allow virtual thread fan-out
        assertEquals(1, observer.posts().size());
        assertEquals("analysis_complete", observer.posts().get(0).content());
    }

    @Test
    @Transactional
    void humanReplies_viaParticipatingBackend_createsObligationInLedger() {
        var ch = tools.listChannels().stream()
                .filter(c -> "e2e-gw-1".equals(c.name())).findFirst().orElseThrow();
        ChannelRef ref = new ChannelRef(ch.channelId(), "e2e-gw-1");
        InboundHumanMessage reply = new InboundHumanMessage(
                "whatsapp-user-99", "Please stop and summarise", Instant.now(), Map.of());

        gateway.receiveHumanMessage(ref, reply);

        // Message should appear in channel with HUMAN sender
        var ledger = tools.listLedgerEntries("e2e-gw-1", null, null, null, null, null, 10);
        assertTrue(ledger.entries().stream()
                .anyMatch(e -> "human:whatsapp-user-99".equals(e.actorId())));
    }

    @Test
    @Transactional
    void observerSignal_appearsAsEvent_notSpeechAct() {
        var ch = tools.listChannels().stream()
                .filter(c -> "e2e-gw-1".equals(c.name())).findFirst().orElseThrow();
        ChannelRef ref = new ChannelRef(ch.channelId(), "e2e-gw-1");
        ObserverSignal signal = new ObserverSignal(
                "dashboard-user", "flag:urgent", Instant.now(), Map.of());

        gateway.receiveObserverSignal(ref, signal);

        var messages = tools.checkMessages("e2e-gw-1", "observer",
                null, null, null, null, null, null, null, true);
        var eventMsg = messages.messages().stream()
                .filter(m -> "human:dashboard-user".equals(m.sender()))
                .findFirst().orElseThrow();
        assertEquals("event", eventMsg.type().toLowerCase());
    }

    @Test
    void deleteChannel_closesAllBackends() throws Exception {
        var ch = tools.listChannels().stream()
                .filter(c -> "e2e-gw-1".equals(c.name())).findFirst().orElseThrow();
        RecordingChannelBackend obs = new RecordingChannelBackend("to-close", ActorType.HUMAN);
        gateway.registerBackend(ch.channelId(), obs, "human_observer");

        tools.deleteChannel("e2e-gw-1", null, false);

        assertEquals(1, obs.closes().size());
        assertTrue(gateway.listBackends(ch.channelId()).isEmpty());
    }
}
```

- [ ] **Step 2: Run E2E tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelGatewayE2ETest -pl runtime -Dno-format 2>&1 | tail -10
```

Expected: all pass.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/qhorus/gateway/ChannelGatewayE2ETest.java
git commit -m "test(gateway): end-to-end tests — fan-out, inbound, observer enforcement, lifecycle

Refs #131"
```

---

## Task 14: Full build + code review

- [ ] **Step 1: Run full project build including examples**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`. All modules including `testing`, `examples/type-system`, `examples/normative-layout` pass.

- [ ] **Step 2: Invoke code review**

Use `superpowers:requesting-code-review` skill to review all changed files before proceeding to documentation.

---

## Task 15: ADR — channel backend abstraction

**Files:**
- Create: `docs/adr/ADR-0006-channel-backend-abstraction.md`

- [ ] **Step 1: Write ADR**

```markdown
# ADR-0006 — Channel Backend Abstraction

**Date:** 2026-05-04  
**Status:** Accepted  
**Refs:** casehubio/qhorus#131

## Context

Qhorus channels were designed for agent-to-agent communication within the Qhorus topology.
Making them backend-agnostic (WhatsApp, Slack, Claudony panel, A2A) required a gateway layer
without breaking existing normative guarantees.

## Decision

Introduce a `ChannelGateway` with a `ChannelBackend` SPI hierarchy:

- `AgentChannelBackend` — always registered (`QhorusChannelBackend`); wraps `MessageService`
- `HumanParticipatingChannelBackend` — at most one per channel; full speech act inbound
- `HumanObserverChannelBackend` — unlimited per channel; inbound capped to `EVENT`

Actor vocabulary aligned with `casehub-ledger`'s `ActorType` enum (`HUMAN`, `AGENT`, `SYSTEM`).

## Alternatives Considered

**Decorated MessageService** — simpler but inbound path was awkward; coherence invariant
unenforceable at type level.

**CDI event bus** — loosely coupled but non-deterministic ordering; enforcing
"at most one participating backend" required a registry anyway.

## At-Most-One HumanParticipatingChannelBackend

Two human participatory surfaces on the same channel produce two independent conversation
threads. An agent cannot know they represent different threads and may honour both —
violating Qhorus's coherence invariant. This constraint is enforced at registration time
with `DuplicateParticipatingBackendException`.

## ActorType Alignment

`HumanParticipatingChannelBackend` and `HumanObserverChannelBackend` are named after
`ActorType.HUMAN`. `AgentChannelBackend` aligns with `ActorType.AGENT`. This ensures
backend type names, ledger entries, and `ActorTypeResolver` share one vocabulary.

## A2A — Protocol Bridge, Not Transport

A2A carries both `role: "user"` (human) and `role: "agent"` (AI). It is a protocol
multiplexer that dispatches into the appropriate gateway inbound path based on resolved
actor type. Tracked in casehubio/qhorus#135 as a separate sub-issue.

## Consequences

- All existing MCP tool behaviour is preserved; gateway is additive above `MessageService`
- External backend fan-out is async (Java 21 virtual threads); failures are non-fatal
- Inbound normalisation is pluggable via `InboundNormaliser` SPI; default always returns QUERY
- Backend registrations are in-memory; persistence deferred to casehubio/qhorus#132
```

- [ ] **Step 2: Commit**

```bash
git add docs/adr/ADR-0006-channel-backend-abstraction.md
git commit -m "docs(adr): ADR-0006 channel backend abstraction decision

Refs #131"
```

---

## Task 16: Documentation sweep

**Files:**
- Modify: `docs/agent-mesh-framework.md`
- Modify: `docs/agent-protocol-comparison.md`
- Modify: `docs/multi-agent-framework-comparison.md`
- Modify: `docs/normative-layer.md`
- Modify: `docs/normative-summary.md`
- Modify: `docs/normative-channel-layout.md`

- [ ] **Step 1: Update `agent-mesh-framework.md`**

Add a new Part after Part 2 (Channels):

**Part N — The Channel Gateway**

Cover: what the gateway is, backend taxonomy table (same as spec section 1), outbound flow diagram, inbound flows (participating vs observer), `InboundNormaliser` SPI, how to implement a backend (implement the appropriate interface, inject `ChannelGateway`, call `registerBackend` on channel open).

- [ ] **Step 2: Update `agent-protocol-comparison.md`**

Replace "Phase 9 optional endpoint" references. Update the A2A section to describe `A2AChannelBackend` as a protocol bridge (separate issue #135). Add ACP note with same framing.

- [ ] **Step 3: Update `multi-agent-framework-comparison.md`**

Update A2A row from `🔶 Phase 9 planned` to `✅ via ChannelGateway — gateway SPI + A2AChannelBackend protocol bridge (#135)`.

- [ ] **Step 4: Update `normative-layer.md`**

Add a paragraph in the infrastructure section: "The `ChannelGateway` preserves normative guarantees regardless of backend — every message, whether from an agent posting via MCP or a human replying via WhatsApp, flows through `MessageService` and `LedgerWriteService`. External backends receive post-persistence fan-out only."

- [ ] **Step 5: Update `normative-summary.md`**

Update the gap analysis section: mark the "channel backend abstraction" gap as closed by this implementation.

- [ ] **Step 6: Update `normative-channel-layout.md`**

Add a note: "Channel layouts are backend-agnostic. The normative layer (speech act semantics, commitment lifecycle, ledger recording) applies regardless of which external transport backs the channel."

- [ ] **Step 7: Cross-reference sweep**

Search for any remaining "Phase 9" references across all docs:
```bash
grep -r "Phase 9" /Users/mdproctor/claude/casehub/qhorus/docs/
```
Fix all occurrences.

Search for `"human"` as a literal sender outside of `Senders.HUMAN`:
```bash
grep -rn '"human"' runtime/src/main/java/
```
Fix any remaining magic strings.

- [ ] **Step 8: Commit**

```bash
git add docs/
git commit -m "docs: gateway documentation sweep — agent-mesh-framework, protocol comparisons, normative docs

Refs #131"
```

---

## Task 17: `CLAUDE.md` + parent repo docs

**Files:**
- Modify: `CLAUDE.md`
- Modify: `~/claude/casehub/parent/docs/repos/casehub-qhorus.md` (if accessible)
- Create: `~/claude/casehub/parent/docs/conventions/qhorus-actor-type-mapping.md` (if accessible)

- [ ] **Step 1: Update `CLAUDE.md` project structure section**

Add to Project Structure:
```
├── runtime/src/main/java/io/casehub/qhorus/runtime/
│   └── gateway/
│       ├── ChannelGateway.java        — registration, outbound fan-out, inbound normalisation
│       ├── QhorusChannelBackend.java  — default AgentChannelBackend, wraps MessageService
│       ├── DefaultInboundNormaliser.java — @DefaultBean, always returns QUERY
│       ├── DuplicateParticipatingBackendException.java
│       └── Senders.java               — named constants (HUMAN = "human")
```

Add to API module structure:
```
└── api/src/main/java/io/casehub/qhorus/api/
    └── gateway/
        ├── ChannelBackend.java, AgentChannelBackend.java
        ├── HumanParticipatingChannelBackend.java, HumanObserverChannelBackend.java
        ├── InboundNormaliser.java, ChannelRef.java
        └── OutboundMessage.java, InboundHumanMessage.java, ObserverSignal.java, NormalisedMessage.java
```

Add to Testing conventions:
- `RecordingChannelBackend` in `casehub-qhorus-testing` — records `post()`, `open()`, `close()` calls for assertion; use in gateway unit and integration tests

Add to MCP tool surface:
- `list_backends(channel_name)` — list registered backends
- `deregister_backend(channel_name, backend_id)` — remove a backend (cannot remove qhorus-internal)

- [ ] **Step 2: Update parent repo docs**

In `casehubio/parent/docs/repos/casehub-qhorus.md`, add to the Key Abstractions section:

```markdown
### Channel Gateway

| Class | Purpose |
|---|---|
| `ChannelGateway` | Routes outbound messages to all registered backends; handles inbound normalisation |
| `ChannelBackend` | SPI base — `AgentChannelBackend`, `HumanParticipatingChannelBackend`, `HumanObserverChannelBackend` |
| `QhorusChannelBackend` | Default `AgentChannelBackend` — always registered, wraps `MessageService` |
| `InboundNormaliser` | SPI — maps raw human inbound to Qhorus `MessageType`; `@DefaultBean` always returns QUERY |
| `Senders` | Named constants: `HUMAN = "human"` |
```

Create `docs/conventions/qhorus-actor-type-mapping.md`:

```markdown
# Qhorus Actor Type Mapping Convention

## ActorType Vocabulary

All casehubio projects use `io.casehub.ledger.api.model.ActorType`:
- `AGENT` — autonomous AI agent acting programmatically
- `HUMAN` — human user acting through a UI, API, or messaging platform
- `SYSTEM` — automated system process (scheduler, rule engine, pipeline)

## A2A Protocol Role Mapping

| A2A role | Qhorus ActorType | Notes |
|---|---|---|
| `"user"` | `HUMAN` | Explicit rule in `ActorTypeResolver` (casehubio/ledger#75) |
| `"agent"` | `AGENT` | Explicit rule in `ActorTypeResolver` (casehubio/ledger#75) |

## Human Sender ID Convention

Inbound messages from human backends must use the format `human:{externalSenderId}` as
the `senderInstanceId`. This ensures `ActorTypeResolver` correctly stamps `ActorType.HUMAN`.
Examples: `human:+447911123456`, `human:slack-U012AB3CD`, `human:whatsapp-user-99`.

## Interop Contract for A2A AI Callers

An A2A caller that is an AI agent (not a biological human) SHOULD signal this by:
1. Registering as a Qhorus Instance before calling
2. Including an Agent Card URL in request metadata
3. Using a sender ID in versioned persona format (`model:persona@version`)
4. Setting the `x-qhorus-actor-type: AGENT` header

Without any of these signals, the caller is conservatively classified as `HUMAN`.
```

- [ ] **Step 3: Commit CLAUDE.md and parent repo docs separately**

```bash
# Qhorus CLAUDE.md
git add CLAUDE.md
git commit -m "docs(claude): update project structure, testing conventions, MCP surface for gateway

Refs #131"

# Parent repo (if accessible)
cd ~/claude/casehub/parent
git add docs/repos/casehub-qhorus.md docs/conventions/qhorus-actor-type-mapping.md
git commit -m "docs: update casehub-qhorus platform doc and add actor-type mapping convention

Refs casehubio/qhorus#131"
git push
cd -
```

---

## Task 18: Final verification + push

- [ ] **Step 1: Full clean build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`. All modules. Zero test failures.

- [ ] **Step 2: Verify test count has grown**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | grep "Tests run:"
```

Expected: total test count greater than 970 (was 970 before this feature).

- [ ] **Step 3: Push to GitHub**

```bash
git push origin main
```

- [ ] **Step 4: Close sub-issues and comment on epic**

```bash
# Close the four sub-issues created in Task 1 (replace NNN with actual issue numbers)
gh issue close NNN --repo casehubio/qhorus --comment "Implemented in this session. All tests pass."
# (repeat for each sub-issue)

# Comment on epic #131
gh issue comment 131 --repo casehubio/qhorus \
  --body "Core implementation complete. Gateway SPI, QhorusChannelBackend, DefaultInboundNormaliser, MCP wiring, full test suite, ADR, and documentation sweep all landed on main.

Remaining open sub-issues:
- casehubio/qhorus#135 — A2A protocol bridge (blocked on casehubio/ledger#75)
- casehubio/qhorus#132 — Delivery guarantees (persistent backend registration)"
```

---

## Self-Review Checklist

### Spec coverage

| Spec section | Task covering it |
|---|---|
| SPI contracts (api module) | Task 2 |
| `RecordingChannelBackend` in testing | Task 3 |
| `DuplicateParticipatingBackendException`, `Senders` | Task 4 |
| `QhorusChannelBackend` | Task 5 |
| `DefaultInboundNormaliser` | Task 6 |
| `ChannelGateway` — registration, outbound, inbound | Task 7 |
| Wire `sendMessage` through gateway | Task 8 |
| Wire `create_channel`, `delete_channel` | Task 9 |
| New MCP tools | Task 10 |
| `Senders.HUMAN` cleanup | Task 11 |
| Integration + robustness tests | Task 12 |
| E2E tests | Task 13 |
| Code review | Task 14 |
| ADR | Task 15 |
| Documentation sweep | Tasks 16–17 |
| Full build + push | Task 18 |

### Type consistency

- `ChannelRef(UUID id, String name)` — used consistently in Tasks 2, 5, 7, 12, 13
- `OutboundMessage(UUID messageId, String sender, MessageType type, String content, UUID correlationId, ActorType senderActorType)` — consistent
- `ChannelGateway.BackendRegistration(String backendId, String backendType, ActorType actorType)` — consistent with `BackendInfo(String backendId, String backendType, String actorType)` in MCP response (actorType lowercased for MCP string output)
- `Senders.HUMAN = "human"` — used in Task 11, referenced in Task 7 inbound flows
- `"qhorus-internal"` — `QhorusChannelBackend.backendId()` constant, deregister guard — consistent

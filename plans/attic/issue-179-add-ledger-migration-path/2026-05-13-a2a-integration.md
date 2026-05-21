# A2A Robust Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix four bugs in A2A integration: gateway bypass, hardcoded message type, no durable task state, and missing explicit actorType on every message.

**Architecture:** Three-phase: (1) foundational `message.actorType` column + `MessageService.send()` signature change across ~90 call sites; (2) new `A2AActorResolver` and `A2AChannelBackend` components with TDD; (3) `A2AResource` refactored as a thin adapter using `CommitmentStore` for durable task state.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache ORM, H2 (tests), AssertJ, JUnit 5. No Mockito — use direct CDI wiring with in-memory stores.

**Build command (all tasks):**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

**Run single test class:**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClassName -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

**Git discipline:** Every commit must be in the project repo (`/Users/mdproctor/claude/casehub/qhorus`). Before any `git` command run: `git -C /Users/mdproctor/claude/casehub/qhorus rev-parse --show-toplevel` to confirm.

**Issue reference:** All commits use `Refs #135` until the final task which uses `Closes #135`.

---

## File Map

| Action | File |
|--------|------|
| Create | `runtime/src/main/resources/db/migration/V8__add_actor_type_to_message.sql` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/message/Message.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AActorResolver.java` |
| Create | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java` |
| Modify | `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java` |
| Create | `runtime/src/test/java/io/casehub/qhorus/api/A2AActorResolverTest.java` |
| Create | `runtime/src/test/java/io/casehub/qhorus/api/A2AChannelBackendIntegrationTest.java` |
| Modify | `runtime/src/test/java/io/casehub/qhorus/api/A2ASendMessageTest.java` |
| Modify | `runtime/src/test/java/io/casehub/qhorus/api/A2AGetTaskTest.java` |
| Modify | ~80 test files (messageService.send() call sites — listed in Task 3) |
| Modify | `/Users/mdproctor/claude/casehub/parent/docs/protocols/qhorus-actor-type-mapping.md` |

---

## Task 1: Flyway V8 + Message.actorType field

**Files:**
- Create: `runtime/src/main/resources/db/migration/V8__add_actor_type_to_message.sql`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/Message.java`

- [ ] **Step 1: Create the Flyway migration**

```sql
-- V8__add_actor_type_to_message.sql
ALTER TABLE message ADD COLUMN actor_type VARCHAR(10) NOT NULL DEFAULT 'HUMAN';
```

- [ ] **Step 2: Add actorType field to Message entity**

Replace the entire `Message.java`:

```java
package io.casehub.qhorus.runtime.message;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.SequenceGenerator;
import jakarta.persistence.Table;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.api.message.MessageType;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity
@Table(name = "message")
@SequenceGenerator(name = "message_seq", sequenceName = "message_seq", allocationSize = 50)
public class Message extends PanacheEntityBase {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "message_seq")
    public Long id;

    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(nullable = false)
    public String sender;

    @Enumerated(EnumType.STRING)
    @Column(name = "message_type", nullable = false)
    public MessageType messageType;

    @Enumerated(EnumType.STRING)
    @Column(name = "actor_type", nullable = false)
    public ActorType actorType;

    @Column(columnDefinition = "TEXT")
    public String content;

    @Column(name = "correlation_id")
    public String correlationId;

    @Column(name = "in_reply_to")
    public Long inReplyTo;

    @Column(name = "reply_count", nullable = false)
    public int replyCount = 0;

    @Column(name = "artefact_refs")
    public String artefactRefs;

    @Column(name = "target")
    public String target;

    @Column(name = "commitment_id")
    public UUID commitmentId;

    @Column(name = "deadline")
    public Instant deadline;

    @Column(name = "acknowledged_at")
    public Instant acknowledgedAt;

    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (createdAt == null) {
            createdAt = Instant.now();
        }
    }
}
```

- [ ] **Step 3: Compile the runtime module to confirm no errors yet**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: COMPILE ERROR — `MessageService.send()` still assigns no actorType. That is expected — fix comes in Task 2.

---

## Task 2: MessageService.send() — new signature + all production callers

This is a coordinated breaking change. All 9 files must be updated together before the build is green.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java`

- [ ] **Step 1: Replace MessageService.java — one canonical send() signature**

Replace the three overloaded `send()` methods with a single one. Keep all other methods (`findById`, `pollAfter`, etc.) unchanged:

```java
@Transactional
public Message send(UUID channelId, String sender, MessageType type, String content,
        String correlationId, Long inReplyTo, String artefactRefs, String target,
        ActorType actorType) {
    channelService.findById(channelId)
            .ifPresent(ch -> messageTypePolicy.validate(ch, type));
    Message message = new Message();
    message.channelId = channelId;
    message.sender = sender;
    message.messageType = type;
    message.actorType = actorType;
    message.content = content;
    message.correlationId = correlationId;
    message.inReplyTo = inReplyTo;
    message.artefactRefs = artefactRefs;
    message.target = target;
    messageStore.put(message);

    if (message.correlationId != null) {
        switch (message.messageType) {
            case QUERY, COMMAND -> commitmentService.open(
                    message.commitmentId != null ? message.commitmentId : UUID.randomUUID(),
                    message.correlationId, message.channelId, message.messageType,
                    message.sender, message.target, message.deadline);
            case STATUS -> commitmentService.acknowledge(message.correlationId);
            case RESPONSE, DONE -> commitmentService.fulfill(message.correlationId);
            case DECLINE -> commitmentService.decline(message.correlationId);
            case FAILURE -> commitmentService.fail(message.correlationId);
            case HANDOFF -> commitmentService.delegate(message.correlationId, message.target);
            case EVENT -> { /* no commitment effect */ }
        }
    }

    if (inReplyTo != null) {
        messageStore.find(inReplyTo).ifPresent(parent -> parent.replyCount++);
    }

    channelService.updateLastActivity(channelId);
    return message;
}
```

Add `import io.casehub.ledger.api.model.ActorType;` to the imports.

- [ ] **Step 2: Update ReactiveMessageService.java — same new signature**

The reactive service returns `Uni<Message>`. Add `ActorType actorType` as the last parameter and set `message.actorType = actorType` after the existing field assignments (before `return messageStore.put(message)`). Add `import io.casehub.ledger.api.model.ActorType;`.

- [ ] **Step 3: Update ChannelGateway.java — both messageService.send() calls**

In `receiveHumanMessage()`, change:
```java
messageService.send(channel.id(), normalised.senderInstanceId(),
        normalised.type(), normalised.content(), null, null);
```
To:
```java
messageService.send(channel.id(), normalised.senderInstanceId(),
        normalised.type(), normalised.content(), null, null,
        null, null, ActorType.HUMAN);
```

In `receiveObserverSignal()`, change:
```java
messageService.send(channel.id(), "human:" + signal.externalSenderId(),
        MessageType.EVENT, signal.content(), null, null);
```
To:
```java
messageService.send(channel.id(), "human:" + signal.externalSenderId(),
        MessageType.EVENT, signal.content(), null, null,
        null, null, ActorType.HUMAN);
```

Add `import io.casehub.ledger.api.model.ActorType;`.

- [ ] **Step 4: Update QhorusChannelBackend.java — pass actorType from OutboundMessage**

Replace the `post()` method:
```java
@Override
public void post(ChannelRef channel, OutboundMessage message) {
    String correlationId = message.correlationId() != null
            ? message.correlationId().toString() : null;
    messageService.send(channel.id(), message.sender(), message.type(),
            message.content(), correlationId, null, null, null,
            message.actorType());
}
```

`OutboundMessage.actorType()` already exists — it was set when the OutboundMessage was constructed in `QhorusMcpTools.sendMessage()`.

- [ ] **Step 5: Update WatchdogEvaluationService.java — fix sender + pass SYSTEM**

In `fireAlert()`, change:
```java
messageService.send(notifChannel.get().id, "watchdog", MessageType.STATUS,
        alertContent, null, null, null, null);
```
To:
```java
messageService.send(notifChannel.get().id, "system:watchdog", MessageType.STATUS,
        alertContent, null, null, null, null, ActorType.SYSTEM);
```

Add `import io.casehub.ledger.api.model.ActorType;`.

- [ ] **Step 6: Update QhorusMcpTools.java — inject InstanceActorIdProvider + fix all 4 call sites**

**6a — Add injection.** In the `@Inject` field block, add:
```java
@Inject
io.casehub.qhorus.api.spi.InstanceActorIdProvider instanceActorIdProvider;
```

**6b — Main send path (line ~640).** Change:
```java
Message msg = messageService.send(ch.id, sender, msgType, content, corrId, inReplyTo, refsStr,
        normalisedTarget);
```
To:
```java
ActorType resolvedActorType = io.casehub.ledger.api.model.ActorTypeResolver.resolve(
        instanceActorIdProvider.resolve(sender));
Message msg = messageService.send(ch.id, sender, msgType, content, corrId, inReplyTo, refsStr,
        normalisedTarget, resolvedActorType);
```

**6c — fanOut OutboundMessage (line ~658).** Change `ActorTypeResolver.resolve(sender)` to `msg.actorType`:
```java
channelGateway.fanOut(ch.id, new OutboundMessage(
        UUID.randomUUID(), sender, msgType, content, corrUuid,
        msg.actorType));
```

**6d — System EVENT sends (lines ~458, ~1278, ~1303).** Each sends with sender `"system"`. Add `ActorType.SYSTEM` as last arg. Example:
```java
messageService.send(ch.id, "system", MessageType.EVENT, auditContent, null, null, null, null,
        ActorType.SYSTEM);
```
Apply this same pattern to all three system-sender `messageService.send()` calls in QhorusMcpTools.

- [ ] **Step 7: Update LedgerWriteService.java — use message.actorType directly**

In `record()`, change:
```java
entry.actorType = ActorTypeResolver.resolve(resolvedActorId);
```
To:
```java
entry.actorType = message.actorType;
```

Remove the `import io.casehub.ledger.api.model.ActorTypeResolver;` line (no longer used in this file).

- [ ] **Step 8: Update ReactiveLedgerWriteService.java — same change**

Find the equivalent `entry.actorType = ActorTypeResolver.resolve(...)` line and replace with `entry.actorType = message.actorType;`. Remove the `ActorTypeResolver` import if unused.

- [ ] **Step 9: Compile the runtime module (not tests yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: BUILD SUCCESS on main sources. Test compilation will fail — that's Task 3.

- [ ] **Step 10: Commit production changes**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/resources/db/migration/V8__add_actor_type_to_message.sql \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/Message.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
refactor(message): explicit ActorType on MessageService.send() — platform-wide

Single canonical send() signature replacing three overloads. Every call site
declares the actor type explicitly. LedgerWriteService reads message.actorType
directly — no more string re-derivation at ledger-write time.

QhorusMcpTools injects InstanceActorIdProvider to correctly classify Claudony
session instanceIds via the enrichment chain before setting message.actorType.
WatchdogEvaluationService sender corrected to "system:watchdog" (was "watchdog"
which classified as HUMAN via catch-all — a latent bug now fixed).

Refs #135

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Test call sites — update all ~80 messageService.send() callers

**Pattern:** Add `ActorTypeResolver.resolve(sender)` as the last argument, or an explicit `ActorType` constant where the actor type is known.

Import needed in each test file: `import io.casehub.ledger.api.model.ActorType;` and/or `import io.casehub.ledger.api.model.ActorTypeResolver;`

**Rule for choosing the argument:**
- Sender is `"alice"`, `"bob"`, `"carol"`, `"orchestrator"`, `"external-orchestrator"`, `"smoke-alice"`, `"smoke-bob"`, or any arbitrary name → `ActorTypeResolver.resolve(sender)` (evaluates to HUMAN via catch-all — correct for test humans)
- Sender is `"agent"`, `"agent-a"`, `"agent-b"`, `"agent-x"`, `"agent-test"`, `"researcher-001"`, `"reviewer-001"`, `"monitor"`, `"writer"`, `"sender"`, `"contributor-*"` → `ActorType.AGENT`
- Sender is `"system"`, `"watchdog"` → `ActorType.SYSTEM`
- Sender starts with `"human:"` → `ActorType.HUMAN`

**Simpler uniform approach:** Use `ActorTypeResolver.resolve(sender)` everywhere. This is always correct — it will evaluate AGENT for "agent", SYSTEM for "system", HUMAN for everything else. This avoids per-site judgment calls.

- [ ] **Step 1: Use IntelliJ build to find all failing test compilations**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep "error:" | head -30
```

This lists each failing call site. Work through them file by file.

- [ ] **Step 2: Update each test file**

For each failing `messageService.send(...)` call, append `, ActorTypeResolver.resolve(sender)` where `sender` is the string literal or variable used as the second argument.

**Example — before:**
```java
messageService.send(channel.id, "alice", MessageType.QUERY, "Question?", corrId, null);
```
**After:**
```java
messageService.send(channel.id, "alice", MessageType.QUERY, "Question?", corrId, null,
        null, null, ActorTypeResolver.resolve("alice"));
```

**Example with 8-arg (already has artefactRefs + target):**
```java
// Before:
messageService.send(channelId, "agent-1", MessageType.QUERY, "content", corrId, null, null, target)
// After:
messageService.send(channelId, "agent-1", MessageType.QUERY, "content", corrId, null, null, target,
        ActorTypeResolver.resolve("agent-1"))
```

Files to update (from the earlier search — apply the pattern to all occurrences in each):
- `SmokeTest.java`
- `ReactiveSmokeTest.java`
- `MessageServiceTest.java`
- `MessageServiceTypeEnforcementTest.java`
- `MessageOrderingTest.java`
- `WaitForReplyTest.java`
- `WaitForReplyEdgeCaseTest.java`
- `WaitForReplyCorrelationIsolationTest.java`
- `WaitManagementTest.java`
- `BarrierConcurrentWriteTest.java`
- `EphemeralEdgeCaseTest.java`
- `EphemeralDoubleDeliveryTest.java`
- `CollectAtomicityTest.java`
- `ChannelToolTest.java`
- `DeleteChannelToolTest.java`
- `GetMessageToolTest.java`
- `LastWriteEdgeCaseTest.java` (read-only references — no send calls)
- `LastWriteArtefactRefsTest.java` (read-only references — no send calls)
- `ChannelServiceTest.java`
- `NormativeLayoutTypeEnforcementTest.java` (in `examples/normative-layout/`)
- `NormativeLayoutObligationTest.java`
- `NormativeLayoutRobustnessTest.java`
- `SecureCodeReviewScenario.java`
- `QhorusChannelBackendTest.java` (if it has send() calls)

- [ ] **Step 3: Verify test compilation green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: BUILD SUCCESS — all modules compile including tests.

- [ ] **Step 4: Run the full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: all previously-passing tests still pass. Count should match or exceed the pre-change count (1011+).

- [ ] **Step 5: Commit test updates**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/test examples/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
test: update messageService.send() call sites with explicit ActorType

Mechanical update: all ~80 test call sites now pass ActorTypeResolver.resolve(sender)
as the explicit actorType argument following the new required-parameter signature.

Refs #135

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: CommitmentService.findByCorrelationId()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/message/CommitmentServiceTest.java` (or create if needed)

- [ ] **Step 1: Write the failing test**

Add to the existing `CommitmentServiceTest` (or nearest relevant test class in the `testing` module):

```java
@Test
void findByCorrelationId_returnsEmptyWhenNotFound() {
    Optional<Commitment> result = commitmentService.findByCorrelationId("no-such-corr-id");
    assertThat(result).isEmpty();
}

@Test
void findByCorrelationId_returnsCommitmentWhenExists() {
    commitmentService.open(UUID.randomUUID(), "test-corr-find", UUID.randomUUID(),
            MessageType.QUERY, "sender-a", null, null);
    Optional<Commitment> result = commitmentService.findByCorrelationId("test-corr-find");
    assertThat(result).isPresent();
    assertThat(result.get().correlationId).isEqualTo("test-corr-find");
}
```

- [ ] **Step 2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CommitmentServiceTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: FAIL — `findByCorrelationId` method does not exist.

- [ ] **Step 3: Add the method to CommitmentService.java**

Add after the existing `expireOverdue()` method:

```java
/**
 * Returns the active Commitment for the given correlation ID, if any.
 * Returns the most recent (by creation) when multiple exist (e.g. after delegation).
 */
public Optional<Commitment> findByCorrelationId(String correlationId) {
    if (correlationId == null || correlationId.isBlank()) {
        return Optional.empty();
    }
    return store.findByCorrelationId(correlationId);
}
```

- [ ] **Step 4: Run to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CommitmentServiceTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java \
  runtime/src/test/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(commitment): expose findByCorrelationId for A2A task state lookup

Refs #135

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: A2AActorResolver — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/api/A2AActorResolverTest.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AActorResolver.java`

- [ ] **Step 1: Write all failing tests**

```java
package io.casehub.qhorus.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.qhorus.runtime.api.A2AActorResolver;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.testing.InMemoryInstanceStore;

/**
 * Pure unit test — no Quarkus context needed.
 */
class A2AActorResolverTest {

    private InstanceService instanceService;
    private A2AActorResolver resolver;

    @BeforeEach
    void setup() {
        InMemoryInstanceStore store = new InMemoryInstanceStore();
        instanceService = new InstanceService();
        instanceService.instanceStore = store;
        resolver = new A2AActorResolver();
        resolver.instanceService = instanceService;
    }

    // ── role:"agent" unconditional ─────────────────────────────────────────────

    @Test
    void roleAgent_noHeader_noMetadata_isAgent() {
        assertThat(resolver.resolve("agent", null, Map.of())).isEqualTo(ActorType.AGENT);
    }

    @Test
    void roleAgent_withHumanHeader_stillAgent() {
        // role:"agent" is unconditional — header cannot override it to HUMAN
        assertThat(resolver.resolve("agent", "HUMAN", Map.of())).isEqualTo(ActorType.AGENT);
    }

    // ── Step 1: explicit header ────────────────────────────────────────────────

    @Test
    void roleUser_headerAgent_isAgent() {
        assertThat(resolver.resolve("user", "AGENT", Map.of())).isEqualTo(ActorType.AGENT);
    }

    @Test
    void roleUser_headerHuman_isHuman() {
        assertThat(resolver.resolve("user", "HUMAN", Map.of())).isEqualTo(ActorType.HUMAN);
    }

    @Test
    void roleUser_headerSystem_isSystem() {
        assertThat(resolver.resolve("user", "SYSTEM", Map.of())).isEqualTo(ActorType.SYSTEM);
    }

    @Test
    void roleUser_invalidHeader_fallsThrough_isHuman() {
        // Invalid header value falls through to chain — no exception
        assertThat(resolver.resolve("user", "BANANA", Map.of())).isEqualTo(ActorType.HUMAN);
    }

    // ── Step 2: Instance registry ──────────────────────────────────────────────

    @Test
    void roleUser_agentIdInRegistry_isAgent() {
        instanceService.register("registry-agent", "test-cap", false);
        assertThat(resolver.resolve("user", null,
                Map.of("agentId", "registry-agent"))).isEqualTo(ActorType.AGENT);
    }

    // ── Step 3: agentCardUrl ───────────────────────────────────────────────────

    @Test
    void roleUser_agentCardUrlPresent_isAgent() {
        assertThat(resolver.resolve("user", null,
                Map.of("agentCardUrl", "https://example.com/.well-known/agent-card.json")))
                .isEqualTo(ActorType.AGENT);
    }

    @Test
    void roleUser_agentCardUrlBlank_fallsThrough() {
        // Blank URL is not a valid signal
        assertThat(resolver.resolve("user", null,
                Map.of("agentCardUrl", ""))).isEqualTo(ActorType.HUMAN);
    }

    // ── Steps 4+5: ActorTypeResolver on agentId ────────────────────────────────

    @Test
    void roleUser_personaAgentId_isAgent() {
        assertThat(resolver.resolve("user", null,
                Map.of("agentId", "claude:orchestrator@v1"))).isEqualTo(ActorType.AGENT);
    }

    @Test
    void roleUser_systemAgentId_isSystem() {
        assertThat(resolver.resolve("user", null,
                Map.of("agentId", "system:scheduler"))).isEqualTo(ActorType.SYSTEM);
    }

    // ── Step 6: default ────────────────────────────────────────────────────────

    @Test
    void roleUser_noSignals_isHuman() {
        assertThat(resolver.resolve("user", null, Map.of())).isEqualTo(ActorType.HUMAN);
    }

    @Test
    void roleUser_nullMetadata_isHuman() {
        // null metadata should not throw NPE
        assertThat(resolver.resolve("user", null, Map.of())).isEqualTo(ActorType.HUMAN);
    }

    @Test
    void unknownRole_noSignals_isHuman() {
        // Unknown roles fall through to the chain, default HUMAN
        assertThat(resolver.resolve("orchestrator", null, Map.of())).isEqualTo(ActorType.HUMAN);
    }
}
```

- [ ] **Step 2: Run to confirm all tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2AActorResolverTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: FAIL — class not found.

- [ ] **Step 3: Implement A2AActorResolver**

```java
package io.casehub.qhorus.runtime.api;

import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.ActorTypeResolver;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.quarkus.arc.properties.UnlessBuildProperty;

/**
 * Resolves {@link ActorType} for inbound A2A messages.
 *
 * <p>For {@code role:"agent"} — unconditional AGENT, chain skipped.
 * For {@code role:"user"} and unknown roles — 6-step chain:
 * explicit header, instance registry, agent card URL,
 * ActorTypeResolver on agentId (covers persona + system), default HUMAN.
 */
@ApplicationScoped
@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)
public class A2AActorResolver {

    @Inject
    InstanceService instanceService;

    public ActorType resolve(String role, String actorTypeHeader, Map<String, String> metadata) {
        // role:"agent" is unconditional — the chain does not apply
        if ("agent".equals(role)) {
            return ActorType.AGENT;
        }

        // Step 1: explicit x-qhorus-actor-type header
        if (actorTypeHeader != null && !actorTypeHeader.isBlank()) {
            try {
                return ActorType.valueOf(actorTypeHeader.toUpperCase());
            } catch (IllegalArgumentException ignored) {
                // invalid value — fall through to chain
            }
        }

        String agentId = metadata.get("agentId");

        // Step 2: Qhorus Instance registry lookup
        if (agentId != null && instanceService.findByInstanceId(agentId).isPresent()) {
            return ActorType.AGENT;
        }

        // Step 3: A2A Agent Card URL (survives relay, requires no Qhorus knowledge)
        String agentCardUrl = metadata.get("agentCardUrl");
        if (agentCardUrl != null && !agentCardUrl.isBlank()) {
            return ActorType.AGENT;
        }

        // Steps 4+5: delegate to canonical ActorTypeResolver (covers persona format + system:*)
        if (agentId != null) {
            ActorType fromId = ActorTypeResolver.resolve(agentId);
            if (fromId != ActorType.HUMAN) {
                return fromId;
            }
        }

        // Step 6: conservative default
        return ActorType.HUMAN;
    }
}
```

- [ ] **Step 4: Run all A2AActorResolverTest tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2AActorResolverTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: 13 tests, 0 failures.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AActorResolver.java \
  runtime/src/test/java/io/casehub/qhorus/api/A2AActorResolverTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(a2a): A2AActorResolver — 6-step sender identity resolution chain

role:"agent" → unconditional AGENT. For role:"user": explicit header,
instance registry, agentCardUrl, ActorTypeResolver on agentId, default HUMAN.

Refs #135

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: A2AChannelBackend — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/api/A2AChannelBackendIntegrationTest.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java`

- [ ] **Step 1: Write failing integration tests**

```java
package io.casehub.qhorus.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.api.A2AChannelBackend;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration tests for A2AChannelBackend.
 * Uses @QuarkusTest to verify gateway registration and message routing end-to-end.
 */
@QuarkusTest
@TestProfile(A2AEnabledProfile.class)
class A2AChannelBackendIntegrationTest {

    @Inject
    A2AChannelBackend a2aBackend;

    @Inject
    ChannelGateway channelGateway;

    @Inject
    QhorusMcpTools tools;

    @Inject
    io.casehub.qhorus.runtime.channel.ChannelService channelService;

    @Test
    void ensureRegistered_calledTwiceSameChannel_registersOnce() {
        tools.createChannel("a2a-backend-reg-1", "Test", "APPEND", null, null, null, null, null, null);
        io.casehub.qhorus.runtime.channel.Channel ch =
                channelService.findByName("a2a-backend-reg-1").orElseThrow();
        UUID channelId = ch.id;
        io.casehub.qhorus.api.gateway.ChannelRef ref =
                new io.casehub.qhorus.api.gateway.ChannelRef(channelId, "a2a-backend-reg-1");

        a2aBackend.ensureRegistered(channelId, ref);
        a2aBackend.ensureRegistered(channelId, ref);

        long a2aCount = channelGateway.listBackends(channelId).stream()
                .filter(b -> "a2a".equals(b.backendId()))
                .count();
        assertThat(a2aCount).isEqualTo(1);
    }

    @Test
    void receive_roleAgent_createsResponseMessage() {
        tools.createChannel("a2a-backend-recv-1", "Test", "APPEND", null, null, null, null, null, null);
        String correlationId = UUID.randomUUID().toString();

        String returned = a2aBackend.receive("a2a-backend-recv-1", "agent",
                "work result", correlationId, Map.of(), null);

        assertThat(returned).isEqualTo(correlationId);
        QhorusMcpTools.CheckResult check = tools.checkMessages("a2a-backend-recv-1", 0L, 10, null, null, null);
        assertThat(check.messages()).hasSize(1);
        assertThat(check.messages().get(0).messageType()).isEqualTo("RESPONSE");
        assertThat(check.messages().get(0).sender()).isEqualTo("agent");
    }

    @Test
    void receive_roleUserNoSignals_createsQueryWithHumanSender() {
        tools.createChannel("a2a-backend-recv-2", "Test", "APPEND", null, null, null, null, null, null);

        a2aBackend.receive("a2a-backend-recv-2", "user",
                "help me", null, Map.of(), null);

        QhorusMcpTools.CheckResult check = tools.checkMessages("a2a-backend-recv-2", 0L, 10, null, null, null);
        assertThat(check.messages()).hasSize(1);
        assertThat(check.messages().get(0).messageType()).isEqualTo("QUERY");
        assertThat(check.messages().get(0).sender()).isEqualTo("human:user");
    }

    @Test
    void receive_roleUserWithPersonaAgentId_createsQueryWithPersonaSender() {
        tools.createChannel("a2a-backend-recv-3", "Test", "APPEND", null, null, null, null, null, null);

        a2aBackend.receive("a2a-backend-recv-3", "user",
                "delegate this", null,
                Map.of("agentId", "claude:orchestrator@v1"), "AGENT");

        QhorusMcpTools.CheckResult check = tools.checkMessages("a2a-backend-recv-3", 0L, 10, null, null, null);
        assertThat(check.messages()).hasSize(1);
        assertThat(check.messages().get(0).sender()).isEqualTo("claude:orchestrator@v1");
    }

    @Test
    void receive_noTaskId_generatesCorrelationId() {
        tools.createChannel("a2a-backend-recv-4", "Test", "APPEND", null, null, null, null, null, null);

        String correlationId = a2aBackend.receive("a2a-backend-recv-4", "user",
                "hello", null, Map.of(), null);

        assertThat(correlationId).isNotBlank();
        assertThat(() -> UUID.fromString(correlationId)).doesNotThrowAnyException();
    }

    @Test
    void post_logsOnly_doesNotThrow() {
        io.casehub.qhorus.api.gateway.ChannelRef ref =
                new io.casehub.qhorus.api.gateway.ChannelRef(UUID.randomUUID(), "test");
        io.casehub.qhorus.api.gateway.OutboundMessage msg =
                new io.casehub.qhorus.api.gateway.OutboundMessage(
                        UUID.randomUUID(), "agent", io.casehub.qhorus.api.message.MessageType.DONE,
                        "done", UUID.randomUUID(), io.casehub.ledger.api.model.ActorType.AGENT);

        // Must not throw
        a2aBackend.post(ref, msg);
    }
}
```

- [ ] **Step 2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2AChannelBackendIntegrationTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: FAIL — class not found.

- [ ] **Step 3: Implement A2AChannelBackend**

```java
package io.casehub.qhorus.runtime.api;

import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.ActorTypeResolver;
import io.casehub.qhorus.api.gateway.ChannelBackend;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.arc.properties.UnlessBuildProperty;

/**
 * Protocol bridge backend that registers A2A as a first-class gateway participant.
 *
 * <p>Handles inbound A2A messages by resolving the sender's actor type and
 * routing through {@link QhorusMcpTools#sendMessage} to get the full pipeline
 * (ledger, fanOut, commitment tracking). {@link #post} is the outbound hook
 * called by {@link ChannelGateway#fanOut} — currently a logging no-op, the
 * correct hook for future SSE streaming (casehubio/qhorus#147).
 */
@ApplicationScoped
@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)
public class A2AChannelBackend implements ChannelBackend {

    private static final Logger LOG = Logger.getLogger(A2AChannelBackend.class);

    private final Set<UUID> registeredChannels = ConcurrentHashMap.newKeySet();

    @Inject
    QhorusMcpTools tools;

    @Inject
    ChannelGateway gateway;

    @Inject
    A2AActorResolver actorResolver;

    @Override
    public String backendId() { return "a2a"; }

    @Override
    public ActorType actorType() { return ActorType.AGENT; }

    @Override
    public void open(ChannelRef channel, Map<String, String> metadata) { /* no-op */ }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        if (message.correlationId() != null) {
            LOG.debugf("A2A backend notified: channel=%s correlationId=%s type=%s",
                    channel.name(), message.correlationId(), message.type());
        }
    }

    @Override
    public void close(ChannelRef channel) {
        registeredChannels.remove(channel.id());
    }

    /**
     * Registers this backend on the channel if not already registered.
     * Thread-safe — uses ConcurrentHashMap.add() semantics; only one caller wins.
     */
    public void ensureRegistered(UUID channelId, ChannelRef ref) {
        if (registeredChannels.add(channelId)) {
            gateway.registerBackend(channelId, this, "agent");
            open(ref, Map.of());
        }
    }

    /**
     * Processes an inbound A2A message: resolves actor type, builds sender, routes via tools.
     *
     * @param channelName    Qhorus channel name (from A2A contextId)
     * @param role           A2A role string ("user", "agent", or custom)
     * @param textContent    extracted text from A2A parts
     * @param taskId         A2A taskId used as correlationId; auto-generated if null
     * @param metadata       A2A message metadata map (may contain agentId, agentCardUrl)
     * @param actorTypeHeader value of x-qhorus-actor-type HTTP header (may be null)
     * @return the correlationId used for this message
     */
    public String receive(String channelName, String role, String textContent,
            String taskId, Map<String, String> metadata, String actorTypeHeader) {
        ActorType resolved = actorResolver.resolve(role, actorTypeHeader, metadata);
        String agentId = metadata.get("agentId");
        String sender = buildSender(resolved, agentId, role);
        String type = "agent".equals(role) ? "response" : "query";
        String correlationId = (taskId != null && !taskId.isBlank())
                ? taskId : UUID.randomUUID().toString();

        tools.sendMessage(channelName, sender, type, textContent,
                correlationId, null, null, null, null);
        return correlationId;
    }

    private String buildSender(ActorType resolved, String agentId, String role) {
        return switch (resolved) {
            case AGENT -> (agentId != null && ActorTypeResolver.resolve(agentId) == ActorType.AGENT)
                    ? agentId : "agent";
            case HUMAN -> "human:" + (agentId != null ? agentId : role);
            case SYSTEM -> agentId != null ? agentId : "system";
        };
    }
}
```

- [ ] **Step 4: Run the integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=A2AChannelBackendIntegrationTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: 5 tests, 0 failures.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java \
  runtime/src/test/java/io/casehub/qhorus/api/A2AChannelBackendIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(a2a): A2AChannelBackend — protocol bridge gateway backend

Registered as ChannelBackend "a2a" via ensureRegistered(). Inbound routing:
resolves ActorType via A2AActorResolver, constructs structured sender, routes
through tools.sendMessage() for full pipeline. post() is the fanOut hook for
future SSE streaming (#147).

Refs #135

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: A2AResource refactor + new/updated tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/api/A2ASendMessageTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/api/A2AGetTaskTest.java`

- [ ] **Step 1: Write new failing tests in A2ASendMessageTest**

Add these tests to `A2ASendMessageTest` (existing tests should still pass; these new ones will fail until the refactor):

```java
@Test
void role_agent_messageTypeIsResponse() {
    tools.createChannel("a2a-type-agent-1", "Test", "APPEND", null, null, null, null, null, null);

    given().urlEncodingEnabled(false)
            .contentType("application/json")
            .body(sendBody("a2a-type-agent-1", "agent", "work done", null))
            .when().post(SEND_PATH)
            .then().statusCode(200);

    QhorusMcpTools.CheckResult check = tools.checkMessages("a2a-type-agent-1", 0L, 10, null, null, null);
    assertEquals("RESPONSE", check.messages().get(0).messageType(),
            "role:agent should produce RESPONSE type");
    assertEquals("agent", check.messages().get(0).sender());
}

@Test
void role_user_noSignals_messageTypeIsQuery_senderIsHumanUser() {
    tools.createChannel("a2a-type-user-1", "Test", "APPEND", null, null, null, null, null, null);

    given().urlEncodingEnabled(false)
            .contentType("application/json")
            .body(sendBody("a2a-type-user-1", "user", "hello", null))
            .when().post(SEND_PATH)
            .then().statusCode(200);

    QhorusMcpTools.CheckResult check = tools.checkMessages("a2a-type-user-1", 0L, 10, null, null, null);
    assertEquals("QUERY", check.messages().get(0).messageType());
    assertEquals("human:user", check.messages().get(0).sender());
}

@Test
void role_user_headerAgent_senderIsAgent() {
    tools.createChannel("a2a-header-agent-1", "Test", "APPEND", null, null, null, null, null, null);

    given().urlEncodingEnabled(false)
            .contentType("application/json")
            .header("x-qhorus-actor-type", "AGENT")
            .body(sendBody("a2a-header-agent-1", "user", "delegate this", null))
            .when().post(SEND_PATH)
            .then().statusCode(200);

    QhorusMcpTools.CheckResult check = tools.checkMessages("a2a-header-agent-1", 0L, 10, null, null, null);
    assertEquals("agent", check.messages().get(0).sender(),
            "AGENT header with no agentId should produce generic 'agent' sender");
}
```

Also update the `sendMessageTypeIsRequest` test — it currently asserts QUERY for role:"user" which should still pass. The `senderIsSetFromRoleField` test asserts sender="orchestrator" for role:"orchestrator" — after the refactor, unknown roles fall to HUMAN, so sender becomes "human:orchestrator". Update that assertion:

```java
// Was: assertEquals("orchestrator", ...)
assertEquals("human:orchestrator", check.messages().get(0).sender(),
        "unknown role should produce human-prefixed sender");
```

- [ ] **Step 2: Write new failing test in A2AGetTaskTest**

Read the current `A2AGetTaskTest` first, then add:

```java
@Test
void getTask_afterDone_returnsCompleted() {
    tools.createChannel("a2a-task-state-1", "Test", "APPEND", null, null, null, null, null, null);
    String taskId = UUID.randomUUID().toString();

    // Send a QUERY (creates commitment OPEN)
    given().urlEncodingEnabled(false)
            .contentType("application/json")
            .body(sendBody("a2a-task-state-1", "user", "please do this", taskId))
            .when().post("/a2a/message:send")
            .then().statusCode(200);

    // Resolve the commitment by sending DONE on the correlationId
    tools.sendMessage("a2a-task-state-1", "agent", "done",
            "all finished", taskId, null, null, null, null);

    // getTask should reflect completed state
    given()
            .when().get("/a2a/tasks/" + taskId)
            .then()
            .statusCode(200)
            .body("status.state", equalTo("completed"));
}
```

- [ ] **Step 3: Run to confirm new tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="A2ASendMessageTest,A2AGetTaskTest" -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: Several new tests fail; existing tests pass.

- [ ] **Step 4: Refactor A2AResource.java**

Replace the full class. Key changes:
- Add `Map<String, String> metadata` field to `A2AMessage` record
- `sendMessage()` delegates to `a2aBackend`
- `getTask()` checks CommitmentService first
- Remove `QhorusMcpTools` injection; add `A2AChannelBackend` and `CommitmentService`

```java
package io.casehub.qhorus.runtime.api;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.Context;
import jakarta.ws.rs.core.HttpHeaders;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.CommitmentService;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.message.MessageType;
import io.quarkus.arc.properties.UnlessBuildProperty;

@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)
@Path("/a2a")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class A2AResource {

    private static final Response A2A_DISABLED = Response
            .status(Response.Status.NOT_IMPLEMENTED)
            .entity("{\"error\":\"A2A endpoint is disabled. Set casehub.qhorus.a2a.enabled=true to activate.\"}")
            .type(MediaType.APPLICATION_JSON)
            .build();

    @Inject
    QhorusConfig config;

    @Inject
    A2AChannelBackend a2aBackend;

    @Inject
    ChannelService channelService;

    @Inject
    MessageService messageService;

    @Inject
    CommitmentService commitmentService;

    @POST
    @Path("/message:send")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response sendMessage(SendMessageRequest request, @Context HttpHeaders headers) {
        if (!config.a2a().enabled()) {
            return A2A_DISABLED;
        }

        if (request == null || request.message() == null) {
            return error400("message is required");
        }
        A2AMessage msg = request.message();

        if (msg.contextId() == null || msg.contextId().isBlank()) {
            return error400("message.contextId (channel name) is required");
        }
        if (msg.parts() == null || msg.parts().isEmpty()) {
            return error400("message.parts must contain at least one text part");
        }
        String text = msg.parts().stream()
                .filter(p -> "text".equals(p.kind()) && p.text() != null)
                .map(A2APart::text)
                .findFirst()
                .orElse(null);
        if (text == null) {
            return error400("message.parts must contain at least one text part with kind=text");
        }

        Channel channel = channelService.findByName(msg.contextId()).orElse(null);
        if (channel == null) {
            return error400("Channel not found: " + msg.contextId());
        }

        ChannelRef ref = new ChannelRef(channel.id, channel.name);
        a2aBackend.ensureRegistered(channel.id, ref);

        String actorTypeHeader = headers.getHeaderString("x-qhorus-actor-type");
        Map<String, String> metadata = msg.metadata() != null ? msg.metadata() : Map.of();

        String correlationId;
        try {
            correlationId = a2aBackend.receive(
                    channel.name, msg.role(), text, msg.taskId(), metadata, actorTypeHeader);
        } catch (Exception e) {
            String cause = e.getCause() != null ? e.getCause().getMessage() : e.getMessage();
            return error400(cause);
        }

        Task task = new Task(correlationId, msg.contextId(), new TaskStatus("submitted"), null);
        return Response.ok(new SendMessageResponse(task)).build();
    }

    @GET
    @Path("/tasks/{id}")
    @Transactional
    public Response getTask(@PathParam("id") String taskId) {
        if (!config.a2a().enabled()) {
            return A2A_DISABLED;
        }

        // Prefer CommitmentStore — durable, survives restarts, semantically accurate
        Commitment commitment = commitmentService.findByCorrelationId(taskId).orElse(null);
        if (commitment != null) {
            List<Message> messages = messageService.findAllByCorrelationId(taskId);
            if (messages.isEmpty()) {
                return notFound(taskId);
            }
            Channel channel = channelService.findById(messages.get(0).channelId)
                    .orElseThrow(() -> new IllegalStateException("Channel not found for task " + taskId));
            return Response.ok(new Task(taskId, channel.name,
                    new TaskStatus(toA2AState(commitment.state)), null)).build();
        }

        // Fallback: derive from message history (handles EVENT-only channels, no commitment)
        List<Message> messages = messageService.findAllByCorrelationId(taskId);
        if (messages.isEmpty()) {
            return notFound(taskId);
        }
        Channel channel = channelService.findById(messages.get(0).channelId)
                .orElseThrow(() -> new IllegalStateException("Channel not found for task " + taskId));
        return Response.ok(new Task(taskId, channel.name,
                new TaskStatus(deriveState(messages)), null)).build();
    }

    private String toA2AState(CommitmentState state) {
        return switch (state) {
            case FULFILLED, DELEGATED -> "completed";
            case FAILED, DECLINED, EXPIRED -> "failed";
            case ACKNOWLEDGED -> "working";
            case OPEN -> "submitted";
        };
    }

    private static String deriveState(List<Message> messages) {
        MessageType lastType = null;
        for (Message m : messages) {
            lastType = m.messageType;
        }
        if (lastType == null) return "submitted";
        return switch (lastType) {
            case RESPONSE, DONE -> "completed";
            case FAILURE, DECLINE -> "failed";
            case STATUS -> "working";
            default -> "submitted";
        };
    }

    private static Response notFound(String taskId) {
        return Response.status(Response.Status.NOT_FOUND)
                .entity("{\"error\":\"Task not found: " + taskId + "\"}")
                .type(MediaType.APPLICATION_JSON)
                .build();
    }

    private static Response error400(String message) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity("{\"error\":\"" + message + "\"}")
                .type(MediaType.APPLICATION_JSON)
                .build();
    }

    // ── A2A data model ─────────────────────────────────────────────────────────

    public record SendMessageRequest(String id, A2AMessage message) {}

    public record A2AMessage(
            String role,
            List<A2APart> parts,
            String messageId,
            String taskId,
            String contextId,
            Map<String, String> metadata) {}

    public record A2APart(String kind, String text) {}

    public record Task(String id, String contextId, TaskStatus status,
            List<A2AMessage> history) {}

    public record TaskStatus(String state) {}

    public record SendMessageResponse(Task task) {}
}
```

- [ ] **Step 5: Update ReactiveA2AResource.java**

Make these targeted changes to `ReactiveA2AResource.java`:

**Remove** the `@Inject QhorusMcpTools tools;` field. **Add:**
```java
@Inject
A2AChannelBackend a2aBackend;

@Inject
CommitmentService commitmentService;
```

**Update the `A2AMessage` record** (same inner record as in A2AResource) to add `Map<String, String> metadata` as the last field. Since ReactiveA2AResource defines its own copy of these records, add the same field there.

**Update `sendMessage()`** — replace the `tools.sendMessage(...)` call with:
```java
return Uni.createFrom().item(() -> {
    Channel channel = channelService.findByName(msg.contextId()).orElse(null);
    if (channel == null) return error400("Channel not found: " + msg.contextId());
    ChannelRef ref = new ChannelRef(channel.id, channel.name);
    a2aBackend.ensureRegistered(channel.id, ref);
    String actorTypeHeader = headers.getHeaderString("x-qhorus-actor-type");
    Map<String, String> metadata = msg.metadata() != null ? msg.metadata() : Map.of();
    try {
        String correlationId = a2aBackend.receive(
                channel.name, msg.role(), text, msg.taskId(), metadata, actorTypeHeader);
        return Response.ok(new SendMessageResponse(
                new Task(correlationId, msg.contextId(), new TaskStatus("submitted"), null))).build();
    } catch (Exception e) {
        String cause = e.getCause() != null ? e.getCause().getMessage() : e.getMessage();
        return error400(cause);
    }
});
```

**Update `getTask()`** — add CommitmentStore check before the existing message-history fallback, wrapping in `Uni.createFrom().item()` for the blocking CommitmentService call:
```java
Commitment commitment = commitmentService.findByCorrelationId(taskId).orElse(null);
if (commitment != null && !messages.isEmpty()) {
    return Response.ok(new Task(taskId, channel.name,
            new TaskStatus(toA2AState(commitment.state)), null)).build();
}
```
Add the `toA2AState()` private method identical to the one in `A2AResource`.

- [ ] **Step 6: Run all A2A tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="A2ASendMessageTest,A2AGetTaskTest,A2AResourceDisabledTest,A2AChannelBackendIntegrationTest" -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: all pass.

- [ ] **Step 7: Run the full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: BUILD SUCCESS, ≥1011 tests passing, 0 failures.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java \
  runtime/src/test/java/io/casehub/qhorus/api/A2ASendMessageTest.java \
  runtime/src/test/java/io/casehub/qhorus/api/A2AGetTaskTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(a2a): A2AResource refactored — thin adapter with CommitmentStore-based getTask

sendMessage() delegates to A2AChannelBackend. A2AMessage gains metadata field
for A2A-native identity signals (agentId, agentCardUrl). getTask() queries
CommitmentStore for durable task state; falls back to message-history deriveState
for channels where no commitment was created.

Closes #135

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Platform convention doc update

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/protocols/qhorus-actor-type-mapping.md`

- [ ] **Step 1: Update the protocol doc**

Open `/Users/mdproctor/claude/casehub/parent/docs/protocols/qhorus-actor-type-mapping.md`. In the A2A Protocol Role Mapping table, the Notes column references `(casehubio/ledger#75)` as pending. Update to reflect that both issues are now implemented:

Change the notes in the table from:
```
| `"user"` | `HUMAN` | Explicit rule in `ActorTypeResolver` (casehubio/ledger#75) |
| `"agent"` | `AGENT` | Explicit rule in `ActorTypeResolver` (casehubio/ledger#75) |
```
To:
```
| `"user"` | `HUMAN` | Explicit rule in `ActorTypeResolver` — implemented in casehubio/ledger#75 |
| `"agent"` | `AGENT` | Explicit rule in `ActorTypeResolver` — implemented in casehubio/ledger#75 |
```

Also verify the A2A interop contract section accurately describes the 6-step resolution chain now implemented in `A2AActorResolver`. If the section lists the steps, verify they match:
1. `x-qhorus-actor-type` header
2. Instance registry (via `metadata.agentId`)
3. Agent Card URL (via `metadata.agentCardUrl`)
4. Persona format check on `metadata.agentId`
5. System check on `metadata.agentId`
6. Default HUMAN

Update any gaps or inaccuracies. Add a cross-reference to casehubio/qhorus#135.

- [ ] **Step 2: Commit in the parent repo**

```bash
git -C /Users/mdproctor/claude/casehub/parent add \
  docs/protocols/qhorus-actor-type-mapping.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "$(cat <<'EOF'
docs: sync qhorus-actor-type-mapping — ledger#75 shipped, qhorus#135 A2AActorResolver

Removes '(pending)' markers, adds 6-step resolution chain details,
cross-references casehubio/qhorus#135.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Final verification

- [ ] **Step 1: Full clean build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml
```

Expected: BUILD SUCCESS, all tests pass, 0 failures.

- [ ] **Step 2: Confirm issue is closed**

```bash
gh issue view 135 --repo casehubio/qhorus --json state -q '.state'
```

Expected: `CLOSED` (the `Closes #135` commit message should have closed it when pushed, or verify manually).

# Deadline Dispatch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `deadline` through both blocking and reactive message dispatch paths so `CommitmentService.open()` receives a real value for the first time, enabling Layer 3 temporal accountability.

**Architecture:** Add `Instant deadline` to `MessageDispatch` (fixing the blocking path in one line); replace `ReactiveMessageService.send()` with `dispatch(MessageDispatch)` returning `Uni<DispatchResult>` (fixing the reactive path and unifying the API); update the three callers mechanically.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny, Hibernate ORM Panache, AssertJ, Mockito

Build command (run from project root unless told otherwise):
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Run a single test class (in `runtime/` module):
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl runtime
```

---

## File Map

**qhorus repo** (`/Users/mdproctor/claude/casehub/qhorus`):

| File | Change |
|------|--------|
| `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java` | Add `Instant deadline` field + builder method |
| `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` | One line: `message.deadline = dispatch.deadline()` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` | Replace `send(9 params)` with `dispatch(MessageDispatch)` returning `Uni<DispatchResult>` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java` | Remove paused check; build `MessageDispatch`; map `DispatchResult → HumanMessageResult` |
| `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchIntegrationTest.java` | Add `dispatch_command_with_deadline_persists_deadline()` |
| `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java` | Update `send()` override to use `svc.dispatch(MessageDispatch)` |
| `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java` | Update `sendHumanMessage_*` tests to mock `dispatch(any())` not `send(...)` |

**claudony repo** (`/Users/mdproctor/claude/casehub/claudony`):

| File | Change |
|------|--------|
| `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java` | Build `MessageDispatch` in `postToChannel()`; call `messageService.dispatch()` |
| `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java` | Update mocks from `send(9 args)` to `dispatch(any(MessageDispatch.class))` |

---

## Task 1: Add `deadline` to `MessageDispatch` + test for blocking path

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java`
- Modify (test): `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchIntegrationTest.java`
- Modify (impl): `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`

- [ ] **Step 1.1 — Write the failing test in `MessageDispatchIntegrationTest`**

Add this test to `MessageDispatchIntegrationTest` (after the existing `dispatch_command_returns_DispatchResult_with_messageId` test). Add `import java.time.Duration;` at the top.

```java
@Test @TestTransaction
void dispatch_command_with_deadline_persists_deadline() {
    UUID channelId = createChannel("deadline-test-" + UUID.randomUUID());
    Instant deadline = Instant.now().plus(Duration.ofHours(1)).truncatedTo(java.time.temporal.ChronoUnit.MILLIS);

    DispatchResult result = messageService.dispatch(MessageDispatch.builder()
            .channelId(channelId).sender("orchestrator").type(MessageType.COMMAND)
            .content("do task").correlationId("corr-deadline-" + UUID.randomUUID())
            .deadline(deadline)
            .actorType(ActorType.SYSTEM).build());

    assertThat(messageService.findById(result.messageId()))
            .isPresent()
            .hasValueSatisfying(m -> assertThat(m.deadline).isEqualTo(deadline));
}
```

- [ ] **Step 1.2 — Verify the test fails to compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageDispatchIntegrationTest -pl runtime 2>&1 | grep -E "ERROR|cannot find|deadline"
```

Expected: compilation error — `deadline(Instant)` method not found on builder.

- [ ] **Step 1.3 — Add `deadline` to `MessageDispatch`**

Replace the full content of `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java`:

```java
package io.casehub.qhorus.api.message;

import java.time.Instant;
import java.util.UUID;
import io.casehub.platform.api.identity.ActorType;

public record MessageDispatch(
        UUID channelId,
        String sender,
        MessageType type,
        String content,
        String correlationId,
        Long inReplyTo,
        String artefactRefs,  // comma-separated UUID strings, matches Message entity storage; nullable
        String target,
        UUID subjectId,
        UUID causedByEntryId,
        ActorType actorType,
        Instant deadline) {

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private UUID channelId;
        private String sender;
        private MessageType type;
        private String content;
        private String correlationId;
        private Long inReplyTo;
        private String artefactRefs;
        private String target;
        private UUID subjectId;
        private UUID causedByEntryId;
        private ActorType actorType;
        private Instant deadline;

        public Builder channelId(UUID v)       { this.channelId = v;       return this; }
        public Builder sender(String v)         { this.sender = v;           return this; }
        public Builder type(MessageType v)      { this.type = v;             return this; }
        public Builder content(String v)        { this.content = v;          return this; }
        public Builder correlationId(String v)  { this.correlationId = v;    return this; }
        public Builder inReplyTo(Long v)        { this.inReplyTo = v;        return this; }
        public Builder artefactRefs(String v)   { this.artefactRefs = v;     return this; }
        public Builder target(String v)         { this.target = v;           return this; }
        public Builder subjectId(UUID v)        { this.subjectId = v;        return this; }
        public Builder causedByEntryId(UUID v)  { this.causedByEntryId = v;  return this; }
        public Builder actorType(ActorType v)   { this.actorType = v;        return this; }
        public Builder deadline(Instant v)      { this.deadline = v;         return this; }

        /**
         * Validates and builds the dispatch. Enforcement matrix:
         * <ul>
         *   <li>DONE, DECLINE, FAILURE — require both {@code inReplyTo} AND {@code correlationId};
         *       they resolve a COMMAND commitment via the correlationId thread</li>
         *   <li>RESPONSE — requires both {@code inReplyTo} AND {@code correlationId};
         *       fulfills a QUERY commitment. The correlationId requirement is identical to
         *       DONE/DECLINE/FAILURE — both are needed for commitment resolution and causal chain
         *       integrity, regardless of whether the original message was a COMMAND or QUERY.</li>
         *   <li>HANDOFF — requires {@code inReplyTo}, {@code correlationId}, AND {@code target}</li>
         *   <li>COMMAND, QUERY, EVENT, STATUS — no required reply fields</li>
         * </ul>
         */
        public MessageDispatch build() {
            if (channelId == null) throw new IllegalArgumentException("channelId is required");
            if (sender == null || sender.isBlank()) throw new IllegalArgumentException("sender is required");
            if (type == null) throw new IllegalArgumentException("type is required");
            if (actorType == null) throw new IllegalArgumentException("actorType is required");

            switch (type) {
                case DONE, DECLINE, FAILURE -> {
                    if (inReplyTo == null)
                        throw new IllegalArgumentException(type.name() + " requires inReplyTo");
                    if (correlationId == null)
                        throw new IllegalArgumentException(
                            type.name() + " requires correlationId for commitment resolution");
                }
                case RESPONSE -> {
                    if (inReplyTo == null)
                        throw new IllegalArgumentException("RESPONSE requires inReplyTo");
                    if (correlationId == null)
                        throw new IllegalArgumentException(
                            "RESPONSE requires correlationId for commitment resolution");
                }
                case HANDOFF -> {
                    if (inReplyTo == null)
                        throw new IllegalArgumentException("HANDOFF requires inReplyTo");
                    if (correlationId == null)
                        throw new IllegalArgumentException("HANDOFF requires correlationId");
                    if (target == null || target.isBlank())
                        throw new IllegalArgumentException("HANDOFF requires target");
                }
                default -> { /* COMMAND, QUERY, EVENT, STATUS — no required reply fields */ }
            }
            return new MessageDispatch(channelId, sender, type, content, correlationId,
                    inReplyTo, artefactRefs, target, subjectId, causedByEntryId, actorType, deadline);
        }
    }
}
```

- [ ] **Step 1.4 — Set `message.deadline` in `MessageService.dispatch()`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`, in the "Normal insert" block, add one line immediately before `messageStore.put(message)`. The block currently looks like:

```java
final Message message = new Message();
message.channelId = dispatch.channelId();
message.sender = dispatch.sender();
message.messageType = dispatch.type();
message.actorType = dispatch.actorType();
message.content = dispatch.content();
message.correlationId = dispatch.correlationId();
message.inReplyTo = dispatch.inReplyTo();
message.artefactRefs = dispatch.artefactRefs();
message.target = dispatch.target();
message.commitmentId = commitmentId;
messageStore.put(message);
```

Change it to:

```java
final Message message = new Message();
message.channelId = dispatch.channelId();
message.sender = dispatch.sender();
message.messageType = dispatch.type();
message.actorType = dispatch.actorType();
message.content = dispatch.content();
message.correlationId = dispatch.correlationId();
message.inReplyTo = dispatch.inReplyTo();
message.artefactRefs = dispatch.artefactRefs();
message.target = dispatch.target();
message.deadline = dispatch.deadline();
message.commitmentId = commitmentId;
messageStore.put(message);
```

- [ ] **Step 1.5 — Run the new test and verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageDispatchIntegrationTest -pl runtime 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 1.6 — Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#192): add deadline to MessageDispatch and wire through blocking dispatch

Refs #192
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Replace `ReactiveMessageService.send()` with `dispatch(MessageDispatch)`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`
- Modify (test): `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java`

- [ ] **Step 2.1 — Update `ReactiveMessageServiceTest.send()` to call `dispatch()` (compilation failure)**

Replace the full content of `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java`:

```java
package io.casehub.qhorus.service;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Disabled;

import io.casehub.platform.api.identity.ActorTypeResolver;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.ReactiveMessageService;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

@Disabled("ReactiveMessageService calls Panache.withTransaction() — requires reactive datasource.")
@QuarkusTest
@TestProfile(ReactiveTestProfile.class)
class ReactiveMessageServiceTest extends MessageServiceContractTest {

    @Inject
    ReactiveMessageService svc;

    @Override
    protected DispatchResult send(UUID channelId, String sender, MessageType type,
            String content, String correlationId, Long inReplyTo) {
        return svc.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(sender)
                .type(type)
                .content(content)
                .correlationId(correlationId)
                .inReplyTo(inReplyTo)
                .actorType(ActorTypeResolver.resolve(sender))
                .build()).await().indefinitely();
    }

    @Override
    protected Optional<Message> findById(Long id) {
        return svc.findById(id).await().indefinitely();
    }

    @Override
    protected List<Message> pollAfter(UUID channelId, Long afterId, int limit) {
        return svc.pollAfter(channelId, afterId, limit).await().indefinitely();
    }
}
```

- [ ] **Step 2.2 — Verify the test file fails to compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | grep -E "ERROR|cannot find|dispatch"
```

Expected: compilation error — `dispatch(MessageDispatch)` not found on `ReactiveMessageService`.

- [ ] **Step 2.3 — Implement `ReactiveMessageService.dispatch(MessageDispatch)`**

Replace the full content of `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`:

```java
package io.casehub.qhorus.runtime.message;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import jakarta.enterprise.inject.Instance;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.store.ReactiveChannelStore;
import io.casehub.qhorus.runtime.store.ReactiveMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;

@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class ReactiveMessageService {

    @Inject
    ReactiveMessageStore messageStore;

    @Inject
    ReactiveChannelStore channelStore;

    @Inject
    CommitmentService commitmentService;

    @Inject
    Instance<MessageObserver> observers;

    /**
     * Dispatches a message to a channel via the reactive path.
     *
     * <p>Applies the paused check before persisting. Full enforcement parity
     * (ACL, rate limit, type policy, LAST_WRITE, ledger write, fanOut) is
     * deferred to issue #193.
     *
     * <p>Returns {@code null} ledger fields ({@code ledgerEntryId},
     * {@code subjectId}, {@code causedByEntryId}) until #193 adds ledger writes
     * to the reactive path.
     */
    public Uni<DispatchResult> dispatch(final MessageDispatch dispatch) {
        return Panache.withTransaction("qhorus", () -> {
            final int[] replyCountHolder = { 0 };

            return channelStore.find(dispatch.channelId())
                    .flatMap(chOpt -> {
                        final String channelName = chOpt.map(ch -> ch.name).orElse(null);

                        // Paused check — moved here from caller (QhorusDashboardService.sendHumanMessage).
                        // Full enforcement (ACL, rate limit, type policy) deferred to #193.
                        chOpt.ifPresent(ch -> {
                            if (ch.paused) {
                                throw new IllegalStateException(
                                        "Channel '" + ch.name
                                                + "' is paused — send_message blocked. Use resume_channel to re-enable.");
                            }
                        });

                        final Message message = new Message();
                        message.channelId = dispatch.channelId();
                        message.sender = dispatch.sender();
                        message.messageType = dispatch.type();
                        message.actorType = dispatch.actorType();
                        message.content = dispatch.content();
                        message.correlationId = dispatch.correlationId();
                        message.inReplyTo = dispatch.inReplyTo();
                        message.artefactRefs = dispatch.artefactRefs();
                        message.target = dispatch.target();
                        message.deadline = dispatch.deadline();
                        message.commitmentId = (dispatch.correlationId() != null &&
                                (dispatch.type() == MessageType.COMMAND
                                        || dispatch.type() == MessageType.QUERY))
                                ? UUID.randomUUID() : null;

                        return messageStore.put(message)
                                .invoke(m -> MessageObserverDispatcher.dispatch(
                                        channelName, dispatch.channelId(), m, observers.handles()))
                                .invoke(m -> {
                                    if (m.correlationId != null) {
                                        switch (m.messageType) {
                                            case QUERY, COMMAND -> commitmentService.open(
                                                    m.commitmentId,
                                                    m.correlationId, m.channelId, m.messageType,
                                                    m.sender, m.target, m.deadline);
                                            case STATUS -> commitmentService.acknowledge(m.correlationId);
                                            case RESPONSE, DONE -> commitmentService.fulfill(m.correlationId);
                                            case DECLINE -> commitmentService.decline(m.correlationId);
                                            case FAILURE -> commitmentService.fail(m.correlationId);
                                            case HANDOFF -> commitmentService.delegate(
                                                    m.correlationId, m.target);
                                            case EVENT -> { /* no commitment effect */ }
                                        }
                                    }
                                })
                                .flatMap(m -> dispatch.inReplyTo() != null
                                        ? messageStore.find(dispatch.inReplyTo())
                                                .invoke(opt -> opt.ifPresent(parent -> {
                                                    parent.replyCount++;
                                                    replyCountHolder[0] = parent.replyCount;
                                                }))
                                                .map(ignored -> m)
                                        : Uni.createFrom().item(m))
                                .invoke(m -> chOpt.ifPresent(ch -> ch.lastActivityAt = Instant.now()))
                                .map(m -> new DispatchResult(
                                        m.id,
                                        dispatch.channelId(),
                                        dispatch.sender(),
                                        dispatch.type(),
                                        dispatch.correlationId(),
                                        dispatch.inReplyTo(),
                                        ArtefactRefParser.parse(dispatch.artefactRefs()),
                                        dispatch.target(),
                                        null,  // ledgerEntryId — deferred to #193
                                        null,  // subjectId — deferred to #193
                                        null,  // causedByEntryId — deferred to #193
                                        replyCountHolder[0]));
                    });
        });
    }

    public Uni<Optional<Message>> findById(Long id) {
        return messageStore.find(id);
    }

    /**
     * Returns messages in channel posted after {@code afterId}, excluding EVENT type.
     */
    public Uni<List<Message>> pollAfter(UUID channelId, Long afterId, int limit) {
        return messageStore.scan(
                MessageQuery.builder()
                        .channelId(channelId)
                        .afterId(afterId)
                        .limit(limit)
                        .excludeTypes(List.of(MessageType.EVENT))
                        .build());
    }

    /**
     * Like {@link #pollAfter} but filters by sender in the query.
     */
    public Uni<List<Message>> pollAfterBySender(UUID channelId, Long afterId, int limit, String sender) {
        return messageStore.scan(
                MessageQuery.builder()
                        .channelId(channelId)
                        .afterId(afterId)
                        .limit(limit)
                        .excludeTypes(List.of(MessageType.EVENT))
                        .sender(sender)
                        .build());
    }
}
```

- [ ] **Step 2.4 — Verify compilation succeeds and build passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`. (`ReactiveMessageServiceTest` is `@Disabled` — it compiles but does not run.)

- [ ] **Step 2.5 — Run the full runtime test suite to check for regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 2.6 — Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#192): replace ReactiveMessageService.send() with dispatch(MessageDispatch)

Unifies reactive API with blocking service. Adds paused check. Returns
Uni<DispatchResult>. Full enforcement parity (ACL, rate limit, ledger)
deferred to #193.

Refs #192
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Update `QhorusDashboardService.sendHumanMessage()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`
- Modify (test): `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java`

- [ ] **Step 3.1 — Update the dashboard service tests first**

The tests mock `messageService.send(...)`. After the change, they must mock `messageService.dispatch(any(MessageDispatch.class))`.

In `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java`, add these imports if not present:
```java
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import static org.mockito.ArgumentMatchers.any;
```

Replace the three `sendHumanMessage_*` test methods (lines 201–252 approximately) with:

```java
// ── sendHumanMessage ──────────────────────────────────────────────────────

@Test
void sendHumanMessage_unknownChannel_throwsIllegalArgumentException() {
    when(channelService.findByName("ghost")).thenReturn(Uni.createFrom().item(Optional.empty()));

    Exception ex = assertThrows(Exception.class, () ->
            service.sendHumanMessage("ghost", "human:alice", MessageType.STATUS, "hello")
                    .await().atMost(Duration.ofSeconds(1)));
    assertTrue(ex instanceof IllegalArgumentException
            || (ex.getCause() instanceof IllegalArgumentException),
            "Expected IllegalArgumentException, got: " + ex);
    String msg = ex instanceof IllegalArgumentException ? ex.getMessage() : ex.getCause().getMessage();
    assertTrue(msg.contains("ghost"), "Message should mention channel name: " + msg);
}

@Test
void sendHumanMessage_pausedChannel_throwsIllegalStateException() {
    Channel ch = channel("oversight", ChannelSemantic.APPEND);
    ch.paused = true;
    when(channelService.findByName("oversight")).thenReturn(Uni.createFrom().item(Optional.of(ch)));
    // Paused check now lives inside ReactiveMessageService.dispatch() — mock the reactive service
    // to throw as it would in production.
    when(messageService.dispatch(any(MessageDispatch.class)))
            .thenReturn(Uni.createFrom().failure(
                    new IllegalStateException("Channel 'oversight' is paused")));

    Exception ex = assertThrows(Exception.class, () ->
            service.sendHumanMessage("oversight", "human:alice", MessageType.STATUS, "hello")
                    .await().atMost(Duration.ofSeconds(1)));
    assertTrue(ex instanceof IllegalStateException
            || (ex.getCause() instanceof IllegalStateException),
            "Expected IllegalStateException, got: " + ex);
    String msg = ex instanceof IllegalStateException ? ex.getMessage() : ex.getCause().getMessage();
    assertTrue(msg.contains("paused"), "Message should mention paused: " + msg);
}

@Test
void sendHumanMessage_success_returnsHumanMessageResultWithCorrectFields() {
    Channel ch = channel("work", ChannelSemantic.APPEND);
    DispatchResult dr = new DispatchResult(42L, ch.id, "human:alice", MessageType.STATUS,
            null, null, List.of(), null, null, null, null, 0);
    when(channelService.findByName("work")).thenReturn(Uni.createFrom().item(Optional.of(ch)));
    when(messageService.dispatch(any(MessageDispatch.class)))
            .thenReturn(Uni.createFrom().item(dr));

    QhorusDashboardService.HumanMessageResult result =
            service.sendHumanMessage("work", "human:alice", MessageType.STATUS, "please prioritise security")
                    .await().atMost(Duration.ofSeconds(1));

    assertEquals(42L, result.messageId());
    assertEquals("work", result.channelName());
    assertEquals("human:alice", result.sender());
    assertEquals("STATUS", result.messageType());
}
```

- [ ] **Step 3.2 — Verify the updated tests fail (mock mismatch)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusDashboardServiceTest -pl runtime 2>&1 | tail -15
```

Expected: `sendHumanMessage_success` and `sendHumanMessage_pausedChannel` fail — `messageService.dispatch()` is stubbed but `sendHumanMessage()` still calls `messageService.send()`.

- [ ] **Step 3.3 — Update `QhorusDashboardService.sendHumanMessage()`**

In `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`, add these imports at the top if not present:
```java
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import java.util.UUID;
```

Replace the `sendHumanMessage()` method (roughly lines 115–133):

```java
/**
 * Sends a human operator message to a named channel.
 *
 * <p>Paused check is enforced by {@link ReactiveMessageService#dispatch}.
 */
public Uni<HumanMessageResult> sendHumanMessage(
        String channelName, String sender, MessageType type, String content) {
    return channelService.findByName(channelName)
            .map(opt -> opt.orElseThrow(
                    () -> new IllegalArgumentException("Channel not found: " + channelName)))
            .flatMap(ch -> messageService.dispatch(
                    MessageDispatch.builder()
                            .channelId(ch.id)
                            .sender(sender)
                            .type(type)
                            .content(content)
                            .actorType(ActorType.HUMAN)
                            .build()))
            .map(result -> new HumanMessageResult(
                    result.messageId(), channelName, result.sender(),
                    result.type() != null ? result.type().name() : null,
                    result.correlationId(), result.inReplyTo(),
                    result.parentReplyCount(),
                    result.artefactRefs().stream().map(UUID::toString).toList(),
                    result.target()));
}
```

- [ ] **Step 3.4 — Run the dashboard tests and verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusDashboardServiceTest -pl runtime 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3.5 — Run full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3.6 — Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "refactor(#192): sendHumanMessage uses MessageDispatch; paused check moves to dispatch()

Refs #192
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: Update Claudony — `ClaudonyReactiveCaseChannelProvider`

**Files (claudony repo at `/Users/mdproctor/claude/casehub/claudony`):**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`
- Modify (test): `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java`

- [ ] **Step 4.1 — Update the Claudony tests to mock `dispatch()` instead of `send()`**

In `casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java`:

Add import at the top:
```java
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.DispatchResult;
import java.util.List;
```

Replace the `postToChannel_*` test methods (lines 172–269 approximately) with:

```java
// ── postToChannel ─────────────────────────────────────────────────────────

@Test
void postToChannel_sendsViaMessageService() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    DispatchResult dr = new DispatchResult(1L, channelId, "alice", MessageType.STATUS,
            null, null, List.of(), null, null, null, null, 0);

    when(messageService.dispatch(any(MessageDispatch.class)))
            .thenReturn(Uni.createFrom().item(dr));

    provider.postToChannel(ch, "alice", "hello", MessageType.STATUS).await().indefinitely();

    verify(messageService).dispatch(argThat(d ->
            channelId.equals(d.channelId()) &&
            "alice".equals(d.sender()) &&
            d.type() == MessageType.STATUS &&
            "hello".equals(d.content()) &&
            d.correlationId() == null &&
            d.deadline() == null));
}

// NOTE: postToChannel_nullType_sendsWithNullType() is intentionally omitted.
// MessageDispatch.builder().build() validates type != null — passing a null type
// to postToChannel() now throws IllegalArgumentException at the builder. Null type
// is semantically invalid (9-type taxonomy, ADR-0005); the test was documenting an
// accidental pass-through, not a required behaviour.

@Test
void postToChannel_commandWithCorrelationId_passesCorrelationIdToDispatch() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    String content = "{\"type\":\"COMMAND\",\"capability\":\"research\","
            + "\"correlationId\":\"42\",\"input\":{}}";
    DispatchResult dr = new DispatchResult(1L, channelId, "engine", MessageType.COMMAND,
            "42", null, List.of(), null, null, null, null, 0);

    when(messageService.dispatch(any(MessageDispatch.class)))
            .thenReturn(Uni.createFrom().item(dr));

    provider.postToChannel(ch, "engine", content, MessageType.COMMAND).await().indefinitely();

    verify(messageService).dispatch(argThat(d ->
            channelId.equals(d.channelId()) &&
            "engine".equals(d.sender()) &&
            d.type() == MessageType.COMMAND &&
            "42".equals(d.correlationId())));
}

@Test
void postToChannel_queryWithCorrelationId_passesCorrelationIdToDispatch() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    String content = "{\"type\":\"QUERY\",\"correlationId\":\"q-99\",\"input\":{}}";
    DispatchResult dr = new DispatchResult(1L, channelId, "engine", MessageType.QUERY,
            "q-99", null, List.of(), null, null, null, null, 0);

    when(messageService.dispatch(any(MessageDispatch.class)))
            .thenReturn(Uni.createFrom().item(dr));

    provider.postToChannel(ch, "engine", content, MessageType.QUERY).await().indefinitely();

    verify(messageService).dispatch(argThat(d ->
            channelId.equals(d.channelId()) &&
            "q-99".equals(d.correlationId())));
}

@Test
void postToChannel_commandMalformedJson_sendsWithNullCorrelationId() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    DispatchResult dr = new DispatchResult(1L, channelId, "engine", MessageType.COMMAND,
            null, null, List.of(), null, null, null, null, 0);

    when(messageService.dispatch(any(MessageDispatch.class)))
            .thenReturn(Uni.createFrom().item(dr));

    provider.postToChannel(ch, "engine", "not-valid-json", MessageType.COMMAND)
            .await().indefinitely();

    verify(messageService).dispatch(argThat(d ->
            d.correlationId() == null));
}

@Test
void postToChannel_nonCommandType_doesNotParseCorrelationId() {
    UUID channelId = UUID.randomUUID();
    CaseChannel ch = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus",
            Map.of("qhorus-name", "case-x/work"));
    String content = "{\"correlationId\":\"should-be-ignored\"}";
    DispatchResult dr = new DispatchResult(1L, channelId, "engine", MessageType.STATUS,
            null, null, List.of(), null, null, null, null, 0);

    when(messageService.dispatch(any(MessageDispatch.class)))
            .thenReturn(Uni.createFrom().item(dr));

    provider.postToChannel(ch, "engine", content, MessageType.STATUS).await().indefinitely();

    verify(messageService).dispatch(argThat(d ->
            d.correlationId() == null));
}
```

Also add the missing import for `argThat` and `DispatchResult` at the top of the test file. The existing imports include `static org.mockito.ArgumentMatchers.*` which covers `argThat`.

- [ ] **Step 4.2 — Verify the tests fail (mock mismatch)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClaudonyReactiveCaseChannelProviderTest -pl casehub 2>&1 | tail -15
```

Expected: tests fail — `dispatch()` method not found or mock mismatch with `send()`.

- [ ] **Step 4.3 — Update `postToChannel()` to build `MessageDispatch`**

In `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java`:

Add import at the top (after existing imports):
```java
import io.casehub.qhorus.api.message.MessageDispatch;
```

Replace the `postToChannel()` method and `extractCorrelationId()` helper with:

```java
@Override
public Uni<Void> postToChannel(CaseChannel channel, String from, String content, MessageType type) {
    UUID channelId = UUID.fromString(channel.id());
    // correlationId extracted from content: workaround until claudony#135 adds it as a first-class SPI param.
    // deadline is null until claudony#135 also adds it to the SPI.
    String correlationId = (type == MessageType.COMMAND || type == MessageType.QUERY)
            ? extractCorrelationId(content) : null;
    return messageService.dispatch(MessageDispatch.builder()
                    .channelId(channelId)
                    .sender(from)
                    .type(type)
                    .content(content)
                    .correlationId(correlationId)
                    .actorType(io.casehub.platform.api.identity.ActorType.AGENT)
                    .build())
            .replaceWithVoid();
}

// Content-coupling workaround: postToChannel() SPI doesn't carry correlationId as a
// first-class parameter. Track claudony#135 for the SPI fix that removes this method.
private static String extractCorrelationId(String content) {
    try {
        JsonNode node = MAPPER.readTree(content);
        JsonNode cid = node.get("correlationId");
        return (cid != null && !cid.isNull()) ? cid.asText() : null;
    } catch (Exception e) {
        log.warnf("Could not parse correlationId from COMMAND/QUERY content — Commitment will not be tracked");
        return null;
    }
}
```

- [ ] **Step 4.4 — Run the Claudony tests and verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClaudonyReactiveCaseChannelProviderTest -pl casehub 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, all `postToChannel_*` tests pass.

- [ ] **Step 4.5 — Run the full Claudony build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4.6 — Commit Claudony changes**

```bash
git -C /Users/mdproctor/claude/casehub/claudony add casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProvider.java casehub/src/test/java/io/casehub/claudony/casehub/ClaudonyReactiveCaseChannelProviderTest.java
git -C /Users/mdproctor/claude/casehub/claudony commit -m "feat(qhorus#192): postToChannel uses MessageDispatch; wires deadline path

Calls ReactiveMessageService.dispatch(MessageDispatch) instead of
send() flat params. correlationId and deadline (#135) remain the only
pending SPI fields.

Refs casehubio/qhorus#192
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Full build verification

- [ ] **Step 5.1 — Full qhorus build from root**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -20
```

Expected: `BUILD SUCCESS` across all modules (api, runtime, testing, deployment, examples).

- [ ] **Step 5.2 — Full Claudony build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -20
```

Run from `/Users/mdproctor/claude/casehub/claudony`.

Expected: `BUILD SUCCESS`.

- [ ] **Step 5.3 — Close the issue (after code review passes)**

```bash
gh issue close 192 --repo casehubio/qhorus --comment "Fixed: deadline added to MessageDispatch and wired through both blocking and reactive dispatch paths. ReactiveMessageService.send() replaced with dispatch(MessageDispatch) returning Uni<DispatchResult>."
```

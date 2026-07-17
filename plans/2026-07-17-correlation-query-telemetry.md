# Correlation Integrity, QUERY Tracking, and Context Telemetry — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #353 — message correlation strengthening
**Issue group:** #353, #362, #363

**Goal:** Make the obligation model type-aware at resolution time, add default QUERY deadlines, and surface context window pressure as observable telemetry.

**Architecture:** New `CorrelationIntegrityChecker` bean added to the dispatch enforcement gate alongside existing policy beans. Default QUERY deadline applied at commitment open time. Context window percentage captured in EVENT ledger entries and evaluated by a new watchdog condition.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (named `qhorus` PU), H2 (tests), Flyway V2002

## Global Constraints

- Advisory enforcement only — no blocking (WARN log + `DispatchResult.advisories()`)
- Flyway migration V2002 for ledger subclass columns (V2001 is `message_ledger_entry.topic`)
- Pre-release platform — breaking changes cost nothing
- Reactive parity deferred (follow-up issue)
- `mvn` not `./mvnw`; `JAVA_HOME=$(/usr/libexec/java_home -v 26)`

---

### Task 1: CorrelationIntegrityChecker — bean and unit tests

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/CorrelationIntegrityChecker.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/CorrelationIntegrityCheckerTest.java`

**Interfaces:**
- Consumes: `CommitmentStore.findByCorrelationId(String)` → `Optional<Commitment>`, `MessageStore.find(Long)` → `Optional<Message>`
- Produces: `List<String> check(MessageDispatch dispatch, UUID channelId)` — returns advisory strings; empty list when no violations

- [ ] **Step 1: Write the unit test class with all check scenarios**

```java
package io.casehub.qhorus.runtime.message;

import io.casehub.qhorus.api.message.*;
import io.casehub.qhorus.api.store.CommitmentStore;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore;
import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class CorrelationIntegrityCheckerTest {

    private CorrelationIntegrityChecker checker;
    private InMemoryCommitmentStore commitmentStore;
    private InMemoryMessageStore messageStore;
    private UUID channelId;

    @BeforeEach
    void setUp() {
        commitmentStore = new InMemoryCommitmentStore();
        messageStore = new InMemoryMessageStore();
        checker = new CorrelationIntegrityChecker();
        checker.commitmentStore = commitmentStore;
        checker.messageStore = messageStore;
        channelId = UUID.randomUUID();
    }

    // --- inReplyTo validation ---

    @Test
    void inReplyTo_nonExistentMessage_advisory() {
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.STATUS)
                .content("update").inReplyTo(999L).actorType(ActorType.AI).build();

        List<String> advisories = checker.check(dispatch, channelId);

        assertThat(advisories).hasSize(1);
        assertThat(advisories.get(0)).contains("inReplyTo").contains("999");
    }

    @Test
    void inReplyTo_wrongChannel_advisory() {
        UUID otherChannel = UUID.randomUUID();
        Message parent = messageStore.put(Message.builder()
                .channelId(otherChannel).sender("agent-b").messageType(MessageType.COMMAND)
                .actorType(ActorType.AI).content("do something").build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.RESPONSE)
                .content("reply").inReplyTo(parent.id()).actorType(ActorType.AI).build();

        List<String> advisories = checker.check(dispatch, channelId);

        assertThat(advisories).hasSize(1);
        assertThat(advisories.get(0)).contains("different channel");
    }

    @Test
    void inReplyTo_validSameChannel_noAdvisory() {
        Message parent = messageStore.put(Message.builder()
                .channelId(channelId).sender("agent-b").messageType(MessageType.COMMAND)
                .actorType(ActorType.AI).content("do something").build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.STATUS)
                .content("working").inReplyTo(parent.id()).actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    @Test
    void inReplyTo_null_noCheck() {
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.STATUS)
                .content("update").actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    // --- Resolution type matching ---

    @Test
    void responseOnCommandObligation_advisory() {
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.RESPONSE)
                .content("answer").correlationId(corrId).actorType(ActorType.AI).build();

        List<String> advisories = checker.check(dispatch, channelId);

        assertThat(advisories).anyMatch(a -> a.contains("RESPONSE") && a.contains("COMMAND"));
    }

    @Test
    void doneOnQueryObligation_advisory() {
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.QUERY).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.DONE)
                .content("finished").correlationId(corrId).actorType(ActorType.AI).build();

        List<String> advisories = checker.check(dispatch, channelId);

        assertThat(advisories).anyMatch(a -> a.contains("DONE") && a.contains("QUERY"));
    }

    @Test
    void doneOnCommandObligation_noAdvisory() {
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.DONE)
                .content("done").correlationId(corrId).actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    @Test
    void declineOnEitherObligationType_noAdvisory() {
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.DECLINE)
                .content("can't").correlationId(corrId).actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    // --- Obligor identity ---

    @Test
    void wrongSenderResolvingObligation_advisory() {
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-b").type(MessageType.DONE)
                .content("done").correlationId(corrId).actorType(ActorType.AI).build();

        List<String> advisories = checker.check(dispatch, channelId);

        assertThat(advisories).anyMatch(a -> a.contains("agent-b") && a.contains("obligor"));
    }

    @Test
    void delegatedToSenderResolvingObligation_noAdvisory() {
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .delegatedTo("agent-b").state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-b").type(MessageType.DONE)
                .content("done").correlationId(corrId).actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    @Test
    void nullObligor_skipIdentityCheck() {
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.QUERY).requester("requester")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("anyone").type(MessageType.RESPONSE)
                .content("answer").correlationId(corrId).actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    // --- Cross-channel resolution ---

    @Test
    void crossChannelResolution_advisory() {
        UUID otherChannel = UUID.randomUUID();
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(otherChannel)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.DONE)
                .content("done").correlationId(corrId).actorType(ActorType.AI).build();

        List<String> advisories = checker.check(dispatch, channelId);

        assertThat(advisories).anyMatch(a -> a.contains("different channel"));
    }

    // --- Edge cases ---

    @Test
    void noCommitmentFound_noAdvisory() {
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.DONE)
                .content("done").correlationId("nonexistent").actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    @Test
    void nonTerminalType_noObligationChecks() {
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.STATUS)
                .content("working").correlationId("some-corr").actorType(ActorType.AI).build();

        assertThat(checker.check(dispatch, channelId)).isEmpty();
    }

    @Test
    void multipleViolations_allReported() {
        UUID otherChannel = UUID.randomUUID();
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(otherChannel)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-b").type(MessageType.RESPONSE)
                .content("answer").correlationId(corrId).inReplyTo(999L)
                .actorType(ActorType.AI).build();

        List<String> advisories = checker.check(dispatch, channelId);

        assertThat(advisories).hasSizeGreaterThanOrEqualTo(3);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CorrelationIntegrityCheckerTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — `CorrelationIntegrityChecker` class does not exist

- [ ] **Step 3: Implement CorrelationIntegrityChecker**

```java
package io.casehub.qhorus.runtime.message;

import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.CommitmentStore;
import io.casehub.qhorus.api.store.MessageStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class CorrelationIntegrityChecker {

    private static final Set<MessageType> TERMINAL_TYPES = Set.of(
            MessageType.DONE, MessageType.FAILURE, MessageType.DECLINE,
            MessageType.RESPONSE, MessageType.HANDOFF);

    @Inject
    CommitmentStore commitmentStore;

    @Inject
    MessageStore messageStore;

    public List<String> check(MessageDispatch dispatch, UUID channelId) {
        List<String> advisories = new ArrayList<>();
        checkInReplyTo(dispatch, channelId, advisories);
        checkObligationIntegrity(dispatch, channelId, advisories);
        return List.copyOf(advisories);
    }

    private void checkInReplyTo(MessageDispatch dispatch, UUID channelId,
                                List<String> advisories) {
        if (dispatch.inReplyTo() == null) return;
        var parent = messageStore.find(dispatch.inReplyTo());
        if (parent.isEmpty()) {
            advisories.add("inReplyTo references non-existent message ID " + dispatch.inReplyTo());
        } else if (!parent.get().channelId().equals(channelId)) {
            advisories.add("inReplyTo references message in different channel");
        }
    }

    private void checkObligationIntegrity(MessageDispatch dispatch, UUID channelId,
                                          List<String> advisories) {
        if (dispatch.correlationId() == null) return;
        if (!TERMINAL_TYPES.contains(dispatch.type())) return;

        var commitment = commitmentStore.findByCorrelationId(dispatch.correlationId());
        if (commitment.isEmpty()) return;

        Commitment c = commitment.get();

        // Resolution type matching
        if (c.messageType() == MessageType.COMMAND && dispatch.type() == MessageType.RESPONSE) {
            advisories.add("RESPONSE used to resolve COMMAND obligation — expected DONE/FAILURE/DECLINE");
        }
        if (c.messageType() == MessageType.QUERY
                && (dispatch.type() == MessageType.DONE || dispatch.type() == MessageType.FAILURE)) {
            advisories.add(dispatch.type() + " used to resolve QUERY obligation — expected RESPONSE/DECLINE");
        }

        // Obligor identity
        if (c.obligor() != null
                && !dispatch.sender().equals(c.obligor())
                && (c.delegatedTo() == null || !dispatch.sender().equals(c.delegatedTo()))) {
            advisories.add("Sender '" + dispatch.sender() + "' is not the obligor ('"
                    + c.obligor() + "') for correlationId '" + dispatch.correlationId() + "'");
        }

        // Cross-channel resolution
        if (!c.channelId().equals(channelId)) {
            advisories.add("Resolving obligation from different channel (obligation channel: "
                    + c.channelId() + ", dispatch channel: " + channelId + ")");
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CorrelationIntegrityCheckerTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(#353): CorrelationIntegrityChecker — advisory validation for correlation integrity

Four advisory checks: inReplyTo existence/channel, resolution type matching
(RESPONSE vs DONE for QUERY vs COMMAND obligations), obligor identity, and
cross-channel resolution detection.

Refs #353
```

---

### Task 2: Wire checker into MessageService + default QUERY deadline

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (inject checker, call in dispatch, add default QUERY deadline)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` (add `default-query-deadline` to `Commitment` interface)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/CorrelationIntegrityDispatchTest.java`

**Interfaces:**
- Consumes: `CorrelationIntegrityChecker.check(MessageDispatch, UUID)` from Task 1
- Produces: Advisories in `DispatchResult.advisories()`, default deadline on QUERY commitments

- [ ] **Step 1: Add `defaultQueryDeadline` to QhorusConfig.Commitment**

In `QhorusConfig.java`, add to the `Commitment` interface:

```java
Optional<java.time.Duration> defaultQueryDeadline();
```

- [ ] **Step 2: Write the integration test**

```java
package io.casehub.qhorus.runtime.message;

import io.casehub.qhorus.api.message.*;
import io.casehub.qhorus.api.store.CommitmentStore;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore;
import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.config.QhorusTracingConfig;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class CorrelationIntegrityDispatchTest {

    private MessageService messageService;
    private InMemoryMessageStore messageStore;
    private InMemoryCommitmentStore commitmentStore;
    private CommitmentService commitmentService;
    private UUID channelId;

    @BeforeEach
    void setUp() {
        messageStore = new InMemoryMessageStore();
        commitmentStore = new InMemoryCommitmentStore();

        QhorusTracingConfig tracingConfig = mock(QhorusTracingConfig.class);
        when(tracingConfig.enabled()).thenReturn(false);

        commitmentService = new CommitmentService();
        commitmentService.store = commitmentStore;
        commitmentService.tracingConfig = tracingConfig;

        messageService = new MessageService();
        messageService.messageStore = messageStore;
        messageService.commitmentService = commitmentService;
        messageService.tracingConfig = tracingConfig;

        // Wire the checker
        CorrelationIntegrityChecker checker = new CorrelationIntegrityChecker();
        checker.commitmentStore = commitmentStore;
        checker.messageStore = messageStore;
        messageService.correlationIntegrityChecker = checker;

        // Wire remaining dependencies — minimal stubs for CDI-free test
        // (channelService, currentPrincipal, crossTenantChannelStore, etc.
        //  are set up by the test to allow dispatch to proceed)
        channelId = UUID.randomUUID();
    }

    @Test
    void dispatch_wrongObligor_advisoryInResult() {
        // Set up: COMMAND creates obligation for agent-a
        String corrId = UUID.randomUUID().toString();
        commitmentStore.save(Commitment.builder()
                .id(UUID.randomUUID()).correlationId(corrId).channelId(channelId)
                .messageType(MessageType.COMMAND).requester("requester").obligor("agent-a")
                .state(CommitmentState.OPEN).build());

        // agent-b tries to resolve it
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-b").type(MessageType.DONE)
                .content("done").correlationId(corrId).actorType(ActorType.AI).build();

        DispatchResult result = messageService.dispatch(dispatch);

        assertThat(result.advisories()).anyMatch(a -> a.contains("obligor"));
    }

    @Test
    void dispatch_queryWithDefaultDeadline_commitmentHasDeadline() {
        String corrId = UUID.randomUUID().toString();
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.QUERY)
                .content("what is X?").correlationId(corrId).actorType(ActorType.AI).build();

        // Config with 5-minute default QUERY deadline
        // (mock config to return the duration)

        DispatchResult result = messageService.dispatch(dispatch);

        Optional<Commitment> commitment = commitmentStore.findByCorrelationId(corrId);
        assertThat(commitment).isPresent();
        assertThat(commitment.get().expiresAt()).isNotNull();
    }

    @Test
    void dispatch_queryWithExplicitDeadline_usesExplicitNotDefault() {
        Instant explicit = Instant.now().plusSeconds(120);
        String corrId = UUID.randomUUID().toString();
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.QUERY)
                .content("what?").correlationId(corrId).deadline(explicit)
                .actorType(ActorType.AI).build();

        DispatchResult result = messageService.dispatch(dispatch);

        Optional<Commitment> commitment = commitmentStore.findByCorrelationId(corrId);
        assertThat(commitment).isPresent();
        assertThat(commitment.get().expiresAt()).isEqualTo(explicit);
    }

    @Test
    void dispatch_commandIgnoresDefaultQueryDeadline() {
        String corrId = UUID.randomUUID().toString();
        MessageDispatch dispatch = MessageDispatch.builder()
                .channelId(channelId).sender("agent-a").type(MessageType.COMMAND)
                .content("do X").correlationId(corrId).target("agent-b")
                .actorType(ActorType.AI).build();

        DispatchResult result = messageService.dispatch(dispatch);

        Optional<Commitment> commitment = commitmentStore.findByCorrelationId(corrId);
        assertThat(commitment).isPresent();
        assertThat(commitment.get().expiresAt()).isNull();
    }
}
```

Note: This test requires wiring MessageService CDI-free. The existing CDI-free unit tests in the codebase (e.g. `CommitmentServiceTest` in `testing/`) set fields directly. The test above is a template — the actual wiring will depend on how many dependencies can be nulled vs mocked vs stubbed. The cross-tenant channel store needs to return a channel for `dispatch.channelId()` lookups.

- [ ] **Step 3: Add CorrelationIntegrityChecker injection to MessageService**

In `MessageService.java`, add a new field:

```java
@Inject
CorrelationIntegrityChecker correlationIntegrityChecker;
```

- [ ] **Step 4: Call checker in dispatch() after message type policy**

After the `messageTypePolicy.advisory()` block (around line 211), add:

```java
if (ch != null) {
    List<String> correlationAdvisories = correlationIntegrityChecker.check(dispatch, ch.id());
    if (!correlationAdvisories.isEmpty()) {
        for (String ca : correlationAdvisories) {
            LOG.warn(ca);
        }
        advisories = new ArrayList<>(advisories);
        advisories.addAll(correlationAdvisories);
    }
}
```

- [ ] **Step 5: Add default QUERY deadline in the commitment switch**

In the commitment switch block (around line 303-316), change the QUERY/COMMAND case to:

```java
case QUERY, COMMAND -> {
    Instant effectiveDeadline = saved.deadline();
    if (effectiveDeadline == null && dispatch.type() == MessageType.QUERY) {
        config.commitment().defaultQueryDeadline().ifPresent(d ->
            // Use a local variable since lambda captures must be effectively final
        );
        // Alternative: inline the check
        var defaultDl = config.commitment().defaultQueryDeadline();
        if (defaultDl.isPresent()) {
            effectiveDeadline = Instant.now().plus(defaultDl.get());
        }
    }
    commitmentService.open(
            storedCommitmentId,
            dispatch.correlationId(), dispatch.channelId(), dispatch.type(),
            dispatch.sender(), dispatch.target(), effectiveDeadline);
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CorrelationIntegrityDispatchTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: ALL PASS

- [ ] **Step 7: Run full test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat(#353,#362): wire CorrelationIntegrityChecker into dispatch gate + default QUERY deadline

Correlation integrity advisories now flow through DispatchResult.advisories().
QUERYs without an explicit deadline receive the configured default
(casehub.qhorus.commitment.default-query-deadline), applied at the Commitment
level only — Message.deadline stays null (preserves sender intent).

Refs #353, #362
```

---

### Task 3: Context window telemetry — schema, parsing, hash

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java` (add `contextWindowPct` field + update `domainContentBytes`)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java` (parse `context_window_pct` in `populateTelemetry()`)
- Create: `db/qhorus/migration/V2002__message_ledger_entry_context_window_pct.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/ContextWindowTelemetryTest.java`

**Interfaces:**
- Consumes: EVENT telemetry JSON with optional `context_window_pct` integer field
- Produces: `MessageLedgerEntry.contextWindowPct` (nullable Integer, 0-100) populated for EVENT entries

- [ ] **Step 1: Write the test for populateTelemetry parsing**

```java
package io.casehub.qhorus.runtime.ledger;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ContextWindowTelemetryTest {

    private LedgerWriteService service;
    private MessageLedgerEntry entry;

    @BeforeEach
    void setUp() {
        service = new LedgerWriteService();
        service.objectMapper = new ObjectMapper();
        entry = new MessageLedgerEntry();
    }

    @Test
    void populateTelemetry_withContextWindowPct() throws Exception {
        var method = LedgerWriteService.class.getDeclaredMethod(
                "populateTelemetry", MessageLedgerEntry.class, String.class);
        method.setAccessible(true);

        String json = "{\"tool_name\":\"search\",\"context_window_pct\":75}";
        method.invoke(service, entry, json);

        assertThat(entry.contextWindowPct).isEqualTo(75);
        assertThat(entry.toolName).isEqualTo("search");
    }

    @Test
    void populateTelemetry_withoutContextWindowPct_remainsNull() throws Exception {
        var method = LedgerWriteService.class.getDeclaredMethod(
                "populateTelemetry", MessageLedgerEntry.class, String.class);
        method.setAccessible(true);

        String json = "{\"tool_name\":\"search\",\"duration_ms\":42}";
        method.invoke(service, entry, json);

        assertThat(entry.contextWindowPct).isNull();
        assertThat(entry.toolName).isEqualTo("search");
    }

    @Test
    void populateTelemetry_contextWindowPctZero() throws Exception {
        var method = LedgerWriteService.class.getDeclaredMethod(
                "populateTelemetry", MessageLedgerEntry.class, String.class);
        method.setAccessible(true);

        String json = "{\"context_window_pct\":0}";
        method.invoke(service, entry, json);

        assertThat(entry.contextWindowPct).isEqualTo(0);
    }

    @Test
    void populateTelemetry_contextWindowPct100() throws Exception {
        var method = LedgerWriteService.class.getDeclaredMethod(
                "populateTelemetry", MessageLedgerEntry.class, String.class);
        method.setAccessible(true);

        String json = "{\"context_window_pct\":100}";
        method.invoke(service, entry, json);

        assertThat(entry.contextWindowPct).isEqualTo(100);
    }

    @Test
    void domainContentBytes_includesContextWindowPct() {
        entry.contextWindowPct = 85;
        entry.channelId = java.util.UUID.randomUUID();
        entry.messageId = 1L;
        entry.messageType = "EVENT";

        byte[] bytes = entry.domainContentBytes();
        String canonical = new String(bytes);

        assertThat(canonical).contains("85");
    }

    @Test
    void domainContentBytes_nullContextWindowPct_emptySegment() {
        entry.contextWindowPct = null;
        entry.channelId = java.util.UUID.randomUUID();
        entry.messageId = 1L;
        entry.messageType = "EVENT";

        byte[] bytes = entry.domainContentBytes();
        // Should not throw and should produce a valid canonical string
        assertThat(bytes).isNotEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ContextWindowTelemetryTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — field `contextWindowPct` does not exist on `MessageLedgerEntry`

- [ ] **Step 3: Add contextWindowPct field to MessageLedgerEntry**

After the `sourceEntity` field (line 86), add:

```java
@Column(name = "context_window_pct")
public Integer contextWindowPct;
```

- [ ] **Step 4: Update domainContentBytes() to include the new field**

In `domainContentBytes()`, add to the `String.join("|", ...)` call, after the `sourceEntity` line:

```java
contextWindowPct != null ? contextWindowPct.toString() : ""
```

- [ ] **Step 5: Add parsing in LedgerWriteService.populateTelemetry()**

After the `source_entity` parsing block (around line 387), add:

```java
final JsonNode cwp = root.get("context_window_pct");
if (cwp != null && cwp.isNumber()) {
    entry.contextWindowPct = cwp.asInt();
}
```

- [ ] **Step 6: Create V2002 Flyway migration**

Create `runtime/src/main/resources/db/qhorus/migration/V2002__message_ledger_entry_context_window_pct.sql`:

```sql
ALTER TABLE message_ledger_entry ADD COLUMN context_window_pct SMALLINT;
```

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ContextWindowTelemetryTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat(#363): context_window_pct telemetry on MessageLedgerEntry

Nullable SMALLINT column (V2002 migration) parsed from EVENT telemetry JSON.
Agents reporting context window usage (0-100%) populate this field
automatically via the existing telemetry pipeline.

Refs #363
```

---

### Task 4: CONTEXT_PRESSURE watchdog condition

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/ContextPressureContext.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java` (add to `permits`)
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogConditionType.java` (add enum value)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java` (new query)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java` (inject repo, add evaluate method, add to switch)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/ContextPressureWatchdogTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntry.contextWindowPct` from Task 3
- Produces: `CONTEXT_PRESSURE` watchdog condition; fires when any agent's latest context usage ≥ threshold

- [ ] **Step 1: Create ContextPressureContext record**

```java
package io.casehub.qhorus.api.watchdog;

import java.util.UUID;

public record ContextPressureContext(
        UUID channelId,
        String channelName,
        String actorId,
        int contextWindowPct
) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() {
        return WatchdogConditionType.CONTEXT_PRESSURE;
    }
}
```

- [ ] **Step 2: Update AlertContext sealed permits**

In `AlertContext.java`, add `ContextPressureContext` to the `permits` clause:

```java
public sealed interface AlertContext
        permits BarrierStuckContext, ApprovalPendingContext,
                AgentStaleContext, ChannelIdleContext, QueueDepthContext,
                ContextPressureContext {
```

- [ ] **Step 3: Add CONTEXT_PRESSURE to WatchdogConditionType enum**

```java
public enum WatchdogConditionType {
    BARRIER_STUCK, APPROVAL_PENDING, AGENT_STALE, CHANNEL_IDLE, QUEUE_DEPTH,
    CONTEXT_PRESSURE
}
```

- [ ] **Step 4: Add repository query for latest context pressure per agent**

In `MessageLedgerEntryRepository.java`, add:

```java
public List<MessageLedgerEntry> findLatestContextPressure(final UUID channelId, final String tenancyId) {
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e WHERE e.subjectId = :cid AND e.tenancyId = :tid" +
            " AND e.messageType = 'EVENT' AND e.contextWindowPct IS NOT NULL" +
            " AND e.sequenceNumber = (SELECT MAX(e2.sequenceNumber) FROM MessageLedgerEntry e2" +
            " WHERE e2.subjectId = :cid AND e2.tenancyId = :tid" +
            " AND e2.messageType = 'EVENT' AND e2.contextWindowPct IS NOT NULL" +
            " AND e2.actorId = e.actorId)",
            MessageLedgerEntry.class)
            .setParameter("cid", channelId)
            .setParameter("tid", tenancyId(tenancyId))
            .getResultList();
}
```

- [ ] **Step 5: Write the watchdog test**

```java
package io.casehub.qhorus.runtime.watchdog;

import io.casehub.qhorus.api.watchdog.*;
import io.casehub.qhorus.persistence.memory.InMemoryCrossTenantChannelStore;
import io.casehub.qhorus.persistence.memory.InMemoryCrossTenantWatchdogStore;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.config.QhorusTracingConfig;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ContextPressureWatchdogTest {

    private WatchdogEvaluationService service;
    private InMemoryCrossTenantChannelStore channelStore;
    private InMemoryCrossTenantWatchdogStore watchdogStore;
    private MessageLedgerEntryRepository messageRepo;
    private Event<WatchdogAlertEvent> alertEvents;
    private UUID channelId;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        channelStore = new InMemoryCrossTenantChannelStore();
        watchdogStore = new InMemoryCrossTenantWatchdogStore();
        messageRepo = mock(MessageLedgerEntryRepository.class);
        alertEvents = mock(Event.class);

        QhorusConfig config = mock(QhorusConfig.class);
        QhorusConfig.Watchdog watchdogConfig = mock(QhorusConfig.Watchdog.class);
        when(config.watchdog()).thenReturn(watchdogConfig);
        when(watchdogConfig.enabled()).thenReturn(true);

        service = new WatchdogEvaluationService();
        service.config = config;
        service.crossTenantChannelStore = channelStore;
        service.crossTenantWatchdogStore = watchdogStore;
        service.messageRepo = messageRepo;
        service.alertEvents = alertEvents;

        QhorusTracingConfig tracingConfig = mock(QhorusTracingConfig.class);
        when(tracingConfig.enabled()).thenReturn(false);

        channelId = UUID.randomUUID();
    }

    @Test
    void contextPressure_aboveThreshold_firesAlert() {
        // Set up channel
        Channel ch = new Channel();
        ch.id = channelId;
        ch.name = "test-channel";
        ch.semantic = ChannelSemantic.APPEND;
        ch.tenancyId = "default";
        channelStore.put(ch);

        // Set up watchdog
        Watchdog wd = Watchdog.builder("CONTEXT_PRESSURE", "test-channel")
                .id(UUID.randomUUID()).thresholdCount(80)
                .notificationChannel("alerts").tenancyId("default").build();
        watchdogStore.put(wd);

        // Mock ledger query returning agent at 90%
        MessageLedgerEntry entry = new MessageLedgerEntry();
        entry.channelId = channelId;
        entry.contextWindowPct = 90;
        entry.messageType = "EVENT";
        entry.actorId = "agent-a";
        when(messageRepo.findLatestContextPressure(channelId, "default"))
                .thenReturn(List.of(entry));

        service.evaluateAll();

        verify(alertEvents).fireAsync(argThat(event ->
                event.summary().contains("CONTEXT_PRESSURE")
                && event.summary().contains("agent-a")
                && event.summary().contains("90")));
    }

    @Test
    void contextPressure_belowThreshold_noAlert() {
        Channel ch = new Channel();
        ch.id = channelId;
        ch.name = "test-channel";
        ch.semantic = ChannelSemantic.APPEND;
        ch.tenancyId = "default";
        channelStore.put(ch);

        Watchdog wd = Watchdog.builder("CONTEXT_PRESSURE", "test-channel")
                .id(UUID.randomUUID()).thresholdCount(80)
                .notificationChannel("alerts").tenancyId("default").build();
        watchdogStore.put(wd);

        MessageLedgerEntry entry = new MessageLedgerEntry();
        entry.channelId = channelId;
        entry.contextWindowPct = 50;
        entry.messageType = "EVENT";
        entry.actorId = "agent-a";
        when(messageRepo.findLatestContextPressure(channelId, "default"))
                .thenReturn(List.of(entry));

        service.evaluateAll();

        verify(alertEvents, never()).fireAsync(any());
    }
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ContextPressureWatchdogTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — `ContextPressureContext` not found, `messageRepo` field not on service

- [ ] **Step 7: Add messageRepo injection to WatchdogEvaluationService**

Add field:

```java
@Inject
MessageLedgerEntryRepository messageRepo;
```

- [ ] **Step 8: Add evaluateContextPressure method to WatchdogEvaluationService**

```java
private boolean evaluateContextPressure(Watchdog w, Instant now) {
    int threshold = w.thresholdCount() != null ? w.thresholdCount() : 80;

    List<Channel> channels = crossTenantChannelStore.listAll().stream()
            .filter(ch -> "*".equals(w.targetName()) || ch.name().equals(w.targetName()))
            .toList();

    boolean fired = false;
    for (Channel ch : channels) {
        List<MessageLedgerEntry> entries = messageRepo.findLatestContextPressure(
                ch.id(), w.tenancyId());
        for (MessageLedgerEntry entry : entries) {
            if (entry.contextWindowPct != null && entry.contextWindowPct >= threshold) {
                String summary = "CONTEXT_PRESSURE: agent='" + entry.actorId
                        + "' at " + entry.contextWindowPct + "% on channel='" + ch.name() + "'";
                fireAlert(w, summary,
                        new ContextPressureContext(ch.id(), ch.name(),
                                entry.actorId, entry.contextWindowPct),
                        now);
                fired = true;
            }
        }
    }
    return fired;
}
```

- [ ] **Step 9: Add CONTEXT_PRESSURE to the evaluateAll switch**

In `evaluateAll()`, add to the switch (around line 100):

```java
case "CONTEXT_PRESSURE" -> evaluateContextPressure(w, now);
```

- [ ] **Step 10: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ContextPressureWatchdogTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: ALL PASS

- [ ] **Step 11: Update get_telemetry_summary MCP tool**

In `QhorusMcpToolsBase.java`, find the `get_telemetry_summary` `@Tool` method. Add context pressure stats to the response — for each agent, include their latest `contextWindowPct` from the ledger. Use the same `findLatestContextPressure` query from the repository. Add the field to the existing telemetry summary response record.

- [ ] **Step 12: Run full build to check all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS (api changes visible to all modules)

- [ ] **Step 13: Commit**

```
feat(#363): CONTEXT_PRESSURE watchdog condition

New condition fires when any agent's latest context_window_pct exceeds
the threshold (default 80%). ContextPressureContext record added to the
sealed AlertContext hierarchy. New ledger query finds latest context
pressure per agent per channel.

Refs #363
```

---

## CLAUDE.md Update

After all tasks complete, update `CLAUDE.md` with:
- `CorrelationIntegrityChecker` in the project structure (under `runtime/message/`)
- `V2002` in the Flyway migration list
- `CONTEXT_PRESSURE` in the watchdog conditions documentation
- `casehub.qhorus.commitment.default-query-deadline` config reference
- Note that `contextWindowPct` is included in `domainContentBytes()` hash

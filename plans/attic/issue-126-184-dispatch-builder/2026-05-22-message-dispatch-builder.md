# MessageDispatch Builder API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the 9-parameter positional `MessageService.send()` with `dispatch(MessageDispatch)`, returning `DispatchResult` with `ledgerEntryId`, and fix `subjectId`/`causedByEntryId` propagation in the ledger for all 9 message types.

**Architecture:** `MessageDispatch` (builder, in `api/message/`) carries all dispatch parameters including the optional `subjectId` and `causedByEntryId`. `MessageService.dispatch()` owns the ledger write (moving it out of `QhorusMcpTools`); `LedgerWriteService.record()` resolves both fields via a two-priority rule: explicit > auto-propagated (correlation root for subject, `inReplyTo` lookup for causation). `DispatchResult` echoes the resolved values so callers can chain subsequent domain entries without a secondary ledger query.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/H2 (tests), `@Transactional`/`REQUIRES_NEW`, Jackson 2.x, `quarkus-mcp-server`

---

## File Map

| Action | File |
|---|---|
| DELETE | `api/src/main/java/io/casehub/qhorus/api/message/MessageResult.java` |
| CREATE | `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java` |
| CREATE | `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java` |
| CREATE | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteOutcome.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveMessageLedgerEntryRepository.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java` |
| MODIFY | `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` |
| CREATE | `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchBuilderTest.java` |
| CREATE | `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/StubMessageLedgerEntryRepository.java` |
| CREATE | `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/LedgerWritePropagationTest.java` |
| CREATE | `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryTestFactory.java` |
| CREATE | `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepositoryTest.java` |
| CREATE | `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchIntegrationTest.java` |
| MODIFY | 28 existing test files (see Task 8) |

---

## Task 1: Create `MessageDispatch` and `DispatchResult`, delete `MessageResult`

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java`
- Delete: `api/src/main/java/io/casehub/qhorus/api/message/MessageResult.java`
- Create (test): `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchBuilderTest.java`

- [ ] **Step 1: Write the failing builder validation test**

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchBuilderTest.java
package io.casehub.qhorus.runtime.message;

import static org.assertj.core.api.Assertions.*;

import java.util.UUID;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;

class MessageDispatchBuilderTest {

    // ── Valid cases ───────────────────────────────────────────────────────────

    @Test void command_minimal_passes() {
        assertThatNoException().isThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("agent").type(MessageType.COMMAND)
                .content("do it").correlationId("c1").actorType(ActorType.AGENT).build());
    }

    @Test void done_with_required_fields_passes() {
        assertThatNoException().isThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("agent").type(MessageType.DONE)
                .content("done").correlationId("c1").inReplyTo(42L).actorType(ActorType.AGENT).build());
    }

    @Test void handoff_with_all_required_passes() {
        assertThatNoException().isThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("agent").type(MessageType.HANDOFF)
                .content("delegate").correlationId("c1").inReplyTo(42L).target("role:specialist")
                .actorType(ActorType.AGENT).build());
    }

    @Test void status_without_reply_passes() {
        assertThatNoException().isThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("agent").type(MessageType.STATUS)
                .content("progress").actorType(ActorType.AGENT).build());
    }

    // ── DONE/DECLINE/FAILURE require inReplyTo ────────────────────────────────

    @Test void done_without_inReplyTo_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.DONE)
                .correlationId("c1").actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("DONE").hasMessageContaining("inReplyTo");
    }

    @Test void decline_without_inReplyTo_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.DECLINE)
                .correlationId("c1").actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("DECLINE").hasMessageContaining("inReplyTo");
    }

    @Test void failure_without_inReplyTo_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.FAILURE)
                .correlationId("c1").actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("FAILURE").hasMessageContaining("inReplyTo");
    }

    // ── DONE/DECLINE/FAILURE require correlationId ────────────────────────────

    @Test void done_without_correlationId_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.DONE)
                .inReplyTo(1L).actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("DONE").hasMessageContaining("correlationId");
    }

    // ── RESPONSE requires inReplyTo ───────────────────────────────────────────

    @Test void response_without_inReplyTo_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.RESPONSE)
                .content("answer").actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("RESPONSE").hasMessageContaining("inReplyTo");
    }

    // ── HANDOFF requires inReplyTo + correlationId + target ──────────────────

    @Test void handoff_without_target_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.HANDOFF)
                .inReplyTo(1L).correlationId("c1").actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("HANDOFF").hasMessageContaining("target");
    }

    @Test void handoff_without_correlationId_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.HANDOFF)
                .inReplyTo(1L).target("role:x").actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("HANDOFF").hasMessageContaining("correlationId");
    }

    @Test void handoff_blank_target_throws() {
        assertThatThrownBy(() ->
            MessageDispatch.builder()
                .channelId(UUID.randomUUID()).sender("a").type(MessageType.HANDOFF)
                .inReplyTo(1L).correlationId("c1").target("  ").actorType(ActorType.AGENT).build())
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("HANDOFF").hasMessageContaining("target");
    }
}
```

- [ ] **Step 2: Run — expect compile failure** (`MessageDispatch` does not exist yet)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageDispatchBuilderTest -pl runtime 2>&1 | tail -20
```
Expected: compilation error — `MessageDispatch cannot be found`

- [ ] **Step 3: Create `MessageDispatch.java`**

```java
// api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java
package io.casehub.qhorus.api.message;

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
        ActorType actorType) {

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
                    inReplyTo, artefactRefs, target, subjectId, causedByEntryId, actorType);
        }
    }
}
```

- [ ] **Step 4: Create `DispatchResult.java`**

```java
// api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java
package io.casehub.qhorus.api.message;

import java.util.List;
import java.util.UUID;

import com.fasterxml.jackson.annotation.JsonInclude;
import jakarta.annotation.Nullable;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record DispatchResult(
        Long messageId,
        UUID channelId,
        String sender,
        MessageType type,
        String correlationId,
        Long inReplyTo,
        @JsonInclude(JsonInclude.Include.NON_EMPTY) List<UUID> artefactRefs,
        String target,
        @Nullable UUID ledgerEntryId,    // null when ledger writes suppressed
        @Nullable UUID subjectId,        // resolved value actually written to ledger
        @Nullable UUID causedByEntryId   // resolved value actually written to ledger
) {
    public DispatchResult {
        artefactRefs = (artefactRefs == null) ? List.of() : List.copyOf(artefactRefs);
    }

    /** Parse a comma-separated artefact refs string (from Message entity) into List<UUID>. */
    public static List<UUID> parseArtefactRefs(String raw) {
        if (raw == null || raw.isBlank()) return List.of();
        return java.util.Arrays.stream(raw.split(","))
                .map(String::trim)
                .filter(s -> !s.isBlank())
                .map(UUID::fromString)
                .toList();
    }
}
```

- [ ] **Step 5: Delete `MessageResult.java`**

```bash
rm /path/to/qhorus/api/src/main/java/io/casehub/qhorus/api/message/MessageResult.java
```

- [ ] **Step 6: Run builder tests — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageDispatchBuilderTest -pl runtime
```
Expected: all 11 tests PASS. Compilation errors expected elsewhere (call sites still reference `MessageResult` and `send()`) — that is fine, we are working task by task. Run only this test class.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/main/java/io/casehub/qhorus/api/message/
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchBuilderTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): MessageDispatch builder + DispatchResult; delete MessageResult

Builder enforces type-specific required fields at runtime (build() invocation).
DispatchResult carries resolved ledgerEntryId, subjectId, causedByEntryId.

Refs #184"
```

---

## Task 2: `LedgerWriteOutcome` + two new repository queries

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteOutcome.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryTestFactory.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepositoryTest.java`

- [ ] **Step 1: Create `LedgerWriteOutcome.java`**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteOutcome.java
package io.casehub.qhorus.runtime.ledger;

import java.util.UUID;
import jakarta.annotation.Nullable;

/** Return carrier from {@link LedgerWriteService#record}. Carries resolved field values back to {@link io.casehub.qhorus.runtime.message.MessageService}. */
public record LedgerWriteOutcome(
        @Nullable UUID entryId,
        @Nullable UUID subjectId,
        @Nullable UUID causedByEntryId) {

    /** Sentinel returned when ledger writes are suppressed via config. */
    public static final LedgerWriteOutcome DISABLED = new LedgerWriteOutcome(null, null, null);
}
```

- [ ] **Step 2: Write the failing repository tests**

First create the test factory:

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryTestFactory.java
package io.casehub.qhorus.runtime.ledger;

import java.time.Instant;
import java.util.UUID;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.platform.api.identity.ActorType;

/** Builds {@link MessageLedgerEntry} instances with required fields populated. */
public final class MessageLedgerEntryTestFactory {

    private MessageLedgerEntryTestFactory() {}

    public static MessageLedgerEntry entry(String messageType) {
        return entry(UUID.randomUUID(), 1L, messageType, UUID.randomUUID(), null);
    }

    public static MessageLedgerEntry entry(UUID subjectId, Long messageId, String messageType,
            UUID channelId, String correlationId) {
        MessageLedgerEntry e = new MessageLedgerEntry();
        e.id = UUID.randomUUID();
        e.subjectId = subjectId;
        e.channelId = channelId;
        e.messageId = messageId;
        e.messageType = messageType;
        e.correlationId = correlationId;
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.COMMAND;
        e.actorId = "test-actor";
        e.actorType = ActorType.AGENT;
        e.occurredAt = Instant.now();
        return e;
    }
}
```

Then write the repository tests:

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepositoryTest.java
package io.casehub.qhorus.runtime.ledger;

import static org.assertj.core.api.Assertions.*;

import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;

@QuarkusTest
class MessageLedgerEntryRepositoryTest {

    @Inject MessageLedgerEntryRepository repository;

    @Test @TestTransaction
    void findByMessageId_returns_entry_for_known_messageId() {
        UUID channelId = UUID.randomUUID();
        MessageLedgerEntry e = MessageLedgerEntryTestFactory.entry(
                channelId, 99L, "COMMAND", channelId, "corr-1");
        e.sequenceNumber = 1;
        repository.save(e);

        Optional<MessageLedgerEntry> found = repository.findByMessageId(99L);
        assertThat(found).isPresent();
        assertThat(found.get().messageId).isEqualTo(99L);
    }

    @Test @TestTransaction
    void findByMessageId_returns_empty_for_unknown_messageId() {
        assertThat(repository.findByMessageId(-999L)).isEmpty();
    }

    @Test @TestTransaction
    void findEarliestWithSubjectByCorrelationId_returns_first_by_sequenceNumber() {
        UUID subjectId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();

        // seq 1 — first entry
        MessageLedgerEntry first = MessageLedgerEntryTestFactory.entry(
                subjectId, 1L, "COMMAND", channelId, "corr-x");
        first.sequenceNumber = 1;
        repository.save(first);

        // seq 2 — later entry, same correlation, same subject
        MessageLedgerEntry second = MessageLedgerEntryTestFactory.entry(
                subjectId, 2L, "STATUS", channelId, "corr-x");
        second.sequenceNumber = 2;
        repository.save(second);

        Optional<MessageLedgerEntry> found =
                repository.findEarliestWithSubjectByCorrelationId("corr-x");
        assertThat(found).isPresent();
        assertThat(found.get().messageId).isEqualTo(1L); // seq 1 wins
        assertThat(found.get().subjectId).isEqualTo(subjectId);
    }

    @Test @TestTransaction
    void findEarliestWithSubjectByCorrelationId_returns_empty_when_no_match() {
        assertThat(repository.findEarliestWithSubjectByCorrelationId("no-such-corr")).isEmpty();
    }
}
```

- [ ] **Step 3: Run — expect compile failure** (methods don't exist yet)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageLedgerEntryRepositoryTest -pl runtime 2>&1 | tail -20
```
Expected: compile error — `findByMessageId` and `findEarliestWithSubjectByCorrelationId` not found.

- [ ] **Step 4: Add the two new queries to `MessageLedgerEntryRepository`**

Add these two methods to `MessageLedgerEntryRepository.java` (after `findLatestByCorrelationId`):

```java
/**
 * Finds the ledger entry whose {@code messageId} matches the given message entity ID.
 * Used at ledger write time to resolve {@code causedByEntryId} from {@code inReplyTo}.
 */
public Optional<MessageLedgerEntry> findByMessageId(final Long messageId) {
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e WHERE e.messageId = :mid",
            MessageLedgerEntry.class)
            .setParameter("mid", messageId)
            .setMaxResults(1)
            .getResultStream()
            .findFirst();
}

/**
 * Returns the earliest entry in a correlation thread that has a non-null {@code subjectId}.
 * Used at write time to propagate the domain subject ({@code subjectId}) from the originating
 * COMMAND to all subsequent messages in the same correlation thread.
 *
 * <p>Ordered by {@code sequenceNumber ASC} — monotonic MMR sequence, clock-skew-safe.
 */
public Optional<MessageLedgerEntry> findEarliestWithSubjectByCorrelationId(
        final String correlationId) {
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e " +
                    "WHERE e.correlationId = :corr AND e.subjectId IS NOT NULL " +
                    "ORDER BY e.sequenceNumber ASC",
            MessageLedgerEntry.class)
            .setParameter("corr", correlationId)
            .setMaxResults(1)
            .getResultStream()
            .findFirst();
}
```

- [ ] **Step 5: Run — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageLedgerEntryRepositoryTest -pl runtime
```
Expected: 4 tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteOutcome.java
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/test/java/io/casehub/qhorus/runtime/ledger/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): LedgerWriteOutcome + findByMessageId + findEarliestWithSubjectByCorrelationId

New repository queries for causedByEntryId auto-link (inReplyTo lookup)
and subjectId propagation (correlation root, sequenceNumber ASC ordering).

Refs #184"
```

---

## Task 3: `LedgerWriteService.record()` — new signature and propagation

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/StubMessageLedgerEntryRepository.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/LedgerWritePropagationTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java`

- [ ] **Step 1: Create `StubMessageLedgerEntryRepository`**

This stub is used in CDI-free propagation unit tests. It stores entries in-memory and supports the two new queries.

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/ledger/StubMessageLedgerEntryRepository.java
package io.casehub.qhorus.runtime.ledger;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;

/** In-memory stub of {@link MessageLedgerEntryRepository} for CDI-free unit tests. */
class StubMessageLedgerEntryRepository extends MessageLedgerEntryRepository {

    final List<MessageLedgerEntry> entries = new ArrayList<>();

    StubMessageLedgerEntryRepository() {
        // no CDI, no EntityManager needed — override every used method
    }

    @Override public LedgerEntry save(LedgerEntry entry) {
        entries.add((MessageLedgerEntry) entry);
        return entry;
    }

    @Override public Optional<LedgerEntry> findLatestBySubjectId(UUID subjectId) {
        return entries.stream()
                .filter(e -> subjectId.equals(e.subjectId))
                .max(Comparator.comparingInt(e -> e.sequenceNumber))
                .map(e -> (LedgerEntry) e);
    }

    @Override public Optional<MessageLedgerEntry> findByMessageId(Long messageId) {
        return entries.stream().filter(e -> messageId.equals(e.messageId)).findFirst();
    }

    @Override public Optional<MessageLedgerEntry> findEarliestWithSubjectByCorrelationId(String correlationId) {
        return entries.stream()
                .filter(e -> correlationId.equals(e.correlationId) && e.subjectId != null)
                .min(Comparator.comparingInt(e -> e.sequenceNumber));
    }

    @Override public Optional<LedgerEntry> findEntryById(UUID id) {
        return entries.stream().filter(e -> id.equals(e.id)).map(e -> (LedgerEntry) e).findFirst();
    }

    @Override public LedgerAttestation saveAttestation(LedgerAttestation a) { return a; }

    // remaining abstract methods — no-op stubs
    @Override public List<LedgerEntry> findBySubjectId(UUID s) { return List.of(); }
    @Override public List<LedgerEntry> listAll() { return List.of(); }
    @Override public List<LedgerEntry> findAllEvents() { return List.of(); }
    @Override public List<LedgerAttestation> findAttestationsByEntryId(UUID id) { return List.of(); }
    @Override public Map<UUID, List<LedgerAttestation>> findAttestationsForEntries(Set<UUID> ids) { return Map.of(); }
    @Override public List<LedgerAttestation> findAttestationsByEntryIdAndCapabilityTag(UUID id, String tag) { return List.of(); }
    @Override public List<LedgerAttestation> findAttestationsByEntryIdGlobal(UUID id) { return List.of(); }
    @Override public List<LedgerAttestation> findAttestationsByAttestorIdAndCapabilityTag(String a, String t) { return List.of(); }
    @Override public List<LedgerEntry> findByActorId(String a, Instant f, Instant t) { return List.of(); }
    @Override public List<LedgerEntry> findByActorRole(String r, Instant f, Instant t) { return List.of(); }
    @Override public List<LedgerEntry> findBySubjectIdAndTimeRange(UUID s, Instant f, Instant t) { return List.of(); }
    @Override public List<LedgerEntry> findByTimeRange(Instant f, Instant t) { return List.of(); }
    @Override public List<LedgerEntry> findCausedBy(UUID id) { return List.of(); }
}
```

- [ ] **Step 2: Write the failing propagation tests**

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/ledger/LedgerWritePropagationTest.java
package io.casehub.qhorus.runtime.ledger;

import static org.assertj.core.api.Assertions.*;

import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.CommitmentAttestationPolicy;
import io.casehub.qhorus.api.spi.InstanceActorIdProvider;

class LedgerWritePropagationTest {

    private StubMessageLedgerEntryRepository repository;
    private LedgerWriteService service;

    @BeforeEach void setUp() {
        repository = new StubMessageLedgerEntryRepository();
        service = new LedgerWriteService();
        service.repository = repository;
        service.config = () -> true;  // ledger enabled
        service.actorIdProvider = id -> id;
        service.attestationPolicy = (t, a) -> Optional.empty();
        service.objectMapper = new ObjectMapper();
    }

    // ── subjectId: Priority 1 (explicit caller value) ─────────────────────────

    @Test void subjectId_explicit_is_used_as_is() {
        UUID subject = UUID.randomUUID();
        UUID channel = UUID.randomUUID();
        MessageDispatch d = MessageDispatch.builder()
                .channelId(channel).sender("a").type(MessageType.COMMAND)
                .correlationId("c1").subjectId(subject).actorType(ActorType.AGENT).build();

        LedgerWriteOutcome outcome = service.record(d, 1L, null);

        assertThat(outcome.subjectId()).isEqualTo(subject);
        assertThat(repository.entries).hasSize(1);
        assertThat(repository.entries.get(0).subjectId).isEqualTo(subject);
    }

    // ── subjectId: Priority 2 (correlation root) ──────────────────────────────

    @Test void subjectId_inherits_from_correlation_root_when_not_explicit() {
        UUID channel = UUID.randomUUID();
        UUID rootSubject = UUID.randomUUID();

        // Pre-populate a COMMAND entry (the correlation root) with a subjectId
        MessageLedgerEntry root = MessageLedgerEntryTestFactory.entry(
                rootSubject, 1L, "COMMAND", channel, "corr-z");
        root.sequenceNumber = 1;
        repository.save(root);

        // Now dispatch a DONE without explicit subjectId — should inherit rootSubject
        MessageDispatch done = MessageDispatch.builder()
                .channelId(channel).sender("a").type(MessageType.DONE)
                .correlationId("corr-z").inReplyTo(1L).actorType(ActorType.AGENT).build();

        LedgerWriteOutcome outcome = service.record(done, 2L, null);

        assertThat(outcome.subjectId()).isEqualTo(rootSubject);
    }

    // ── subjectId: Priority 3 (channelId fallback) ───────────────────────────

    @Test void subjectId_falls_back_to_channelId_when_no_correlation_root() {
        UUID channel = UUID.randomUUID();
        MessageDispatch d = MessageDispatch.builder()
                .channelId(channel).sender("a").type(MessageType.EVENT)
                .actorType(ActorType.SYSTEM).build(); // no correlationId, no subjectId

        LedgerWriteOutcome outcome = service.record(d, 3L, null);

        assertThat(outcome.subjectId()).isEqualTo(channel); // fallback
    }

    // ── causedByEntryId: Priority 1 (explicit) ────────────────────────────────

    @Test void causedByEntryId_explicit_is_used_as_is() {
        UUID explicitCause = UUID.randomUUID();
        UUID channel = UUID.randomUUID();
        MessageDispatch d = MessageDispatch.builder()
                .channelId(channel).sender("a").type(MessageType.COMMAND)
                .correlationId("c2").causedByEntryId(explicitCause).actorType(ActorType.AGENT).build();

        LedgerWriteOutcome outcome = service.record(d, 4L, null);

        assertThat(outcome.causedByEntryId()).isEqualTo(explicitCause);
    }

    // ── causedByEntryId: Priority 2 (inReplyTo lookup) ───────────────────────

    @Test void causedByEntryId_auto_linked_from_inReplyTo_when_not_explicit() {
        UUID channel = UUID.randomUUID();
        UUID commandEntryId = UUID.randomUUID();

        // Pre-populate the COMMAND ledger entry (messageId = 10)
        MessageLedgerEntry commandEntry = MessageLedgerEntryTestFactory.entry(
                channel, 10L, "COMMAND", channel, "corr-y");
        commandEntry.id = commandEntryId;
        commandEntry.sequenceNumber = 1;
        repository.save(commandEntry);

        // DONE replies to COMMAND (inReplyTo = 10), no explicit causedByEntryId
        MessageDispatch done = MessageDispatch.builder()
                .channelId(channel).sender("a").type(MessageType.DONE)
                .correlationId("corr-y").inReplyTo(10L).actorType(ActorType.AGENT).build();

        LedgerWriteOutcome outcome = service.record(done, 11L, null);

        assertThat(outcome.causedByEntryId()).isEqualTo(commandEntryId);
    }

    // ── causedByEntryId: Priority 3 (null when no inReplyTo) ─────────────────

    @Test void causedByEntryId_is_null_when_no_inReplyTo_and_not_explicit() {
        UUID channel = UUID.randomUUID();
        MessageDispatch d = MessageDispatch.builder()
                .channelId(channel).sender("a").type(MessageType.COMMAND)
                .correlationId("c3").actorType(ActorType.AGENT).build();

        LedgerWriteOutcome outcome = service.record(d, 5L, null);

        assertThat(outcome.causedByEntryId()).isNull();
    }

    // ── Disabled ledger returns DISABLED sentinel ─────────────────────────────

    @Test void record_returns_disabled_sentinel_when_ledger_off() {
        service.config = () -> false; // disable ledger
        UUID channel = UUID.randomUUID();
        MessageDispatch d = MessageDispatch.builder()
                .channelId(channel).sender("a").type(MessageType.COMMAND)
                .actorType(ActorType.AGENT).build();

        LedgerWriteOutcome outcome = service.record(d, 6L, null);

        assertThat(outcome).isSameAs(LedgerWriteOutcome.DISABLED);
        assertThat(repository.entries).isEmpty();
    }
}
```

- [ ] **Step 3: Run — expect compile failure** (`record(MessageDispatch, Long, @Nullable UUID)` doesn't exist)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerWritePropagationTest -pl runtime 2>&1 | tail -20
```
Expected: compile error.

- [ ] **Step 4: Refactor `LedgerWriteService.record()`**

Replace the existing `record(Channel ch, Message message)` method entirely:

```java
/**
 * Record the given dispatch as an immutable ledger entry.
 *
 * <p>Runs in its own transaction (REQUIRES_NEW). Caller must extract all values from JPA
 * entities BEFORE calling — no entities cross this transaction boundary.
 *
 * <p>subjectId resolution (priority): explicit dispatch.subjectId() > earliest entry in
 * correlationId thread with non-null subject > dispatch.channelId() fallback.
 *
 * <p>causedByEntryId resolution (priority): explicit dispatch.causedByEntryId() > ledger
 * entry of the inReplyTo message > null.
 *
 * @return LedgerWriteOutcome with resolved entryId, subjectId, and causedByEntryId;
 *         or LedgerWriteOutcome.DISABLED when ledger writes are suppressed.
 */
@Transactional(value = Transactional.TxType.REQUIRES_NEW)
public LedgerWriteOutcome record(final MessageDispatch dispatch,
        final Long messageId,
        @Nullable final UUID commitmentId) {
    if (!config.enabled()) {
        return LedgerWriteOutcome.DISABLED;
    }

    // ── Resolve subjectId (Priority 1 > 2 > 3) ───────────────────────────────
    UUID resolvedSubjectId;
    if (dispatch.subjectId() != null) {
        resolvedSubjectId = dispatch.subjectId();
    } else if (dispatch.correlationId() != null) {
        resolvedSubjectId = repository
                .findEarliestWithSubjectByCorrelationId(dispatch.correlationId())
                .map(e -> e.subjectId)
                .orElse(dispatch.channelId());
    } else {
        resolvedSubjectId = dispatch.channelId();
    }

    // ── Resolve causedByEntryId (Priority 1 > 2 > null) ─────────────────────
    UUID resolvedCausedByEntryId;
    if (dispatch.causedByEntryId() != null) {
        resolvedCausedByEntryId = dispatch.causedByEntryId();
    } else if (dispatch.inReplyTo() != null) {
        resolvedCausedByEntryId = repository.findByMessageId(dispatch.inReplyTo())
                .map(e -> e.id)
                .orElse(null);
    } else {
        resolvedCausedByEntryId = null;
    }

    // ── Sequence number (per resolved subject chain) ──────────────────────────
    final UUID subjectForSeq = resolvedSubjectId;
    final int sequenceNumber = repository.findLatestBySubjectId(subjectForSeq)
            .map(e -> e.sequenceNumber + 1).orElse(1);

    final String resolvedActorId = actorIdProvider.resolve(dispatch.sender());

    final MessageLedgerEntry entry = new MessageLedgerEntry();
    entry.subjectId = resolvedSubjectId;
    entry.channelId = dispatch.channelId();
    entry.messageId = messageId;
    entry.commitmentId = commitmentId;
    entry.causedByEntryId = resolvedCausedByEntryId;
    entry.messageType = dispatch.type().name();
    entry.target = dispatch.target();
    entry.correlationId = dispatch.correlationId();
    entry.actorId = resolvedActorId;
    entry.actorType = dispatch.actorType();
    entry.occurredAt = java.time.Instant.now().truncatedTo(java.time.temporal.ChronoUnit.MILLIS);
    entry.sequenceNumber = sequenceNumber;
    entry.entryType = switch (dispatch.type()) {
        case QUERY, COMMAND, HANDOFF -> io.casehub.ledger.api.model.LedgerEntryType.COMMAND;
        default -> io.casehub.ledger.api.model.LedgerEntryType.EVENT;
    };

    if (dispatch.type() == MessageType.EVENT) {
        populateTelemetry(entry, dispatch.content());
    } else {
        entry.content = dispatch.content();
    }

    // ── Attestation for terminal commitment types ─────────────────────────────
    if (ATTESTATION_TYPES.contains(dispatch.type()) && resolvedCausedByEntryId != null) {
        repository.findEntryById(resolvedCausedByEntryId).ifPresent(prior ->
                writeAttestation(resolvedSubjectId, (MessageLedgerEntry) prior,
                        dispatch.type(), resolvedActorId));
    }

    repository.save(entry);
    return new LedgerWriteOutcome(entry.id, resolvedSubjectId, resolvedCausedByEntryId);
}

private void writeAttestation(final UUID subjectId, final MessageLedgerEntry commandEntry,
        final MessageType terminalType, final String resolvedActorId) {
    attestationPolicy.attestationFor(terminalType, resolvedActorId).ifPresent(outcome -> {
        try {
            final io.casehub.ledger.runtime.model.LedgerAttestation attestation =
                    new io.casehub.ledger.runtime.model.LedgerAttestation();
            attestation.ledgerEntryId = commandEntry.id;
            attestation.subjectId = subjectId;
            attestation.attestorId = outcome.attestorId();
            attestation.attestorType = outcome.attestorType();
            attestation.verdict = outcome.verdict();
            attestation.confidence = outcome.confidence();
            attestation.capabilityTag = extractCapabilityTag(commandEntry.content);
            repository.saveAttestation(attestation);
        } catch (final Exception e) {
            LOG.warnf("Could not write attestation for entry %s — trust signal lost",
                    commandEntry.id);
        }
    });
}
```

Also add to imports: `import io.casehub.qhorus.api.message.MessageDispatch;` and `import jakarta.annotation.Nullable;`

Remove the old `record(Channel, Message)` method and the `CAUSAL_TYPES` constant (no longer needed — causation is resolved via inReplyTo lookup, not type-based filtering).

- [ ] **Step 5: Run — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerWritePropagationTest -pl runtime
```
Expected: 7 tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): LedgerWriteService — new record() signature with subjectId/causedByEntryId propagation

Signature: record(MessageDispatch, Long messageId, @Nullable UUID commitmentId)
Returns LedgerWriteOutcome (entryId, resolvedSubjectId, resolvedCausedByEntryId).
No JPA entities cross the REQUIRES_NEW boundary.

subjectId: explicit > correlation root lookup > channelId fallback
causedByEntryId: explicit > inReplyTo ledger entry lookup > null
Per-subject Merkle chains (sequenceNumber scoped to resolved subject).

Refs #184"
```

---

## Task 4: `MessageService.dispatch()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchIntegrationTest.java`

- [ ] **Step 1: Write the failing integration test**

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchIntegrationTest.java
package io.casehub.qhorus.runtime.message;

import static org.assertj.core.api.Assertions.*;

import java.util.UUID;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.store.ChannelStore;

@QuarkusTest
class MessageDispatchIntegrationTest {

    @Inject MessageService messageService;
    @Inject ChannelStore channelStore;
    @Inject ChannelService channelService;

    @Test @TestTransaction
    void dispatch_command_returns_DispatchResult_with_messageId() {
        UUID channelId = createChannel("dispatch-test-" + UUID.randomUUID());

        MessageDispatch d = MessageDispatch.builder()
                .channelId(channelId).sender("agent-1").type(MessageType.COMMAND)
                .content("analyse this").correlationId("corr-1")
                .actorType(ActorType.AGENT).build();

        DispatchResult result = messageService.dispatch(d);

        assertThat(result.messageId()).isNotNull();
        assertThat(result.channelId()).isEqualTo(channelId);
        assertThat(result.sender()).isEqualTo("agent-1");
        assertThat(result.type()).isEqualTo(MessageType.COMMAND);
        assertThat(result.correlationId()).isEqualTo("corr-1");
    }

    @Test @TestTransaction
    void dispatch_with_explicit_subjectId_echoes_resolved_subjectId() {
        UUID channelId = createChannel("subject-test-" + UUID.randomUUID());
        UUID subject = UUID.randomUUID();

        DispatchResult result = messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId).sender("agent-1").type(MessageType.COMMAND)
                .content("work").correlationId("corr-2").subjectId(subject)
                .actorType(ActorType.AGENT).build());

        assertThat(result.subjectId()).isEqualTo(subject);
    }

    @Test @TestTransaction
    void dispatch_done_inherits_subjectId_from_command() {
        UUID channelId = createChannel("inherit-test-" + UUID.randomUUID());
        UUID subject = UUID.randomUUID();

        DispatchResult command = messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId).sender("orchestrator").type(MessageType.COMMAND)
                .content("do task").correlationId("corr-3").subjectId(subject)
                .actorType(ActorType.SYSTEM).build());

        DispatchResult done = messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId).sender("worker").type(MessageType.DONE)
                .content("done").correlationId("corr-3").inReplyTo(command.messageId())
                .actorType(ActorType.AGENT).build());

        // subjectId propagated from COMMAND via correlation root lookup
        assertThat(done.subjectId()).isEqualTo(subject);
        // causedByEntryId auto-linked from COMMAND's ledger entry
        assertThat(done.causedByEntryId()).isEqualTo(command.ledgerEntryId());
    }

    @Test @TestTransaction
    void dispatch_event_without_correlationId_falls_back_to_channelId_as_subject() {
        UUID channelId = createChannel("event-test-" + UUID.randomUUID());

        DispatchResult result = messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId).sender("system").type(MessageType.EVENT)
                .content("{\"tool_name\":\"test\"}").actorType(ActorType.SYSTEM).build());

        assertThat(result.subjectId()).isEqualTo(channelId);
        assertThat(result.causedByEntryId()).isNull();
    }

    private UUID createChannel(String name) {
        io.casehub.qhorus.runtime.channel.Channel ch = new io.casehub.qhorus.runtime.channel.Channel();
        ch.id = UUID.randomUUID();
        ch.name = name;
        ch.semantic = ChannelSemantic.APPEND;
        channelStore.put(ch);
        return ch.id;
    }
}
```

- [ ] **Step 2: Run — expect compile failure** (`dispatch()` does not exist on `MessageService`)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageDispatchIntegrationTest -pl runtime 2>&1 | tail -20
```
Expected: compile error.

- [ ] **Step 3: Add `dispatch()` to `MessageService`, remove `send()`**

In `MessageService.java`:

1. Add `@Inject LedgerWriteService ledgerWriteService;`
2. Add the following new method (and DELETE `send()`):

```java
@Transactional
public DispatchResult dispatch(final MessageDispatch dispatch) {
    final io.casehub.qhorus.runtime.channel.Channel ch =
            channelService.findById(dispatch.channelId()).orElse(null);
    if (ch != null) messageTypePolicy.validate(ch, dispatch.type());

    // Generate commitmentId before persisting so it's stored on the message entity
    final UUID commitmentId = (dispatch.correlationId() != null &&
            (dispatch.type() == MessageType.COMMAND || dispatch.type() == MessageType.QUERY))
            ? UUID.randomUUID() : null;

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

    // Extract primitives before REQUIRES_NEW boundary — no JPA entities cross it
    final Long messageId = message.id;
    final UUID storedCommitmentId = message.commitmentId;

    // Commitment state machine
    if (dispatch.correlationId() != null) {
        switch (dispatch.type()) {
            case QUERY, COMMAND -> commitmentService.open(
                    storedCommitmentId != null ? storedCommitmentId : UUID.randomUUID(),
                    dispatch.correlationId(), dispatch.channelId(), dispatch.type(),
                    dispatch.sender(), dispatch.target(), message.deadline);
            case STATUS -> commitmentService.acknowledge(dispatch.correlationId());
            case RESPONSE, DONE -> commitmentService.fulfill(dispatch.correlationId());
            case DECLINE -> commitmentService.decline(dispatch.correlationId());
            case FAILURE -> commitmentService.fail(dispatch.correlationId());
            case HANDOFF -> commitmentService.delegate(dispatch.correlationId(), dispatch.target());
            case EVENT -> { /* no commitment effect */ }
        }
    }

    if (dispatch.inReplyTo() != null) {
        messageStore.find(dispatch.inReplyTo()).ifPresent(parent -> parent.replyCount++);
    }

    channelService.updateLastActivity(dispatch.channelId());

    // Ledger write (REQUIRES_NEW — commits independently; failure rolls back outer tx)
    final LedgerWriteOutcome ledgerOutcome =
            ledgerWriteService.record(dispatch, messageId, storedCommitmentId);

    // Observer fan-out
    MessageObserverDispatcher.dispatch(
            ch != null ? ch.name : null, dispatch.channelId(), message, observers.handles());

    return new DispatchResult(
            messageId,
            dispatch.channelId(),
            dispatch.sender(),
            dispatch.type(),
            dispatch.correlationId(),
            dispatch.inReplyTo(),
            DispatchResult.parseArtefactRefs(dispatch.artefactRefs()),
            dispatch.target(),
            ledgerOutcome.entryId(),
            ledgerOutcome.subjectId(),
            ledgerOutcome.causedByEntryId());
}
```

Add imports: `import io.casehub.qhorus.api.message.MessageDispatch;`, `import io.casehub.qhorus.api.message.DispatchResult;`, `import io.casehub.qhorus.runtime.ledger.LedgerWriteService;`, `import io.casehub.qhorus.runtime.ledger.LedgerWriteOutcome;`

- [ ] **Step 4: Run integration tests — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageDispatchIntegrationTest -pl runtime
```
Expected: 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): MessageService.dispatch() — replaces send()

dispatch() is @Transactional; ledger write (REQUIRES_NEW) moves from
QhorusMcpTools into MessageService. commitmentId assigned before
messageStore.put() so it lands on the stored message and crosses the
REQUIRES_NEW boundary as a primitive. Returns DispatchResult with
resolved ledgerEntryId, subjectId, causedByEntryId.

Refs #184"
```

---

## Task 5: Migrate `QhorusMcpTools.sendMessage()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

- [ ] **Step 1: Update `sendMessage()` in `QhorusMcpTools`**

The method keeps all its pre-flight checks (rate limit, ACL, LAST_WRITE, artefact validation, etc.). Changes:
- Add `subject_id` and `caused_by_entry_id` optional params
- Replace `messageService.send(...)` + `ledgerWriteService.record(...)` with `messageService.dispatch(MessageDispatch.builder()...build())`
- Remove the `ledgerWriteService` injection and its try/catch block
- Return `DispatchResult` instead of `MessageResult`
- Remove `parentReplyCount` lookup (not in `DispatchResult`)

Find and replace the `@Tool`-annotated `sendMessage` method signature (line ~507). The existing pre-flight block (paused check, read-only check, type validation, ACL check, rate limit, artefact validation, LAST_WRITE) stays unchanged. Only the core dispatch block and return change:

```java
// BEFORE (replace these lines, roughly line 652 onward):
//   Message msg = messageService.send(ch.id, sender, msgType, content, corrId, ...);
//   ... ledgerWriteService.record(ch, msg); ...
//   return new MessageResult(msg.id, ch.name, ...);

// AFTER — build dispatch and call dispatch():
DispatchResult dispatchResult = messageService.dispatch(
    MessageDispatch.builder()
        .channelId(ch.id)
        .sender(sender)
        .type(msgType)
        .content(content)
        .correlationId(corrId)
        .inReplyTo(inReplyTo)
        .artefactRefs(refsStr)
        .target(normalisedTarget)
        .subjectId(subjectId != null && !subjectId.isBlank()
                ? UUID.fromString(subjectId) : null)
        .causedByEntryId(causedByEntryId != null && !causedByEntryId.isBlank()
                ? UUID.fromString(causedByEntryId) : null)
        .actorType(resolvedActorType)
        .build());

Message msg = messageService.findById(dispatchResult.messageId()).orElseThrow();
```

Add the two new `@ToolArg` parameters to `sendMessage()`:
```java
@ToolArg(name = "subject_id", description = "UUID of the domain aggregate being processed (e.g. transaction ID, case ID). Auto-propagated from correlation root if omitted.", required = false) String subjectId,
@ToolArg(name = "caused_by_entry_id", description = "UUID of the ledger entry that caused this dispatch (cross-domain causal chain). Auto-linked from in_reply_to when omitted.", required = false) String causedByEntryId,
```

Add `subject_id` and `caused_by_entry_id` UUID parsing with error handling (parse inside the method after the existing `corrId` block):
```java
// Parse optional UUID params — fail early if malformed
UUID subjectIdUuid = null;
if (subjectId != null && !subjectId.isBlank()) {
    try { subjectIdUuid = UUID.fromString(subjectId); }
    catch (IllegalArgumentException e) {
        throw new IllegalArgumentException("subject_id is not a valid UUID: " + subjectId);
    }
}
UUID causedByEntryIdUuid = null;
if (causedByEntryId != null && !causedByEntryId.isBlank()) {
    try { causedByEntryIdUuid = UUID.fromString(causedByEntryId); }
    catch (IllegalArgumentException e) {
        throw new IllegalArgumentException("caused_by_entry_id is not a valid UUID: " + causedByEntryId);
    }
}
```

Update the fanOut and auto-release blocks to use `dispatchResult.messageId()` and `dispatchResult.correlationId()` instead of `msg.*`.

Change return type of `sendMessage()` to `DispatchResult` and remove `MessageResult` import.

Remove `@Inject LedgerWriteService ledgerWriteService;` from the class (the field and injection).

Change the 3 internal EVENT audit calls (`messageService.send(...)` at lines ~466, ~1290, ~1316) to use `dispatch()`:

```java
// e.g. line 466:
messageService.dispatch(MessageDispatch.builder()
    .channelId(ch.id).sender("system").type(MessageType.EVENT)
    .content(auditContent).actorType(ActorType.SYSTEM).build());
```

Also update the LAST_WRITE early-return at line ~642 — it now returns a `DispatchResult` built from the overwritten message:
```java
return new DispatchResult(last.id, ch.id, last.sender, last.messageType,
        last.correlationId, last.inReplyTo, DispatchResult.parseArtefactRefs(last.artefactRefs),
        last.target, null, null, null); // ledger fields null (no ledger write for LAST_WRITE overwrite)
```

- [ ] **Step 2: Build the runtime module to catch compilation errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | tail -30
```
Expected: compile succeeds. Fix any remaining `MessageResult` or `send()` references.

- [ ] **Step 3: Run the full runtime test suite — many tests will fail** (call sites not yet migrated)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -40
```
Expected: failures in tests that still call `messageService.send()` directly. Record which test classes fail.

- [ ] **Step 4: Commit what compiles**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): QhorusMcpTools.sendMessage() — dispatch(), DispatchResult, subject_id/caused_by_entry_id params

Ledger write removed from MCP layer (now in MessageService.dispatch()).
Internal audit EVENT sends migrated to dispatch().
LAST_WRITE overwrite returns DispatchResult with null ledger fields.

Refs #184"
```

---

## Task 6: Migrate internal production `send()` call sites

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/QhorusChannelBackend.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`

- [ ] **Step 1: Migrate `QhorusDashboardService`** (line 137 — sends a message)

Replace `messageService.send(ch.id, sender, type, content, corrId, inReplyTo, null, null, actorType)` with:
```java
messageService.dispatch(MessageDispatch.builder()
    .channelId(ch.id).sender(sender).type(type).content(content)
    .correlationId(corrId).inReplyTo(inReplyTo).actorType(actorType).build())
```

- [ ] **Step 2: Migrate `WatchdogEvaluationService`** (line 211 — sends STATUS)

```java
messageService.dispatch(MessageDispatch.builder()
    .channelId(notifChannel.get().id)
    .sender("system:watchdog").type(MessageType.STATUS)
    .content(alertContent).actorType(ActorType.SYSTEM).build());
```

- [ ] **Step 3: Migrate `QhorusChannelBackend`** (line 37)

```java
messageService.dispatch(MessageDispatch.builder()
    .channelId(channel.id()).sender(message.sender()).type(message.type())
    .content(message.content()).correlationId(/* extract from message */)
    .actorType(message.actorType()).build());
```

Check what fields `NormalisedMessage` provides and map them. `NormalisedMessage` has `type, content, senderInstanceId, correlationId, inReplyTo, artefactRefs, target` (from CLAUDE.md).

- [ ] **Step 4: Migrate `ChannelGateway`** (lines 175, 183)

Normalised inbound human messages:
```java
// line 175 — inbound normalised agent message
messageService.dispatch(MessageDispatch.builder()
    .channelId(channel.id()).sender(n.senderInstanceId()).type(n.type())
    .content(n.content()).correlationId(n.correlationId()).inReplyTo(n.inReplyTo())
    .artefactRefs(n.artefactRefs() != null ? String.join(",",
            n.artefactRefs().stream().map(UUID::toString).toList()) : null)
    .target(n.target()).actorType(ActorType.AGENT).build());

// line 183 — inbound human signal
messageService.dispatch(MessageDispatch.builder()
    .channelId(channel.id())
    .sender("human:" + signal.externalSenderId()).type(MessageType.QUERY)
    .content(signal.content()).correlationId(/* signal correlationId */)
    .actorType(ActorType.HUMAN).build());
```

- [ ] **Step 5: Build to confirm all compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | tail -20
```
Expected: clean build.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/dashboard/
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/gateway/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): migrate internal send() call sites to dispatch()

Dashboard, Watchdog, QhorusChannelBackend, ChannelGateway.
Refs #184"
```

---

## Task 7: Migrate test call sites (134 call sites, 28 files)

**Files:** All 28 test files listed below

The migration pattern is mechanical. For each `messageService.send(channelId, sender, type, content, correlationId, inReplyTo, artefactRefs, target, actorType)` call:

```java
// BEFORE:
messageService.send(channelId, "agent", MessageType.COMMAND, "content",
        "corr-1", null, null, null, ActorType.AGENT);

// AFTER:
messageService.dispatch(MessageDispatch.builder()
    .channelId(channelId).sender("agent").type(MessageType.COMMAND)
    .content("content").correlationId("corr-1").actorType(ActorType.AGENT).build());
```

**Latent protocol violations to fix (not suppress):** If any test does DONE without `correlationId` + `inReplyTo`, or HANDOFF without `correlationId` + `inReplyTo` + `target`, the builder will throw. Fix the test to use the correct protocol — add the missing fields. Do not suppress by setting null or catching the exception.

**Files to migrate** (run in this order — test files near the service first):

1. `runtime/src/test/java/io/casehub/qhorus/service/MessageServiceTest.java`
2. `runtime/src/test/java/io/casehub/qhorus/service/ReactiveMessageServiceTest.java`
3. `runtime/src/test/java/io/casehub/qhorus/message/MessageServiceTest.java`
4. `runtime/src/test/java/io/casehub/qhorus/message/MessageServiceTypeEnforcementTest.java`
5. `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayTest.java`
6. `runtime/src/test/java/io/casehub/qhorus/gateway/QhorusChannelBackendTest.java`
7. `runtime/src/test/java/io/casehub/qhorus/runtime/dashboard/QhorusDashboardServiceTest.java`
8. `runtime/src/test/java/io/casehub/qhorus/channel/ChannelServiceTest.java`
9. `runtime/src/test/java/io/casehub/qhorus/mcp/ChannelToolTest.java`
10. `runtime/src/test/java/io/casehub/qhorus/mcp/MessageOrderingTest.java`
11. `runtime/src/test/java/io/casehub/qhorus/mcp/LastWriteEdgeCaseTest.java`
12. `runtime/src/test/java/io/casehub/qhorus/mcp/LastWriteArtefactRefsTest.java`
13. `runtime/src/test/java/io/casehub/qhorus/mcp/BarrierConcurrentWriteTest.java`
14. `runtime/src/test/java/io/casehub/qhorus/mcp/DeleteChannelToolTest.java`
15. `runtime/src/test/java/io/casehub/qhorus/mcp/WaitForReplyCorrelationIsolationTest.java`
16. `runtime/src/test/java/io/casehub/qhorus/mcp/WaitForReplyTest.java`
17. `runtime/src/test/java/io/casehub/qhorus/mcp/WaitManagementTest.java`
18. `runtime/src/test/java/io/casehub/qhorus/mcp/WaitForReplyEdgeCaseTest.java`
19. `runtime/src/test/java/io/casehub/qhorus/mcp/GetMessageToolTest.java`
20. `runtime/src/test/java/io/casehub/qhorus/mcp/EphemeralEdgeCaseTest.java`
21. `runtime/src/test/java/io/casehub/qhorus/mcp/EphemeralDoubleDeliveryTest.java`
22. `runtime/src/test/java/io/casehub/qhorus/mcp/CollectAtomicityTest.java`
23. `runtime/src/test/java/io/casehub/qhorus/SmokeTest.java`
24. `runtime/src/test/java/io/casehub/qhorus/ReactiveSmokeTest.java`
25. `examples/normative-layout/src/test/java/io/casehub/qhorus/examples/normativelayout/NormativeLayoutTypeEnforcementTest.java`
26. `examples/normative-layout/src/test/java/io/casehub/qhorus/examples/normativelayout/NormativeLayoutObligationTest.java`
27. `examples/normative-layout/src/test/java/io/casehub/qhorus/examples/normativelayout/NormativeLayoutRobustnessTest.java`
28. `examples/normative-layout/src/test/java/io/casehub/qhorus/examples/normativelayout/SecureCodeReviewScenario.java`

- [ ] **Step 1: Migrate files 1–8 (service and gateway tests)**

Use IntelliJ Find & Replace or grep to locate all `messageService.send(` and `\.send(` usages. For each, apply the builder pattern. Add required imports: `import io.casehub.qhorus.api.message.MessageDispatch;`

- [ ] **Step 2: Run files 1–8**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="MessageServiceTest,ReactiveMessageServiceTest,MessageServiceTypeEnforcementTest,ChannelGatewayTest,QhorusChannelBackendTest,QhorusDashboardServiceTest,ChannelServiceTest" -pl runtime
```
Expected: all pass. Fix any latent protocol violations exposed by builder validation.

- [ ] **Step 3: Migrate files 9–24 (MCP and smoke tests)**

Same mechanical substitution for remaining test files.

- [ ] **Step 4: Run files 9–24**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all runtime tests pass (1494 tests).

- [ ] **Step 5: Migrate examples (files 25–28)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout
```

- [ ] **Step 6: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests 2>&1 | tail -20
```
Expected: BUILD SUCCESS.

- [ ] **Step 7: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```
Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): migrate all send() test call sites to dispatch()

134 call sites across 28 files. Latent protocol violations fixed (DONE/DECLINE
without correlationId, HANDOFF without target) rather than suppressed.

Refs #184"
```

---

## Task 8: Reactive parity

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveMessageLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Add two new queries to `ReactiveMessageLedgerEntryRepository`**

```java
// Add to ReactiveMessageLedgerEntryRepository
public Uni<Optional<MessageLedgerEntry>> findByMessageId(final Long messageId) {
    return Panache.withTransaction("qhorus", () ->
            MessageLedgerEntry.<MessageLedgerEntry>find("messageId", messageId)
                    .firstResultOptional());
}

public Uni<Optional<MessageLedgerEntry>> findEarliestWithSubjectByCorrelationId(
        final String correlationId) {
    return Panache.withTransaction("qhorus", () ->
            MessageLedgerEntry.<MessageLedgerEntry>find(
                    "correlationId = ?1 AND subjectId IS NOT NULL ORDER BY sequenceNumber ASC",
                    correlationId)
                    .firstResultOptional());
}
```

- [ ] **Step 2: Update `ReactiveLedgerWriteService.record()` — new signature**

Replace `record(Channel ch, Message message)` with:

```java
@Transactional(value = Transactional.TxType.REQUIRES_NEW)
public Uni<LedgerWriteOutcome> record(final MessageDispatch dispatch,
        final Long messageId,
        @Nullable final UUID commitmentId) {
    if (!config.enabled()) {
        return Uni.createFrom().item(LedgerWriteOutcome.DISABLED);
    }

    // Resolve subjectId — Priority 1 (explicit) > Priority 2 (correlation root) > Priority 3 (channelId)
    Uni<UUID> subjectIdUni;
    if (dispatch.subjectId() != null) {
        subjectIdUni = Uni.createFrom().item(dispatch.subjectId());
    } else if (dispatch.correlationId() != null) {
        subjectIdUni = reactiveRepository
                .findEarliestWithSubjectByCorrelationId(dispatch.correlationId())
                .map(opt -> opt.map(e -> e.subjectId).orElse(dispatch.channelId()));
    } else {
        subjectIdUni = Uni.createFrom().item(dispatch.channelId());
    }

    // Resolve causedByEntryId — Priority 1 (explicit) > Priority 2 (inReplyTo) > null
    Uni<UUID> causedByUni;
    if (dispatch.causedByEntryId() != null) {
        causedByUni = Uni.createFrom().item(dispatch.causedByEntryId());
    } else if (dispatch.inReplyTo() != null) {
        causedByUni = reactiveRepository.findByMessageId(dispatch.inReplyTo())
                .map(opt -> opt.map(e -> e.id).orElse(null));
    } else {
        causedByUni = Uni.createFrom().nullItem();
    }

    return Uni.combine().all().unis(subjectIdUni, causedByUni).asTuple()
            .chain(t -> {
                UUID resolvedSubject = t.getItem1();
                UUID resolvedCause = t.getItem2();
                return reactiveRepository.findLatestBySubjectId(resolvedSubject)
                        .map(latest -> latest.map(e -> e.sequenceNumber + 1).orElse(1))
                        .chain(seq -> {
                            final String resolvedActorId = actorIdProvider.resolve(dispatch.sender());
                            final MessageLedgerEntry entry = new MessageLedgerEntry();
                            entry.subjectId = resolvedSubject;
                            entry.channelId = dispatch.channelId();
                            entry.messageId = messageId;
                            entry.commitmentId = commitmentId;
                            entry.causedByEntryId = resolvedCause;
                            entry.messageType = dispatch.type().name();
                            entry.target = dispatch.target();
                            entry.correlationId = dispatch.correlationId();
                            entry.actorId = resolvedActorId;
                            entry.actorType = dispatch.actorType();
                            entry.occurredAt = java.time.Instant.now()
                                    .truncatedTo(java.time.temporal.ChronoUnit.MILLIS);
                            entry.sequenceNumber = seq;
                            entry.entryType = switch (dispatch.type()) {
                                case QUERY, COMMAND, HANDOFF ->
                                        io.casehub.ledger.api.model.LedgerEntryType.COMMAND;
                                default -> io.casehub.ledger.api.model.LedgerEntryType.EVENT;
                            };
                            if (dispatch.type() == MessageType.EVENT) {
                                populateTelemetry(entry, dispatch.content());
                            } else {
                                entry.content = dispatch.content();
                            }
                            return reactiveRepository.save(entry)
                                    .map(saved -> new LedgerWriteOutcome(
                                            saved.id, resolvedSubject, resolvedCause));
                        });
            });
}
```

Also add `import io.casehub.qhorus.api.message.MessageDispatch;` and remove the old `record(Channel, Message)` method.

- [ ] **Step 3: Update `ReactiveMessageService.dispatch()`**

Replace `send()` with:

```java
public Uni<DispatchResult> dispatch(final MessageDispatch dispatch) {
    return Panache.withTransaction("qhorus", () -> {
        final io.casehub.qhorus.runtime.channel.Channel ch =
                channelService.findById(dispatch.channelId()).orElse(null);
        if (ch != null) messageTypePolicy.validate(ch, dispatch.type());

        final UUID commitmentId = (dispatch.correlationId() != null &&
                (dispatch.type() == MessageType.COMMAND || dispatch.type() == MessageType.QUERY))
                ? UUID.randomUUID() : null;

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

        return messageStore.put(message)
                .chain(msg -> {
                    final Long msgId = msg.id;
                    final UUID storedCommitmentId = msg.commitmentId;

                    if (dispatch.correlationId() != null) {
                        switch (dispatch.type()) {
                            case QUERY, COMMAND -> commitmentService.open(
                                    storedCommitmentId != null ? storedCommitmentId : UUID.randomUUID(),
                                    dispatch.correlationId(), dispatch.channelId(), dispatch.type(),
                                    dispatch.sender(), dispatch.target(), msg.deadline);
                            case STATUS -> commitmentService.acknowledge(dispatch.correlationId());
                            case RESPONSE, DONE -> commitmentService.fulfill(dispatch.correlationId());
                            case DECLINE -> commitmentService.decline(dispatch.correlationId());
                            case FAILURE -> commitmentService.fail(dispatch.correlationId());
                            case HANDOFF -> commitmentService.delegate(
                                    dispatch.correlationId(), dispatch.target());
                            case EVENT -> { }
                        }
                    }

                    channelService.updateLastActivity(dispatch.channelId());

                    return reactiveLedgerWriteService.record(dispatch, msgId, storedCommitmentId)
                            .map(outcome -> new DispatchResult(
                                    msgId, dispatch.channelId(), dispatch.sender(), dispatch.type(),
                                    dispatch.correlationId(), dispatch.inReplyTo(),
                                    DispatchResult.parseArtefactRefs(dispatch.artefactRefs()),
                                    dispatch.target(), outcome.entryId(),
                                    outcome.subjectId(), outcome.causedByEntryId()));
                });
    });
}
```

- [ ] **Step 4: Update `ReactiveQhorusMcpTools.sendMessage()`**

Change return type to `Uni<DispatchResult>`. Add `subject_id` and `caused_by_entry_id` params (same as Task 5). Replace internal `send()` + `ledgerWriteService.record()` with `dispatch()`. Remove `ledgerWriteService` injection. Internal EVENT audit calls use `dispatch()` as in Task 5.

- [ ] **Step 5: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | tail -20
```
Expected: clean. Reactive integration tests remain `@Disabled` — Docker required.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#184): reactive parity — ReactiveMessageService.dispatch(), ReactiveLedgerWriteService

Full reactive implementation with Uni chains. Priority resolution for subjectId
and causedByEntryId mirrors blocking stack. @Disabled tests unchanged (Docker required).

Refs #184"
```

---

## Task 9: Final verification + `ToolOverloadDiscoverabilityTest`

- [ ] **Step 1: Extend `ToolOverloadDiscoverabilityTest`**

Find the existing test and add an assertion that no public non-`@Tool` method named `dispatch` exists on `QhorusMcpTools` or `ReactiveQhorusMcpTools`:

```java
// In ToolOverloadDiscoverabilityTest — add alongside existing send() checks:
@Test void dispatch_has_no_public_non_tool_overload() {
    var toolMethods = Arrays.stream(QhorusMcpTools.class.getMethods())
            .filter(m -> m.getName().equals("dispatch"))
            .filter(m -> m.isAnnotationPresent(io.quarkiverse.mcp.server.Tool.class))
            .count();
    var publicMethods = Arrays.stream(QhorusMcpTools.class.getMethods())
            .filter(m -> m.getName().equals("dispatch"))
            .count();
    // dispatch() is NOT a @Tool method (it's on MessageService, not QhorusMcpTools) — no overload risk
    assertThat(publicMethods).as("No public dispatch() on QhorusMcpTools").isEqualTo(0);
}
```

- [ ] **Step 2: Run the full build and all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -30
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 3: Confirm claudony#126 changes are staged** (from earlier in the session)

```bash
git -C /Users/mdproctor/claude/casehub/claudony status
```
Expected: `app/src/main/resources/application.properties` and `app/src/test/resources/application.properties` modified.

- [ ] **Step 4: Final commit for any remaining cleanup**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#184): ToolOverloadDiscoverabilityTest — guard dispatch() overload

Refs #184"
```

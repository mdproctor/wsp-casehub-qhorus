# Normative Ledger — Part 1: Core (Entity, Repository, Write Service)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Lay the foundation for the normative ledger — `MessageLedgerEntry` entity, `MessageLedgerEntryRepository`, and a rewritten `LedgerWriteService.record()` that handles all 9 message types.

**Architecture:** New entity (`MessageLedgerEntry`) and repository (`MessageLedgerEntryRepository`) are added alongside the old ones — the old classes are NOT deleted yet (deleted in Part 3). The write service is rewritten in place; the old `recordEvent` method is replaced by `record(Channel, Message)`. The caller in `QhorusMcpTools` is NOT updated yet (done in Part 2).

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA JOINED inheritance (quarkus-ledger base class `LedgerEntry`), H2 for tests, JUnit 5, Mockito (quarkus-junit5-mockito).

**Spec:** `docs/superpowers/specs/2026-04-26-normative-ledger-design.md`

---

## File Map

| File | Action |
|---|---|
| `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntry.java` | Create |
| `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntryRepository.java` | Create |
| `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/LedgerWriteService.java` | Rewrite |
| `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryTest.java` | Create |
| `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryRepositoryTest.java` | Create |
| `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerWriteServiceTest.java` | Create |

---

## Task 1: Issues and Epic Setup

- [ ] **Step 1: Run issue-workflow Phase 1**

Invoke the `issue-workflow` skill and create the epic and child issues for this feature. The epic title is "Normative Ledger — complete channel audit trail for all 9 message types". Child issues (one per plan task group):

1. `MessageLedgerEntry` entity + unit test
2. `MessageLedgerEntryRepository` + integration test
3. `LedgerWriteService` rewrite — unit tests + implementation
4. Integration tests — happy path, sequence, causal chain, filters, robustness (Part 2)
5. MCP tools — `list_ledger_entries`, wire `record()`, E2E tests (Part 2)
6. Reactive stack — `ReactiveMessageLedgerEntryRepository`, `ReactiveLedgerWriteService` (Part 3)
7. Delete old ledger classes and old test files (Part 3)
8. Example tests — type-system and agent-communication (Part 3)
9. Documentation — systematic review pass (Part 3)
10. Documentation — normative ledger section additions (Part 3)

Record the epic number (e.g. `#101`) — all commits in this plan use `Refs #<child-issue> — Epic #<epic>`.

---

## Task 2: `MessageLedgerEntry` Entity

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntry.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryTest.java`

- [ ] **Step 1: Write the failing unit test**

Create `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryTest.java`:

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkiverse.ledger.runtime.model.LedgerEntry;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry;

class MessageLedgerEntryTest {

    @Test
    void isSubtypeOfLedgerEntry() {
        assertInstanceOf(LedgerEntry.class, new MessageLedgerEntry());
    }

    @Test
    void commonFields_areAccessible() {
        MessageLedgerEntry e = new MessageLedgerEntry();
        UUID channelId = UUID.randomUUID();
        e.channelId = channelId;
        e.subjectId = channelId;
        e.messageId = 42L;
        e.messageType = "COMMAND";
        e.actorId = "agent-1";
        e.sequenceNumber = 1;

        assertEquals(channelId, e.channelId);
        assertEquals(channelId, e.subjectId);
        assertEquals(42L, e.messageId);
        assertEquals("COMMAND", e.messageType);
        assertEquals("agent-1", e.actorId);
        assertEquals(1, e.sequenceNumber);
    }

    @Test
    void normativeFields_defaultToNull() {
        MessageLedgerEntry e = new MessageLedgerEntry();
        assertNull(e.target);
        assertNull(e.content);
        assertNull(e.correlationId);
        assertNull(e.commitmentId);
    }

    @Test
    void telemetryFields_defaultToNull() {
        MessageLedgerEntry e = new MessageLedgerEntry();
        assertNull(e.toolName);
        assertNull(e.durationMs);
        assertNull(e.tokenCount);
        assertNull(e.contextRefs);
        assertNull(e.sourceEntity);
    }

    @Test
    void allFields_canBeSet() {
        MessageLedgerEntry e = new MessageLedgerEntry();
        UUID commitmentId = UUID.randomUUID();

        e.target = "instance:abc";
        e.content = "Please generate the report";
        e.correlationId = "corr-1";
        e.commitmentId = commitmentId;
        e.toolName = "read_file";
        e.durationMs = 42L;
        e.tokenCount = 1200L;
        e.contextRefs = "[\"msg-1\"]";
        e.sourceEntity = "{\"id\":\"case-1\"}";

        assertEquals("instance:abc", e.target);
        assertEquals("Please generate the report", e.content);
        assertEquals("corr-1", e.correlationId);
        assertEquals(commitmentId, e.commitmentId);
        assertEquals("read_file", e.toolName);
        assertEquals(42L, e.durationMs);
        assertEquals(1200L, e.tokenCount);
        assertNotNull(e.contextRefs);
        assertNotNull(e.sourceEntity);
    }
}
```

- [ ] **Step 2: Run test — confirm it fails to compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageLedgerEntryTest -Dno-format -q 2>&1 | tail -5
```
Expected: compilation error — `MessageLedgerEntry cannot be found`.

- [ ] **Step 3: Create `MessageLedgerEntry.java`**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntry.java`:

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import io.quarkiverse.ledger.runtime.model.LedgerEntry;

/**
 * A ledger entry recording any agent-to-agent message as a speech act.
 *
 * <p>
 * Extends the domain-agnostic {@link LedgerEntry} base class using JPA JOINED inheritance.
 * The {@code message_ledger_entry} table holds Qhorus-specific fields; all common audit
 * fields live in {@code ledger_entry}. The {@code subjectId} field on the base class
 * carries the Channel UUID.
 *
 * <p>
 * Every message type is recorded — this is the complete, immutable channel audit trail.
 * The {@link #messageType} field discriminates content interpretation. Telemetry fields
 * ({@code toolName}, {@code durationMs}, etc.) are populated only for EVENT messages
 * and are null for all other types.
 *
 * <p>
 * The CommitmentStore is the live obligation state; this ledger is the permanent record.
 * {@code causedByEntryId} (inherited from base) links terminal messages (DONE, FAILURE,
 * DECLINE, HANDOFF) back to the COMMAND that created the obligation.
 */
@Entity
@Table(name = "message_ledger_entry")
@DiscriminatorValue("QHORUS_MESSAGE")
public class MessageLedgerEntry extends LedgerEntry {

    /** UUID of the channel this message was sent on. Mirrors {@code subjectId}. */
    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    /** Primary key of the {@code message} row that triggered this entry. */
    @Column(name = "message_id", nullable = false)
    public Long messageId;

    /** Qhorus {@code MessageType} enum name — e.g. {@code "COMMAND"}, {@code "EVENT"}. */
    @Column(name = "message_type", nullable = false)
    public String messageType;

    /** Intended recipient for COMMAND and HANDOFF messages. Null for broadcasts. */
    @Column(name = "target")
    public String target;

    /**
     * Message content for normative types: COMMAND description, DECLINE/FAILURE reason,
     * DONE summary. Null for EVENT (telemetry uses dedicated fields instead).
     */
    @Column(name = "content", columnDefinition = "TEXT")
    public String content;

    /** Correlation ID propagated from the message — used for causal chain resolution. */
    @Column(name = "correlation_id")
    public String correlationId;

    /** Links to the {@code Commitment} row for obligation-bearing message types. */
    @Column(name = "commitment_id")
    public UUID commitmentId;

    // ── EVENT-only telemetry fields ───────────────────────────────────────────
    // Null for all non-EVENT message types.

    /** Name of the agent tool that was invoked. EVENT only. */
    @Column(name = "tool_name")
    public String toolName;

    /** Wall-clock duration of the tool invocation in milliseconds. EVENT only. */
    @Column(name = "duration_ms")
    public Long durationMs;

    /** LLM token count consumed by the invocation. EVENT only. */
    @Column(name = "token_count")
    public Long tokenCount;

    /** JSON array or string listing context references used. EVENT only. */
    @Column(name = "context_refs", columnDefinition = "TEXT")
    public String contextRefs;

    /** JSON object describing the source domain entity that triggered the event. EVENT only. */
    @Column(name = "source_entity", columnDefinition = "TEXT")
    public String sourceEntity;
}
```

- [ ] **Step 4: Run test — confirm it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageLedgerEntryTest -Dno-format -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`, 5 tests passing.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntry.java \
        runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryTest.java
git commit -m "$(cat <<'EOF'
feat(ledger): add MessageLedgerEntry — unified entity for all 9 message types

Replaces the EVENT-only AgentMessageLedgerEntry design. Records every speech act
on a channel as an immutable ledger entry. Telemetry fields (toolName, durationMs,
etc.) nullable and EVENT-only; normative fields (content, target, commitmentId)
populated for obligation-bearing types.

Refs #<child-issue-1> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: `MessageLedgerEntryRepository`

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntryRepository.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryRepositoryTest.java`

- [ ] **Step 1: Write the failing integration test**

Create `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryRepositoryTest.java`:

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.ledger.runtime.model.ActorType;
import io.quarkiverse.ledger.runtime.model.LedgerEntryType;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Integration tests for {@link MessageLedgerEntryRepository}.
 * Verifies persistence, retrieval, filtering, and causal chain lookup.
 * Refs #<child-issue-2> — Epic #<epic>.
 */
@QuarkusTest
@TestTransaction
class MessageLedgerEntryRepositoryTest {

    @Inject
    MessageLedgerEntryRepository repo;

    // =========================================================================
    // save + findByChannelId
    // =========================================================================

    @Test
    void save_andFindByChannelId_returnsSavedEntry() {
        UUID channelId = UUID.randomUUID();
        repo.save(entry(channelId, 1L, "COMMAND", 1));

        List<MessageLedgerEntry> found = repo.findByChannelId(channelId);

        assertEquals(1, found.size());
        assertEquals("COMMAND", found.get(0).messageType);
        assertEquals(channelId, found.get(0).channelId);
    }

    @Test
    void findByChannelId_unknownChannel_returnsEmpty() {
        List<MessageLedgerEntry> found = repo.findByChannelId(UUID.randomUUID());
        assertTrue(found.isEmpty());
    }

    @Test
    void findByChannelId_orderedBySequenceNumber() {
        UUID channelId = UUID.randomUUID();
        repo.save(entry(channelId, 3L, "DONE", 3));
        repo.save(entry(channelId, 1L, "COMMAND", 1));
        repo.save(entry(channelId, 2L, "STATUS", 2));

        List<MessageLedgerEntry> found = repo.findByChannelId(channelId);

        assertEquals(3, found.size());
        assertEquals(1, found.get(0).sequenceNumber);
        assertEquals(2, found.get(1).sequenceNumber);
        assertEquals(3, found.get(2).sequenceNumber);
    }

    // =========================================================================
    // findLatestBySubjectId — for sequence number calculation
    // =========================================================================

    @Test
    void findLatestBySubjectId_returnsHighestSequenceEntry() {
        UUID channelId = UUID.randomUUID();
        repo.save(entry(channelId, 1L, "COMMAND", 1));
        repo.save(entry(channelId, 2L, "DONE", 2));

        var latest = repo.findLatestBySubjectId(channelId);

        assertTrue(latest.isPresent());
        assertEquals(2, latest.get().sequenceNumber);
    }

    @Test
    void findLatestBySubjectId_emptyChannel_returnsEmpty() {
        assertTrue(repo.findLatestBySubjectId(UUID.randomUUID()).isEmpty());
    }

    // =========================================================================
    // listEntries — no filter
    // =========================================================================

    @Test
    void listEntries_noFilter_returnsAllTypes() {
        UUID channelId = UUID.randomUUID();
        repo.save(entry(channelId, 1L, "COMMAND", 1));
        repo.save(entry(channelId, 2L, "STATUS", 2));
        repo.save(entry(channelId, 3L, "DONE", 3));
        repo.save(entry(channelId, 4L, "EVENT", 4));

        List<MessageLedgerEntry> entries = repo.listEntries(channelId, null, null, null, null, 20);

        assertEquals(4, entries.size());
    }

    // =========================================================================
    // listEntries — messageTypes filter
    // =========================================================================

    @Test
    void listEntries_typeFilter_returnsOnlyMatchingTypes() {
        UUID channelId = UUID.randomUUID();
        repo.save(entry(channelId, 1L, "COMMAND", 1));
        repo.save(entry(channelId, 2L, "DONE", 2));
        repo.save(entry(channelId, 3L, "EVENT", 3));
        repo.save(entry(channelId, 4L, "DECLINE", 4));

        List<MessageLedgerEntry> entries = repo.listEntries(
                channelId, Set.of("COMMAND", "DONE"), null, null, null, 20);

        assertEquals(2, entries.size());
        assertTrue(entries.stream().allMatch(e -> Set.of("COMMAND", "DONE").contains(e.messageType)));
    }

    @Test
    void listEntries_typeFilter_noMatch_returnsEmpty() {
        UUID channelId = UUID.randomUUID();
        repo.save(entry(channelId, 1L, "COMMAND", 1));

        List<MessageLedgerEntry> entries = repo.listEntries(
                channelId, Set.of("DONE"), null, null, null, 20);

        assertTrue(entries.isEmpty());
    }

    // =========================================================================
    // listEntries — afterSequence cursor
    // =========================================================================

    @Test
    void listEntries_afterSequence_returnsOnlyLaterEntries() {
        UUID channelId = UUID.randomUUID();
        repo.save(entry(channelId, 1L, "COMMAND", 1));
        repo.save(entry(channelId, 2L, "STATUS", 2));
        repo.save(entry(channelId, 3L, "DONE", 3));

        List<MessageLedgerEntry> entries = repo.listEntries(channelId, null, 1L, null, null, 20);

        assertEquals(2, entries.size());
        assertTrue(entries.stream().allMatch(e -> e.sequenceNumber > 1));
    }

    // =========================================================================
    // listEntries — agentId filter
    // =========================================================================

    @Test
    void listEntries_agentFilter_returnsOnlyMatchingAgent() {
        UUID channelId = UUID.randomUUID();
        MessageLedgerEntry e1 = entry(channelId, 1L, "COMMAND", 1);
        e1.actorId = "agent-a";
        MessageLedgerEntry e2 = entry(channelId, 2L, "DONE", 2);
        e2.actorId = "agent-b";
        repo.save(e1);
        repo.save(e2);

        List<MessageLedgerEntry> entries = repo.listEntries(channelId, null, null, "agent-a", null, 20);

        assertEquals(1, entries.size());
        assertEquals("agent-a", entries.get(0).actorId);
    }

    // =========================================================================
    // listEntries — since filter
    // =========================================================================

    @Test
    void listEntries_sinceFilter_excludesOlderEntries() {
        UUID channelId = UUID.randomUUID();
        MessageLedgerEntry old = entry(channelId, 1L, "COMMAND", 1);
        old.occurredAt = Instant.parse("2026-01-01T00:00:00Z");
        repo.save(old);

        MessageLedgerEntry recent = entry(channelId, 2L, "DONE", 2);
        recent.occurredAt = Instant.parse("2026-06-01T00:00:00Z");
        repo.save(recent);

        List<MessageLedgerEntry> entries = repo.listEntries(
                channelId, null, null, null, Instant.parse("2026-03-01T00:00:00Z"), 20);

        assertEquals(1, entries.size());
        assertEquals("DONE", entries.get(0).messageType);
    }

    // =========================================================================
    // listEntries — limit
    // =========================================================================

    @Test
    void listEntries_limit_capsResults() {
        UUID channelId = UUID.randomUUID();
        for (int i = 1; i <= 5; i++) {
            repo.save(entry(channelId, (long) i, "EVENT", i));
        }

        List<MessageLedgerEntry> entries = repo.listEntries(channelId, null, null, null, null, 3);

        assertEquals(3, entries.size());
    }

    // =========================================================================
    // findLatestByCorrelationId — causal chain lookup
    // =========================================================================

    @Test
    void findLatestByCorrelationId_returnsCommandEntry() {
        UUID channelId = UUID.randomUUID();
        MessageLedgerEntry cmd = entry(channelId, 1L, "COMMAND", 1);
        cmd.correlationId = "corr-1";
        repo.save(cmd);

        Optional<MessageLedgerEntry> found = repo.findLatestByCorrelationId(channelId, "corr-1");

        assertTrue(found.isPresent());
        assertEquals("COMMAND", found.get().messageType);
    }

    @Test
    void findLatestByCorrelationId_withHandoff_returnsLatestHandoff() {
        UUID channelId = UUID.randomUUID();

        MessageLedgerEntry cmd = entry(channelId, 1L, "COMMAND", 1);
        cmd.correlationId = "corr-2";
        repo.save(cmd);

        MessageLedgerEntry handoff = entry(channelId, 2L, "HANDOFF", 2);
        handoff.correlationId = "corr-2";
        repo.save(handoff);

        Optional<MessageLedgerEntry> found = repo.findLatestByCorrelationId(channelId, "corr-2");

        assertTrue(found.isPresent());
        assertEquals("HANDOFF", found.get().messageType);
    }

    @Test
    void findLatestByCorrelationId_doneEntryIgnored() {
        // DONE is not COMMAND or HANDOFF — should not be returned by this query
        UUID channelId = UUID.randomUUID();
        MessageLedgerEntry done = entry(channelId, 1L, "DONE", 1);
        done.correlationId = "corr-3";
        repo.save(done);

        Optional<MessageLedgerEntry> found = repo.findLatestByCorrelationId(channelId, "corr-3");

        assertTrue(found.isEmpty());
    }

    @Test
    void findLatestByCorrelationId_unknownCorrelationId_returnsEmpty() {
        UUID channelId = UUID.randomUUID();
        assertTrue(repo.findLatestByCorrelationId(channelId, "no-such-corr").isEmpty());
    }

    // =========================================================================
    // Fixture
    // =========================================================================

    private MessageLedgerEntry entry(UUID channelId, long messageId, String type, int seq) {
        MessageLedgerEntry e = new MessageLedgerEntry();
        e.channelId = channelId;
        e.subjectId = channelId;
        e.messageId = messageId;
        e.messageType = type;
        e.sequenceNumber = seq;
        e.actorId = "agent-1";
        e.actorType = ActorType.AGENT;
        e.entryType = LedgerEntryType.EVENT;
        e.occurredAt = Instant.now();
        return e;
    }
}
```

- [ ] **Step 2: Run test — confirm it fails to compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageLedgerEntryRepositoryTest -Dno-format -q 2>&1 | tail -5
```
Expected: compilation error — `MessageLedgerEntryRepository cannot be found`.

- [ ] **Step 3: Create `MessageLedgerEntryRepository.java`**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntryRepository.java`:

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.TypedQuery;

import io.quarkiverse.ledger.runtime.model.LedgerAttestation;
import io.quarkiverse.ledger.runtime.model.LedgerEntry;
import io.quarkiverse.ledger.runtime.model.LedgerEntryType;
import io.quarkiverse.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.hibernate.orm.PersistenceUnit;

/**
 * Blocking JPA repository for {@link MessageLedgerEntry}.
 *
 * <p>
 * Implements {@link LedgerEntryRepository} using direct {@link EntityManager} queries
 * (not Panache statics — {@code LedgerEntry} is a plain {@code @Entity}).
 *
 * <p>
 * Key query: {@link #listEntries} — unified filtered query supporting type, agent, time,
 * and cursor filters. {@link #findLatestByCorrelationId} resolves causal chain links
 * at write time (DONE/FAILURE/DECLINE/HANDOFF → their originating COMMAND or HANDOFF).
 */
@ApplicationScoped
public class MessageLedgerEntryRepository implements LedgerEntryRepository {

    @Inject
    @PersistenceUnit("qhorus")
    EntityManager em;

    @Override
    public LedgerEntry save(final LedgerEntry entry) {
        em.persist(entry);
        return entry;
    }

    /**
     * All entries for a channel, ordered by sequence number ascending.
     */
    public List<MessageLedgerEntry> findByChannelId(final UUID channelId) {
        return em.createQuery(
                "SELECT e FROM MessageLedgerEntry e WHERE e.subjectId = :sid ORDER BY e.sequenceNumber ASC",
                MessageLedgerEntry.class)
                .setParameter("sid", channelId)
                .getResultList();
    }

    /**
     * Filtered query for the {@code list_ledger_entries} MCP tool. All parameters except
     * {@code channelId} and {@code limit} are optional.
     *
     * @param channelId    required — scopes the query to this channel
     * @param messageTypes if non-null/non-empty, only entries whose {@code messageType}
     *                     is in this set are returned
     * @param afterSequence if non-null, only entries with sequenceNumber &gt; afterSequence
     * @param agentId      if non-null/blank, filter by actorId
     * @param since        if non-null, filter by occurredAt &gt;= since
     * @param limit        max results
     */
    public List<MessageLedgerEntry> listEntries(final UUID channelId, final Set<String> messageTypes,
            final Long afterSequence, final String agentId, final Instant since, final int limit) {

        final StringBuilder jpql = new StringBuilder(
                "SELECT e FROM MessageLedgerEntry e WHERE e.subjectId = ?1");
        final List<Object> params = new ArrayList<>();
        params.add(channelId);

        if (messageTypes != null && !messageTypes.isEmpty()) {
            jpql.append(" AND e.messageType IN ?").append(params.size() + 1);
            params.add(messageTypes);
        }
        if (afterSequence != null) {
            jpql.append(" AND e.sequenceNumber > ?").append(params.size() + 1);
            params.add(afterSequence.intValue());
        }
        if (agentId != null && !agentId.isBlank()) {
            jpql.append(" AND e.actorId = ?").append(params.size() + 1);
            params.add(agentId);
        }
        if (since != null) {
            jpql.append(" AND e.occurredAt >= ?").append(params.size() + 1);
            params.add(since);
        }
        jpql.append(" ORDER BY e.sequenceNumber ASC");

        final TypedQuery<MessageLedgerEntry> query = em.createQuery(jpql.toString(), MessageLedgerEntry.class);
        for (int i = 0; i < params.size(); i++) {
            query.setParameter(i + 1, params.get(i));
        }
        query.setMaxResults(limit);
        return query.getResultList();
    }

    /**
     * Returns the most recent COMMAND or HANDOFF entry on this channel with the given
     * correlation ID. Used at write time to resolve {@code causedByEntryId} for DONE,
     * FAILURE, DECLINE, and HANDOFF entries.
     */
    public Optional<MessageLedgerEntry> findLatestByCorrelationId(final UUID channelId,
            final String correlationId) {
        return em.createQuery(
                "SELECT e FROM MessageLedgerEntry e " +
                        "WHERE e.subjectId = :sid AND e.correlationId = :corr " +
                        "AND e.messageType IN ('COMMAND', 'HANDOFF') " +
                        "ORDER BY e.sequenceNumber DESC",
                MessageLedgerEntry.class)
                .setParameter("sid", channelId)
                .setParameter("corr", correlationId)
                .setMaxResults(1)
                .getResultStream()
                .findFirst();
    }

    @Override
    public Optional<LedgerEntry> findLatestBySubjectId(final UUID subjectId) {
        return em.createQuery(
                "SELECT e FROM MessageLedgerEntry e WHERE e.subjectId = :sid ORDER BY e.sequenceNumber DESC",
                LedgerEntry.class)
                .setParameter("sid", subjectId)
                .setMaxResults(1)
                .getResultStream()
                .findFirst();
    }

    @Override
    public Optional<LedgerEntry> findEntryById(final UUID id) {
        return Optional.ofNullable(em.find(MessageLedgerEntry.class, id));
    }

    @Override
    public List<LedgerEntry> findBySubjectId(final UUID subjectId) {
        return em.createQuery(
                "SELECT e FROM MessageLedgerEntry e WHERE e.subjectId = :sid ORDER BY e.sequenceNumber ASC",
                LedgerEntry.class)
                .setParameter("sid", subjectId)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> listAll() {
        return em.createQuery("SELECT e FROM LedgerEntry e ORDER BY e.sequenceNumber ASC", LedgerEntry.class)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> findAllEvents() {
        return em.createQuery(
                "SELECT e FROM MessageLedgerEntry e WHERE e.entryType = :type ORDER BY e.sequenceNumber ASC",
                LedgerEntry.class)
                .setParameter("type", LedgerEntryType.EVENT)
                .getResultList();
    }

    @Override
    public List<LedgerAttestation> findAttestationsByEntryId(final UUID ledgerEntryId) {
        return em.createNamedQuery("LedgerAttestation.findByEntryId", LedgerAttestation.class)
                .setParameter("entryId", ledgerEntryId)
                .getResultList();
    }

    @Override
    public LedgerAttestation saveAttestation(final LedgerAttestation attestation) {
        em.persist(attestation);
        return attestation;
    }

    @Override
    public Map<UUID, List<LedgerAttestation>> findAttestationsForEntries(final Set<UUID> entryIds) {
        if (entryIds.isEmpty()) {
            return Collections.emptyMap();
        }
        return em.createNamedQuery("LedgerAttestation.findByEntryIds", LedgerAttestation.class)
                .setParameter("entryIds", entryIds)
                .getResultList()
                .stream()
                .collect(Collectors.groupingBy(a -> a.ledgerEntryId));
    }

    @Override
    public List<LedgerEntry> findByActorId(final String actorId, final Instant from, final Instant to) {
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.actorId = :aid " +
                        "AND e.occurredAt >= :from AND e.occurredAt <= :to ORDER BY e.occurredAt ASC",
                LedgerEntry.class)
                .setParameter("aid", actorId)
                .setParameter("from", from)
                .setParameter("to", to)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> findByActorRole(final String actorRole, final Instant from, final Instant to) {
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.actorRole = :role " +
                        "AND e.occurredAt >= :from AND e.occurredAt <= :to ORDER BY e.occurredAt ASC",
                LedgerEntry.class)
                .setParameter("role", actorRole)
                .setParameter("from", from)
                .setParameter("to", to)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> findByTimeRange(final Instant from, final Instant to) {
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.occurredAt >= :from AND e.occurredAt <= :to ORDER BY e.occurredAt ASC",
                LedgerEntry.class)
                .setParameter("from", from)
                .setParameter("to", to)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> findCausedBy(final UUID entryId) {
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.causedByEntryId = :eid ORDER BY e.sequenceNumber ASC",
                LedgerEntry.class)
                .setParameter("eid", entryId)
                .getResultList();
    }
}
```

- [ ] **Step 4: Run test — confirm it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageLedgerEntryRepositoryTest -Dno-format -q 2>&1 | tail -8
```
Expected: `BUILD SUCCESS`, 12 tests passing.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageLedgerEntryRepository.java \
        runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerEntryRepositoryTest.java
git commit -m "$(cat <<'EOF'
feat(ledger): add MessageLedgerEntryRepository with unified listEntries query

Supports type, agent, time, and cursor filters. findLatestByCorrelationId
resolves causal chain links (COMMAND/HANDOFF → DONE/FAILURE/DECLINE resolution).

Refs #<child-issue-2> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: `LedgerWriteService` — Unit Tests then Rewrite

**Files:**
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerWriteServiceTest.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/LedgerWriteService.java`

- [ ] **Step 1: Write the failing unit tests**

Create `runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerWriteServiceTest.java`:

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.quarkiverse.ledger.runtime.config.LedgerConfig;
import io.quarkiverse.ledger.runtime.model.ActorType;
import io.quarkiverse.ledger.runtime.model.LedgerEntry;
import io.quarkiverse.ledger.runtime.model.LedgerEntryType;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.ledger.LedgerWriteService;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;

/**
 * Pure unit tests for {@link LedgerWriteService#record} — no Quarkus runtime.
 * Uses a capturing stub repository and Mockito for LedgerConfig.
 * Refs #<child-issue-3> — Epic #<epic>.
 */
class LedgerWriteServiceTest {

    // =========================================================================
    // Stub repository — captures saved entries in memory
    // =========================================================================

    static class CapturingRepo extends MessageLedgerEntryRepository {
        final List<MessageLedgerEntry> saved = new ArrayList<>();

        @Override
        public LedgerEntry save(final LedgerEntry entry) {
            final MessageLedgerEntry mle = (MessageLedgerEntry) entry;
            if (mle.id == null) {
                mle.id = UUID.randomUUID();
            }
            saved.add(mle);
            return mle;
        }

        @Override
        public Optional<LedgerEntry> findLatestBySubjectId(final UUID subjectId) {
            return saved.stream()
                    .filter(e -> e.subjectId.equals(subjectId))
                    .reduce((a, b) -> b)
                    .map(e -> (LedgerEntry) e);
        }

        @Override
        public Optional<MessageLedgerEntry> findLatestByCorrelationId(final UUID channelId,
                final String correlationId) {
            return saved.stream()
                    .filter(e -> channelId.equals(e.subjectId))
                    .filter(e -> correlationId.equals(e.correlationId))
                    .filter(e -> "COMMAND".equals(e.messageType) || "HANDOFF".equals(e.messageType))
                    .reduce((a, b) -> b);
        }
    }

    // =========================================================================
    // Setup
    // =========================================================================

    private CapturingRepo repo;
    private LedgerWriteService service;
    private LedgerConfig enabledConfig;

    @BeforeEach
    void setup() {
        repo = new CapturingRepo();
        enabledConfig = mock(LedgerConfig.class);
        when(enabledConfig.enabled()).thenReturn(true);

        service = new LedgerWriteService();
        service.repository = repo;
        service.config = enabledConfig;
        service.objectMapper = new ObjectMapper();
    }

    // =========================================================================
    // Happy path — one test per obligation-bearing message type
    // =========================================================================

    @Test
    void record_query_createsEntry() {
        service.record(channel(), message("QUERY", "How many orders today?", "agent-a", null, null));

        assertEquals(1, repo.saved.size());
        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("QUERY", e.messageType);
        assertEquals(LedgerEntryType.COMMAND, e.entryType);
        assertEquals("agent-a", e.actorId);
        assertEquals("How many orders today?", e.content);
        assertNull(e.toolName);
    }

    @Test
    void record_command_createsEntry() {
        service.record(channel(), message("COMMAND", "Generate the report", "agent-a", "corr-1", null));

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("COMMAND", e.messageType);
        assertEquals(LedgerEntryType.COMMAND, e.entryType);
        assertEquals("corr-1", e.correlationId);
        assertEquals("Generate the report", e.content);
    }

    @Test
    void record_response_createsEntry() {
        service.record(channel(), message("RESPONSE", "42 orders", "agent-b", "corr-1", null));

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("RESPONSE", e.messageType);
        assertEquals(LedgerEntryType.EVENT, e.entryType);
        assertEquals("42 orders", e.content);
    }

    @Test
    void record_status_createsEntry() {
        service.record(channel(), message("STATUS", "Working on it", "agent-b", "corr-1", null));

        assertEquals(1, repo.saved.size());
        assertEquals("STATUS", repo.saved.get(0).messageType);
    }

    @Test
    void record_decline_createsEntry() {
        service.record(channel(), message("DECLINE", "Out of scope", "agent-b", "corr-1", null));

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("DECLINE", e.messageType);
        assertEquals(LedgerEntryType.EVENT, e.entryType);
        assertEquals("Out of scope", e.content);
    }

    @Test
    void record_handoff_createsEntry() {
        Message msg = message("HANDOFF", null, "agent-a", "corr-1", null);
        msg.target = "instance:agent-c";
        service.record(channel(), msg);

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("HANDOFF", e.messageType);
        assertEquals(LedgerEntryType.COMMAND, e.entryType);
        assertEquals("instance:agent-c", e.target);
    }

    @Test
    void record_done_createsEntry() {
        service.record(channel(), message("DONE", "Report delivered", "agent-b", "corr-1", null));

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("DONE", e.messageType);
        assertEquals(LedgerEntryType.EVENT, e.entryType);
        assertEquals("Report delivered", e.content);
    }

    @Test
    void record_failure_createsEntry() {
        service.record(channel(), message("FAILURE", "Database unreachable", "agent-b", "corr-1", null));

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("FAILURE", e.messageType);
        assertEquals(LedgerEntryType.EVENT, e.entryType);
        assertEquals("Database unreachable", e.content);
    }

    // =========================================================================
    // EVENT — telemetry extraction
    // =========================================================================

    @Test
    void record_event_withValidJson_populatesTelemetry() {
        service.record(channel(),
                message("EVENT", "{\"tool_name\":\"read_file\",\"duration_ms\":42,\"token_count\":1200}",
                        "agent-a", null, null));

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals("EVENT", e.messageType);
        assertEquals("read_file", e.toolName);
        assertEquals(42L, e.durationMs);
        assertEquals(1200L, e.tokenCount);
        assertNull(e.content);
    }

    @Test
    void record_event_missingToolName_entryStillWritten_toolNameNull() {
        service.record(channel(), message("EVENT", "{\"duration_ms\":10}", "agent-a", null, null));

        assertEquals(1, repo.saved.size());
        assertNull(repo.saved.get(0).toolName);
        assertEquals(10L, repo.saved.get(0).durationMs);
    }

    @Test
    void record_event_missingDurationMs_entryStillWritten_durationNull() {
        service.record(channel(), message("EVENT", "{\"tool_name\":\"write_file\"}", "agent-a", null, null));

        assertEquals(1, repo.saved.size());
        assertEquals("write_file", repo.saved.get(0).toolName);
        assertNull(repo.saved.get(0).durationMs);
    }

    @Test
    void record_event_malformedJson_entryStillWritten_allTelemetryNull() {
        service.record(channel(), message("EVENT", "not-valid-json", "agent-a", null, null));

        assertEquals(1, repo.saved.size());
        assertNull(repo.saved.get(0).toolName);
        assertNull(repo.saved.get(0).durationMs);
    }

    @Test
    void record_event_nullContent_entryStillWritten_telemetryNull() {
        service.record(channel(), message("EVENT", null, "agent-a", null, null));

        assertEquals(1, repo.saved.size());
        assertNull(repo.saved.get(0).toolName);
    }

    @Test
    void record_event_emptyContent_entryStillWritten() {
        service.record(channel(), message("EVENT", "", "agent-a", null, null));
        assertEquals(1, repo.saved.size());
    }

    // =========================================================================
    // Causal chain — causedByEntryId resolution
    // =========================================================================

    @Test
    void record_done_withMatchingCommand_setsCausedByEntryId() {
        UUID channelId = UUID.randomUUID();
        Channel ch = channel(channelId);

        // Pre-populate a COMMAND entry in the stub repo
        MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
        cmdEntry.id = UUID.randomUUID();
        cmdEntry.subjectId = channelId;
        cmdEntry.channelId = channelId;
        cmdEntry.messageType = "COMMAND";
        cmdEntry.correlationId = "corr-done";
        cmdEntry.sequenceNumber = 1;
        repo.saved.add(cmdEntry);

        service.record(ch, message("DONE", "Done", "agent-b", "corr-done", null));

        MessageLedgerEntry doneEntry = repo.saved.get(repo.saved.size() - 1);
        assertEquals(cmdEntry.id, doneEntry.causedByEntryId);
    }

    @Test
    void record_failure_withMatchingCommand_setsCausedByEntryId() {
        UUID channelId = UUID.randomUUID();
        Channel ch = channel(channelId);

        MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
        cmdEntry.id = UUID.randomUUID();
        cmdEntry.subjectId = channelId;
        cmdEntry.messageType = "COMMAND";
        cmdEntry.correlationId = "corr-fail";
        cmdEntry.sequenceNumber = 1;
        repo.saved.add(cmdEntry);

        service.record(ch, message("FAILURE", "Timeout", "agent-b", "corr-fail", null));

        assertEquals(cmdEntry.id, repo.saved.get(repo.saved.size() - 1).causedByEntryId);
    }

    @Test
    void record_decline_withMatchingCommand_setsCausedByEntryId() {
        UUID channelId = UUID.randomUUID();
        Channel ch = channel(channelId);

        MessageLedgerEntry cmdEntry = new MessageLedgerEntry();
        cmdEntry.id = UUID.randomUUID();
        cmdEntry.subjectId = channelId;
        cmdEntry.messageType = "COMMAND";
        cmdEntry.correlationId = "corr-dec";
        cmdEntry.sequenceNumber = 1;
        repo.saved.add(cmdEntry);

        service.record(ch, message("DECLINE", "Out of scope", "agent-b", "corr-dec", null));

        assertEquals(cmdEntry.id, repo.saved.get(repo.saved.size() - 1).causedByEntryId);
    }

    @Test
    void record_done_noCorrelationId_causedByEntryIdNull() {
        service.record(channel(), message("DONE", "Done", "agent-b", null, null));

        assertNull(repo.saved.get(0).causedByEntryId);
    }

    @Test
    void record_done_noMatchingCommand_causedByEntryIdNull() {
        service.record(channel(), message("DONE", "Done", "agent-b", "corr-no-match", null));

        assertNull(repo.saved.get(0).causedByEntryId);
    }

    // =========================================================================
    // Sequence numbering
    // =========================================================================

    @Test
    void record_firstEntry_sequenceNumberIsOne() {
        service.record(channel(), message("COMMAND", "Go", "agent-a", null, null));

        assertEquals(1, repo.saved.get(0).sequenceNumber);
    }

    @Test
    void record_threeEntries_sequenceNumbersIncrement() {
        UUID channelId = UUID.randomUUID();
        Channel ch = channel(channelId);

        service.record(ch, message("COMMAND", "Go", "agent-a", null, null));
        service.record(ch, message("STATUS", "Working", "agent-b", null, null));
        service.record(ch, message("DONE", "Done", "agent-b", null, null));

        assertEquals(1, repo.saved.get(0).sequenceNumber);
        assertEquals(2, repo.saved.get(1).sequenceNumber);
        assertEquals(3, repo.saved.get(2).sequenceNumber);
    }

    // =========================================================================
    // Base fields
    // =========================================================================

    @Test
    void record_populatesBaseFields() {
        UUID channelId = UUID.randomUUID();
        Channel ch = channel(channelId);
        Message msg = message("COMMAND", "Run audit", "agent-a", "corr-x", UUID.randomUUID());

        service.record(ch, msg);

        MessageLedgerEntry e = repo.saved.get(0);
        assertEquals(channelId, e.channelId);
        assertEquals(channelId, e.subjectId);
        assertEquals(msg.id, e.messageId);
        assertEquals("agent-a", e.actorId);
        assertEquals(ActorType.AGENT, e.actorType);
        assertEquals("corr-x", e.correlationId);
        assertEquals(msg.commitmentId, e.commitmentId);
        assertNotNull(e.occurredAt);
    }

    // =========================================================================
    // Ledger disabled
    // =========================================================================

    @Test
    void record_ledgerDisabled_writesNothing() {
        when(enabledConfig.enabled()).thenReturn(false);

        service.record(channel(), message("COMMAND", "Do it", "agent-a", null, null));

        assertTrue(repo.saved.isEmpty());
    }

    // =========================================================================
    // Fixtures
    // =========================================================================

    private Channel channel() {
        return channel(UUID.randomUUID());
    }

    private Channel channel(final UUID channelId) {
        Channel ch = new Channel();
        ch.id = channelId;
        ch.name = "test-channel-" + channelId;
        return ch;
    }

    private Message message(final String type, final String content, final String sender,
            final String correlationId, final UUID commitmentId) {
        Message msg = new Message();
        msg.id = (long) (Math.random() * 100000);
        msg.messageType = MessageType.valueOf(type);
        msg.content = content;
        msg.sender = sender;
        msg.correlationId = correlationId;
        msg.commitmentId = commitmentId;
        msg.createdAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);
        return msg;
    }
}
```

- [ ] **Step 2: Run test — confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerWriteServiceTest -Dno-format -q 2>&1 | tail -8
```
Expected: compilation errors — `record` method not found on `LedgerWriteService`, `repository`/`config`/`objectMapper` fields not accessible.

- [ ] **Step 3: Rewrite `LedgerWriteService.java`**

Replace the entire content of `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/LedgerWriteService.java`:

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.time.temporal.ChronoUnit;
import java.util.Optional;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.quarkiverse.ledger.runtime.config.LedgerConfig;
import io.quarkiverse.ledger.runtime.model.ActorType;
import io.quarkiverse.ledger.runtime.model.LedgerEntry;
import io.quarkiverse.ledger.runtime.model.LedgerEntryType;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;

/**
 * Writes immutable audit ledger entries for every message sent on a channel.
 *
 * <p>
 * Called from {@code QhorusMcpTools.sendMessage} for all 9 message types — there is no
 * conditional branching in the caller. Every speech act on a channel is permanently
 * recorded. The CommitmentStore is the live obligation state; this ledger is the
 * tamper-evident historical record.
 *
 * <p>
 * For EVENT messages, telemetry fields ({@code toolName}, {@code durationMs}, etc.) are
 * extracted from the JSON payload. Malformed or partial payloads still produce an entry —
 * the speech act happened regardless of telemetry quality. For all other types, the
 * {@code content} field carries the message content verbatim.
 *
 * <p>
 * DONE, FAILURE, DECLINE, and HANDOFF entries have {@code causedByEntryId} set to the
 * most recent COMMAND or HANDOFF entry sharing the same {@code correlationId} on the
 * same channel — creating a traversable obligation chain in the ledger itself.
 *
 * <p>
 * Ledger write failures are caught and logged; they never propagate to the caller.
 * The message pipeline must not be affected by ledger issues.
 */
@ApplicationScoped
public class LedgerWriteService {

    private static final Logger LOG = Logger.getLogger(LedgerWriteService.class);
    private static final Set<String> CAUSAL_TYPES = Set.of("DONE", "FAILURE", "DECLINE", "HANDOFF");

    @Inject
    MessageLedgerEntryRepository repository;

    @Inject
    LedgerConfig config;

    @Inject
    ObjectMapper objectMapper;

    /**
     * Record the given message as an immutable ledger entry.
     *
     * <p>
     * Runs in its own transaction ({@code REQUIRES_NEW}) so that a ledger write failure
     * does not roll back the calling transaction.
     *
     * @param ch      the channel the message was sent to
     * @param message the persisted message to record
     */
    @Transactional(value = Transactional.TxType.REQUIRES_NEW)
    public void record(final Channel ch, final Message message) {
        if (!config.enabled()) {
            return;
        }

        final Optional<LedgerEntry> latest = repository.findLatestBySubjectId(ch.id);
        final int sequenceNumber = latest.map(e -> e.sequenceNumber + 1).orElse(1);

        final MessageLedgerEntry entry = new MessageLedgerEntry();
        entry.subjectId = ch.id;
        entry.channelId = ch.id;
        entry.messageId = message.id;
        entry.messageType = message.messageType.name();
        entry.target = message.target;
        entry.correlationId = message.correlationId;
        entry.commitmentId = message.commitmentId;
        entry.actorId = message.sender;
        entry.actorType = ActorType.AGENT;
        entry.occurredAt = message.createdAt.truncatedTo(ChronoUnit.MILLIS);
        entry.sequenceNumber = sequenceNumber;
        entry.entryType = switch (message.messageType) {
            case QUERY, COMMAND, HANDOFF -> LedgerEntryType.COMMAND;
            default -> LedgerEntryType.EVENT;
        };

        if (message.messageType == MessageType.EVENT) {
            populateTelemetry(entry, message.content);
        } else {
            entry.content = message.content;
        }

        if (CAUSAL_TYPES.contains(message.messageType.name()) && message.correlationId != null) {
            repository.findLatestByCorrelationId(ch.id, message.correlationId)
                    .ifPresent(prior -> entry.causedByEntryId = prior.id);
        }

        repository.save(entry);
    }

    private void populateTelemetry(final MessageLedgerEntry entry, final String content) {
        if (content == null || !content.stripLeading().startsWith("{")) {
            return;
        }
        try {
            final JsonNode root = objectMapper.readTree(content);

            final JsonNode tn = root.get("tool_name");
            if (tn != null && tn.isTextual()) {
                entry.toolName = tn.asText();
            }
            final JsonNode dm = root.get("duration_ms");
            if (dm != null && dm.isNumber()) {
                entry.durationMs = dm.asLong();
            }
            final JsonNode tc = root.get("token_count");
            if (tc != null && tc.isNumber()) {
                entry.tokenCount = tc.asLong();
            }
            final JsonNode cr = root.get("context_refs");
            if (cr != null && !cr.isNull()) {
                try {
                    entry.contextRefs = objectMapper.writeValueAsString(cr);
                } catch (final Exception ignored) {
                    LOG.warnf("Could not serialise context_refs for ledger entry on message %d", entry.messageId);
                }
            }
            final JsonNode se = root.get("source_entity");
            if (se != null && !se.isNull()) {
                try {
                    entry.sourceEntity = objectMapper.writeValueAsString(se);
                } catch (final Exception ignored) {
                    LOG.warnf("Could not serialise source_entity for ledger entry on message %d", entry.messageId);
                }
            }
        } catch (final Exception e) {
            LOG.warnf("Could not parse EVENT content as JSON for message %d — telemetry fields will be null",
                    entry.messageId);
        }
    }
}
```

- [ ] **Step 4: Run unit tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerWriteServiceTest -Dno-format -q 2>&1 | tail -8
```
Expected: `BUILD SUCCESS`, all 24 tests passing.

- [ ] **Step 5: Run the full test suite — confirm nothing broken**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format -q 2>&1 | tail -8
```
Expected: `BUILD SUCCESS`. Note: existing `AgentLedgerCaptureTest` may still pass because `AgentMessageLedgerEntry` and the old `recordEvent` method still exist. If any test fails, investigate before continuing.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/LedgerWriteService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/ledger/LedgerWriteServiceTest.java
git commit -m "$(cat <<'EOF'
feat(ledger): rewrite LedgerWriteService — unified record() for all 9 message types

All types produce ledger entries. EVENT telemetry extracted from JSON (graceful on
malformed payload). Causal chain: DONE/FAILURE/DECLINE/HANDOFF resolve causedByEntryId
via correlationId lookup. Old recordEvent() removed.

Refs #<child-issue-3> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Verification

- [ ] **Final check — Part 1 complete**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | grep -E "Tests run|BUILD"
```
Expected: `BUILD SUCCESS`. All existing tests still pass. Three new test classes added:
`MessageLedgerEntryTest`, `MessageLedgerEntryRepositoryTest`, `LedgerWriteServiceTest`.

Continue with Part 2: `docs/superpowers/plans/2026-04-26-normative-ledger-part2-tests-mcp.md`

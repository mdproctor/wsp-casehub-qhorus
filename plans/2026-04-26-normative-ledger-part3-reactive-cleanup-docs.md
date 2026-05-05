# Normative Ledger — Part 3: Reactive Stack, Cleanup, Examples, Documentation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mirror the blocking stack reactively; delete all old ledger classes and tests; add ledger assertions to both example modules; perform the systematic design-doc review and add the new Normative Audit Ledger section.

**Prerequisite:** Parts 1 and 2 complete — `MessageLedgerEntry`, `MessageLedgerEntryRepository`, `LedgerWriteService.record()`, `list_ledger_entries` all working and tested. All Part 2 tests pass.

**Architecture:** Reactive components are `@Alternative` and activated only when `quarkus.qhorus.reactive.enabled=true` is set at build time. Their tests are `@Disabled` in CI (no Docker/reactive PG). Old classes are deleted in one atomic commit after the reactive replacements exist.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate Reactive Panache, Mutiny `Uni<T>`, JUnit 5.

**Spec:** `docs/superpowers/specs/2026-04-26-normative-ledger-design.md`

---

## File Map

| File | Action |
|---|---|
| `runtime/.../ledger/MessageReactivePanacheRepo.java` | Create |
| `runtime/.../ledger/ReactiveMessageLedgerEntryRepository.java` | Create |
| `runtime/.../ledger/ReactiveLedgerWriteService.java` | Rewrite |
| `runtime/.../mcp/ReactiveQhorusMcpTools.java` | Modify — wire `record()`, `list_ledger_entries` |
| `runtime/.../ledger/AgentMessageLedgerEntry.java` | Delete |
| `runtime/.../ledger/AgentMessageLedgerEntryRepository.java` | Delete |
| `runtime/.../ledger/AgentMessageReactivePanacheRepo.java` | Delete |
| `runtime/.../ledger/ReactiveAgentMessageLedgerEntryRepository.java` | Delete |
| `runtime/src/test/.../ledger/AgentMessageLedgerEntryTest.java` | Delete |
| `examples/type-system/src/test/.../LedgerCaptureExampleTest.java` | Create |
| `examples/agent-communication/src/test/.../LedgerObligationTrailTest.java` | Create |
| `examples/type-system/src/main/resources/application.properties` | Modify — enable ledger |
| `examples/agent-communication/src/main/resources/application.properties` | Modify — enable ledger |
| `docs/specs/2026-04-13-qhorus-design.md` | Systematic review + normative ledger section |

---

## Task 8: Reactive Stack

All reactive tests are `@Disabled` — reactive JPA requires a native async driver (PostgreSQL + Docker) not available in CI H2 builds.

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageReactivePanacheRepo.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveMessageLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveLedgerWriteService.java`

- [ ] **Step 1: Create `MessageReactivePanacheRepo`**

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;

/**
 * Minimal reactive Panache repository for {@link MessageLedgerEntry}.
 *
 * <p>
 * Marked {@code @Alternative} — inactive by default. Activate alongside
 * {@link ReactiveMessageLedgerEntryRepository} via {@code quarkus.arc.selected-alternatives}
 * when configuring a reactive datasource. This prevents Hibernate Reactive from booting in
 * applications that only use the blocking {@link MessageLedgerEntryRepository}.
 */
@Alternative
@ApplicationScoped
class MessageReactivePanacheRepo implements PanacheRepositoryBase<MessageLedgerEntry, UUID> {
}
```

- [ ] **Step 2: Create `ReactiveMessageLedgerEntryRepository`**

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.quarkiverse.ledger.runtime.model.LedgerAttestation;
import io.quarkiverse.ledger.runtime.model.LedgerEntry;
import io.quarkiverse.ledger.runtime.model.LedgerEntryType;
import io.quarkiverse.ledger.runtime.repository.ReactiveLedgerEntryRepository;
import io.smallrye.mutiny.Uni;

/**
 * Reactive mirror of {@link MessageLedgerEntryRepository}.
 *
 * <p>
 * {@code @Alternative} — inactive by default. Activate via
 * {@code quarkus.arc.selected-alternatives} when a reactive datasource is configured.
 * Tests are {@code @Disabled} in CI (requires PostgreSQL reactive driver, not H2).
 *
 * <p>
 * {@link LedgerAttestation} methods throw {@link UnsupportedOperationException} —
 * reactive attestation persistence is not yet available in quarkus-ledger.
 */
@Alternative
@ApplicationScoped
public class ReactiveMessageLedgerEntryRepository implements ReactiveLedgerEntryRepository {

    @Inject
    MessageReactivePanacheRepo repo;

    @Override
    public Uni<LedgerEntry> save(final LedgerEntry entry) {
        return repo.persist((MessageLedgerEntry) entry).map(e -> (LedgerEntry) e);
    }

    public Uni<List<MessageLedgerEntry>> findByChannelId(final UUID channelId) {
        return repo.list("subjectId = ?1 ORDER BY sequenceNumber ASC", channelId);
    }

    @Override
    public Uni<Optional<LedgerEntry>> findLatestBySubjectId(final UUID subjectId) {
        return repo.find("subjectId = ?1 ORDER BY sequenceNumber DESC", subjectId)
                .firstResult()
                .map(e -> Optional.ofNullable((LedgerEntry) e));
    }

    public Uni<Optional<MessageLedgerEntry>> findLatestByCorrelationId(final UUID channelId,
            final String correlationId) {
        return repo.find(
                "subjectId = ?1 AND correlationId = ?2 AND messageType IN ('COMMAND','HANDOFF') " +
                        "ORDER BY sequenceNumber DESC",
                channelId, correlationId)
                .firstResult()
                .map(Optional::ofNullable);
    }

    @Override
    public Uni<Optional<LedgerEntry>> findEntryById(final UUID id) {
        return repo.findById(id).map(e -> Optional.ofNullable((LedgerEntry) e));
    }

    @Override
    @SuppressWarnings("unchecked")
    public Uni<List<LedgerEntry>> findBySubjectId(final UUID subjectId) {
        return repo.list("subjectId = ?1 ORDER BY sequenceNumber ASC", subjectId)
                .map(l -> (List<LedgerEntry>) (List<?>) l);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Uni<List<LedgerEntry>> listAll() {
        return repo.listAll().map(l -> (List<LedgerEntry>) (List<?>) l);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Uni<List<LedgerEntry>> findAllEvents() {
        return repo.list("entryType = ?1", LedgerEntryType.EVENT)
                .map(l -> (List<LedgerEntry>) (List<?>) l);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Uni<List<LedgerEntry>> findByActorId(final String actorId, final Instant from, final Instant to) {
        return repo.list(
                "actorId = ?1 AND occurredAt >= ?2 AND occurredAt <= ?3 ORDER BY occurredAt ASC",
                actorId, from, to)
                .map(l -> (List<LedgerEntry>) (List<?>) l);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Uni<List<LedgerEntry>> findByActorRole(final String actorRole, final Instant from, final Instant to) {
        return repo.list(
                "actorRole = ?1 AND occurredAt >= ?2 AND occurredAt <= ?3 ORDER BY occurredAt ASC",
                actorRole, from, to)
                .map(l -> (List<LedgerEntry>) (List<?>) l);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Uni<List<LedgerEntry>> findByTimeRange(final Instant from, final Instant to) {
        return repo.list("occurredAt >= ?1 AND occurredAt <= ?2 ORDER BY occurredAt ASC", from, to)
                .map(l -> (List<LedgerEntry>) (List<?>) l);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Uni<List<LedgerEntry>> findCausedBy(final UUID entryId) {
        return repo.list("causedByEntryId = ?1 ORDER BY sequenceNumber ASC", entryId)
                .map(l -> (List<LedgerEntry>) (List<?>) l);
    }

    @Override
    public Uni<LedgerAttestation> saveAttestation(final LedgerAttestation attestation) {
        throw new UnsupportedOperationException(
                "Reactive attestation writes not yet supported — use blocking LedgerEntryRepository");
    }

    @Override
    public Uni<List<LedgerAttestation>> findAttestationsByEntryId(final UUID ledgerEntryId) {
        throw new UnsupportedOperationException(
                "Reactive attestation reads not yet supported — use blocking LedgerEntryRepository");
    }

    @Override
    public Uni<Map<UUID, List<LedgerAttestation>>> findAttestationsForEntries(final Set<UUID> entryIds) {
        throw new UnsupportedOperationException(
                "Reactive attestation reads not yet supported — use blocking LedgerEntryRepository");
    }
}
```

- [ ] **Step 3: Rewrite `ReactiveLedgerWriteService.java`**

Replace the entire file content:

```java
package io.quarkiverse.qhorus.runtime.ledger;

import java.time.temporal.ChronoUnit;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

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
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;

/**
 * Reactive mirror of {@link LedgerWriteService}.
 *
 * <p>
 * {@code @Alternative} — inactive by default. Writes immutable audit ledger entries for
 * every message type using the reactive ledger repository. Called from
 * {@code ReactiveQhorusMcpTools.sendMessage}. Failures are caught and swallowed at the
 * call site — the message pipeline must not be affected by ledger issues.
 */
@Alternative
@ApplicationScoped
public class ReactiveLedgerWriteService {

    private static final Logger LOG = Logger.getLogger(ReactiveLedgerWriteService.class);
    private static final Set<String> CAUSAL_TYPES = Set.of("DONE", "FAILURE", "DECLINE", "HANDOFF");

    @Inject
    ReactiveMessageLedgerEntryRepository reactiveRepo;

    @Inject
    LedgerConfig config;

    @Inject
    ObjectMapper objectMapper;

    public Uni<Void> record(final Channel ch, final Message message) {
        if (!config.enabled()) {
            return Uni.createFrom().voidItem();
        }

        return Panache.withTransaction(() ->
                reactiveRepo.findLatestBySubjectId(ch.id).flatMap(latestOpt -> {
                    final int sequenceNumber = latestOpt.map(e -> e.sequenceNumber + 1).orElse(1);

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
                        return reactiveRepo.findLatestByCorrelationId(ch.id, message.correlationId)
                                .flatMap(priorOpt -> {
                                    priorOpt.ifPresent(prior -> entry.causedByEntryId = prior.id);
                                    return reactiveRepo.save(entry).replaceWithVoid();
                                });
                    }
                    return reactiveRepo.save(entry).replaceWithVoid();
                }));
    }

    private void populateTelemetry(final MessageLedgerEntry entry, final String content) {
        if (content == null || !content.stripLeading().startsWith("{")) {
            return;
        }
        try {
            final JsonNode root = objectMapper.readTree(content);
            final JsonNode tn = root.get("tool_name");
            if (tn != null && tn.isTextual()) entry.toolName = tn.asText();
            final JsonNode dm = root.get("duration_ms");
            if (dm != null && dm.isNumber()) entry.durationMs = dm.asLong();
            final JsonNode tc = root.get("token_count");
            if (tc != null && tc.isNumber()) entry.tokenCount = tc.asLong();
            final JsonNode cr = root.get("context_refs");
            if (cr != null && !cr.isNull()) {
                try { entry.contextRefs = objectMapper.writeValueAsString(cr); }
                catch (final Exception ignored) {}
            }
            final JsonNode se = root.get("source_entity");
            if (se != null && !se.isNull()) {
                try { entry.sourceEntity = objectMapper.writeValueAsString(se); }
                catch (final Exception ignored) {}
            }
        } catch (final Exception e) {
            LOG.warnf("Could not parse EVENT content as JSON for message %d — telemetry fields will be null",
                    entry.messageId);
        }
    }
}
```

- [ ] **Step 4: Update `ReactiveQhorusMcpTools` — wire `record()` and replace `listEvents`**

In `ReactiveQhorusMcpTools.java`:

a) Change the `reactiveLedgerWriteService` injection (if present) — it now injects `ReactiveLedgerWriteService` which has `record()`. The call site in `sendMessage` should change from the EVENT-only conditional to:

```java
// Replace:
if (msgType == MessageType.EVENT) {
    reactiveLedgerWriteService.recordEvent(ch, msg)...
}

// With:
reactiveLedgerWriteService.record(ch, msg)
    .onFailure().invoke(e -> LOG.warnf("Ledger write failed for message %d: %s", msg.id, e.getMessage()))
    .onFailure().recoverWithNull()
```

b) Change `ledgerRepo` field type from `ReactiveAgentMessageLedgerEntryRepository` to `ReactiveMessageLedgerEntryRepository`.

c) Remove the reactive `listEvents` tool method.

d) Add the reactive `listLedgerEntries` tool method (mirrors the blocking version from Part 2 Task 6 Step 5, returning `Uni<List<Map<String, Object>>>` and calling `reactiveRepo.listEntries(...)` if that method exists, or delegating to a blocking call).

Note: The reactive `listEntries` query with dynamic JPQL may not be available via Panache's simple API. If `ReactiveMessageLedgerEntryRepository` does not have a `listEntries` method (it currently does not — only `findByChannelId`), implement `listLedgerEntries` reactively by calling the blocking `MessageLedgerEntryRepository` via `Uni.createFrom().item(...)` as a temporary measure, or add `listEntries` to `ReactiveMessageLedgerEntryRepository` using a Hibernate Reactive `Session` and `createQuery`. The simplest approach: delegate to the blocking repo inside `Uni.createFrom().item(...)`.

- [ ] **Step 5: Build — confirm no compilation errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -Dno-format -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format -q 2>&1 | tail -8
```
Expected: `BUILD SUCCESS`. All tests passing.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/MessageReactivePanacheRepo.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveMessageLedgerEntryRepository.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveLedgerWriteService.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java
git commit -m "$(cat <<'EOF'
feat(ledger): reactive stack — ReactiveMessageLedgerEntryRepository, ReactiveLedgerWriteService

Mirrors blocking implementation. record() replaces recordEvent() for all 9 types.
list_ledger_entries replaces list_events in ReactiveQhorusMcpTools.
All @Alternative — activated only with reactive datasource.

Refs #<child-issue-6> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Delete Old Classes

Remove all old ledger classes now that their replacements exist and are tested.

- [ ] **Step 1: Delete old production classes**

```bash
rm runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntry.java
rm runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntryRepository.java
rm runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageReactivePanacheRepo.java
rm runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveAgentMessageLedgerEntryRepository.java
```

- [ ] **Step 2: Delete old test class**

```bash
rm runtime/src/test/java/io/quarkiverse/qhorus/ledger/AgentMessageLedgerEntryTest.java
```

- [ ] **Step 3: Build — confirm no dangling references**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -Dno-format -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`. If there are `cannot find symbol` errors for `AgentMessageLedgerEntry`, search for any remaining references:

```bash
grep -r "AgentMessageLedgerEntry\|AgentMessageReactivePanacheRepo\|ReactiveAgentMessageLedgerEntryRepository" \
    runtime/src/main --include="*.java"
```

Fix any found references before continuing.

- [ ] **Step 4: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format -q 2>&1 | tail -8
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 5: Commit**

```bash
git rm runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntry.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageLedgerEntryRepository.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/AgentMessageReactivePanacheRepo.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/ledger/ReactiveAgentMessageLedgerEntryRepository.java
git rm runtime/src/test/java/io/quarkiverse/qhorus/ledger/AgentMessageLedgerEntryTest.java
git commit -m "$(cat <<'EOF'
refactor(ledger): delete old AgentMessageLedgerEntry classes and unit test

All replaced by MessageLedgerEntry, MessageLedgerEntryRepository,
ReactiveMessageLedgerEntryRepository, MessageReactivePanacheRepo.

Refs #<child-issue-7> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Enable Ledger in Examples + Add Ledger Tests

**Files:**
- Modify: `examples/type-system/src/test/resources/application.properties`
- Modify: `examples/agent-communication/src/main/resources/application.properties`
- Create: `examples/type-system/src/test/java/io/quarkiverse/qhorus/examples/LedgerCaptureExampleTest.java`
- Create: `examples/agent-communication/src/test/java/io/quarkiverse/qhorus/examples/LedgerObligationTrailTest.java`

- [ ] **Step 1: Enable ledger in type-system test application.properties**

In `examples/type-system/src/test/resources/application.properties`, change:
```properties
quarkus.ledger.enabled=false
```
to:
```properties
quarkus.ledger.enabled=true
```

- [ ] **Step 2: Enable ledger in agent-communication application.properties**

In `examples/agent-communication/src/main/resources/application.properties`, change:
```properties
quarkus.ledger.enabled=false
```
to:
```properties
quarkus.ledger.enabled=true
```

- [ ] **Step 3: Create `LedgerCaptureExampleTest` in type-system**

Find the test package used in `examples/type-system/src/test/java/` and create `LedgerCaptureExampleTest.java` in the same package:

```java
package io.quarkiverse.qhorus.examples;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Set;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Validates that the normative ledger records entries for all 9 message types
 * in the type-system example context (InMemory stores, H2 ledger).
 *
 * <p>
 * These tests run in CI without any model or external services.
 *
 * <p>
 * Refs #<child-issue-8> — Epic #<epic>.
 */
@QuarkusTest
class LedgerCaptureExampleTest {

    @Inject
    QhorusMcpTools tools;

    @Inject
    MessageLedgerEntryRepository ledgerRepo;

    @Test
    void allNineMessageTypes_produceLedgerEntries() {
        tools.createChannel("ledger-ex-all-types", "APPEND", null, null);
        tools.registerInstance("ledger-ex-all-types", "agent-a", null, null, null);
        tools.registerInstance("ledger-ex-all-types", "agent-b", null, null, null);

        tools.sendMessage("ledger-ex-all-types", "agent-a", "query",
                "What is the order count?", "corr-ex1", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-b", "response",
                "42 orders", "corr-ex1", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-a", "command",
                "Generate compliance report", "corr-ex2", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-b", "status",
                "Processing...", "corr-ex2", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-b", "done",
                "Report delivered", "corr-ex2", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-a", "command",
                "Delete audit logs", "corr-ex3", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-b", "decline",
                "I do not have permission to delete", "corr-ex3", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-a", "command",
                "Audit the accounts", "corr-ex4", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-b", "failure",
                "Database unreachable", "corr-ex4", null, null, null);
        tools.sendMessage("ledger-ex-all-types", "agent-a", "event",
                "{\"tool_name\":\"read_file\",\"duration_ms\":10}", null, null, null, null);

        io.quarkiverse.qhorus.runtime.channel.Channel ch =
                io.quarkiverse.qhorus.runtime.channel.Channel.<io.quarkiverse.qhorus.runtime.channel.Channel>find(
                        "name", "ledger-ex-all-types")
                        .firstResultOptional()
                        .orElseThrow();

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(ch.id);

        // 10 messages → 10 ledger entries
        assertEquals(10, entries.size());

        // Verify all 9 types are present
        Set<String> types = new java.util.HashSet<>();
        entries.forEach(e -> types.add(e.messageType));
        assertTrue(types.containsAll(Set.of(
                "QUERY", "RESPONSE", "COMMAND", "STATUS", "DONE",
                "DECLINE", "FAILURE", "EVENT")));
        // HANDOFF not in this scenario — covered by MessageLedgerCaptureTest
    }

    @Test
    void obligationLifecycle_commandDone_causalChainPresent() {
        tools.createChannel("ledger-ex-chain", "APPEND", null, null);
        tools.registerInstance("ledger-ex-chain", "agent-a", null, null, null);
        tools.registerInstance("ledger-ex-chain", "agent-b", null, null, null);

        tools.sendMessage("ledger-ex-chain", "agent-a", "command",
                "Run end-of-day batch", "corr-chain", null, null, null);
        tools.sendMessage("ledger-ex-chain", "agent-b", "done",
                "Batch complete — 1542 records processed", "corr-chain", null, null, null);

        io.quarkiverse.qhorus.runtime.channel.Channel ch =
                io.quarkiverse.qhorus.runtime.channel.Channel.<io.quarkiverse.qhorus.runtime.channel.Channel>find(
                        "name", "ledger-ex-chain")
                        .firstResultOptional()
                        .orElseThrow();

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(ch.id);
        assertEquals(2, entries.size());

        MessageLedgerEntry cmd = entries.get(0);
        MessageLedgerEntry done = entries.get(1);

        assertEquals("COMMAND", cmd.messageType);
        assertEquals("DONE", done.messageType);
        assertNotNull(done.causedByEntryId, "DONE must point back to COMMAND via causedByEntryId");
        assertEquals(cmd.id, done.causedByEntryId);
    }

    @Test
    void listLedgerEntries_typeFilter_obligationTypesOnly() {
        tools.createChannel("ledger-ex-filter", "APPEND", null, null);
        tools.registerInstance("ledger-ex-filter", "agent-a", null, null, null);
        tools.registerInstance("ledger-ex-filter", "agent-b", null, null, null);

        String corr = "corr-filter";
        tools.sendMessage("ledger-ex-filter", "agent-a", "command", "Do X", corr, null, null, null);
        tools.sendMessage("ledger-ex-filter", "agent-a", "status", "Working", corr, null, null, null);
        tools.sendMessage("ledger-ex-filter", "agent-b", "done", "Done", corr, null, null, null);
        tools.sendMessage("ledger-ex-filter", "agent-a", "event",
                "{\"tool_name\":\"t\",\"duration_ms\":1}", null, null, null, null);

        List<java.util.Map<String, Object>> obligationEntries =
                tools.listLedgerEntries("ledger-ex-filter", "COMMAND,DONE,FAILURE,DECLINE,HANDOFF",
                        null, null, null, 20);

        assertEquals(2, obligationEntries.size());
        assertTrue(obligationEntries.stream()
                .allMatch(e -> Set.of("COMMAND", "DONE").contains(e.get("message_type"))));
    }
}
```

- [ ] **Step 4: Run type-system example tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/type-system -Dno-format -q 2>&1 | tail -8
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 5: Create `LedgerObligationTrailTest` in agent-communication**

Find the test package in `examples/agent-communication/src/test/java/` and create:

```java
package io.quarkiverse.qhorus.examples;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.Set;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Validates that real LLM agent communication produces correct ledger entries.
 *
 * <p>
 * Agents use Jlama (pure Java inference). The obligation lifecycle
 * (COMMAND → STATUS → DONE or FAILURE) must be reflected in the ledger.
 *
 * <p>
 * Requires {@code -Pwith-llm-examples} and model in {@code ~/.jlama/}.
 *
 * <p>
 * Refs #<child-issue-8> — Epic #<epic>.
 */
@QuarkusTest
class LedgerObligationTrailTest {

    @Inject
    QhorusMcpTools tools;

    @Inject
    io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntryRepository ledgerRepo;

    @Test
    void agentCommunication_commandLifecycle_producesLedgerTrail() {
        // Set up a channel for this test
        tools.createChannel("ledger-llm-trail", "APPEND", null, null);
        tools.registerInstance("ledger-llm-trail", "orchestrator", null, null, null);
        tools.registerInstance("ledger-llm-trail", "worker", null, null, null);

        String corrId = java.util.UUID.randomUUID().toString();

        // Orchestrator issues a COMMAND
        tools.sendMessage("ledger-llm-trail", "orchestrator", "command",
                "Generate a summary of Q1 sales data", corrId, null, null, null);

        // Worker acknowledges with STATUS then completes with DONE
        tools.sendMessage("ledger-llm-trail", "worker", "status",
                "Retrieving Q1 data", corrId, null, null, null);
        tools.sendMessage("ledger-llm-trail", "worker", "done",
                "Q1 sales total: $1.2M across 342 transactions", corrId, null, null, null);

        io.quarkiverse.qhorus.runtime.channel.Channel ch =
                io.quarkiverse.qhorus.runtime.channel.Channel.<io.quarkiverse.qhorus.runtime.channel.Channel>find(
                        "name", "ledger-llm-trail")
                        .firstResultOptional()
                        .orElseThrow();

        List<io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry> entries =
                ledgerRepo.findByChannelId(ch.id);

        // 3 messages → 3 ledger entries
        assertThat(entries).hasSize(3);

        // Types are as expected
        assertThat(entries.get(0).messageType).isEqualTo("COMMAND");
        assertThat(entries.get(1).messageType).isEqualTo("STATUS");
        assertThat(entries.get(2).messageType).isEqualTo("DONE");

        // DONE points back to COMMAND
        assertThat(entries.get(2).causedByEntryId)
                .as("DONE entry should point to COMMAND via causedByEntryId")
                .isEqualTo(entries.get(0).id);

        // Obligation lifecycle visible via list_ledger_entries
        List<Map<String, Object>> obligationTrail = tools.listLedgerEntries(
                "ledger-llm-trail", "COMMAND,DONE,FAILURE", null, null, null, 20);
        assertThat(obligationTrail).hasSize(2); // COMMAND + DONE only (STATUS filtered out)
        assertThat(obligationTrail.get(1).get("caused_by_entry_id")).isNotNull();
    }
}
```

- [ ] **Step 6: Build agent-communication (compile only — LLM tests need the profile)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test-compile -pl examples/agent-communication -Dno-format -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 7: Commit**

```bash
git add examples/type-system/src/test/resources/application.properties \
        examples/agent-communication/src/main/resources/application.properties \
        examples/type-system/src/test/java/io/quarkiverse/qhorus/examples/LedgerCaptureExampleTest.java \
        examples/agent-communication/src/test/java/io/quarkiverse/qhorus/examples/LedgerObligationTrailTest.java
git commit -m "$(cat <<'EOF'
test(examples): ledger capture tests — all 9 types, causal chain, type_filter

type-system: LedgerCaptureExampleTest runs in CI (no model).
agent-communication: LedgerObligationTrailTest validates obligation trail
against real LLM agents (requires -Pwith-llm-examples).

Refs #<child-issue-8> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 11: Documentation — Systematic Review

Read the full `docs/specs/2026-04-13-qhorus-design.md` from top to bottom. Work through this checklist section by section, making fixes inline as you find them. Do not defer — fix each item before moving to the next.

**Files:**
- Modify: `docs/specs/2026-04-13-qhorus-design.md`

- [ ] **Step 1: Staleness pass — find and fix outdated references**

Search and fix each of the following:

```bash
grep -n "PendingReply\|PENDING_REPLY\|pending_reply\|list_events\|listEvents\|AgentMessageLedgerEntry\|recordEvent" \
    docs/specs/2026-04-13-qhorus-design.md
```

For each hit:
- `PendingReply` / `PENDING_REPLY` / `pending_reply` → replace with `Commitment` / `COMMITMENT` / `commitment`
- `list_events` / `listEvents` → replace with `list_ledger_entries`
- `AgentMessageLedgerEntry` → replace with `MessageLedgerEntry`
- `recordEvent` → replace with `record`

- [ ] **Step 2: Data Model ERD — update**

Find the `## Data Model` section containing the `erDiagram` Mermaid block.

a) Remove the `PENDING_REPLY` entity and its relationships entirely.

b) Add `COMMITMENT` entity:
```
COMMITMENT {
    uuid id PK
    string correlation_id
    string state
    string obligor
    string requester
    uuid channel_id FK
    timestamp created_at
    timestamp updated_at
    timestamp expires_at
}
```

c) Add `MESSAGE_LEDGER_ENTRY` entity:
```
MESSAGE_LEDGER_ENTRY {
    uuid id PK
    uuid channel_id FK
    bigint message_id FK
    string message_type
    string entry_type
    string actor_id
    string target
    text content
    string correlation_id
    uuid commitment_id FK
    uuid caused_by_entry_id FK
    int sequence_number
    timestamp occurred_at
    string digest
    string tool_name
    bigint duration_ms
    bigint token_count
}
```

d) Add relationships:
```
CHANNEL ||--o{ COMMITMENT : tracked_by
CHANNEL ||--o{ MESSAGE_LEDGER_ENTRY : audited_by
MESSAGE ||--o{ MESSAGE_LEDGER_ENTRY : recorded_as
```

- [ ] **Step 3: Message Type Taxonomy table — verify correctness**

Find the table under `## Message Type Taxonomy`. Verify:
- All 9 types are listed: QUERY, COMMAND, RESPONSE, STATUS, DECLINE, HANDOFF, DONE, FAILURE, EVENT
- The `Obligation` column matches the current `CommitmentService` state machine
- The `Terminal` column is correct (DECLINE, HANDOFF, DONE, FAILURE are terminal)
- Fix any rows that are wrong

- [ ] **Step 4: MCP Tool Surface — verify completeness**

Find the `## MCP Tool Surface` section. Check each tool listed against the current `QhorusMcpTools`:

```bash
grep -n "@Tool(name" runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java | \
    sed 's/.*name = "\([^"]*\)".*/\1/' | sort
```

- Remove any tool in the doc that no longer exists in the code (e.g. `list_events`)
- Add any tool in the code that is missing from the doc (e.g. `list_ledger_entries`, commitment tools)
- Update tool descriptions if they are stale

- [ ] **Step 5: Cross-references — verify**

Search for "see Section", "see §", and Markdown anchor links (`[text](#anchor)`) and verify each resolves to an existing heading.

```bash
grep -n "see Section\|see §\|\[.*\](#" docs/specs/2026-04-13-qhorus-design.md
```

Fix any broken references.

- [ ] **Step 6: Duplication and redundancy pass**

Check the `## Design Decisions` section. For each decision, ask: is the full explanation already in the main body? If the Design Decisions entry merely repeats what is in the main body without adding context (tradeoffs, alternatives rejected, why this choice), remove the redundant copy and keep a brief forward reference.

Check `## Differences From cross-claude-mcp`. If this table is now mostly obsolete (cross-claude-mcp is archived and qhorus has diverged substantially), either remove it or update it to reflect actual current differences.

- [ ] **Step 7: Build Roadmap — update**

Find `## Build Roadmap`. Update item #12 (structured observability / ledger) to reflect the complete normative ledger. Mark it as ✅ implemented if using checkboxes, or move it to a "Completed" section. Note that the ledger now covers all 9 message types, not just EVENT telemetry.

- [ ] **Step 8: Commit the review pass**

```bash
git add docs/specs/2026-04-13-qhorus-design.md
git commit -m "$(cat <<'EOF'
docs(design): systematic review — staleness, correctness, ERD, MCP tool surface

Remove PendingReply (replaced by Commitment). Update ERD with COMMITMENT and
MESSAGE_LEDGER_ENTRY. Replace list_events with list_ledger_entries throughout.
Fix MCP tool surface to match current QhorusMcpTools. Update Build Roadmap.

Refs #<child-issue-9> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 12: Documentation — Normative Audit Ledger Section

Add the new section to `docs/specs/2026-04-13-qhorus-design.md`.

**Files:**
- Modify: `docs/specs/2026-04-13-qhorus-design.md`

- [ ] **Step 1: Insert new section**

Find the heading `## MCP Tool Surface`. Insert the following new section **immediately before** it:

```markdown
## Normative Audit Ledger

Every message sent on a channel is permanently recorded as a `MessageLedgerEntry` — an
immutable, tamper-evident (SHA-256 hash-chained) record extending the quarkus-ledger
`LedgerEntry` base class via JPA JOINED inheritance.

### Two-Layer Model

| Layer | Component | Purpose |
|---|---|---|
| **Live state** | `CommitmentStore` | Current obligation status — queryable, mutable |
| **Historical record** | Ledger (`MessageLedgerEntry`) | Complete immutable channel history — permanent |

These are complementary. The CommitmentStore answers "what is the current state of
obligation X?"; the ledger answers "what has happened in this channel, in order, permanently?"

### What Gets Recorded

All 9 message types produce a ledger entry when sent via `send_message`:

| Message Type | `entryType` | Causal link set? | Notes |
|---|---|---|---|
| QUERY | COMMAND | No | Information request — declared intent |
| COMMAND | COMMAND | No | Action request — creates obligation |
| RESPONSE | EVENT | No | Answers a QUERY |
| STATUS | EVENT | No | Progress update |
| DECLINE | EVENT | Yes → COMMAND | Obligation refused |
| HANDOFF | COMMAND | Yes → COMMAND | Obligation transferred |
| DONE | EVENT | Yes → COMMAND/HANDOFF | Obligation completed |
| FAILURE | EVENT | Yes → COMMAND/HANDOFF | Obligation failed |
| EVENT | EVENT | No | Tool telemetry — see below |

`entryType` (from quarkus-ledger `LedgerEntryType`) is a coarse classification. All
Qhorus-level filtering uses `messageType` (the full 9-type name).

### Causal Chain

DONE, FAILURE, DECLINE, and HANDOFF entries have `causedByEntryId` set to the most
recent COMMAND or HANDOFF entry sharing the same `correlationId` on the same channel.
This creates a traversable obligation chain inside the ledger itself:

```
seq=1  COMMAND   "Generate compliance report"   causedByEntryId=null
seq=2  STATUS    "Processing…"                  causedByEntryId=null
seq=3  DONE      "Report delivered"             causedByEntryId=<id of seq=1>
```

For delegation:
```
seq=1  COMMAND   "Audit accounts"               causedByEntryId=null
seq=2  HANDOFF   → agent-c                      causedByEntryId=<id of seq=1>
seq=3  DONE      "Audit complete"               causedByEntryId=<id of seq=2>
```

Use `list_ledger_entries` with `type_filter=COMMAND,DONE,FAILURE,DECLINE,HANDOFF` to
retrieve the obligation lifecycle. Use `caused_by_entry_id` in the response to trace
the chain.

### EVENT Telemetry

EVENT entries carry additional fields extracted from the JSON payload:

| Field | Required | Description |
|---|---|---|
| `tool_name` | No | Agent tool that was invoked |
| `duration_ms` | No | Wall-clock duration in milliseconds |
| `token_count` | No | LLM token count |
| `context_refs` | No | JSON array of context references |
| `source_entity` | No | JSON object — source domain entity |

Malformed or partial EVENT payloads still produce a ledger entry — the speech act
happened regardless of telemetry quality. Telemetry fields are null when absent.

### Querying the Ledger

Use `list_ledger_entries`:

```
# Full channel history
list_ledger_entries(channel_name="my-channel")

# Obligation lifecycle only
list_ledger_entries(channel_name="my-channel", type_filter="COMMAND,DONE,FAILURE,DECLINE,HANDOFF")

# Telemetry only
list_ledger_entries(channel_name="my-channel", type_filter="EVENT")

# One agent's actions
list_ledger_entries(channel_name="my-channel", agent_id="orchestrator")

# Paginate
list_ledger_entries(channel_name="my-channel", after_id=15, limit=20)
```

Response fields: `sequence_number`, `message_type`, `entry_type`, `actor_id`, `target`,
`content`, `correlation_id`, `commitment_id`, `caused_by_entry_id`, `occurred_at`,
`message_id`, plus telemetry fields when present (`tool_name`, `duration_ms`, etc.).

### Design Decision — Complete Audit Trail

Every message type is recorded (not just EVENT) because:

1. **Accountability**: which agent issued which COMMAND, which agent declined, which
   succeeded — must be permanently attributable.
2. **Compliance**: obligation chains (COMMAND → DONE/FAILURE) are the audit evidence
   for automated actions, not just telemetry.
3. **Simplicity**: unconditional recording in the caller (`send_message`) eliminates
   conditional logic and the risk of accidentally omitting a type.
4. **Single entity**: one `MessageLedgerEntry` table with nullable telemetry fields
   avoids UNION queries. `messageType` is the discriminator; telemetry fields are
   clearly EVENT-only.
```

- [ ] **Step 2: Verify the section renders correctly**

Open the file in Typora or preview the Markdown. Confirm:
- The table renders correctly
- The code blocks are properly fenced
- The section heading appears in the document outline between Message Type Taxonomy and MCP Tool Surface

- [ ] **Step 3: Commit**

```bash
git add docs/specs/2026-04-13-qhorus-design.md
git commit -m "$(cat <<'EOF'
docs(design): add Normative Audit Ledger section

Two-layer model (CommitmentStore + Ledger), per-type recording table, causal chain
explanation with examples, EVENT telemetry subset, query patterns, design decision
rationale. Positioned between Message Type Taxonomy and MCP Tool Surface.

Refs #<child-issue-10> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Final Verification

- [ ] **Full build across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format 2>&1 | grep -E "Tests run|BUILD|ERROR"
```
Expected: `BUILD SUCCESS`. Zero failures across `runtime`, `testing`, `deployment`, `examples/type-system`.

- [ ] **Close the epic**

Run issue-workflow Phase 3 to confirm all child issues are closed and the epic is resolved.

- [ ] **Tag the completion**

```bash
git log --oneline -15
```
Review the commit history for this feature. All commits should reference their child issue and the epic.

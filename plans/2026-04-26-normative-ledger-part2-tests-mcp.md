# Normative Ledger — Part 2: Integration Tests + MCP Tools

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integration tests covering all 9 types, causal chains, filters and robustness; MCP wiring so `sendMessage` calls `record()` for every type; new `list_ledger_entries` tool replaces `list_events`; old test files deleted.

**Prerequisite:** Part 1 complete — `MessageLedgerEntry`, `MessageLedgerEntryRepository`, and `LedgerWriteService.record()` all exist and unit-tested.

**Architecture:** Tasks 5 and 7 write the tests RED (they reference the new repo and new MCP tool that don't work yet). Task 6 is the single wiring commit that turns everything GREEN and deletes the now-broken old test files in the same atomic step.

**Tech Stack:** Java 21, Quarkus 3.32.2, `@QuarkusTest @TestTransaction`, H2, JUnit 5, AssertJ.

**Spec:** `docs/superpowers/specs/2026-04-26-normative-ledger-design.md`

---

## File Map

| File | Action |
|---|---|
| `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerCaptureTest.java` | Create |
| `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java` | Modify — `toLedgerEntryMap` replaces `toEventMap` |
| `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java` | Modify — ledgerRepo type, sendMessage wiring, `list_ledger_entries` replaces `list_events` |
| `runtime/src/test/java/io/quarkiverse/qhorus/ledger/AgentLedgerCaptureTest.java` | Delete |
| `runtime/src/test/java/io/quarkiverse/qhorus/ledger/ListEventsTest.java` | Delete |
| `runtime/src/test/java/io/quarkiverse/qhorus/ledger/ListLedgerEntriesTest.java` | Create |

---

## Task 5: Integration Tests — RED Phase

Write `MessageLedgerCaptureTest`. These tests call `tools.sendMessage(...)` and then query via `ledgerRepo`. They will **fail** for non-EVENT types until Task 6 wires `sendMessage` to call `record()`.

**Files:**
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerCaptureTest.java`

- [ ] **Step 1: Create the test class**

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry;
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Integration tests for the normative ledger — verifies that every message type
 * sent via {@code sendMessage} produces a {@link MessageLedgerEntry} with correct
 * fields, sequence numbers, causal chain links, and filter behaviour.
 *
 * <p>
 * Uses unique channel names per test for isolation. {@code @TestTransaction} rolls back
 * Message table writes; ledger entries are committed by their own {@code REQUIRES_NEW}
 * transactions but are scoped to unique channel UUIDs so they don't interfere across tests.
 *
 * <p>
 * Refs #<child-issue-4> — Epic #<epic>.
 */
@QuarkusTest
@TestTransaction
class MessageLedgerCaptureTest {

    @Inject
    QhorusMcpTools tools;

    @Inject
    MessageLedgerEntryRepository ledgerRepo;

    // =========================================================================
    // Happy path — one test per message type
    // =========================================================================

    @Test
    void sendQuery_createsLedgerEntry() {
        setup("mlc-query-1", "agent-a");
        tools.sendMessage("mlc-query-1", "agent-a", "query", "How many orders today?",
                "corr-q1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-query-1"));
        assertEquals(1, entries.size());
        MessageLedgerEntry e = entries.get(0);
        assertEquals("QUERY", e.messageType);
        assertEquals("agent-a", e.actorId);
        assertEquals("corr-q1", e.correlationId);
        assertEquals("How many orders today?", e.content);
        assertNull(e.toolName);
    }

    @Test
    void sendCommand_createsLedgerEntry() {
        setup("mlc-cmd-1", "agent-a");
        tools.sendMessage("mlc-cmd-1", "agent-a", "command", "Generate the monthly report",
                "corr-c1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-cmd-1"));
        assertEquals(1, entries.size());
        assertEquals("COMMAND", entries.get(0).messageType);
        assertEquals("Generate the monthly report", entries.get(0).content);
    }

    @Test
    void sendResponse_createsLedgerEntry() {
        setup("mlc-resp-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-resp-1", "agent-a", "query", "Status?", "corr-r1", null, null, null);
        tools.sendMessage("mlc-resp-1", "agent-b", "response", "All good", "corr-r1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-resp-1"));
        assertEquals(2, entries.size());
        assertEquals("RESPONSE", entries.get(1).messageType);
        assertEquals("All good", entries.get(1).content);
    }

    @Test
    void sendStatus_createsLedgerEntry() {
        setup("mlc-status-1", "agent-a");
        tools.sendMessage("mlc-status-1", "agent-a", "command", "Run migration",
                "corr-s1", null, null, null);
        tools.sendMessage("mlc-status-1", "agent-a", "status", "50% complete",
                "corr-s1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-status-1"));
        assertEquals(2, entries.size());
        assertEquals("STATUS", entries.get(1).messageType);
    }

    @Test
    void sendDecline_createsLedgerEntry() {
        setup("mlc-dec-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-dec-1", "agent-a", "command", "Delete all records",
                "corr-d1", null, null, null);
        tools.sendMessage("mlc-dec-1", "agent-b", "decline", "I do not have write permissions",
                "corr-d1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-dec-1"));
        assertEquals(2, entries.size());
        assertEquals("DECLINE", entries.get(1).messageType);
        assertEquals("I do not have write permissions", entries.get(1).content);
    }

    @Test
    void sendHandoff_createsLedgerEntry() {
        setup("mlc-hand-1", "agent-a", "agent-b", "agent-c");
        tools.sendMessage("mlc-hand-1", "agent-a", "command", "Audit the accounts",
                "corr-h1", null, null, null);
        tools.sendMessage("mlc-hand-1", "agent-b", "handoff", null,
                "corr-h1", null, null, "instance:agent-c");

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-hand-1"));
        assertEquals(2, entries.size());
        MessageLedgerEntry handoff = entries.get(1);
        assertEquals("HANDOFF", handoff.messageType);
        assertEquals("instance:agent-c", handoff.target);
    }

    @Test
    void sendDone_createsLedgerEntry() {
        setup("mlc-done-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-done-1", "agent-a", "command", "Process refunds",
                "corr-done1", null, null, null);
        tools.sendMessage("mlc-done-1", "agent-b", "done", "All 42 refunds processed",
                "corr-done1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-done-1"));
        assertEquals(2, entries.size());
        assertEquals("DONE", entries.get(1).messageType);
        assertEquals("All 42 refunds processed", entries.get(1).content);
    }

    @Test
    void sendFailure_createsLedgerEntry() {
        setup("mlc-fail-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-fail-1", "agent-a", "command", "Run batch job",
                "corr-fail1", null, null, null);
        tools.sendMessage("mlc-fail-1", "agent-b", "failure", "Database connection lost",
                "corr-fail1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-fail-1"));
        assertEquals(2, entries.size());
        assertEquals("FAILURE", entries.get(1).messageType);
        assertEquals("Database connection lost", entries.get(1).content);
    }

    @Test
    void sendEvent_withValidPayload_createsTelemetryEntry() {
        setup("mlc-event-1", "agent-a");
        tools.sendMessage("mlc-event-1", "agent-a", "event",
                "{\"tool_name\":\"read_file\",\"duration_ms\":42,\"token_count\":1200}",
                null, null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-event-1"));
        assertEquals(1, entries.size());
        MessageLedgerEntry e = entries.get(0);
        assertEquals("EVENT", e.messageType);
        assertEquals("read_file", e.toolName);
        assertEquals(42L, e.durationMs);
        assertEquals(1200L, e.tokenCount);
        assertNull(e.content);
    }

    @Test
    void sendEvent_malformedJson_entryStillCreated() {
        setup("mlc-event-malformed-1", "agent-a");
        tools.sendMessage("mlc-event-malformed-1", "agent-a", "event",
                "not-json", null, null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-event-malformed-1"));
        assertEquals(1, entries.size());
        assertNull(entries.get(0).toolName);
        assertNull(entries.get(0).durationMs);
    }

    @Test
    void sendEvent_missingMandatoryTelemetryFields_entryStillCreated() {
        setup("mlc-event-partial-1", "agent-a");
        tools.sendMessage("mlc-event-partial-1", "agent-a", "event",
                "{\"duration_ms\":10}", null, null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-event-partial-1"));
        assertEquals(1, entries.size());
        assertNull(entries.get(0).toolName);
        assertEquals(10L, entries.get(0).durationMs);
    }

    // =========================================================================
    // Sequence numbering
    // =========================================================================

    @Test
    void multipleMessages_sequenceNumbersIncrement() {
        setup("mlc-seq-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-seq-1", "agent-a", "command", "Go", "corr-seq1", null, null, null);
        tools.sendMessage("mlc-seq-1", "agent-a", "status", "Working", "corr-seq1", null, null, null);
        tools.sendMessage("mlc-seq-1", "agent-b", "done", "Done", "corr-seq1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-seq-1"));
        assertEquals(3, entries.size());
        assertEquals(1, entries.get(0).sequenceNumber);
        assertEquals(2, entries.get(1).sequenceNumber);
        assertEquals(3, entries.get(2).sequenceNumber);
    }

    @Test
    void sequenceNumbers_independentAcrossChannels() {
        setup("mlc-seq-2a", "agent-a");
        setup("mlc-seq-2b", "agent-b");

        tools.sendMessage("mlc-seq-2a", "agent-a", "command", "X", null, null, null, null);
        tools.sendMessage("mlc-seq-2b", "agent-b", "command", "Y", null, null, null, null);

        List<MessageLedgerEntry> a = ledgerRepo.findByChannelId(channelId("mlc-seq-2a"));
        List<MessageLedgerEntry> b = ledgerRepo.findByChannelId(channelId("mlc-seq-2b"));

        assertEquals(1, a.get(0).sequenceNumber);
        assertEquals(1, b.get(0).sequenceNumber);
    }

    // =========================================================================
    // Causal chain — causedByEntryId
    // =========================================================================

    @Test
    void commandThenDone_donePointsToCommand() {
        setup("mlc-causal-done-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-causal-done-1", "agent-a", "command", "Run report",
                "corr-cd1", null, null, null);
        tools.sendMessage("mlc-causal-done-1", "agent-b", "done", "Report delivered",
                "corr-cd1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-causal-done-1"));
        assertEquals(2, entries.size());

        MessageLedgerEntry cmd = entries.get(0);
        MessageLedgerEntry done = entries.get(1);

        assertEquals("COMMAND", cmd.messageType);
        assertEquals("DONE", done.messageType);
        assertNotNull(done.causedByEntryId);
        assertEquals(cmd.id, done.causedByEntryId);
    }

    @Test
    void commandThenFailure_failurePointsToCommand() {
        setup("mlc-causal-fail-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-causal-fail-1", "agent-a", "command", "Run migration",
                "corr-cf1", null, null, null);
        tools.sendMessage("mlc-causal-fail-1", "agent-b", "failure", "DB error",
                "corr-cf1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-causal-fail-1"));
        assertEquals(cmd(entries).id, terminal(entries, "FAILURE").causedByEntryId);
    }

    @Test
    void commandThenDecline_declinePointsToCommand() {
        setup("mlc-causal-dec-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-causal-dec-1", "agent-a", "command", "Delete everything",
                "corr-cdec1", null, null, null);
        tools.sendMessage("mlc-causal-dec-1", "agent-b", "decline", "Out of scope",
                "corr-cdec1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-causal-dec-1"));
        assertEquals(cmd(entries).id, terminal(entries, "DECLINE").causedByEntryId);
    }

    @Test
    void commandHandoffDone_fullChain() {
        setup("mlc-causal-chain-1", "agent-a", "agent-b", "agent-c");
        tools.sendMessage("mlc-causal-chain-1", "agent-a", "command", "Audit",
                "corr-chain1", null, null, null);
        tools.sendMessage("mlc-causal-chain-1", "agent-b", "handoff", null,
                "corr-chain1", null, null, "instance:agent-c");
        tools.sendMessage("mlc-causal-chain-1", "agent-c", "done", "Audit complete",
                "corr-chain1", null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-causal-chain-1"));
        assertEquals(3, entries.size());

        MessageLedgerEntry cmd = entries.get(0);
        MessageLedgerEntry handoff = entries.get(1);
        MessageLedgerEntry done = entries.get(2);

        assertEquals("COMMAND", cmd.messageType);
        assertEquals("HANDOFF", handoff.messageType);
        assertEquals("DONE", done.messageType);

        // HANDOFF points to COMMAND
        assertEquals(cmd.id, handoff.causedByEntryId);
        // DONE points to HANDOFF (most recent COMMAND-or-HANDOFF)
        assertEquals(handoff.id, done.causedByEntryId);
    }

    @Test
    void doneWithNoCorrelationId_causedByEntryIdNull() {
        setup("mlc-causal-nocorr-1", "agent-a");
        tools.sendMessage("mlc-causal-nocorr-1", "agent-a", "done", "Done with no correlation",
                null, null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-causal-nocorr-1"));
        assertEquals(1, entries.size());
        assertNull(entries.get(0).causedByEntryId);
    }

    // =========================================================================
    // listEntries filter correctness (via repository directly)
    // =========================================================================

    @Test
    void listEntries_typeFilter_commandAndDone_excludesOtherTypes() {
        setup("mlc-filter-type-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-filter-type-1", "agent-a", "command", "Go", "corr-ft1", null, null, null);
        tools.sendMessage("mlc-filter-type-1", "agent-a", "status", "Working", "corr-ft1", null, null, null);
        tools.sendMessage("mlc-filter-type-1", "agent-b", "done", "Done", "corr-ft1", null, null, null);

        UUID chId = channelId("mlc-filter-type-1");
        List<MessageLedgerEntry> entries = ledgerRepo.listEntries(
                chId, Set.of("COMMAND", "DONE"), null, null, null, 20);

        assertEquals(2, entries.size());
        assertTrue(entries.stream().allMatch(e -> Set.of("COMMAND", "DONE").contains(e.messageType)));
    }

    @Test
    void listEntries_agentFilter_returnsOnlyThatAgent() {
        setup("mlc-filter-agent-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-filter-agent-1", "agent-a", "command", "Go", "corr-fa1", null, null, null);
        tools.sendMessage("mlc-filter-agent-1", "agent-b", "done", "Done", "corr-fa1", null, null, null);

        UUID chId = channelId("mlc-filter-agent-1");
        List<MessageLedgerEntry> entries = ledgerRepo.listEntries(chId, null, null, "agent-a", null, 20);

        assertEquals(1, entries.size());
        assertEquals("agent-a", entries.get(0).actorId);
    }

    @Test
    void listEntries_afterSequenceCursor_returnsLaterEntries() {
        setup("mlc-cursor-1", "agent-a", "agent-b");
        tools.sendMessage("mlc-cursor-1", "agent-a", "command", "Go", "corr-cur1", null, null, null);
        tools.sendMessage("mlc-cursor-1", "agent-a", "status", "Working", "corr-cur1", null, null, null);
        tools.sendMessage("mlc-cursor-1", "agent-b", "done", "Done", "corr-cur1", null, null, null);

        UUID chId = channelId("mlc-cursor-1");
        List<MessageLedgerEntry> page2 = ledgerRepo.listEntries(chId, null, 1L, null, null, 20);

        assertEquals(2, page2.size());
        assertTrue(page2.stream().allMatch(e -> e.sequenceNumber > 1));
    }

    @Test
    void listEntries_limit_capsResults() {
        setup("mlc-limit-1", "agent-a");
        for (int i = 0; i < 5; i++) {
            tools.sendMessage("mlc-limit-1", "agent-a", "event",
                    "{\"tool_name\":\"t\",\"duration_ms\":1}", null, null, null, null);
        }

        UUID chId = channelId("mlc-limit-1");
        List<MessageLedgerEntry> entries = ledgerRepo.listEntries(chId, null, null, null, null, 3);

        assertEquals(3, entries.size());
    }

    // =========================================================================
    // Robustness — pipeline not affected by ledger issues
    // =========================================================================

    @Test
    void sendMessage_eventWithEmptyContent_doesNotThrow() {
        setup("mlc-robust-1", "agent-a");
        assertDoesNotThrow(() ->
                tools.sendMessage("mlc-robust-1", "agent-a", "event", "", null, null, null, null));

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-robust-1"));
        assertEquals(1, entries.size());
    }

    @Test
    void sendMessage_allTypesProduceLedgerEntry() {
        setup("mlc-all-types-1", "agent-a", "agent-b");
        String corr = "corr-all";
        tools.sendMessage("mlc-all-types-1", "agent-a", "query", "Status?", corr, null, null, null);
        tools.sendMessage("mlc-all-types-1", "agent-b", "response", "Good", corr, null, null, null);
        tools.sendMessage("mlc-all-types-1", "agent-a", "command", "Go", corr, null, null, null);
        tools.sendMessage("mlc-all-types-1", "agent-b", "status", "Working", corr, null, null, null);
        tools.sendMessage("mlc-all-types-1", "agent-b", "done", "Done", corr, null, null, null);
        tools.sendMessage("mlc-all-types-1", "agent-a", "event",
                "{\"tool_name\":\"t\",\"duration_ms\":1}", null, null, null, null);

        List<MessageLedgerEntry> entries = ledgerRepo.findByChannelId(channelId("mlc-all-types-1"));
        assertEquals(6, entries.size());
    }

    // =========================================================================
    // Fixtures
    // =========================================================================

    private void setup(final String channel, final String... agents) {
        tools.createChannel(channel, "APPEND", null, null);
        for (final String agent : agents) {
            tools.registerInstance(channel, agent, null, null, null);
        }
    }

    private UUID channelId(final String channelName) {
        return Channel.<Channel> find("name", channelName)
                .firstResultOptional()
                .map(ch -> ch.id)
                .orElseThrow(() -> new IllegalStateException("Channel not found: " + channelName));
    }

    private MessageLedgerEntry cmd(final List<MessageLedgerEntry> entries) {
        return entries.stream().filter(e -> "COMMAND".equals(e.messageType)).findFirst()
                .orElseThrow(() -> new AssertionError("No COMMAND entry found"));
    }

    private MessageLedgerEntry terminal(final List<MessageLedgerEntry> entries, final String type) {
        return entries.stream().filter(e -> type.equals(e.messageType)).findFirst()
                .orElseThrow(() -> new AssertionError("No " + type + " entry found"));
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail for non-EVENT types**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageLedgerCaptureTest -Dno-format -q 2>&1 | tail -10
```
Expected: `BUILD FAILURE` — tests that send COMMAND, QUERY, RESPONSE, STATUS, DECLINE, HANDOFF, DONE, FAILURE produce 0 ledger entries instead of 1. EVENT tests should pass. Causal chain tests fail because non-EVENT entries don't exist.

- [ ] **Step 3: Commit the RED-phase tests**

```bash
git add runtime/src/test/java/io/quarkiverse/qhorus/ledger/MessageLedgerCaptureTest.java
git commit -m "$(cat <<'EOF'
test(ledger): MessageLedgerCaptureTest RED — all 9 types, causal chain, filters

Tests fail for non-EVENT types until sendMessage is wired to call record() for all types.

Refs #<child-issue-4> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Wire MCP Tools — GREEN Phase

This is an atomic commit: update `QhorusMcpToolsBase` and `QhorusMcpTools`, then immediately delete the broken old test files. All `MessageLedgerCaptureTest` tests go green in this step.

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Delete: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/AgentLedgerCaptureTest.java`
- Delete: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/ListEventsTest.java`

- [ ] **Step 1: Update `QhorusMcpToolsBase` — replace `toEventMap` with `toLedgerEntryMap`**

In `QhorusMcpToolsBase.java`, find the `toEventMap` method (around line 374) and replace it entirely with:

```java
protected Map<String, Object> toLedgerEntryMap(final MessageLedgerEntry e) {
    final Map<String, Object> m = new java.util.LinkedHashMap<>();
    m.put("sequence_number", e.sequenceNumber);
    m.put("message_type", e.messageType);
    m.put("entry_type", e.entryType != null ? e.entryType.name() : null);
    m.put("actor_id", e.actorId);
    m.put("target", e.target);
    m.put("content", e.content);
    m.put("correlation_id", e.correlationId);
    m.put("commitment_id", e.commitmentId != null ? e.commitmentId.toString() : null);
    m.put("caused_by_entry_id", e.causedByEntryId != null ? e.causedByEntryId.toString() : null);
    m.put("occurred_at", e.occurredAt != null ? e.occurredAt.toString() : null);
    m.put("message_id", e.messageId);
    // Telemetry — only include keys when values are present (EVENT-only fields)
    if (e.toolName != null) {
        m.put("tool_name", e.toolName);
    }
    if (e.durationMs != null) {
        m.put("duration_ms", e.durationMs);
    }
    if (e.tokenCount != null) {
        m.put("token_count", e.tokenCount);
    }
    if (e.contextRefs != null) {
        m.put("context_refs", e.contextRefs);
    }
    if (e.sourceEntity != null) {
        m.put("source_entity", e.sourceEntity);
    }
    return m;
}
```

Also add the import at the top of `QhorusMcpToolsBase.java`:

```java
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntry;
```

Remove the import for `AgentMessageLedgerEntry` from `QhorusMcpToolsBase.java` if it exists.

- [ ] **Step 2: Update `QhorusMcpTools` — change `ledgerRepo` field type**

Find the `ledgerRepo` field declaration (around line 81) and change it:

```java
// Before:
io.quarkiverse.qhorus.runtime.ledger.AgentMessageLedgerEntryRepository ledgerRepo;

// After:
@Inject
MessageLedgerEntryRepository ledgerRepo;
```

Add the import:
```java
import io.quarkiverse.qhorus.runtime.ledger.MessageLedgerEntryRepository;
```

Remove the import for `AgentMessageLedgerEntryRepository`.

- [ ] **Step 3: Update `sendMessage` — remove EVENT conditional, wire `record()` for all types**

Find the ledger write block in `sendMessage` (the `if (msgType == MessageType.EVENT)` block, around line 565) and replace it:

```java
// Before:
if (msgType == MessageType.EVENT) {
    try {
        ledgerWriteService.recordEvent(ch, msg);
    } catch (Exception e) {
        LOG.warnf("Ledger write failed for EVENT message %d in channel '%s': %s",
                msg.id, ch.name, e.getMessage());
    }
}

// After:
try {
    ledgerWriteService.record(ch, msg);
} catch (Exception e) {
    LOG.warnf("Ledger write failed for message %d in channel '%s': %s",
            msg.id, ch.name, e.getMessage());
}
```

- [ ] **Step 4: Remove `listEvents` method from `QhorusMcpTools`**

Delete the entire `listEvents` method (from `@Tool(name = "list_events"` through its closing `}`). It is approximately 30 lines starting around line 1270.

- [ ] **Step 5: Add `listLedgerEntries` method to `QhorusMcpTools`**

Add the following method in the "Ledger event audit trail tools" section (where `listEvents` was):

```java
@Tool(name = "list_ledger_entries", description = "Query the immutable audit ledger for a channel. "
        + "Returns all ledger entries in chronological order — every speech act, every tool invocation. "
        + "Use type_filter to narrow by message type: 'COMMAND,DONE,FAILURE' for obligation lifecycle, "
        + "'EVENT' for telemetry only, omit for the full channel history. "
        + "Supports optional filters for agent_id, since (ISO-8601), and cursor-based pagination via after_id.")
@Transactional
public List<Map<String, Object>> listLedgerEntries(
        @ToolArg(name = "channel_name", description = "Name of the channel to query") String channelName,
        @ToolArg(name = "type_filter", description = "Comma-separated MessageType names to include "
                + "(e.g. 'COMMAND,DONE,FAILURE'). Omit to return all types.", required = false) String typeFilter,
        @ToolArg(name = "agent_id", description = "Filter by sender — returns only entries from this agent",
                required = false) String agentId,
        @ToolArg(name = "since", description = "ISO-8601 timestamp — return only entries at or after this time",
                required = false) String since,
        @ToolArg(name = "after_id", description = "Return entries with sequence_number > after_id (cursor pagination)",
                required = false) Long afterId,
        @ToolArg(name = "limit", description = "Maximum entries to return (default 20, max 100)",
                required = false) Integer limit) {

    final Channel ch = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));

    java.util.Set<String> types = null;
    if (typeFilter != null && !typeFilter.isBlank()) {
        types = java.util.Arrays.stream(typeFilter.split(","))
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .collect(java.util.stream.Collectors.toSet());
    }

    final int effectiveLimit = (limit != null && limit > 0) ? Math.min(limit, 100) : 20;

    java.time.Instant sinceInstant = null;
    if (since != null && !since.isBlank()) {
        try {
            sinceInstant = java.time.Instant.parse(since);
        } catch (final java.time.format.DateTimeParseException ex) {
            throw new IllegalArgumentException(
                    "Invalid 'since' timestamp '" + since + "' — use ISO-8601 format, e.g. 2026-04-15T10:00:00Z");
        }
    }

    final List<MessageLedgerEntry> entries = ledgerRepo.listEntries(
            ch.id, types, afterId, agentId, sinceInstant, effectiveLimit);

    return entries.stream().map(this::toLedgerEntryMap).toList();
}
```

- [ ] **Step 6: Delete old test files**

```bash
rm runtime/src/test/java/io/quarkiverse/qhorus/ledger/AgentLedgerCaptureTest.java
rm runtime/src/test/java/io/quarkiverse/qhorus/ledger/ListEventsTest.java
```

- [ ] **Step 7: Build and run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format -q 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`. `MessageLedgerCaptureTest` all passing. `ChannelTimelineTest` still passing (unchanged). No compilation errors from deleted files.

If there are compilation errors, check:
- `QhorusMcpToolsBase` still references `toEventMap` anywhere — replace with `toLedgerEntryMap`
- `QhorusMcpTools` still references `AgentMessageLedgerEntryRepository` — update to `MessageLedgerEntryRepository`
- Any test still imports `AgentLedgerCaptureTest` or `ListEventsTest` — those files are deleted

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java
git rm runtime/src/test/java/io/quarkiverse/qhorus/ledger/AgentLedgerCaptureTest.java \
       runtime/src/test/java/io/quarkiverse/qhorus/ledger/ListEventsTest.java
git commit -m "$(cat <<'EOF'
feat(ledger): wire record() for all types; add list_ledger_entries; remove list_events

sendMessage now calls LedgerWriteService.record() for every message type — no more
EVENT-only conditional. list_events removed; list_ledger_entries replaces it with
full type_filter support. Old AgentLedgerCaptureTest and ListEventsTest deleted.

Refs #<child-issue-4> #<child-issue-5> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: E2E Tests — `list_ledger_entries` MCP Tool

Write end-to-end tests exercising the `list_ledger_entries` MCP tool through the full stack.

**Files:**
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/ledger/ListLedgerEntriesTest.java`

- [ ] **Step 1: Create the test class**

```java
package io.quarkiverse.qhorus.ledger;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Map;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.mcp.server.ToolCallException;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

/**
 * End-to-end tests for the {@code list_ledger_entries} MCP tool.
 *
 * <p>
 * Verifies the full stack: sendMessage → LedgerWriteService.record() →
 * MessageLedgerEntryRepository → list_ledger_entries response shape.
 *
 * <p>
 * Refs #<child-issue-5> — Epic #<epic>.
 */
@QuarkusTest
@TestTransaction
class ListLedgerEntriesTest {

    @Inject
    QhorusMcpTools tools;

    // =========================================================================
    // Happy path — basic retrieval
    // =========================================================================

    @Test
    void listLedgerEntries_allTypes_returnsAll() {
        setup("lle-basic-1", "agent-a", "agent-b");
        tools.sendMessage("lle-basic-1", "agent-a", "command", "Run audit", "corr-1", null, null, null);
        tools.sendMessage("lle-basic-1", "agent-b", "done", "Audit done", "corr-1", null, null, null);
        tools.sendMessage("lle-basic-1", "agent-a", "event",
                "{\"tool_name\":\"read\",\"duration_ms\":10}", null, null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries("lle-basic-1", null, null, null, null, 20);

        assertEquals(3, entries.size());
    }

    @Test
    void listLedgerEntries_returnsRequiredFields() {
        setup("lle-fields-1", "agent-a");
        tools.sendMessage("lle-fields-1", "agent-a", "command", "Do X", "corr-f1", null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries("lle-fields-1", null, null, null, null, 20);

        assertEquals(1, entries.size());
        Map<String, Object> e = entries.get(0);
        assertEquals("COMMAND", e.get("message_type"));
        assertEquals("agent-a", e.get("actor_id"));
        assertEquals("corr-f1", e.get("correlation_id"));
        assertEquals("Do X", e.get("content"));
        assertNotNull(e.get("sequence_number"));
        assertNotNull(e.get("occurred_at"));
        assertNotNull(e.get("message_id"));
    }

    @Test
    void listLedgerEntries_eventEntry_includesTelemetryFields() {
        setup("lle-telemetry-1", "agent-a");
        tools.sendMessage("lle-telemetry-1", "agent-a", "event",
                "{\"tool_name\":\"analyze\",\"duration_ms\":42,\"token_count\":500}",
                null, null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries("lle-telemetry-1", null, null, null, null, 20);

        Map<String, Object> e = entries.get(0);
        assertEquals("EVENT", e.get("message_type"));
        assertEquals("analyze", e.get("tool_name"));
        assertEquals(42L, e.get("duration_ms"));
        assertEquals(500L, e.get("token_count"));
        assertNull(e.get("content"));
    }

    @Test
    void listLedgerEntries_nonEventEntry_doesNotIncludeTelemetryKeys() {
        setup("lle-no-telemetry-1", "agent-a");
        tools.sendMessage("lle-no-telemetry-1", "agent-a", "command", "Do it", null, null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries("lle-no-telemetry-1", null, null, null, null, 20);

        Map<String, Object> e = entries.get(0);
        assertFalse(e.containsKey("tool_name"), "Non-EVENT entries must not include tool_name key");
        assertFalse(e.containsKey("duration_ms"), "Non-EVENT entries must not include duration_ms key");
    }

    @Test
    void listLedgerEntries_returnedInChronologicalOrder() {
        setup("lle-order-1", "agent-a", "agent-b");
        tools.sendMessage("lle-order-1", "agent-a", "command", "first", "corr-ord", null, null, null);
        tools.sendMessage("lle-order-1", "agent-a", "status", "second", "corr-ord", null, null, null);
        tools.sendMessage("lle-order-1", "agent-b", "done", "third", "corr-ord", null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries("lle-order-1", null, null, null, null, 20);

        assertEquals(1, entries.get(0).get("sequence_number"));
        assertEquals(2, entries.get(1).get("sequence_number"));
        assertEquals(3, entries.get(2).get("sequence_number"));
    }

    @Test
    void listLedgerEntries_unknownChannel_throws() {
        assertThrows(ToolCallException.class,
                () -> tools.listLedgerEntries("no-such-channel", null, null, null, null, 20));
    }

    // =========================================================================
    // type_filter
    // =========================================================================

    @Test
    void listLedgerEntries_typeFilter_obligationLifecycle() {
        setup("lle-type-1", "agent-a", "agent-b");
        tools.sendMessage("lle-type-1", "agent-a", "command", "Go", "corr-tf1", null, null, null);
        tools.sendMessage("lle-type-1", "agent-a", "status", "Working", "corr-tf1", null, null, null);
        tools.sendMessage("lle-type-1", "agent-b", "done", "Done", "corr-tf1", null, null, null);
        tools.sendMessage("lle-type-1", "agent-a", "event",
                "{\"tool_name\":\"t\",\"duration_ms\":1}", null, null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries(
                "lle-type-1", "COMMAND,DONE", null, null, null, 20);

        assertEquals(2, entries.size());
        assertTrue(entries.stream().allMatch(e ->
                "COMMAND".equals(e.get("message_type")) || "DONE".equals(e.get("message_type"))));
    }

    @Test
    void listLedgerEntries_typeFilter_eventOnly() {
        setup("lle-type-event-1", "agent-a");
        tools.sendMessage("lle-type-event-1", "agent-a", "command", "Go", null, null, null, null);
        tools.sendMessage("lle-type-event-1", "agent-a", "event",
                "{\"tool_name\":\"t\",\"duration_ms\":1}", null, null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries(
                "lle-type-event-1", "EVENT", null, null, null, 20);

        assertEquals(1, entries.size());
        assertEquals("EVENT", entries.get(0).get("message_type"));
    }

    @Test
    void listLedgerEntries_typeFilter_noMatch_returnsEmpty() {
        setup("lle-type-empty-1", "agent-a");
        tools.sendMessage("lle-type-empty-1", "agent-a", "command", "Go", null, null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries(
                "lle-type-empty-1", "DECLINE", null, null, null, 20);

        assertTrue(entries.isEmpty());
    }

    // =========================================================================
    // agent_id filter
    // =========================================================================

    @Test
    void listLedgerEntries_agentFilter_returnsOnlyThatAgent() {
        setup("lle-agent-1", "agent-a", "agent-b");
        tools.sendMessage("lle-agent-1", "agent-a", "command", "Go", "corr-ag1", null, null, null);
        tools.sendMessage("lle-agent-1", "agent-b", "done", "Done", "corr-ag1", null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries(
                "lle-agent-1", null, "agent-a", null, null, 20);

        assertEquals(1, entries.size());
        assertEquals("agent-a", entries.get(0).get("actor_id"));
    }

    // =========================================================================
    // Cursor pagination
    // =========================================================================

    @Test
    void listLedgerEntries_limit_capsResults() {
        setup("lle-limit-1", "agent-a");
        for (int i = 0; i < 5; i++) {
            tools.sendMessage("lle-limit-1", "agent-a", "event",
                    "{\"tool_name\":\"t\",\"duration_ms\":1}", null, null, null, null);
        }

        List<Map<String, Object>> page = tools.listLedgerEntries("lle-limit-1", null, null, null, null, 3);

        assertEquals(3, page.size());
    }

    @Test
    void listLedgerEntries_afterId_returnsNextPage() {
        setup("lle-cursor-1", "agent-a", "agent-b");
        tools.sendMessage("lle-cursor-1", "agent-a", "command", "Go", "corr-cur", null, null, null);
        tools.sendMessage("lle-cursor-1", "agent-a", "status", "Working", "corr-cur", null, null, null);
        tools.sendMessage("lle-cursor-1", "agent-b", "done", "Done", "corr-cur", null, null, null);

        List<Map<String, Object>> page1 = tools.listLedgerEntries("lle-cursor-1", null, null, null, null, 2);
        assertEquals(2, page1.size());

        Long cursor = (Long) (Object) page1.get(1).get("sequence_number");
        List<Map<String, Object>> page2 = tools.listLedgerEntries("lle-cursor-1", null, null, null, cursor, 2);

        assertEquals(1, page2.size());
        assertEquals("DONE", page2.get(0).get("message_type"));
    }

    // =========================================================================
    // Causal chain visible in MCP response
    // =========================================================================

    @Test
    void listLedgerEntries_causalChain_causedByEntryIdPopulated() {
        setup("lle-causal-1", "agent-a", "agent-b");
        tools.sendMessage("lle-causal-1", "agent-a", "command", "Run", "corr-c1", null, null, null);
        tools.sendMessage("lle-causal-1", "agent-b", "done", "Done", "corr-c1", null, null, null);

        List<Map<String, Object>> entries = tools.listLedgerEntries("lle-causal-1", null, null, null, null, 20);
        assertEquals(2, entries.size());

        Map<String, Object> cmd = entries.get(0);
        Map<String, Object> done = entries.get(1);

        assertNull(cmd.get("caused_by_entry_id"), "COMMAND should have no causal predecessor");
        assertNotNull(done.get("caused_by_entry_id"), "DONE should point to its COMMAND");
    }

    // =========================================================================
    // Invalid since parameter
    // =========================================================================

    @Test
    void listLedgerEntries_invalidSinceTimestamp_throws() {
        setup("lle-since-bad-1", "agent-a");

        assertThrows(ToolCallException.class,
                () -> tools.listLedgerEntries("lle-since-bad-1", null, null, "not-a-date", null, 20));
    }

    // =========================================================================
    // Fixture
    // =========================================================================

    private void setup(final String channel, final String... agents) {
        tools.createChannel(channel, "APPEND", null, null);
        for (final String agent : agents) {
            tools.registerInstance(channel, agent, null, null, null);
        }
    }
}
```

- [ ] **Step 2: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ListLedgerEntriesTest -Dno-format -q 2>&1 | tail -10
```
Expected: `BUILD SUCCESS`, all tests passing.

- [ ] **Step 3: Run the full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format -q 2>&1 | grep -E "Tests run|BUILD"
```
Expected: `BUILD SUCCESS`. Zero failures.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/quarkiverse/qhorus/ledger/ListLedgerEntriesTest.java
git commit -m "$(cat <<'EOF'
test(ledger): ListLedgerEntriesTest — E2E coverage for list_ledger_entries MCP tool

Happy path, type_filter, agent_id, cursor pagination, causal chain visibility,
event telemetry fields, robustness against invalid parameters.

Refs #<child-issue-5> — Epic #<epic>

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Verification

- [ ] **Part 2 complete — full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | grep -E "Tests run|BUILD"
```
Expected: `BUILD SUCCESS`. Three new passing test classes: `MessageLedgerCaptureTest`, `ListLedgerEntriesTest`. `ChannelTimelineTest` unchanged and passing.

Continue with Part 3: `docs/superpowers/plans/2026-04-26-normative-ledger-part3-reactive-cleanup-docs.md`

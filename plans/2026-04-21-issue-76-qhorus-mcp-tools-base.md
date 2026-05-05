# Issue #76 — QhorusMcpToolsBase Refactor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract all shared code from `QhorusMcpTools` into a new abstract `QhorusMcpToolsBase` — all 23 public response records, 6 entity-to-DTO mappers, and 4 validation/helper methods — so that `ReactiveQhorusMcpTools` (Issue #78) can extend the base without duplicating anything.

**Architecture:** `QhorusMcpToolsBase` is a plain abstract class — no CDI annotations, no `@Tool`, no service injection. It holds records, mappers, and validators only. `QhorusMcpTools extends QhorusMcpToolsBase` and retains all `@Inject`, `@Tool`, and private channel-semantic helpers. Because Java inherits static nested types through `extends`, all existing `QhorusMcpTools.CheckResult` / import references in tests continue to compile without changes.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5 (666 existing tests are the correctness oracle — no new tests needed for a pure refactor)

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`

---

## File Map

**New:**
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java`

**Modified:**
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java` — add `extends QhorusMcpToolsBase`, remove extracted members, update two call sites

---

## Task 1: Create QhorusMcpToolsBase with all 23 records, extend QhorusMcpTools

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`

- [ ] **Step 1: Verify baseline — 666 tests green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` with 666 tests run, 0 failures.

- [ ] **Step 2: Create QhorusMcpToolsBase.java with all 23 records**

```java
// runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java
package io.quarkiverse.qhorus.runtime.mcp;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

public abstract class QhorusMcpToolsBase {

    public record RegisterResponse(
            String instanceId,
            List<ChannelSummary> activeChannels,
            List<InstanceInfo> onlineInstances) {
    }

    public record ChannelSummary(String name, String description, String semantic) {
    }

    public record InstanceInfo(
            String instanceId,
            String description,
            String status,
            List<String> capabilities,
            String lastSeen) {
    }

    public record ChannelDetail(
            UUID channelId,
            String name,
            String description,
            String semantic,
            String barrierContributors,
            long messageCount,
            String lastActivityAt,
            boolean paused,
            /** Comma-separated allowed-writer entries, or null if the channel is open to all writers. */
            String allowedWriters,
            /** Comma-separated admin instance IDs, or null if management is open to any caller. */
            String adminInstances,
            /** Max messages per minute across all senders. Null = unlimited. */
            Integer rateLimitPerChannel,
            /** Max messages per minute from a single sender. Null = unlimited. */
            Integer rateLimitPerInstance) {
    }

    public record MessageResult(
            Long messageId,
            String channelName,
            String sender,
            String messageType,
            String correlationId,
            Long inReplyTo,
            int parentReplyCount,
            List<String> artefactRefs,
            /** Addressing target: null (broadcast), instance:<id>, capability:<tag>, or role:<name>. */
            String target) {
    }

    public record MessageSummary(
            Long messageId,
            String sender,
            String messageType,
            String content,
            String correlationId,
            Long inReplyTo,
            String createdAt,
            List<String> artefactRefs,
            /** Addressing target: null (broadcast), instance:<id>, capability:<tag>, or role:<name>. */
            String target) {
    }

    public record CheckResult(
            List<MessageSummary> messages,
            Long lastId,
            /** Non-null on BARRIER channels that have not yet released — lists pending contributors. */
            String barrierStatus) {
    }

    public record ArtefactDetail(
            UUID artefactId,
            String key,
            String description,
            String createdBy,
            String content,
            boolean complete,
            long sizeBytes,
            String updatedAt) {
    }

    public record WaitResult(
            boolean found,
            boolean timedOut,
            String correlationId,
            /** The matching response message, or null on timeout. */
            MessageSummary message,
            String status) {
    }

    public record ApprovalSummary(
            String correlationId,
            String channelName,
            String expiresAt,
            long timeRemainingSeconds) {
    }

    public record PendingWaitSummary(
            String correlationId,
            String channelName,
            String expiresAt,
            long timeRemainingSeconds) {
    }

    public record CancelWaitResult(
            String correlationId,
            boolean cancelled,
            String message) {
    }

    public record ForceReleaseResult(
            String channelName,
            String semantic,
            int messageCount,
            List<MessageSummary> messages) {
    }

    public record RevokeResult(
            String artefactId,
            String key,
            String createdBy,
            long sizeBytes,
            int claimsReleased,
            boolean revoked,
            String message) {
    }

    public record DeleteMessageResult(
            Long messageId,
            boolean deleted,
            String sender,
            String messageType,
            String contentPreview,
            String message) {
    }

    public record ClearChannelResult(
            String channelName,
            int messagesDeleted,
            boolean cleared) {
    }

    public record DeregisterResult(
            String instanceId,
            boolean deregistered,
            String message) {
    }

    public record MessagePreview(
            Long messageId,
            String sender,
            String messageType,
            String contentPreview,
            String createdAt) {
    }

    public record WatchdogSummary(
            String id,
            String conditionType,
            String targetName,
            Integer thresholdSeconds,
            Integer thresholdCount,
            String notificationChannel,
            String createdBy,
            String createdAt,
            String lastFiredAt) {
    }

    public record ObserverRegistration(
            String observerId,
            Set<String> channelNames) {
    }

    public record DeregisterObserverResult(
            String observerId,
            boolean deregistered,
            String message) {
    }

    public record DeleteWatchdogResult(
            String watchdogId,
            boolean deleted,
            String message) {
    }

    public record ChannelDigest(
            String channelName,
            String semantic,
            boolean paused,
            long messageCount,
            Map<String, Integer> senderBreakdown,
            Map<String, Integer> typeBreakdown,
            int artefactRefCount,
            List<String> activeAgents,
            List<MessagePreview> recentMessages,
            String oldestMessageAt,
            String newestMessageAt) {
    }
}
```

- [ ] **Step 3: Update QhorusMcpTools — add `extends QhorusMcpToolsBase`, remove records**

In `QhorusMcpTools.java`:

Change line 44:
```java
public class QhorusMcpTools {
```
to:
```java
public class QhorusMcpTools extends QhorusMcpToolsBase {
```

Delete the entire records section (lines 76–276 in the original file), which starts with the comment `// Return-type records — public so tests can reference them` and ends after the `ChannelDigest` record closing brace. These are now in `QhorusMcpToolsBase` and inherited.

Remove unused imports no longer needed after record removal (Java compiler will flag these; remove any that produce "unused import" warnings — only `java.util.Set` from the records section, since `Set` is still used in the method body elsewhere):
- Check that `java.util.Set` is still imported (it's used in `checkMessagesBarrier`) — keep it.

- [ ] **Step 4: Compile and run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` — 666 tests, 0 failures.

> **Java inheritance note:** Static nested types (including records) declared in a parent class are inherited members of the subclass. `QhorusMcpTools.CheckResult` and `import ...QhorusMcpTools.CheckResult` both resolve correctly without any test file changes.

---

## Task 2: Move mappers and toolError to QhorusMcpToolsBase

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — add methods
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java` — remove methods

- [ ] **Step 1: Add imports and 7 methods to QhorusMcpToolsBase**

Add these imports to `QhorusMcpToolsBase.java` (after the existing `java.util.*` imports):

```java
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.ledger.AgentMessageLedgerEntry;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;
```

Add these methods to `QhorusMcpToolsBase` (before the closing `}`):

```java
    protected String toolError(Exception e) {
        return "Error: " + e.getMessage();
    }

    protected ArtefactDetail toArtefactDetail(SharedData d) {
        return new ArtefactDetail(d.id, d.key, d.description, d.createdBy,
                d.content, d.complete, d.sizeBytes, d.updatedAt.toString());
    }

    protected MessageSummary toMessageSummary(Message m) {
        List<String> refs = (m.artefactRefs != null && !m.artefactRefs.isBlank())
                ? List.of(m.artefactRefs.split(","))
                : List.of();
        return new MessageSummary(m.id, m.sender, m.messageType.name(), m.content,
                m.correlationId, m.inReplyTo, m.createdAt.toString(), refs, m.target);
    }

    protected ChannelDetail toChannelDetail(Channel ch, long messageCount) {
        return new ChannelDetail(
                ch.id,
                ch.name,
                ch.description,
                ch.semantic.name(),
                ch.barrierContributors,
                messageCount,
                ch.lastActivityAt.toString(),
                ch.paused,
                ch.allowedWriters,
                ch.adminInstances,
                ch.rateLimitPerChannel,
                ch.rateLimitPerInstance);
    }

    protected WatchdogSummary toWatchdogSummary(Watchdog w) {
        return new WatchdogSummary(
                w.id.toString(),
                w.conditionType,
                w.targetName,
                w.thresholdSeconds,
                w.thresholdCount,
                w.notificationChannel,
                w.createdBy,
                w.createdAt != null ? w.createdAt.toString() : null,
                w.lastFiredAt != null ? w.lastFiredAt.toString() : null);
    }

    protected Map<String, Object> toEventMap(AgentMessageLedgerEntry e) {
        Map<String, Object> m = new java.util.LinkedHashMap<>();
        m.put("tool_name", e.toolName);
        m.put("duration_ms", e.durationMs);
        m.put("token_count", e.tokenCount);
        m.put("agent_id", e.actorId);
        m.put("occurred_at", e.occurredAt != null ? e.occurredAt.toString() : null);
        m.put("message_id", e.messageId);
        m.put("correlation_id", e.correlationId);
        m.put("context_refs", e.contextRefs);
        m.put("source_entity", e.sourceEntity);
        m.put("digest", e.digest);
        m.put("sequence_number", (long) e.sequenceNumber);
        return m;
    }

    protected Map<String, Object> toTimelineEntry(Message m) {
        Map<String, Object> entry = new java.util.LinkedHashMap<>();
        entry.put("id", m.id);
        if (m.messageType == MessageType.EVENT) {
            entry.put("type", "EVENT");
            entry.put("created_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("occurred_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("agent_id", m.sender);
            entry.put("message_type", null);
            String toolName = null;
            if (m.content != null) {
                try {
                    com.fasterxml.jackson.databind.JsonNode node = new com.fasterxml.jackson.databind.ObjectMapper()
                            .readTree(m.content);
                    com.fasterxml.jackson.databind.JsonNode tn = node.get("tool_name");
                    if (tn != null && tn.isTextual()) {
                        toolName = tn.asText();
                    }
                } catch (Exception ignored) {
                }
            }
            entry.put("tool_name", toolName);
        } else {
            entry.put("type", "MESSAGE");
            entry.put("created_at", m.createdAt != null ? m.createdAt.toString() : null);
            entry.put("sender", m.sender);
            entry.put("message_type", m.messageType != null ? m.messageType.name().toLowerCase() : null);
            entry.put("content", m.content);
            entry.put("correlation_id", m.correlationId);
            entry.put("tool_name", null);
        }
        return entry;
    }
```

- [ ] **Step 2: Remove the 7 methods from QhorusMcpTools**

Delete these methods from `QhorusMcpTools.java` (they are now inherited from the base):
- `private String toolError(Exception e)` — the entire method body (in the "Tool error helper" section)
- `private ArtefactDetail toArtefactDetail(...)` — in the Helpers section near line 1672
- `private MessageSummary toMessageSummary(Message m)` — near line 1677
- `private ChannelDetail toChannelDetail(Channel ch, long messageCount)` — near line 1685
- `private WatchdogSummary toWatchdogSummary(...)` — near line 1701
- `private Map<String, Object> toEventMap(...)` — near line 1548
- `private Map<String, Object> toTimelineEntry(Message m)` — near line 1565

Also remove the "Tool error helper" comment block that preceded `toolError`.

These methods are inherited — `this::toMessageSummary`, `this.toChannelDetail(...)`, etc. in the remaining `@Tool` methods continue to work with no call site changes.

- [ ] **Step 3: Compile and run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` — 666 tests, 0 failures.

---

## Task 3: Move validators to QhorusMcpToolsBase (Supplier signature refactor)

`checkAdminAccess` is pure (no service calls) and moves as-is. `isAllowedWriter` and `isVisibleToReader` call `instanceService` internally; they are refactored to accept a `Supplier<List<String>>` for the lazy tag lookup, which preserves the lazy-loading semantics while removing the service dependency.

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — add validator methods
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java` — remove validators, update call sites

- [ ] **Step 1: Add Supplier import and 3 validator methods to QhorusMcpToolsBase**

Add import to `QhorusMcpToolsBase.java`:
```java
import java.util.function.Supplier;
```

Add these methods to `QhorusMcpToolsBase` (before the closing `}`):

```java
    /**
     * Throws {@link IllegalStateException} if the channel has an {@code admin_instances} list
     * and {@code callerInstanceId} is not in it (or is null).
     */
    protected static void checkAdminAccess(Channel ch, String callerInstanceId, String toolName) {
        if (ch.adminInstances == null || ch.adminInstances.isBlank()) {
            return;
        }
        if (callerInstanceId == null || callerInstanceId.isBlank()) {
            throw new IllegalStateException(
                    "Channel '" + ch.name + "' requires a caller_instance_id for " + toolName
                            + " — it has an admin_instances list.");
        }
        for (String raw : ch.adminInstances.split(",")) {
            if (raw.strip().equals(callerInstanceId)) {
                return;
            }
        }
        throw new IllegalStateException(
                "Caller '" + callerInstanceId + "' is not permitted to invoke " + toolName
                        + " on channel '" + ch.name + "'. Not in admin_instances list.");
    }

    /**
     * Returns true if {@code sender} is permitted to write to a channel with the given
     * {@code allowedWriters} ACL string. Null or blank ACL = open to all.
     * {@code senderTagsSupplier} is invoked lazily — only if a capability/role entry is present.
     */
    protected static boolean isAllowedWriter(String sender, String allowedWriters,
            Supplier<List<String>> senderTagsSupplier) {
        if (allowedWriters == null || allowedWriters.isBlank()) {
            return true;
        }
        List<String> senderTags = null;
        for (String raw : allowedWriters.split(",")) {
            String entry = raw.strip();
            if (entry.isEmpty()) {
                continue;
            }
            if (entry.startsWith("capability:") || entry.startsWith("role:")) {
                if (senderTags == null) {
                    senderTags = senderTagsSupplier.get();
                }
                if (senderTags.contains(entry)) {
                    return true;
                }
            } else {
                if (entry.equals(sender)) {
                    return true;
                }
            }
        }
        return false;
    }

    /**
     * Returns true if the message is visible to the given reader.
     * {@code readerTagsSupplier} is invoked lazily — only if the target is a capability/role prefix.
     */
    protected static boolean isVisibleToReader(Message m, String readerInstanceId,
            Supplier<List<String>> readerTagsSupplier) {
        if (readerInstanceId == null || readerInstanceId.isBlank()) {
            return true;
        }
        if (m.messageType == MessageType.EVENT) {
            return true;
        }
        if (m.target == null) {
            return true;
        }
        if (m.target.equals("instance:" + readerInstanceId)) {
            return true;
        }
        if (m.target.startsWith("capability:") || m.target.startsWith("role:")) {
            return readerTagsSupplier.get().contains(m.target);
        }
        return false;
    }
```

- [ ] **Step 2: Remove the 3 validator methods from QhorusMcpTools**

Delete these private methods from `QhorusMcpTools.java`:
- `private void checkAdminAccess(Channel ch, String callerInstanceId, String toolName)` — the full method including its Javadoc
- `private boolean isAllowedWriter(String sender, String allowedWriters)` — the full method including its Javadoc
- `private boolean isVisibleToReader(Message m, String readerInstanceId)` — the full method including both Javadoc blocks

- [ ] **Step 3: Update the one isAllowedWriter call site in sendMessage**

In `QhorusMcpTools.sendMessage`, find:
```java
        if (msgType != MessageType.EVENT && !isAllowedWriter(sender, ch.allowedWriters)) {
```
Replace with:
```java
        if (msgType != MessageType.EVENT && !isAllowedWriter(sender, ch.allowedWriters,
                () -> instanceService.findCapabilityTagsForInstance(sender))) {
```

- [ ] **Step 4: Update the 6 isVisibleToReader call sites**

Each call is inside a `filter` lambda. Replace all 6 occurrences of:
```java
.filter(m -> isVisibleToReader(m, readerInstanceId))
```
with:
```java
.filter(m -> isVisibleToReader(m, readerInstanceId,
        () -> instanceService.findCapabilityTagsForInstance(readerInstanceId)))
```

The 6 locations are in:
- `checkMessagesEphemeral` (1 occurrence)
- `checkMessagesCollect` (1 occurrence)
- `checkMessagesBarrier` (1 occurrence)
- `checkMessagesAppend` (1 occurrence)
- `getReplies` (1 occurrence — `readerInstanceId` is the method parameter)
- `searchMessages` (1 occurrence — `readerInstanceId` is the method parameter)

- [ ] **Step 5: Compile and run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```
Expected: `BUILD SUCCESS` — 666 tests, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java
git commit -m "$(cat <<'EOF'
refactor(mcp): extract QhorusMcpToolsBase — records, mappers, validators

Prerequisite for ReactiveQhorusMcpTools (#78). Moves all 23 response
records, 7 mapper/helper methods, and 3 validator methods from
QhorusMcpTools into new abstract QhorusMcpToolsBase. No behaviour
changes. isAllowedWriter and isVisibleToReader gain Supplier<List<String>>
parameter for lazy tag lookup so the base class is service-free.

Refs #76, Refs #73

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- ✅ `QhorusMcpToolsBase` contains all response records (23) — Task 1
- ✅ `QhorusMcpToolsBase` contains all entity→DTO mappers — Task 2
- ✅ `QhorusMcpToolsBase` contains all input validators/helpers — Task 3
- ✅ `QhorusMcpTools extends QhorusMcpToolsBase` — Task 1, Step 3
- ✅ No `@Tool`, CDI, or service injection in base class — validators use Supplier, no direct injection
- ✅ All 666 tests green — verified at end of each task

**What does NOT move (stays in QhorusMcpTools):**
- All `@Inject` fields — CDI, not allowed in base
- `buildInstanceInfoList(List<Instance>)` — calls `Capability.find(...)` Panache static; blocking-specific, reactive variant (#78) will override
- `checkMessagesEphemeral/Collect/Barrier/Append` — use Panache statics and injected services
- `requireWatchdogEnabled()` — uses injected `qhorusConfig`
- All `@Tool` methods and convenience overloads

**Placeholder scan:** None. All code is complete.

**Type consistency:** Validator signatures in base match call sites in QhorusMcpTools at every updated location.

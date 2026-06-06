# Channel Parameter Rename Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename all MCP tool `channel_name` parameters to `channel` (accepting UUID-or-name) across `QhorusMcpTools` and `ReactiveQhorusMcpTools`, and upgrade `resolveChannel()` in the base class to return `Channel` (eliminating double lookups).

**Architecture:** `QhorusMcpToolsBase.resolveChannel(String)` is changed to return `Channel` instead of `UUID` — one lookup covers both UUID and name inputs, and the entity is available to all callers. A matching `resolveChannelAsync(String) → Uni<Channel>` is added to `ReactiveQhorusMcpTools` for the pure-reactive (Category A) tools. Category B reactive tools use the blocking `resolveChannel()` unchanged, since they are already `@Blocking`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Panache, Mutiny (`Uni<T>`), `@QuarkusTest`, `@TestTransaction`. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`.

---

## Files Changed

| File | Change |
|------|--------|
| `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` | `resolveChannel()` return type `UUID` → `Channel` |
| `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` | 27 param renames + structural changes |
| `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` | 26 param renames + `resolveChannelAsync()` + structural changes |
| `runtime/src/test/java/io/casehub/qhorus/mcp/ChannelToolTest.java` | Add UUID resolution tests |
| `runtime/src/test/java/io/casehub/qhorus/mcp/MessagingToolTest.java` | Add UUID `send_message` test |
| `runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryToolsTest.java` | Add UUID ledger test |
| `runtime/src/test/java/io/casehub/qhorus/mcp/ToolErrorHandlingTest.java` | Add non-existent UUID test |

---

## Task 1: Write Failing UUID-Resolution Tests

These tests document the new behaviour and will fail until the implementation is complete.

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/ChannelToolTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/MessagingToolTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryToolsTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/ToolErrorHandlingTest.java`

- [ ] **Step 1: Add UUID resolution tests to ChannelToolTest**

Add at the end of the class:
```java
@Test
@TestTransaction
void pauseChannel_acceptsChannelUuid() {
    ChannelDetail created = tools.createChannel("uuid-pause-test", "Test", null, null, null, null, null, null, null, null, null, null, null, null);
    String uuid = created.channelId().toString();

    ChannelDetail result = tools.pauseChannel(uuid, null);

    assertEquals("uuid-pause-test", result.name());
}

@Test
@TestTransaction
void resumeChannel_acceptsChannelUuid() {
    ChannelDetail created = tools.createChannel("uuid-resume-test", "Test", null, null, null, null, null, null, null, null, null, null, null, null);
    String uuid = created.channelId().toString();
    tools.pauseChannel(uuid, null);

    ChannelDetail result = tools.resumeChannel(uuid, null);

    assertEquals("uuid-resume-test", result.name());
}
```

- [ ] **Step 2: Add UUID test to MessagingToolTest**

Add at the end of the class (add `import io.casehub.qhorus.api.channel.ChannelDetail;` if not already present):
```java
@Test
@TestTransaction
void sendMessage_acceptsChannelUuid() {
    ChannelDetail created = tools.createChannel("uuid-msg-test", "Test", null, null, null, null, null, null, null, null, null, null, null, null);
    String uuid = created.channelId().toString();

    DispatchResult result = tools.sendMessage(uuid, "alice", "query", "hello via uuid", null, null, null, null, null, null, null);

    assertNotNull(result);
}
```

- [ ] **Step 3: Add UUID test to LedgerQueryToolsTest**

Add at the end of the class:
```java
@Test
@TestTransaction
void listLedgerEntries_acceptsChannelUuid() {
    ChannelDetail created = tools.createChannel("uuid-ledger-test", "Test", null, null, null, null, null, null, null, null, null, null, null, null);
    tools.sendMessage("uuid-ledger-test", "alice", "query", "hello", null, null, null, null, null, null, null);
    String uuid = created.channelId().toString();

    List<Map<String, Object>> entries = tools.listLedgerEntries(uuid, null, null, null, null, null, null, 20);

    assertFalse(entries.isEmpty());
}
```
Add import `import io.casehub.qhorus.api.channel.ChannelDetail;` if not already present.

- [ ] **Step 4: Add non-existent UUID test to ToolErrorHandlingTest**

Add at the end of the class:
```java
@Test
@TestTransaction
void pauseChannel_nonExistentUuid_throwsToolCallException() {
    String nonExistentUuid = java.util.UUID.randomUUID().toString();
    assertThrows(ToolCallException.class, () -> tools.pauseChannel(nonExistentUuid, null));
}
```

- [ ] **Step 5: Run tests to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ChannelToolTest#pauseChannel_acceptsChannelUuid+resumeChannel_acceptsChannelUuid,MessagingToolTest#sendMessage_acceptsChannelUuid,LedgerQueryToolsTest#listLedgerEntries_acceptsChannelUuid,ToolErrorHandlingTest#pauseChannel_nonExistentUuid_throwsToolCallException"
```
Expected: failures with "Channel not found: `<uuid-string>`" — confirming the tests are real.

---

## Task 2: QhorusMcpToolsBase — `resolveChannel()` Returns `Channel`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`

- [ ] **Step 1: Change return type and body of `resolveChannel()`**

Find the method (currently around line 302):
```java
UUID resolveChannel(final String channel) {
    final UUID parsedUuid = ChannelSlugValidator.tryParseUuid(channel);
    if (parsedUuid == null) {
        return channelService.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel))
                .id;
    }
    return channelService.findById(parsedUuid)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel))
            .id;
}
```

Replace with:
```java
Channel resolveChannel(final String channel) {
    final UUID parsedUuid = ChannelSlugValidator.tryParseUuid(channel);
    if (parsedUuid == null) {
        return channelService.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel));
    }
    return channelService.findById(parsedUuid)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel));
}
```

- [ ] **Step 2: Verify the import for `Channel` is present in the base class**

`io.casehub.qhorus.runtime.channel.Channel` must be imported. It likely already is since `channelService` is typed on it. Confirm the import is present.

- [ ] **Step 3: Build to surface all compile errors from the return-type change**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | head -30
```
Expected: compile errors in `QhorusMcpTools` and `ReactiveQhorusMcpTools` wherever `resolveChannel()` is called. These are fixed in subsequent tasks.

---

## Task 3: Fix Existing `resolveChannel()` Callers in `QhorusMcpTools`

Two tools already use `channel` (not `channel_name`) and call `resolveChannel()`. They must be adapted first to restore compile, before the 27-tool rename begins.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

- [ ] **Step 1: Fix `setChannelTypeConstraints`**

Find (currently around line 336):
```java
UUID channelId = resolveChannel(channel);
Channel ch = channelService.setTypeConstraints(channelId, allowedTypes, deniedTypes);
```
Replace with:
```java
Channel resolved = resolveChannel(channel);
Channel ch = channelService.setTypeConstraints(resolved.id, allowedTypes, deniedTypes);
```

- [ ] **Step 2: Fix `projectChannel`**

Find the `projectChannel` method. It calls `resolveChannel(channel)` and uses the result as `UUID channelId`. Change:
```java
UUID channelId = resolveChannel(channel);
```
to:
```java
UUID channelId = resolveChannel(channel).id;
```

- [ ] **Step 3: Build to confirm compile errors reduced (only reactive class remains broken)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep -v "ReactiveQhorusMcpTools" | head -20
```
Expected: no errors outside `ReactiveQhorusMcpTools`.

---

## Task 4: `QhorusMcpTools` — Service-by-Name Pattern Tools

These tools call `channelService.setXxx(channelName, ...)` which takes a String name. After resolving the channel entity, pass `ch.name` to the service.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Tools:** `set_channel_rate_limits`, `set_channel_writers`, `set_channel_admins`

- [ ] **Step 1: Migrate `setChannelRateLimits`**

Before:
```java
public ChannelDetail setChannelRateLimits(
        @ToolArg(name = "channel_name", description = "Name of the channel to update") String channelName,
        @ToolArg(name = "rate_limit_per_channel", ...) Integer rateLimitPerChannel,
        @ToolArg(name = "rate_limit_per_instance", ...) Integer rateLimitPerInstance) {
    Channel ch = channelService.setRateLimits(channelName, rateLimitPerChannel, rateLimitPerInstance);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

After:
```java
public ChannelDetail setChannelRateLimits(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "rate_limit_per_channel", ...) Integer rateLimitPerChannel,
        @ToolArg(name = "rate_limit_per_instance", ...) Integer rateLimitPerInstance) {
    Channel resolved = resolveChannel(channel);
    Channel ch = channelService.setRateLimits(resolved.name, rateLimitPerChannel, rateLimitPerInstance);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

- [ ] **Step 2: Migrate `setChannelWriters`**

Before:
```java
public ChannelDetail setChannelWriters(
        @ToolArg(name = "channel_name", description = "Name of the channel to update") String channelName,
        @ToolArg(name = "allowed_writers", ...) String allowedWriters) {
    Channel ch = channelService.setAllowedWriters(channelName, allowedWriters);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

After:
```java
public ChannelDetail setChannelWriters(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "allowed_writers", ...) String allowedWriters) {
    Channel resolved = resolveChannel(channel);
    Channel ch = channelService.setAllowedWriters(resolved.name, allowedWriters);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

- [ ] **Step 3: Migrate `setChannelAdmins`**

Before:
```java
public ChannelDetail setChannelAdmins(
        @ToolArg(name = "channel_name", description = "Name of the channel to update") String channelName,
        @ToolArg(name = "admin_instances", ...) String adminInstances) {
    Channel ch = channelService.setAdminInstances(channelName, adminInstances);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

After:
```java
public ChannelDetail setChannelAdmins(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "admin_instances", ...) String adminInstances) {
    Channel resolved = resolveChannel(channel);
    Channel ch = channelService.setAdminInstances(resolved.name, adminInstances);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

- [ ] **Step 4: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep "QhorusMcpTools.java" | head -20
```
Expected: no errors in `QhorusMcpTools` for these three methods.

---

## Task 5: `QhorusMcpTools` — Entity-Pattern Channel Management Tools

These tools call `channelService.findByName(channelName)` to get the entity, then operate using `ch.name` or `ch.id`. Replace the `findByName` call with `resolveChannel(channel)`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Tools:** `update_channel_binding`, `pause_channel`, `resume_channel`, `delete_channel`, `list_backends`, `deregister_backend`, `register_backend`, `force_release_channel`

- [ ] **Step 1: Migrate `updateChannelBinding`**

Before:
```java
public ChannelDetail updateChannelBinding(
        @ToolArg(name = "channel_name", description = "Name of the channel whose binding to update") String channelName, ...) {
    Channel ch = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
    channelService.updateConnectorBinding(ch.id, outboundConnectorId, outboundDestination);
    return toChannelDetail(ch, Message.<Message>count("channelId", ch.id));
}
```

After:
```java
public ChannelDetail updateChannelBinding(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    Channel ch = resolveChannel(channel);
    channelService.updateConnectorBinding(ch.id, outboundConnectorId, outboundDestination);
    return toChannelDetail(ch, Message.<Message>count("channelId", ch.id));
}
```

- [ ] **Step 2: Migrate `pauseChannel` (the `@Tool` method)**

Before:
```java
public ChannelDetail pauseChannel(
        @ToolArg(name = "channel_name", description = "Name of the channel to pause") String channelName,
        @ToolArg(name = "caller_instance_id", ...) String callerInstanceId) {
    Channel ch = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
    checkAdminAccess(ch, callerInstanceId, "pause_channel");
    ch = channelService.pause(channelName);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

After:
```java
public ChannelDetail pauseChannel(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "caller_instance_id", ...) String callerInstanceId) {
    Channel ch = resolveChannel(channel);
    checkAdminAccess(ch, callerInstanceId, "pause_channel");
    ch = channelService.pause(ch.name);
    return toChannelDetail(ch, Message.<Message> count("channelId", ch.id));
}
```

Also update the package-private convenience overload (no `@Tool`) directly above:
```java
// Before:
ChannelDetail pauseChannel(String channelName) {
    return pauseChannel(channelName, null);
}

// After (param rename only — still passes straight through):
ChannelDetail pauseChannel(String channel) {
    return pauseChannel(channel, null);
}
```

- [ ] **Step 3: Migrate `resumeChannel` (the `@Tool` method and its convenience overload)**

Same pattern as `pause_channel`. Find the `@Tool` method and apply:
```java
// @Tool method — before:
@ToolArg(name = "channel_name", ...) String channelName
Channel ch = channelService.findByName(channelName).orElseThrow(...);
checkAdminAccess(ch, callerInstanceId, "resume_channel");
ch = channelService.resume(channelName);

// @Tool method — after:
@ToolArg(name = "channel", description = "Channel name or UUID") String channel
Channel ch = resolveChannel(channel);
checkAdminAccess(ch, callerInstanceId, "resume_channel");
ch = channelService.resume(ch.name);
```

Update the package-private convenience overload parameter from `channelName` to `channel`.

- [ ] **Step 4: Migrate `deleteChannel`**

Before:
```java
public String deleteChannel(
        @ToolArg(name = "channel_name", ...) String channelName,
        @ToolArg(name = "force", ...) Boolean force,
        @ToolArg(name = "caller_instance_id", ...) String callerInstanceId) {
    Channel ch = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
    checkAdminAccess(ch, callerInstanceId, "delete_channel");
    ...uses channelName in service calls and result...
}
```

After (read the current body carefully and replace all `channelName` references):
```java
public String deleteChannel(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "force", ...) Boolean force,
        @ToolArg(name = "caller_instance_id", ...) String callerInstanceId) {
    Channel ch = resolveChannel(channel);
    checkAdminAccess(ch, callerInstanceId, "delete_channel");
    // use ch.name or ch.id wherever channelName appeared in service calls and results
    ...
}
```

- [ ] **Step 5: Migrate `listBackends`, `deregisterBackend`, `registerBackend`**

Each follows the same entity-pattern. For each:
- Rename `@ToolArg(name = "channel_name", ...)` → `@ToolArg(name = "channel", description = "Channel name or UUID")`
- Rename `String channelName` → `String channel`
- Replace `channelService.findByName(channelName).orElseThrow(...)` → `resolveChannel(channel)`
- Replace all downstream uses of `channelName` with `ch.name` or `ch.id` as appropriate

- [ ] **Step 6: Migrate `forceReleaseChannel`**

Same entity-pattern. Find the current body, apply the same changes.

- [ ] **Step 7: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep "QhorusMcpTools.java" | head -20
```
Expected: no remaining errors in these tools.

---

## Task 6: `QhorusMcpTools` — Messaging Tools

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Tools:** `send_message`, `check_messages`, `wait_for_reply`

All three follow the same entity-pattern: `channelService.findByName(channelName).orElseThrow(...)` → `resolveChannel(channel)`. The resolved `ch.id` is used downstream.

- [ ] **Step 1: Migrate `sendMessage`**

Before (lines 535–547 approximately):
```java
@ToolArg(name = "channel_name", description = "Target channel name") String channelName,
...
Channel ch = channelService.findByName(channelName)
        .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
```

After:
```java
@ToolArg(name = "channel", description = "Channel name or UUID") String channel,
...
Channel ch = resolveChannel(channel);
```

No other uses of `channelName` in the method body — the resolved `ch` is used throughout. Verify this by reading the full method body.

- [ ] **Step 2: Migrate `checkMessages`**

Same pattern. Find the method, apply:
- `@ToolArg(name = "channel_name", ...) String channelName` → `@ToolArg(name = "channel", description = "Channel name or UUID") String channel`
- `channelService.findByName(channelName).orElseThrow(...)` → `resolveChannel(channel)`
- Any downstream `channelName` reference → `ch.name` or `ch.id`

- [ ] **Step 3: Migrate `waitForReply`**

Same pattern. Apply the entity-pattern replacement.

- [ ] **Step 4: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep "QhorusMcpTools.java" | head -20
```

---

## Task 7: `QhorusMcpTools` — Special Cases: `request_approval`, `respond_to_approval`, `search_messages`, `list_my_commitments`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

- [ ] **Step 1: Migrate `requestApproval` — resolve once, pass name to helper**

Before:
```java
public WaitResult requestApproval(
        @ToolArg(name = "channel_name", ...) String channelName, ...) {
    String correlationId = UUID.randomUUID().toString();
    return requestApprovalWithCorrelationId(channelName, content, correlationId, timeoutS);
}
```

After:
```java
public WaitResult requestApproval(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    Channel ch = resolveChannel(channel);
    String correlationId = UUID.randomUUID().toString();
    return requestApprovalWithCorrelationId(ch.name, content, correlationId, timeoutS);
}
```

`requestApprovalWithCorrelationId` is NOT a `@Tool` method — its `channelName` parameter keeps the name `channelName` (it already receives a resolved name). No change needed there.

- [ ] **Step 2: Migrate `respondToApproval`**

Before:
```java
public DispatchResult respondToApproval(
        ...,
        @ToolArg(name = "channel_name", ...) String channelName) {
    Channel ch = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
    ...uses ch.id for dispatch...
}
```

After:
```java
public DispatchResult respondToApproval(
        ...,
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel) {
    Channel ch = resolveChannel(channel);
    ...uses ch.id for dispatch (unchanged)...
}
```

- [ ] **Step 3: Migrate `searchMessages` — nullable channel param**

Before:
```java
public List<MessageSummary> searchMessages(
        @ToolArg(name = "query", ...) String query,
        @ToolArg(name = "channel_name", ..., required = false) String channelName,
        ...) {
    // uses channelName as filter — may be null
}
```

After:
```java
public List<MessageSummary> searchMessages(
        @ToolArg(name = "query", ...) String query,
        @ToolArg(name = "channel", description = "Channel name or UUID", required = false) String channel,
        ...) {
    Channel ch = null;
    if (channel != null && !channel.isBlank()) {
        ch = resolveChannel(channel);
    }
    UUID channelId = ch != null ? ch.id : null;
    // pass channelId wherever channelName/channel UUID was used as a filter
}
```

- [ ] **Step 4: Migrate `listMyCommitments`**

Before:
```java
@ToolArg(name = "channel_name", description = "Channel to query") String channelName
// uses channelName → ch.id for query
```

After:
```java
@ToolArg(name = "channel", description = "Channel name or UUID") String channel
Channel ch = resolveChannel(channel);
// use ch.id for query
```

- [ ] **Step 5: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep "QhorusMcpTools.java" | head -10
```
Expected: no errors in `QhorusMcpTools`.

---

## Task 8: `QhorusMcpTools` — Utility and Ledger Tools

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Tools:** `clear_channel`, `get_channel_digest`, `list_ledger_entries`, `get_obligation_chain`, `get_causal_chain`, `list_stalled_obligations`, `get_obligation_stats`, `get_telemetry_summary`, `get_channel_timeline`

All use the entity-pattern. The ledger tools use `ch.id` for ledger queries. The changes are mechanical:
- Rename param: `channel_name` → `channel` (description: "Channel name or UUID")
- Rename Java param: `channelName` → `channel`
- Replace lookup: `channelService.findByName(channelName).orElseThrow(...)` → `resolveChannel(channel)`
- Downstream: `channelName` references in service calls → `ch.name`; `ch.id` stays as-is

- [ ] **Step 1: Migrate `clearChannel`, `getChannelDigest`**

For each: apply the entity-pattern replacement described above.

- [ ] **Step 2: Migrate all 7 ledger tools**

`list_ledger_entries` (shown in full as the template for the group):

Before:
```java
public List<Map<String, Object>> listLedgerEntries(
        @ToolArg(name = "channel_name", description = "Name of the channel to query") String channelName, ...) {
    final Channel ch = channelService.findByName(channelName)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelName));
    ...uses ch.id for ledger query...
}
```

After:
```java
public List<Map<String, Object>> listLedgerEntries(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    final Channel ch = resolveChannel(channel);
    ...uses ch.id for ledger query (unchanged)...
}
```

Apply identical changes to: `getObligationChain`, `getCausalChain`, `listStalledObligations`, `getObligationStats`, `getTelemetrySummary`, `getChannelTimeline`.

- [ ] **Step 3: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep "QhorusMcpTools.java" | head -10
```
Expected: zero errors in `QhorusMcpTools`.

- [ ] **Step 4: Run all QhorusMcpTools tests including the new UUID tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ChannelToolTest,MessagingToolTest,ToolErrorHandlingTest,LedgerQueryToolsTest,CommitmentToolTest,ToolOverloadDiscoverabilityTest"
```
Expected: all pass, including the four new UUID-resolution tests added in Task 1.

- [ ] **Step 5: Commit QhorusMcpTools changes**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java runtime/src/test/java/io/casehub/qhorus/mcp/ChannelToolTest.java runtime/src/test/java/io/casehub/qhorus/mcp/MessagingToolTest.java runtime/src/test/java/io/casehub/qhorus/mcp/ToolErrorHandlingTest.java runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryToolsTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#237): rename channel_name→channel in QhorusMcpTools; resolveChannel() returns Channel

Refs #237"
```

---

## Task 9: `ReactiveQhorusMcpTools` — Add `resolveChannelAsync()` and Fix `set_channel_type_constraints`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Add `resolveChannelAsync()` private method**

Add this method inside the class body (near the top, after field declarations):
```java
private Uni<Channel> resolveChannelAsync(String channel) {
    UUID parsed = ChannelSlugValidator.tryParseUuid(channel);
    if (parsed != null) {
        return channelService.findById(parsed)
                .map(opt -> opt.orElseThrow(() ->
                        new IllegalArgumentException("Channel not found: " + channel)));
    }
    return channelService.findByName(channel)
            .map(opt -> opt.orElseThrow(() ->
                    new IllegalArgumentException("Channel not found: " + channel)));
}
```

`channelService` here is the reactive `ReactiveChannelService` already injected.
`ChannelSlugValidator` is `io.casehub.qhorus.runtime.channel.ChannelSlugValidator` — add the import if not present.

- [ ] **Step 2: Fix `setChannelTypeConstraints` — remove `@Blocking`, switch to `resolveChannelAsync()`**

Before:
```java
@Tool(name = "set_channel_type_constraints", ...)
@Blocking
public Uni<ChannelDetail> setChannelTypeConstraints(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    UUID channelId = resolveChannel(channel);
    return channelService.setTypeConstraints(channelId, allowedTypes, deniedTypes)
            .flatMap(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())));
}
```

After (remove `@Blocking`, adapt body):
```java
@Tool(name = "set_channel_type_constraints", ...)
public Uni<ChannelDetail> setChannelTypeConstraints(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    return resolveChannelAsync(channel)
            .flatMap(ch -> channelService.setTypeConstraints(ch.id, allowedTypes, deniedTypes))
            .flatMap(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())));
}
```

- [ ] **Step 3: Fix `projectChannel` in `ReactiveQhorusMcpTools`**

`project_channel` already uses `channel` param and calls `resolveChannel(channel)` (blocking). It has `@Blocking`. Change:
```java
UUID channelId = resolveChannel(channel);
```
to:
```java
UUID channelId = resolveChannel(channel).id;
```
Keep `@Blocking` — `projectAndRender()` is also blocking.

- [ ] **Step 4: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | head -20
```
Expected: no new errors; remaining errors are from the 26 unrenamed `channel_name` params.

---

## Task 10: `ReactiveQhorusMcpTools` — Category A Tools

Category A tools have no `@Blocking` and use `resolveChannelAsync()`. They currently start with `channelService.findByName(channelName)` inline.

**Tools:** `set_channel_rate_limits`, `set_channel_writers`, `set_channel_admins`, `pause_channel`, `resume_channel`, `delete_channel`, `list_backends`, `deregister_backend`, `register_backend`

- [ ] **Step 1: Migrate `setChannelRateLimits`, `setChannelWriters`, `setChannelAdmins`**

These call reactive service methods by name. Pattern:

Before (`set_channel_rate_limits`):
```java
public Uni<ChannelDetail> setChannelRateLimits(
        @ToolArg(name = "channel_name", ...) String channelName, ...) {
    return channelService.setRateLimits(channelName, rateLimitPerChannel, rateLimitPerInstance)
            .flatMap(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())));
}
```

After:
```java
public Uni<ChannelDetail> setChannelRateLimits(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    return resolveChannelAsync(channel)
            .flatMap(resolved -> channelService.setRateLimits(resolved.name, rateLimitPerChannel, rateLimitPerInstance))
            .flatMap(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())));
}
```

Apply the same structure to `setChannelWriters` (`setAllowedWriters(resolved.name, ...)`) and `setChannelAdmins` (`setAdminInstances(resolved.name, ...)`).

- [ ] **Step 2: Migrate `pauseChannel`, `resumeChannel`**

Before (`pause_channel`):
```java
public Uni<ChannelDetail> pauseChannel(
        @ToolArg(name = "channel_name", ...) String channelName, ...) {
    return channelService.findByName(channelName)
            .map(opt -> opt.orElseThrow(...))
            .invoke(ch -> checkAdminAccess(ch, callerInstanceId, "pause_channel"))
            .flatMap(ignored -> channelService.pause(channelName))
            .flatMap(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())));
}
```

After:
```java
public Uni<ChannelDetail> pauseChannel(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    return resolveChannelAsync(channel)
            .invoke(ch -> checkAdminAccess(ch, callerInstanceId, "pause_channel"))
            .flatMap(ch -> channelService.pause(ch.name))
            .flatMap(ch -> messageStore.countByChannel(ch.id)
                    .map(count -> toChannelDetail(ch, count.longValue())));
}
```

Apply the same structure to `resumeChannel` (`channelService.resume(ch.name)`).

- [ ] **Step 3: Migrate `deleteChannel` — nested flatMap (scope fix)**

Before:
```java
public Uni<DeleteChannelResult> deleteChannel(
        @ToolArg(name = "channel_name", ...) String channelName, ...) {
    return channelService.findByName(channelName)
            .map(opt -> opt.orElseThrow(...))
            .invoke(ch -> checkAdminAccess(ch, callerInstanceId, "delete_channel"))
            .invoke(ch -> commitmentStore.deleteAll(ch.id))
            .flatMap(ch -> channelService.delete(channelName, Boolean.TRUE.equals(force))
                    .invoke(ignored -> channelGateway.closeChannel(ch.id, new ChannelRef(ch.id, ch.name))))
            .map(deleted -> new DeleteChannelResult(channelName, deleted, "deleted"));
}
```

After (terminal `.map()` nested inside `.flatMap()` to keep `ch` in scope):
```java
public Uni<DeleteChannelResult> deleteChannel(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    return resolveChannelAsync(channel)
            .invoke(ch -> checkAdminAccess(ch, callerInstanceId, "delete_channel"))
            .invoke(ch -> commitmentStore.deleteAll(ch.id))
            .flatMap(ch -> channelService.delete(ch.name, Boolean.TRUE.equals(force))
                    .invoke(ignored -> channelGateway.closeChannel(ch.id, new ChannelRef(ch.id, ch.name)))
                    .map(deleted -> new DeleteChannelResult(ch.name, deleted, "deleted")));
}
```

- [ ] **Step 4: Migrate `listBackends`, `deregisterBackend`, `registerBackend`**

All three call `channelService.findByName(channelName)` inline. Apply the entity-pattern:
```java
// Before:
return channelService.findByName(channelName)
        .map(opt -> opt.orElseThrow(...))
        .map(channel -> ...uses channel.id...);

// After:
return resolveChannelAsync(channel)
        .map(ch -> ...uses ch.id...);
```

The variable name `channel` inside the lambda conflicts with the parameter. Use `ch` consistently in lambdas.

- [ ] **Step 5: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep "ReactiveQhorusMcpTools" | head -20
```
Expected: errors only on the remaining Category B tools.

---

## Task 11: `ReactiveQhorusMcpTools` — Category B Tools (Part 1: Management + Messaging)

Category B tools are `@Blocking` and delegate to private `blockingXxx` helpers. The public `@Tool` method resolves the channel, then passes `ch.name` to the private helper. The private helper stays name-only.

**Tools:** `update_channel_binding`, `force_release_channel`, `search_messages`, `send_message`, `check_messages`, `wait_for_reply`, `request_approval`, `respond_to_approval`

- [ ] **Step 1: Migrate `updateChannelBinding`**

Before:
```java
@Blocking
public ChannelDetail updateChannelBinding(
        @ToolArg(name = "channel_name", ...) String channelName, ...) {
    Channel ch = blockingChannelService.findByName(channelName)
            .orElseThrow(...);
    blockingChannelService.updateConnectorBinding(ch.id, ...);
    return toChannelDetail(ch, 0L);
}
```

After:
```java
@Blocking
public ChannelDetail updateChannelBinding(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    Channel ch = resolveChannel(channel);
    blockingChannelService.updateConnectorBinding(ch.id, outboundConnectorId, outboundDestination);
    return toChannelDetail(ch, 0L);
}
```

- [ ] **Step 2: Migrate `forceReleaseChannel`**

Before:
```java
@Blocking
public Uni<ForceReleaseResult> forceReleaseChannel(
        @ToolArg(name = "channel_name", ...) String channelName, ...) {
    return Uni.createFrom().item(() -> blockingForceReleaseChannel(channelName, reason, callerInstanceId));
}

private ForceReleaseResult blockingForceReleaseChannel(String channelName, ...) {
    Channel ch = blockingChannelService.findByName(channelName).orElseThrow(...);
    ...
}
```

After:
```java
@Blocking
public Uni<ForceReleaseResult> forceReleaseChannel(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    Channel ch = resolveChannel(channel);
    return Uni.createFrom().item(() -> blockingForceReleaseChannel(ch.name, reason, callerInstanceId));
}

// private helper unchanged — receives resolved name
private ForceReleaseResult blockingForceReleaseChannel(String channelName, ...) { ... }
```

- [ ] **Step 3: Migrate `searchMessages` — nullable, @Blocking**

Before:
```java
@Blocking
public Uni<List<MessageSummary>> searchMessages(
        @ToolArg(name = "query", ...) String query,
        @ToolArg(name = "channel_name", ..., required = false) String channelName, ...) {
    return Uni.createFrom().item(() -> blockingSearchMessages(query, channelName, limit, readerInstanceId));
}
```

After:
```java
@Blocking
public Uni<List<MessageSummary>> searchMessages(
        @ToolArg(name = "query", ...) String query,
        @ToolArg(name = "channel", description = "Channel name or UUID", required = false) String channel, ...) {
    String resolvedName = null;
    if (channel != null && !channel.isBlank()) {
        resolvedName = resolveChannel(channel).name;
    }
    final String channelName = resolvedName;
    return Uni.createFrom().item(() -> blockingSearchMessages(query, channelName, limit, readerInstanceId));
}

// private helper unchanged — receives resolved name or null
private List<MessageSummary> blockingSearchMessages(String query, String channelName, ...) { ... }
```

- [ ] **Step 4: Migrate `sendMessage`, `checkMessages`, `waitForReply`**

Each delegates to a private `blockingXxx` helper. Apply the resolve-at-boundary pattern:
```java
@Blocking
public Uni<DispatchResult> sendMessage(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "sender", ...) String sender, ...) {
    Channel ch = resolveChannel(channel);
    return Uni.createFrom().item(() -> blockingSendMessage(ch.name, sender, ...));
}
```

Apply the same structure to `checkMessages` and `waitForReply`.

- [ ] **Step 5: Migrate `requestApproval` — compound, resolve once**

Before:
```java
@Blocking
public Uni<WaitResult> requestApproval(
        @ToolArg(name = "channel_name", ...) String channelName, ...) {
    return Uni.createFrom().item(() -> blockingRequestApproval(channelName, content, timeoutS));
}

private WaitResult blockingRequestApproval(String channelName, ...) {
    blockingSendMessage(channelName, ...);
    return blockingWaitForReply(channelName, ...);
}
```

After:
```java
@Blocking
public Uni<WaitResult> requestApproval(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    Channel ch = resolveChannel(channel);
    return Uni.createFrom().item(() -> blockingRequestApproval(ch.name, content, timeoutS));
}

// private helper unchanged — receives resolved name
private WaitResult blockingRequestApproval(String channelName, ...) {
    blockingSendMessage(channelName, ...);
    return blockingWaitForReply(channelName, ...);
}
```

- [ ] **Step 6: Migrate `respondToApproval`**

Same entity-pattern as blocking class. Rename param, replace lookup:
```java
@Blocking
public Uni<DispatchResult> respondToApproval(
        ...,
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel) {
    Channel ch = resolveChannel(channel);
    return Uni.createFrom().item(() -> blockingRespondToApproval(..., ch.name));
}
```

- [ ] **Step 7: Build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | grep "ReactiveQhorusMcpTools" | head -20
```

---

## Task 12: `ReactiveQhorusMcpTools` — Category B Tools (Part 2: Utility + Ledger) and Final Verification

**Tools:** `clear_channel`, `get_channel_digest`, `get_channel_timeline`, `list_ledger_entries`, `get_obligation_chain`, `get_causal_chain`, `list_stalled_obligations`, `get_obligation_stats`, `get_telemetry_summary`

- [ ] **Step 1: Migrate all 9 utility and ledger tools**

Each follows the resolve-at-boundary pattern for Category B. Template:

```java
@Blocking
public Uni<SomeResult> someChannelTool(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    Channel ch = resolveChannel(channel);
    return Uni.createFrom().item(() -> blockingSomeChannelTool(ch.name, ...));
    // or if the private helper uses ch.id:
    // return Uni.createFrom().item(() -> blockingSomeChannelTool(ch.id, ...));
}
```

For ledger tools, the private helpers use `channelId` (UUID). Resolve `ch.id` before the lambda:
```java
@Blocking
public Uni<List<Map<String, Object>>> listLedgerEntries(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel, ...) {
    Channel ch = resolveChannel(channel);
    return Uni.createFrom().item(() -> blockingListLedgerEntries(ch.id, typeFilter, agentId, since, afterId, correlationId, sort, limit));
}
```

Apply to each of the 9 tools by reading the current method body and adapting the helper call accordingly.

- [ ] **Step 2: Full build — zero errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | grep "error:" | head -20
```
Expected: zero errors.

- [ ] **Step 3: Run full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests pass (including the 4 new UUID tests and `ToolOverloadDiscoverabilityTest`).

- [ ] **Step 4: Run full project build to catch cross-module effects**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
Expected: BUILD SUCCESS across all modules (api, runtime, deployment, testing, examples).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#237): rename channel_name→channel in ReactiveQhorusMcpTools; add resolveChannelAsync()

- Category A tools use resolveChannelAsync() — no @Blocking
- Category B tools resolve at boundary via resolveChannel(), pass ch.name to private helpers
- set_channel_type_constraints drops @Blocking (sole reason was blocking resolveChannel)
- delete_channel: terminal map nested inside flatMap to keep ch in scope

Closes #237"
```

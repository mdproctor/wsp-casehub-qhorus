# MCP Tool Gaps (#125 + #126) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three missing MCP tools: `delete_channel` (with force guard), `get_instance`, and `get_message` — all additive, non-breaking.

**Architecture:** `ChannelService.delete(name, force)` handles the force check and message purge before channel deletion (required because `fk_message_channel` has no CASCADE). `get_instance` and `get_message` are thin wrappers over existing `findByInstanceId` / `findById` service methods using the existing `buildInstanceInfoList` / `toMessageSummary` mappers. All three tools are mirrored in `ReactiveQhorusMcpTools`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5 (NO AssertJ in runtime tests — use `import static org.junit.jupiter.api.Assertions.*`), `QuarkusTransaction.requiringNew()` for test isolation.

**Test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml`
**Specific:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl runtime -f WORKTREE/pom.xml`

**Issue linkage:** All commits must include `Refs #125` or `Refs #126` as appropriate.

---

## File Map

**Modify:**
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java` — add `delete(name, force)` returning `long`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ReactiveChannelService.java` — add `delete(name, force)` returning `Uni<Long>`
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — add `DeleteChannelResult` record
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java` — add 3 `@Tool` methods
- `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — mirror 3 tools
- `runtime/src/test/java/io/quarkiverse/qhorus/channel/ChannelServiceTest.java` — extend with delete tests

**Create:**
- `runtime/src/test/java/io/quarkiverse/qhorus/mcp/DeleteChannelToolTest.java`
- `runtime/src/test/java/io/quarkiverse/qhorus/mcp/GetInstanceToolTest.java`
- `runtime/src/test/java/io/quarkiverse/qhorus/mcp/GetMessageToolTest.java`

---

### Task 1: `ChannelService.delete()` + `ReactiveChannelService.delete()` + service tests

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ReactiveChannelService.java`
- Modify: `runtime/src/test/java/io/quarkiverse/qhorus/channel/ChannelServiceTest.java`

- [ ] **Step 1: Add failing tests to `ChannelServiceTest`**

Read the existing `ChannelServiceTest` first to understand the pattern (uses `@QuarkusTest` + `QuarkusTransaction.requiringNew()`). Then add these tests:

```java
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.MessageService;
import io.quarkiverse.qhorus.runtime.message.MessageType;
// (add imports alongside existing ones)

@Inject
MessageService messageService;

@Test
void delete_emptyChannel_succeeds() {
    String name = "del-empty-" + System.nanoTime();
    QuarkusTransaction.requiringNew().run(() ->
            channelService.create(name, "Test", ChannelSemantic.APPEND, null));

    QuarkusTransaction.requiringNew().run(() -> {
        long deleted = channelService.delete(name, false);
        assertEquals(0L, deleted);
    });

    QuarkusTransaction.requiringNew().run(() ->
            assertTrue(channelService.findByName(name).isEmpty()));
}

@Test
void delete_notFound_throwsIllegalArgument() {
    assertThrows(IllegalArgumentException.class, () ->
            QuarkusTransaction.requiringNew().run(() ->
                    channelService.delete("no-such-channel-" + System.nanoTime(), false)));
}

@Test
void delete_withMessages_forceTrue_deletesMessagesAndChannel() {
    String name = "del-force-" + System.nanoTime();
    QuarkusTransaction.requiringNew().run(() ->
            channelService.create(name, "Test", ChannelSemantic.APPEND, null));

    UUID[] chId = new UUID[1];
    QuarkusTransaction.requiringNew().run(() ->
            chId[0] = channelService.findByName(name).orElseThrow().id);

    QuarkusTransaction.requiringNew().run(() ->
            messageService.send(chId[0], "agent-a", MessageType.STATUS, "hi", null, null));

    QuarkusTransaction.requiringNew().run(() -> {
        long deleted = channelService.delete(name, true);
        assertEquals(1L, deleted);
    });

    QuarkusTransaction.requiringNew().run(() ->
            assertTrue(channelService.findByName(name).isEmpty()));
}

@Test
void delete_withMessages_forceFalse_throwsIllegalState() {
    String name = "del-noforce-" + System.nanoTime();
    QuarkusTransaction.requiringNew().run(() ->
            channelService.create(name, "Test", ChannelSemantic.APPEND, null));

    UUID[] chId = new UUID[1];
    QuarkusTransaction.requiringNew().run(() ->
            chId[0] = channelService.findByName(name).orElseThrow().id);

    QuarkusTransaction.requiringNew().run(() ->
            messageService.send(chId[0], "agent-a", MessageType.STATUS, "hi", null, null));

    Exception ex = assertThrows(Exception.class, () ->
            QuarkusTransaction.requiringNew().run(() ->
                    channelService.delete(name, false)));
    assertTrue(ex.getMessage().contains("1") && ex.getMessage().contains("force=true"),
            "Error should mention message count and force=true: " + ex.getMessage());
}
```

- [ ] **Step 2: Run tests — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelServiceTest -pl runtime -f WORKTREE/pom.xml
```
Expected: compilation failure — `channelService.delete(name, force)` does not exist.

- [ ] **Step 3: Add `delete(String name, boolean force)` to `ChannelService`**

First add `@Inject MessageStore messageStore;` after the existing `@Inject ChannelStore channelStore;`. Add import: `import io.quarkiverse.qhorus.runtime.store.MessageStore;`

Then add the method at the end of the public API section (before private methods):

```java
/**
 * Delete a channel by name.
 *
 * @param name the channel name
 * @param force when true, deletes all messages in the channel before deleting; when false,
 *              throws if the channel has any messages
 * @return the number of messages that were deleted
 * @throws IllegalArgumentException if the channel does not exist
 * @throws IllegalStateException if force=false and the channel has messages
 */
@Transactional
public long delete(final String name, final boolean force) {
    Channel ch = findByName(name)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + name));
    long messageCount = messageStore.countByChannel(ch.id);
    if (messageCount > 0 && !force) {
        throw new IllegalStateException(
                "Channel '" + name + "' has " + messageCount
                        + " messages. Pass force=true to delete anyway.");
    }
    if (messageCount > 0) {
        messageStore.deleteAll(ch.id);
    }
    channelStore.delete(ch.id);
    return messageCount;
}
```

- [ ] **Step 4: Add `delete(String name, boolean force)` to `ReactiveChannelService`**

Add `@Inject public MessageStore messageStore;` (blocking store — needed for count and deleteAll, no reactive equivalents). Add import: `import io.quarkiverse.qhorus.runtime.store.MessageStore;`

Then add the method:

```java
/**
 * Delete a channel by name.
 *
 * @param name the channel name
 * @param force when true, deletes all messages before deleting the channel
 * @return number of messages deleted
 */
public Uni<Long> delete(final String name, final boolean force) {
    return Panache.withTransaction(() -> channelStore.findByName(name)
            .map(opt -> opt.orElseThrow(
                    () -> new IllegalArgumentException("Channel not found: " + name)))
            .map(ch -> {
                long messageCount = messageStore.countByChannel(ch.id);
                if (messageCount > 0 && !force) {
                    throw new IllegalStateException(
                            "Channel '" + name + "' has " + messageCount
                                    + " messages. Pass force=true to delete anyway.");
                }
                if (messageCount > 0) {
                    messageStore.deleteAll(ch.id);
                }
                channelStore.delete(ch.id);
                return messageCount;
            }));
}
```

- [ ] **Step 5: Run tests — expect 4 new tests PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelServiceTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 6: Run full suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 7: Commit**

```bash
git -C WORKTREE add \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ReactiveChannelService.java \
  runtime/src/test/java/io/quarkiverse/qhorus/channel/ChannelServiceTest.java
git -C WORKTREE commit -m "feat(channel): ChannelService.delete(name, force) — purges messages then deletes channel

Refs #125"
```

---

### Task 2: `DeleteChannelResult` record + `delete_channel` MCP tool

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/mcp/DeleteChannelToolTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.quarkiverse.qhorus.mcp;

import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.DeleteChannelResult;
import io.quarkiverse.qhorus.runtime.message.MessageService;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class DeleteChannelToolTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;

    @Test
    void deleteChannel_emptyChannel_returnsSuccessWithZeroMessages() {
        String name = "del-tool-empty-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Test", ChannelSemantic.APPEND, null));

        DeleteChannelResult result = QuarkusTransaction.requiringNew().run(() ->
                tools.deleteChannel(name, false));

        assertEquals(name, result.channelName());
        assertEquals(0L, result.messagesDeleted());
        assertEquals("deleted", result.status());
    }

    @Test
    void deleteChannel_withMessages_forceFalse_throwsWithCount() {
        String name = "del-tool-guard-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Test", ChannelSemantic.APPEND, null));

        UUID[] chId = new UUID[1];
        QuarkusTransaction.requiringNew().run(() ->
                chId[0] = channelService.findByName(name).orElseThrow().id);
        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(chId[0], "agent-a", MessageType.STATUS, "hi", null, null));

        Exception ex = assertThrows(Exception.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        tools.deleteChannel(name, false)));
        assertTrue(ex.getMessage().contains("1"),
                "Error message should include message count: " + ex.getMessage());
    }

    @Test
    void deleteChannel_withMessages_forceTrue_deletesAllAndReturnsCount() {
        String name = "del-tool-force-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Test", ChannelSemantic.APPEND, null));

        UUID[] chId = new UUID[1];
        QuarkusTransaction.requiringNew().run(() ->
                chId[0] = channelService.findByName(name).orElseThrow().id);
        QuarkusTransaction.requiringNew().run(() -> {
            messageService.send(chId[0], "agent-a", MessageType.STATUS, "one", null, null);
            messageService.send(chId[0], "agent-b", MessageType.STATUS, "two", null, null);
        });

        DeleteChannelResult result = QuarkusTransaction.requiringNew().run(() ->
                tools.deleteChannel(name, true));

        assertEquals(2L, result.messagesDeleted());
        assertEquals("deleted", result.status());
    }

    @Test
    void deleteChannel_notFound_throwsIllegalArgument() {
        assertThrows(Exception.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        tools.deleteChannel("no-such-" + System.nanoTime(), false)));
    }

    @Test
    void deleteChannel_afterDeletion_channelNoLongerListed() {
        String name = "del-tool-gone-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Test", ChannelSemantic.APPEND, null));
        QuarkusTransaction.requiringNew().run(() ->
                tools.deleteChannel(name, false));

        QuarkusTransaction.requiringNew().run(() ->
                assertTrue(channelService.findByName(name).isEmpty()));
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeleteChannelToolTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 3: Add `DeleteChannelResult` record to `QhorusMcpToolsBase`**

In `QhorusMcpToolsBase.java`, add this record alongside the other response records (e.g. after `ChannelDetail`):

```java
public record DeleteChannelResult(
        String channelName,
        long messagesDeleted,
        String status) {
}
```

- [ ] **Step 4: Add `delete_channel` tool to `QhorusMcpTools`**

Add this method in the channel management section (e.g. after `resumeChannel`). The tool is `@Transactional` to ensure the channel service call runs in a transaction:

```java
@Tool(name = "delete_channel", description = "Delete a named channel. "
        + "Rejects with an error if the channel has messages unless force=true. "
        + "When force=true, all messages in the channel are deleted before the channel is removed.")
@Transactional
public DeleteChannelResult deleteChannel(
        @ToolArg(name = "channel_name", description = "Name of the channel to delete") String channelName,
        @ToolArg(name = "force", description = "When true, deletes all messages in the channel then deletes the channel. "
                + "When false (default), rejects if messages exist.", required = false) Boolean force) {
    long deleted = channelService.delete(channelName, Boolean.TRUE.equals(force));
    return new DeleteChannelResult(channelName, deleted, "deleted");
}
```

- [ ] **Step 5: Run tests — expect 5 PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeleteChannelToolTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 6: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 7: Commit**

```bash
git -C WORKTREE add \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/test/java/io/quarkiverse/qhorus/mcp/DeleteChannelToolTest.java
git -C WORKTREE commit -m "feat(mcp): delete_channel tool with force guard

Refs #125"
```

---

### Task 3: `get_instance` + `get_message` MCP tools

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/mcp/GetInstanceToolTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/mcp/GetMessageToolTest.java`

- [ ] **Step 1: Write failing tests for `get_instance`**

```java
package io.quarkiverse.qhorus.mcp;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.InstanceInfo;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class GetInstanceToolTest {

    @Inject QhorusMcpTools tools;
    @Inject InstanceService instanceService;

    @Test
    void getInstance_knownId_returnsCorrectInfo() {
        String instanceId = "inst-get-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                instanceService.register(instanceId, "Test agent",
                        List.of("search", "analysis"), null));

        InstanceInfo info = QuarkusTransaction.requiringNew().run(() ->
                tools.getInstance(instanceId));

        assertEquals(instanceId, info.instanceId());
        assertEquals("Test agent", info.description());
        assertTrue(info.capabilities().contains("search"));
        assertTrue(info.capabilities().contains("analysis"));
    }

    @Test
    void getInstance_unknownId_throwsIllegalArgument() {
        Exception ex = assertThrows(Exception.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        tools.getInstance("no-such-instance-" + System.nanoTime())));
        assertTrue(ex.getMessage().contains("not found"),
                "Error should say 'not found': " + ex.getMessage());
    }
}
```

- [ ] **Step 2: Write failing tests for `get_message`**

Create `runtime/src/test/java/io/quarkiverse/qhorus/mcp/GetMessageToolTest.java`:

```java
package io.quarkiverse.qhorus.mcp;

import static org.junit.jupiter.api.Assertions.*;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageService;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.MessageSummary;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class GetMessageToolTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;

    @Test
    void getMessage_knownId_returnsCorrectSummary() {
        String channelName = "msg-get-ch-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND, null));

        Long[] msgId = new Long[1];
        QuarkusTransaction.requiringNew().run(() -> {
            var ch = channelService.findByName(channelName).orElseThrow();
            Message msg = messageService.send(ch.id, "agent-a", MessageType.STATUS,
                    "hello world", null, null);
            msgId[0] = msg.id;
        });

        MessageSummary summary = QuarkusTransaction.requiringNew().run(() ->
                tools.getMessage(msgId[0]));

        assertEquals(msgId[0], summary.messageId());
        assertEquals("agent-a", summary.sender());
        assertEquals("STATUS", summary.messageType());
        assertEquals("hello world", summary.content());
    }

    @Test
    void getMessage_unknownId_throwsIllegalArgument() {
        Exception ex = assertThrows(Exception.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        tools.getMessage(Long.MAX_VALUE)));
        assertTrue(ex.getMessage().contains("not found"),
                "Error should say 'not found': " + ex.getMessage());
    }
}
```

- [ ] **Step 3: Run both test files — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=GetInstanceToolTest+GetMessageToolTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 4: Add `get_instance` tool to `QhorusMcpTools`**

Add in the instance management section (near `list_instances`). `buildInstanceInfoList` is a private method in `QhorusMcpTools` — callable directly since `getInstance` is in the same class:

```java
@Tool(name = "get_instance", description = "Look up a registered instance by its ID. "
        + "Returns full instance details including capabilities and status. "
        + "Throws an error if the instance is not found.")
@Transactional
public InstanceInfo getInstance(
        @ToolArg(name = "instance_id", description = "Instance ID to look up") String instanceId) {
    io.quarkiverse.qhorus.runtime.instance.Instance instance =
            instanceService.findByInstanceId(instanceId)
                    .orElseThrow(() -> new IllegalArgumentException(
                            "Instance not found: " + instanceId));
    return buildInstanceInfoList(java.util.List.of(instance)).get(0);
}
```

- [ ] **Step 5: Add `get_message` tool to `QhorusMcpTools`**

Add in the messaging section (near `search_messages`). `toMessageSummary` is a protected method from `QhorusMcpToolsBase`:

```java
@Tool(name = "get_message", description = "Look up a message by its numeric ID. "
        + "Returns the message summary including content, type, sender, and metadata. "
        + "Throws an error if the message is not found.")
public MessageSummary getMessage(
        @ToolArg(name = "message_id", description = "Numeric message ID") Long messageId) {
    io.quarkiverse.qhorus.runtime.message.Message message = messageService.findById(messageId)
            .orElseThrow(() -> new IllegalArgumentException(
                    "Message not found: " + messageId));
    return toMessageSummary(message);
}
```

- [ ] **Step 6: Run tests — expect 4 tests PASS (2 + 2)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=GetInstanceToolTest+GetMessageToolTest -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 7: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 8: Commit**

```bash
git -C WORKTREE add \
  runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/test/java/io/quarkiverse/qhorus/mcp/GetInstanceToolTest.java \
  runtime/src/test/java/io/quarkiverse/qhorus/mcp/GetMessageToolTest.java
git -C WORKTREE commit -m "feat(mcp): get_instance and get_message single-item lookup tools

Refs #126"
```

---

### Task 4: Mirror all three tools in `ReactiveQhorusMcpTools`

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Read `ReactiveQhorusMcpTools.java`**

Understand the existing pattern — Category A tools return `Uni<T>` (pure reactive); Category B tools are `@Blocking` and delegate to private helpers. The reactive channel service (`ReactiveChannelService`) is already injected.

- [ ] **Step 2: Add `delete_channel` to `ReactiveQhorusMcpTools`**

`ReactiveChannelService.delete()` returns `Uni<Long>`. The tool returns `Uni<DeleteChannelResult>`:

```java
@Tool(name = "delete_channel", description = "Delete a named channel. "
        + "Rejects with an error if the channel has messages unless force=true. "
        + "When force=true, all messages in the channel are deleted before the channel is removed.")
public Uni<DeleteChannelResult> deleteChannel(
        @ToolArg(name = "channel_name", description = "Name of the channel to delete") String channelName,
        @ToolArg(name = "force", description = "When true, deletes all messages then the channel. "
                + "When false (default), rejects if messages exist.", required = false) Boolean force) {
    return channelService.delete(channelName, Boolean.TRUE.equals(force))
            .map(deleted -> new DeleteChannelResult(channelName, deleted, "deleted"));
}
```

Add the missing import if needed:
```java
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.DeleteChannelResult;
```

- [ ] **Step 3: Add `get_instance` to `ReactiveQhorusMcpTools`**

`ReactiveInstanceService.findByInstanceId` may or may not exist. Check the reactive service first. If it doesn't exist, use `@Blocking` and delegate to the blocking `instanceService`. The safest approach — add `@Blocking` since instance lookups are infrequent:

Check: `grep -n "findByInstanceId" runtime/src/main/java/io/quarkiverse/qhorus/runtime/instance/ReactiveInstanceService.java`

If method exists, use reactive call. If not, use blocking approach:

```java
@Blocking
@Tool(name = "get_instance", description = "Look up a registered instance by its ID. "
        + "Returns full instance details including capabilities and status.")
public InstanceInfo getInstance(
        @ToolArg(name = "instance_id", description = "Instance ID to look up") String instanceId) {
    // Uses blocking InstanceService — infrequent admin lookup
    io.quarkiverse.qhorus.runtime.instance.Instance instance =
            instanceService.findByInstanceId(instanceId)
                    .orElseThrow(() -> new IllegalArgumentException(
                            "Instance not found: " + instanceId));
    // Build InstanceInfo inline — reactive class doesn't have buildInstanceInfoList
    return buildInstanceInfoList(java.util.List.of(instance)).get(0);
}
```

Note: `ReactiveQhorusMcpTools` injects `ReactiveInstanceService` — check if it also has access to blocking `InstanceService`. If not, inject it:
```java
@Inject
io.quarkiverse.qhorus.runtime.instance.InstanceService instanceServiceBlocking;
```
and use `instanceServiceBlocking.findByInstanceId(instanceId)`.

**Important:** Read the existing class carefully and follow the pattern used for other blocking lookups before implementing.

- [ ] **Step 4: Add `get_message` to `ReactiveQhorusMcpTools`**

`messageService.findById(Long id)` is available on the blocking `MessageService`. The reactive class already injects `ReactiveMessageStore` but `MessageService` may be accessible too. Use `@Blocking`:

```java
@Blocking
@Tool(name = "get_message", description = "Look up a message by its numeric ID. "
        + "Returns the message summary including content, type, sender, and metadata.")
public MessageSummary getMessage(
        @ToolArg(name = "message_id", description = "Numeric message ID") Long messageId) {
    io.quarkiverse.qhorus.runtime.message.Message message = messageService.findById(messageId)
            .orElseThrow(() -> new IllegalArgumentException(
                    "Message not found: " + messageId));
    return toMessageSummary(message);
}
```

Check what `messageService` is in `ReactiveQhorusMcpTools` — it may be `ReactiveMessageStore` (not `MessageService`). If so, inject `MessageService` directly:
```java
@Inject
io.quarkiverse.qhorus.runtime.message.MessageService messageService;
```

**Read the class before making assumptions.**

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C WORKTREE add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java
git -C WORKTREE commit -m "feat(mcp): mirror delete_channel, get_instance, get_message in ReactiveQhorusMcpTools

Refs #125, #126"
```

---

### Task 5: Documentation — CLAUDE.md + design doc

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/specs/2026-04-13-qhorus-design.md`

- [ ] **Step 1: Read both files**

Read `CLAUDE.md` and `docs/specs/2026-04-13-qhorus-design.md` to understand current state.

- [ ] **Step 2: Update CLAUDE.md**

In the MCP tool surface section (wherever `create_channel`, `pause_channel`, etc. are listed), add:
```
- `delete_channel(channel_name, force=false)` — deletes channel; rejects if messages exist unless force=true; purges messages before delete
- `get_instance(instance_id)` — direct lookup by instance ID; throws if not found
- `get_message(message_id)` — direct lookup by message numeric ID; throws if not found
```

Update the tool count annotation if present (e.g. "~47 tools" → "~50 tools").

Also add to testing conventions:
```
- `delete_channel` with `force=true` calls `messageStore.deleteAll(channelId)` before `channelStore.delete(channelId)` — required because `fk_message_channel` has no CASCADE. Tests must use `QuarkusTransaction.requiringNew()` chains (not `@TestTransaction`) to avoid the ledger REQUIRES_NEW visibility issue.
```

- [ ] **Step 3: Update design doc**

In `docs/specs/2026-04-13-qhorus-design.md`, in the MCP tool surface section, add entries for the three new tools and note that `ChannelService` now injects `MessageStore` for the delete operation.

- [ ] **Step 4: Run full suite — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f WORKTREE/pom.xml
```

- [ ] **Step 5: Commit**

```bash
git -C WORKTREE add CLAUDE.md docs/specs/2026-04-13-qhorus-design.md
git -C WORKTREE commit -m "docs: document delete_channel, get_instance, get_message tools

Closes #125, Closes #126, Refs #119"
```

---

## Self-Review

**Spec coverage:**
- ✅ `ChannelService.delete(name, force)` returning `long` — Task 1
- ✅ `ReactiveChannelService.delete(name, force)` returning `Uni<Long>` — Task 1
- ✅ `MessageStore.deleteAll()` called before `channelStore.delete()` when force=true — Task 1
- ✅ `DeleteChannelResult` record (channelName, messagesDeleted, status) — Task 2
- ✅ `delete_channel` tool (blocking + reactive) — Tasks 2, 4
- ✅ `get_instance` tool (blocking + reactive) — Tasks 3, 4
- ✅ `get_message` tool (blocking + reactive) — Tasks 3, 4
- ✅ Force guard: rejects with message count when force=false and messages exist — Tasks 1, 2
- ✅ Not-found error for all three tools — Tasks 2, 3
- ✅ Documentation — Task 5

**Type consistency:** `DeleteChannelResult` defined in Task 2 (QhorusMcpToolsBase), used in Tasks 2 and 4. `ChannelService.delete()` returns `long` defined in Task 1, called in Task 2. `ReactiveChannelService.delete()` returns `Uni<Long>` defined in Task 1, called in Task 4.

**No placeholders:** Task 4 explicitly tells the implementer to read the existing class before making assumptions about injected services — this is guidance, not a placeholder, because the pattern varies depending on what's injected.

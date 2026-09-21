# Delete QhorusMcpTools — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #452 — chore: delete QhorusMcpTools — migrate 87 test files to direct service calls
**Issue group:** #452, #453

**Goal:** Replace all test usages of the inert `QhorusMcpTools` and `QhorusMcpToolsBase` classes with a new `QhorusTestHelper` in the `testing/` module, then delete both classes.

**Architecture:** Create `QhorusTestHelper` as an `@ApplicationScoped` CDI bean in `testing/` that wraps `ChannelManager`, `MessageDispatcher`, and store interfaces with overloaded convenience methods matching the old QhorusMcpTools names. Migrate all ~87 test files from `tools.*` calls to `helper.*` calls, replacing MCP inner types with native domain types. Fix compliance-report build failure (#453) by adding `@HandWrittenEndpoint` annotations.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, CDI

## Global Constraints

- All test files are in `runtime/src/test/` (verified by audit)
- `testing/` already depends on `casehub-qhorus` (runtime) — no new dependency needed
- `QhorusTestHelper` must be `@ApplicationScoped` (same CDI lifecycle as QhorusMcpTools was)
- Method names match QhorusMcpTools for mechanical migration
- Return native domain types, not MCP DTOs
- `checkMessages` does simple `messageStore.scan()` — no semantic-specific behavior (EPHEMERAL delete, COLLECT clear, BARRIER wait). Tests that need semantic reads call the @McpDomain MessagingApi directly
- Every commit references `Refs #452` or `Refs #453`
- Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` after each migration batch
- Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` for final verification

---

## Batch 1: Foundation + Compliance Fix

### Task 1: Create QhorusTestHelper

**Files:**
- Create: `testing/src/main/java/io/casehub/qhorus/testing/QhorusTestHelper.java`
- Test: `testing/src/test/java/io/casehub/qhorus/testing/QhorusTestHelperTest.java`

**Interfaces:**
- Produces: `QhorusTestHelper` — CDI bean with `createChannel()`, `sendMessage()`, `checkMessages()`, `registerInstance()`, `shareArtefact()`, `registerWatchdog()`, `deleteChannel()` overloads. All test migration tasks depend on this.

- [ ] **Step 1: Write the test**

```java
package io.casehub.qhorus.testing;

import io.casehub.qhorus.api.channel.ChannelManager;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.store.*;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.message.Message;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkus.test.TestTransaction;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class QhorusTestHelperTest {

    @Inject QhorusTestHelper helper;

    @Test
    @TestTransaction
    void createChannel_simpleName_createsAppendChannel() {
        var channel = helper.createChannel("test-ch-1");
        assertThat(channel).isNotNull();
        assertThat(channel.name).isEqualTo("test-ch-1");
        assertThat(channel.semantic).isEqualTo(ChannelSemantic.APPEND);
    }

    @Test
    @TestTransaction
    void createChannel_withSemantic_respectsSemantic() {
        var channel = helper.createChannel("test-barrier-1", ChannelSemantic.BARRIER,
                List.of("agent-a", "agent-b"));
        assertThat(channel.semantic).isEqualTo(ChannelSemantic.BARRIER);
    }

    @Test
    @TestTransaction
    void sendMessage_simple_dispatches() {
        helper.createChannel("test-msg-ch");
        var result = helper.sendMessage("test-msg-ch", "alice", "status", "Hello");
        assertThat(result.messageId()).isNotNull();
        assertThat(result.sender()).isEqualTo("alice");
    }

    @Test
    @TestTransaction
    void checkMessages_returnsMessageViews() {
        helper.createChannel("test-check-ch");
        helper.sendMessage("test-check-ch", "alice", "status", "msg1");
        helper.sendMessage("test-check-ch", "bob", "status", "msg2");
        var messages = helper.checkMessages("test-check-ch", 0L, 10);
        assertThat(messages).hasSize(2);
        assertThat(messages.get(0).content()).isEqualTo("msg1");
    }

    @Test
    @TestTransaction
    void registerInstance_createsInstance() {
        var instance = helper.registerInstance("agent-1", "Test agent", "skill-a", "skill-b");
        assertThat(instance).isNotNull();
        assertThat(instance.instanceId).isEqualTo("agent-1");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=QhorusTestHelperTest`
Expected: FAIL — `QhorusTestHelper` class does not exist yet.

- [ ] **Step 3: Implement QhorusTestHelper**

```java
package io.casehub.qhorus.testing;

import io.casehub.qhorus.api.channel.ChannelManager;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.message.MessageResult;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.api.store.CommitmentStore;
import io.casehub.qhorus.api.store.DataStore;
import io.casehub.qhorus.api.store.InstanceStore;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.WatchdogStore;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.data.SharedData;
import io.casehub.qhorus.runtime.instance.Capability;
import io.casehub.qhorus.runtime.instance.Instance;
import io.casehub.qhorus.runtime.message.ActorType;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.DispatchResult;
import io.casehub.qhorus.runtime.message.MessageDispatch;
import io.casehub.qhorus.runtime.message.MessageQuery;
import io.casehub.qhorus.runtime.message.MessageType;
import io.casehub.qhorus.runtime.message.MessageView;
import io.casehub.qhorus.runtime.watchdog.Watchdog;
import io.casehub.qhorus.runtime.watchdog.WatchdogAction;
import io.casehub.qhorus.runtime.watchdog.WatchdogConditionType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class QhorusTestHelper {

    @Inject ChannelManager channelManager;
    @Inject ChannelStore channelStore;
    @Inject MessageDispatcher messageDispatcher;
    @Inject MessageStore messageStore;
    @Inject InstanceStore instanceStore;
    @Inject DataStore dataStore;
    @Inject WatchdogStore watchdogStore;
    @Inject CommitmentStore commitmentStore;

    // --- Channel creation ---

    public Channel createChannel(String name) {
        return channelManager.create(ChannelCreateRequest.builder(name).build());
    }

    public Channel createChannel(String name, ChannelSemantic semantic) {
        return channelManager.create(ChannelCreateRequest.builder(name)
                .semantic(semantic)
                .build());
    }

    public Channel createChannel(String name, ChannelSemantic semantic,
                                  List<String> barrierContributors) {
        return channelManager.create(ChannelCreateRequest.builder(name)
                .semantic(semantic)
                .barrierContributors(barrierContributors)
                .build());
    }

    public Channel createChannel(ChannelCreateRequest request) {
        return channelManager.create(request);
    }

    // --- Message sending ---

    public DispatchResult sendMessage(String channelName, String sender,
                                       String type, String content) {
        return sendMessage(channelName, sender, type, content, null, null);
    }

    public DispatchResult sendMessage(String channelName, String sender,
                                       String type, String content,
                                       String correlationId) {
        return sendMessage(channelName, sender, type, content, correlationId, null);
    }

    public DispatchResult sendMessage(String channelName, String sender,
                                       String type, String content,
                                       String correlationId, Long inReplyTo) {
        return sendMessage(channelName, sender, type, content,
                null, correlationId, inReplyTo, null, null, null, null, null, null);
    }

    public DispatchResult sendMessage(String channelName, String sender,
                                       String type, String content,
                                       String payload, String correlationId,
                                       Long inReplyTo, String artefactRefs,
                                       String target, String deadline,
                                       String subjectId, String causedByEntryId,
                                       String topic) {
        UUID channelId = resolveChannelId(channelName);
        var builder = MessageDispatch.builder()
                .channelId(channelId)
                .sender(sender)
                .type(MessageType.valueOf(type.toUpperCase()))
                .actorType(ActorType.AGENT);

        if (content != null) builder.content(content);
        if (payload != null) builder.payload(payload);
        if (correlationId != null) builder.correlationId(correlationId);
        if (inReplyTo != null) builder.inReplyTo(inReplyTo);
        if (target != null) builder.target(target);
        if (topic != null) builder.topic(topic);
        if (subjectId != null) builder.subjectId(UUID.fromString(subjectId));
        if (causedByEntryId != null) builder.causedByEntryId(UUID.fromString(causedByEntryId));

        return messageDispatcher.dispatch(builder.build());
    }

    public DispatchResult dispatch(MessageDispatch dispatch) {
        return messageDispatcher.dispatch(dispatch);
    }

    // --- Message queries ---

    public List<MessageView> checkMessages(String channelName, Long afterId, int limit) {
        return checkMessages(channelName, afterId, limit, null, null, null);
    }

    public List<MessageView> checkMessages(String channelName, Long afterId, int limit,
                                            String sender, String readerInstanceId,
                                            Boolean includeEvents) {
        UUID channelId = resolveChannelId(channelName);
        // findRecent returns List<MessageView> — simplest path
        // For filtered queries (sender, events), fall back to scan + convert
        var views = messageStore.findRecent(channelId, limit);
        if (afterId != null && afterId > 0) {
            views = views.stream().filter(v -> v.messageId() > afterId).toList();
        }
        if (sender != null) {
            views = views.stream().filter(v -> sender.equals(v.sender())).toList();
        }
        if (includeEvents == null || !includeEvents) {
            views = views.stream()
                    .filter(v -> v.type() != MessageType.EVENT)
                    .toList();
        }
        return views;
    }

    // --- Instance management ---

    public Instance registerInstance(String instanceId, String description,
                                      String... capabilities) {
        var instance = new Instance();
        instance.instanceId = instanceId;
        instance.description = description;
        instance.status = "online";
        instanceStore.put(instance);
        if (capabilities != null) {
            for (String cap : capabilities) {
                var capability = new Capability();
                capability.instance = instance;
                capability.tag = cap;
                instanceStore.putCapability(capability);
            }
        }
        return instance;
    }

    // --- Artefact management ---

    public SharedData shareArtefact(String key, String description,
                                     String content, String createdBy) {
        var data = new SharedData(key, description, createdBy, content, true,
                content != null ? content.length() : 0, null, null, null, null, null);
        dataStore.put(data);
        return data;
    }

    // --- Watchdog ---

    public Watchdog registerWatchdog(String conditionType, String targetName,
                                      Integer thresholdSeconds, Integer thresholdCount,
                                      Integer similarityPct, String notificationChannel,
                                      String createdBy, String action) {
        var watchdog = new Watchdog();
        watchdog.conditionType = WatchdogConditionType.valueOf(conditionType);
        watchdog.targetName = targetName;
        watchdog.thresholdSeconds = thresholdSeconds;
        watchdog.thresholdCount = thresholdCount;
        watchdog.similarityPct = similarityPct;
        watchdog.notificationChannel = notificationChannel;
        watchdog.createdBy = createdBy;
        watchdog.action = action != null ? WatchdogAction.valueOf(action) : WatchdogAction.ALERT;
        watchdogStore.save(watchdog);
        return watchdog;
    }

    // --- Channel operations ---

    public void deleteChannel(String channelName, boolean force) {
        UUID channelId = resolveChannelId(channelName);
        channelManager.delete(channelId, force);
    }

    public void pauseChannel(String channelName) {
        UUID channelId = resolveChannelId(channelName);
        channelManager.pause(channelId);
    }

    public void resumeChannel(String channelName) {
        UUID channelId = resolveChannelId(channelName);
        channelManager.resume(channelId);
    }

    // --- Utilities ---

    public UUID resolveChannelId(String channelName) {
        return channelStore.findByName(channelName)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Channel not found: " + channelName))
                .id;
    }

    public Channel findChannel(String channelName) {
        return channelStore.findByName(channelName)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Channel not found: " + channelName));
    }
}
```

**Note:** The exact field names, constructor signatures, and method names on domain types (Instance, SharedData, Watchdog, Capability, etc.) must be verified against the current codebase at implementation time. The above is based on audit findings — field visibility (public vs accessor) and exact constructor arities may differ. The implementer should read each domain class before writing the helper method that constructs it.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=QhorusTestHelperTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add testing/src/main/java/io/casehub/qhorus/testing/QhorusTestHelper.java \
       testing/src/test/java/io/casehub/qhorus/testing/QhorusTestHelperTest.java
git commit -m "feat(#452): add QhorusTestHelper — test convenience bean wrapping services/stores

Refs #452

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Fix compliance-report @HandWrittenEndpoint (#453)

**Files:**
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java`
- Modify: `compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceScheduleResource.java`

**Interfaces:**
- Consumes: nothing from Task 1
- Produces: compilable compliance-report module (unblocks full `mvn install`)

- [ ] **Step 1: Add @HandWrittenEndpoint to ComplianceReportResource**

Add the annotation to the class declaration:

```java
@HandWrittenEndpoint("multipart upload, content negotiation, binary signature downloads")
@Path("/api/compliance")
@ApplicationScoped
public class ComplianceReportResource {
```

The `@HandWrittenEndpoint` import comes from the platform APT generator's API module (verify exact package at implementation time — likely `io.casehub.platform.api.rest.HandWrittenEndpoint` or similar).

- [ ] **Step 2: Add @HandWrittenEndpoint to ComplianceScheduleResource**

```java
@HandWrittenEndpoint("CRUD resource paired with hand-written ComplianceReportResource")
@Path("/api/compliance/schedules")
@ApplicationScoped
public class ComplianceScheduleResource {
```

- [ ] **Step 3: Verify the compliance-report module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl compliance-report`
Expected: BUILD SUCCESS (no more "Hand-written @Path" error)

- [ ] **Step 4: Commit**

```bash
git add compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceReportResource.java \
       compliance-report/src/main/java/io/casehub/qhorus/compliance/api/ComplianceScheduleResource.java
git commit -m "fix(#453): add @HandWrittenEndpoint to compliance REST resources

ComplianceReportResource uses multipart upload, content negotiation,
and binary .p7s downloads — features @McpDomain cannot express.
ComplianceScheduleResource is paired CRUD. Both are already covered
by ComplianceApi @McpDomain(\"compliance\") for the MCP channel.

Closes #453

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Migrate runtime test files

### Task 3: Migrate all test files from QhorusMcpTools to QhorusTestHelper

**Files:**
- Modify: all ~87 test files in `runtime/src/test/` that import `QhorusMcpTools` or `QhorusMcpToolsBase`
- Test: each file's existing tests must pass after migration

**Interfaces:**
- Consumes: `QhorusTestHelper` from Task 1

**Discovery — find all files to migrate:**

```bash
grep -rl "QhorusMcpTools\|QhorusMcpToolsBase" runtime/src/test/ --include="*.java"
```

**Migration rules — apply to EVERY file:**

**Rule 1: Replace injection**
```java
// BEFORE
@Inject QhorusMcpTools tools;
// AFTER
@Inject QhorusTestHelper helper;
```

**Rule 2: Replace imports**
```java
// REMOVE
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.casehub.qhorus.runtime.mcp.QhorusMcpToolsBase;
// ADD
import io.casehub.qhorus.testing.QhorusTestHelper;
```

**Rule 3: Replace createChannel calls**
```java
// BEFORE (19 args, mostly null)
tools.createChannel("name", "desc", null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null);
// AFTER
helper.createChannel("name");

// BEFORE (with semantic)
tools.createChannel("name", "desc", "BARRIER", "a,b", null, null, null, null, null, null, null, null, null, null, null, null, null, null, null);
// AFTER
helper.createChannel("name", ChannelSemantic.BARRIER, List.of("a", "b"));

// BEFORE (with constraints — allowedTypes, deniedTypes, etc.)
tools.createChannel("name", "desc", "APPEND", null, "writer1", null, null, null, "COMMAND,QUERY", null, null, null, null, null, null, null, null, null, null);
// AFTER — use the full builder
helper.createChannel(ChannelCreateRequest.builder("name")
        .allowedWriters(List.of("writer1"))
        .allowedTypes(Set.of(MessageType.COMMAND, MessageType.QUERY))
        .build());
```

**Rule 4: Replace sendMessage calls**
```java
// BEFORE (13 args, mostly null)
tools.sendMessage("channel", "sender", "status", "content", null, null, null, null, null, null, null, null, null);
// AFTER
helper.sendMessage("channel", "sender", "status", "content");

// BEFORE (with correlationId)
tools.sendMessage("channel", "sender", "command", "content", null, "corr-1", null, null, null, null, null, null, null);
// AFTER
helper.sendMessage("channel", "sender", "command", "content", "corr-1");

// BEFORE (with correlationId + inReplyTo — terminal types)
tools.sendMessage("channel", "sender", "done", "content", null, "corr-1", 42L, null, null, null, null, null, null);
// AFTER
helper.sendMessage("channel", "sender", "done", "content", "corr-1", 42L);

// BEFORE (with many non-null args — target, topic, etc.)
tools.sendMessage("channel", "sender", "command", "content", null, "corr-1", null, null, "role:specialist", null, null, null, "my-topic");
// AFTER — use the full 13-arg overload
helper.sendMessage("channel", "sender", "command", "content", null, "corr-1", null, null, "role:specialist", null, null, null, "my-topic");
```

**Rule 5: Replace checkMessages calls**
```java
// BEFORE
QhorusMcpTools.CheckResult result = tools.checkMessages("channel", 0L, 10, null, null, null);
result.messages().get(0).content();
// AFTER
List<MessageView> messages = helper.checkMessages("channel", 0L, 10);
messages.get(0).content();

// Add import:
import io.casehub.qhorus.runtime.message.MessageView;
import java.util.List;
```

**Rule 6: Replace inner type references**
```java
// BEFORE
QhorusMcpTools.CheckResult → List<MessageView>
QhorusMcpTools.MessageSummary → MessageView
QhorusMcpTools.ArtefactDetail → SharedData
QhorusMcpTools.WatchdogSummary → Watchdog
QhorusMcpTools.RegisterResponse → Instance (just use the returned instance)
QhorusMcpTools.CommitmentDetail → Commitment
QhorusMcpToolsBase.ChannelInfo → ChannelDetail
```

**Rule 7: Replace registerInstance calls**
```java
// BEFORE
tools.register("agent-1", "description", "cap-a,cap-b", false);
// AFTER
helper.registerInstance("agent-1", "description", "cap-a", "cap-b");
```

**Rule 8: Replace shareArtefact calls**
```java
// BEFORE
tools.shareArtefact("key", "desc", "creator", "content", false, true);
// AFTER
helper.shareArtefact("key", "desc", "content", "creator");
```

**Rule 9: Replace registerWatchdog calls**
```java
// BEFORE
tools.registerWatchdog("BARRIER_STUCK", "*", 30, null, null, "alerts", "system", null);
// AFTER
helper.registerWatchdog("BARRIER_STUCK", "*", 30, null, null, "alerts", "system", null);
```

**Rule 10: Replace tools. → helper. for remaining calls**

Any remaining `tools.` references (deleteChannel, pauseChannel, listChannels, etc.) become `helper.` calls. If the helper doesn't have a matching method, add one following the same pattern (delegate to the appropriate service/store).

- [ ] **Step 1: Discover all files to migrate**

```bash
grep -rl "QhorusMcpTools\|QhorusMcpToolsBase" runtime/src/test/ --include="*.java" | sort
```

Record the list. Process files in alphabetical order within each package.

- [ ] **Step 2: Migrate each file applying rules 1-10**

For each file:
1. Read the file
2. Identify which QhorusMcpTools methods and inner types it uses
3. Apply the applicable rules
4. If the file uses a method the helper doesn't have yet, add the method to QhorusTestHelper first
5. Save the file

**Important edge cases:**
- Some tests import `QhorusMcpToolsBase` directly for inner types — replace with domain type imports
- Some tests cast or destructure `CheckResult` — replace with `List<MessageView>` operations
- Tests that call `tools.checkMessages()` on BARRIER/COLLECT/EPHEMERAL channels for semantic-read behavior: replace with direct `messageStore.scan()` or the appropriate @McpDomain API call. These are rare — most tests use APPEND channels
- Tests that reference `result.lastId()` from `CheckResult`: use `messages.get(messages.size() - 1).messageId()` or equivalent
- Tests that reference `result.barrierStatus()` from `CheckResult`: these test BARRIER semantics — migrate to call the MessagingApi @McpDomain tool directly or query the channel state

- [ ] **Step 3: Run the full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests pass. Fix any failures before proceeding.

- [ ] **Step 4: Commit in logical groups**

Commit after each package/subdomain batch (e.g., all channel tests, all message tests). Example:

```bash
git add runtime/src/test/
git commit -m "chore(#452): migrate runtime test files to QhorusTestHelper

Replace QhorusMcpTools injection with QhorusTestHelper across all
runtime test files. MCP inner types replaced with native domain types.

Refs #452

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

If the migration is done in sub-batches (by package), commit each sub-batch separately with a more specific message.

---

## Batch 3: Migrate tests in other modules

### Task 4: Migrate non-runtime test files

**Files:**
- Discover: any test files in modules outside `runtime/` that import QhorusMcpTools

```bash
grep -rl "QhorusMcpTools\|QhorusMcpToolsBase" --include="*.java" \
  connector-backend/ slack-channel/ websocket-observer/ webhook-observer/ \
  compliance-report/ a2a-push-notification/ notification-bridge/ \
  a2a-outbound/ agent-card-signing/ postgres-broadcaster/ \
  examples/ persistence-memory/ 2>/dev/null
```

**Interfaces:**
- Consumes: `QhorusTestHelper` from Task 1

- [ ] **Step 1: Discover files**

Run the grep command above. If no files found, skip this task entirely.

- [ ] **Step 2: Migrate each file using the same rules from Task 3**

Apply rules 1-10 from Task 3. Each module may need `casehub-qhorus-testing` as a test dependency — verify the module's `pom.xml` includes it.

- [ ] **Step 3: Run tests for each affected module**

For each module with migrated files:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module-name>
```

- [ ] **Step 4: Commit**

```bash
git commit -m "chore(#452): migrate non-runtime test files to QhorusTestHelper

Refs #452

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Delete + Verify

### Task 5: Delete QhorusMcpTools and QhorusMcpToolsBase

**Files:**
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`

**Interfaces:**
- Consumes: all test files migrated (Tasks 3-4 complete)
- Produces: clean codebase with no QhorusMcpTools references

- [ ] **Step 1: Verify no remaining references**

```bash
grep -r "QhorusMcpTools\|QhorusMcpToolsBase" --include="*.java" runtime/ testing/ \
  connector-backend/ slack-channel/ websocket-observer/ webhook-observer/ \
  compliance-report/ a2a-push-notification/ notification-bridge/ \
  a2a-outbound/ examples/ persistence-memory/ api/ deployment/
```

Expected: no results (or only the files about to be deleted).

- [ ] **Step 2: Delete the files**

Use `ide_refactor_safe_delete` for each file to ensure no remaining references:

```
ide_refactor_safe_delete:
  path: runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java

ide_refactor_safe_delete:
  path: runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java
```

If safe_delete reports remaining usages, fix them first.

- [ ] **Step 3: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 4: Check profile-gated modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -Pwith-llm-examples -f examples/agent-communication/pom.xml
```

Expected: compiles successfully (no stale QhorusMcpTools references).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore(#452): delete QhorusMcpTools and QhorusMcpToolsBase

All 87 test files migrated to QhorusTestHelper. No remaining references
to the inert MCP tool classes.

Closes #452

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-21-delete-qhorus-mcp-tools-design.md] — design spec this plan implements
- [runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java] — source to delete
- [runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java] — source to delete
- [testing/src/main/java/io/casehub/qhorus/testing/] — home for QhorusTestHelper
- [api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java] — channel service interface
- [api/src/main/java/io/casehub/qhorus/api/message/MessageDispatcher.java] — dispatch interface
- [GitHub #452] — focal issue
- [GitHub #453] — compliance build failure
- [GitHub #451] — predecessor @McpDomain migration

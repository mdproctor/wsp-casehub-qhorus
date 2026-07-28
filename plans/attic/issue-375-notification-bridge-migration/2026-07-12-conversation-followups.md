# Conversation Model Followups Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #339 — get_reactions_batch MCP tool
**Issue group:** #339, #336, #335, #340, #333

**Goal:** Add batch reaction fetching, topic merge/move operations, ArtefactType.DEBATE, and presence with heartbeat degradation to the Qhorus communication mesh.

**Architecture:** #339 wraps existing `ReactionService.getReactionsBatch()` as an MCP tool. #336 and #335 add `TopicService.merge()` and `TopicService.move()` — merge is intra-channel reorganization; move is cross-channel with commitment gate and semantic validation. #340 adds a DEBATE enum value. #333 introduces ephemeral presence via Caffeine cache with lazy timeout degradation, computed on read — no scheduler, no database.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM (H2 tests), Caffeine cache, Jackson ObjectMapper, quarkus-mcp-server 1.11.1

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
- Use `mvn` not `./mvnw`
- After API visibility changes, run `mvn install` from project root (not scoped `mvn test`)
- Named datasource `qhorus` — all JPA entities, stores, and queries use the qhorus PU
- `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")` gates reactive beans
- `@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)` gates blocking beans that conflict with reactive counterparts
- InMemory stores: `@Alternative @Priority(1)` in `persistence-memory/`
- CDI-free unit tests: set `service.tracingConfig` to disabled-tracing impl
- `ToolOverloadDiscoverabilityTest` — non-`@Tool` public overloads of `@Tool` method names silently drop the tool. Use package-private visibility.
- `import-qhorus-test.sql` for non-JPA tables in test H2
- Pre-release — breaking changes cost nothing
- MCP tool channel resolution: resolve at `@Tool` boundary via `resolveChannel()` (blocking) or `resolveChannelAsync()` (reactive). See PP-20260606-f899bc.
- Audit EVENTs emitted at MCP tool layer (not service layer) — follows `rename_topic` pattern.
- InMemory store implementations must not mutate PanacheEntity fields in-place. See PP-20260618-100368.

---

### Task 1: get_reactions_batch MCP tool (#339)

Wraps existing `ReactionService.getReactionsBatch()` as an MCP tool with batch-size validation.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add `getReactionsBatch` `@Tool` method
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — add reactive counterpart
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/GetReactionsBatchToolTest.java`

**Interfaces:**
- Consumes: `ReactionService.getReactionsBatch(Collection<Long>)` (existing)
- Produces: MCP tools `get_reactions_batch` (blocking + reactive)

- [ ] **Step 1: Write failing test**

```java
package io.casehub.qhorus.runtime.mcp;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import io.casehub.qhorus.api.message.ReactionGroup;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.runtime.message.ReactionService;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class GetReactionsBatchToolTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject ReactionService reactionService;

    @Test
    void batchReturnsGroupedReactions() {
        var ch = channelService.create(
            io.casehub.qhorus.runtime.channel.ChannelCreateRequest.builder("batch-react-test").build());
        var msg1 = messageService.dispatch(MessageDispatch.builder()
            .channelId(ch.id()).sender("agent-a").type(MessageType.STATUS)
            .content("hello").actorType(ActorType.AGENT).build());
        var msg2 = messageService.dispatch(MessageDispatch.builder()
            .channelId(ch.id()).sender("agent-a").type(MessageType.STATUS)
            .content("world").actorType(ActorType.AGENT).build());

        reactionService.react(msg1.messageId(), "👍", "user-1", null);
        reactionService.react(msg2.messageId(), "❤️", "user-2", null);

        Map<Long, List<ReactionGroup>> result = tools.getReactionsBatch(
            List.of(msg1.messageId(), msg2.messageId()));

        assertThat(result).containsKeys(msg1.messageId(), msg2.messageId());
        assertThat(result.get(msg1.messageId())).hasSize(1);
        assertThat(result.get(msg1.messageId()).get(0).emoji()).isEqualTo("👍");
    }

    @Test
    void emptyListRejected() {
        assertThatThrownBy(() -> tools.getReactionsBatch(List.of()))
            .isInstanceOf(io.quarkiverse.mcp.server.ToolCallException.class);
    }

    @Test
    void nullListRejected() {
        assertThatThrownBy(() -> tools.getReactionsBatch(null))
            .isInstanceOf(io.quarkiverse.mcp.server.ToolCallException.class);
    }

    @Test
    void oversizedListRejected() {
        var ids = java.util.stream.LongStream.rangeClosed(1, 201).boxed().toList();
        assertThatThrownBy(() -> tools.getReactionsBatch(ids))
            .isInstanceOf(io.quarkiverse.mcp.server.ToolCallException.class);
    }

    @Test
    void missingMessagesReturnEmptyReactions() {
        Map<Long, List<ReactionGroup>> result = tools.getReactionsBatch(List.of(999999L));
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=GetReactionsBatchToolTest -pl runtime`
Expected: compilation error — `getReactionsBatch` method does not exist on `QhorusMcpTools`

- [ ] **Step 3: Implement blocking MCP tool**

Add to `QhorusMcpTools.java`, near the existing `getReactions` method (around line 1877):

```java
@Tool(name = "get_reactions_batch", description = "Get reactions for multiple messages in one call, grouped by emoji with actor lists per message")
public Map<Long, List<ReactionGroup>> getReactionsBatch(
        @ToolArg(name = "message_ids", description = "List of message IDs to fetch reactions for (max 200)") List<Long> messageIds) {
    if (messageIds == null || messageIds.isEmpty()) {
        throw new IllegalArgumentException("message_ids must be non-null and non-empty");
    }
    if (messageIds.size() > 200) {
        throw new IllegalArgumentException("message_ids cannot exceed 200 entries");
    }
    return reactionService.getReactionsBatch(messageIds);
}
```

- [ ] **Step 4: Implement reactive MCP tool**

Add to `ReactiveQhorusMcpTools.java`, matching the pattern of other reactive tools:

```java
@Tool(name = "get_reactions_batch", description = "Get reactions for multiple messages in one call, grouped by emoji with actor lists per message")
@Blocking
public Map<Long, List<ReactionGroup>> getReactionsBatch(
        @ToolArg(name = "message_ids", description = "List of message IDs to fetch reactions for (max 200)") List<Long> messageIds) {
    if (messageIds == null || messageIds.isEmpty()) {
        throw new IllegalArgumentException("message_ids must be non-null and non-empty");
    }
    if (messageIds.size() > 200) {
        throw new IllegalArgumentException("message_ids cannot exceed 200 entries");
    }
    return reactionService.getReactionsBatch(messageIds);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=GetReactionsBatchToolTest -pl runtime`
Expected: all 5 tests PASS

- [ ] **Step 6: Verify ToolOverloadDiscoverabilityTest still passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ToolOverloadDiscoverabilityTest -pl runtime`
Expected: PASS — `getReactionsBatch` has no non-`@Tool` public overloads

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java runtime/src/test/java/io/casehub/qhorus/runtime/mcp/GetReactionsBatchToolTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#339): get_reactions_batch MCP tool with 200-entry limit"
```

---

### Task 2: mergeTopics service + MCP tool (#336)

Adds `TopicService.merge()` and `merge_topics` MCP tool. Uses existing `MessageStore.updateTopicName()` and `TopicStore.delete()`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicService.java` — add `merge()` method and `MergeResult` record
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add `mergeTopics` `@Tool` method
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — add reactive counterpart
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/TopicMergeTest.java` — service unit tests
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/MergeTopicsToolTest.java` — MCP integration test

**Interfaces:**
- Consumes: `TopicStore.find()`, `TopicStore.delete()`, `MessageStore.updateTopicName()` (all existing)
- Produces: `TopicService.merge(UUID channelId, String sourceTopic, String targetTopic, String actorId) → MergeResult`; `TopicService.MergeResult(String sourceTopic, String targetTopic, int messagesUpdated)`

- [ ] **Step 1: Write TopicService.merge() unit tests**

```java
package io.casehub.qhorus.runtime.message;

import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.qhorus.persistence.memory.InMemoryTopicStore;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.Topic;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class TopicMergeTest {

    private TopicService topicService;
    private InMemoryTopicStore topicStore;
    private InMemoryMessageStore messageStore;

    @BeforeEach
    void setUp() {
        topicStore = new InMemoryTopicStore();
        messageStore = new InMemoryMessageStore();
        topicService = new TopicService();
        topicService.topicStore = topicStore;
        topicService.messageStore = messageStore;
    }

    private UUID channelId = UUID.randomUUID();

    private void addTopic(String name) {
        topicStore.put(new Topic(null, channelId, name, false, null, null, Instant.now(), null));
    }

    private void addMessage(String topic) {
        messageStore.put(Message.builder()
            .channelId(channelId).sender("agent").messageType(MessageType.STATUS)
            .actorType(ActorType.AGENT).content("msg").topic(topic)
            .build());
    }

    @Test
    void mergeUpdatesMessagesAndDeletesSource() {
        addTopic("bugs");
        addTopic("issues");
        addMessage("bugs");
        addMessage("bugs");
        addMessage("issues");

        TopicService.MergeResult result = topicService.merge(channelId, "bugs", "issues", "admin");

        assertThat(result.sourceTopic()).isEqualTo("bugs");
        assertThat(result.targetTopic()).isEqualTo("issues");
        assertThat(result.messagesUpdated()).isEqualTo(2);
        assertThat(topicStore.find(channelId, "bugs")).isEmpty();
        assertThat(topicStore.find(channelId, "issues")).isPresent();
    }

    @Test
    void mergeIntoGeneralAllowed() {
        addTopic("cleanup");
        topicService.ensureExists(channelId, "general", null);
        addMessage("cleanup");

        TopicService.MergeResult result = topicService.merge(channelId, "cleanup", "general", "admin");

        assertThat(result.messagesUpdated()).isEqualTo(1);
        assertThat(topicStore.find(channelId, "cleanup")).isEmpty();
    }

    @Test
    void mergeFromGeneralRejected() {
        topicService.ensureExists(channelId, "general", null);
        addTopic("target");

        assertThatThrownBy(() -> topicService.merge(channelId, "general", "target", "admin"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("general");
    }

    @Test
    void mergeSourceNotFoundRejected() {
        addTopic("target");

        assertThatThrownBy(() -> topicService.merge(channelId, "nonexistent", "target", "admin"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("not found");
    }

    @Test
    void mergeTargetNotFoundRejected() {
        addTopic("source");

        assertThatThrownBy(() -> topicService.merge(channelId, "source", "nonexistent", "admin"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("not found");
    }

    @Test
    void mergeSameTopicRejected() {
        addTopic("same");

        assertThatThrownBy(() -> topicService.merge(channelId, "same", "same", "admin"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void mergeNormalisesNames() {
        addTopic("bugs");
        addTopic("issues");
        addMessage("bugs");

        TopicService.MergeResult result = topicService.merge(channelId, "  Bugs  ", "  Issues  ", "admin");

        assertThat(result.sourceTopic()).isEqualTo("bugs");
        assertThat(result.targetTopic()).isEqualTo("issues");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TopicMergeTest -pl runtime`
Expected: compilation error — `merge()` and `MergeResult` do not exist

- [ ] **Step 3: Implement TopicService.merge()**

Add to `TopicService.java` after the `rename` method:

```java
public MergeResult merge(UUID channelId, String sourceTopic, String targetTopic, String actorId) {
    String normalSource = normalise(sourceTopic);
    String normalTarget = normalise(targetTopic);
    if (DEFAULT_TOPIC.equalsIgnoreCase(normalSource)) {
        throw new IllegalArgumentException("Cannot merge from the default topic 'general'");
    }
    if (normalSource.equalsIgnoreCase(normalTarget)) {
        throw new IllegalArgumentException("Source and target topics must be different");
    }
    if (topicStore.find(channelId, normalSource).isEmpty()) {
        throw new IllegalArgumentException("Source topic '" + normalSource + "' not found");
    }
    if (topicStore.find(channelId, normalTarget).isEmpty()) {
        throw new IllegalArgumentException("Target topic '" + normalTarget + "' not found");
    }
    int messagesUpdated = messageStore.updateTopicName(channelId, normalSource, normalTarget);
    topicStore.delete(channelId, normalSource);
    return new MergeResult(normalSource, normalTarget, messagesUpdated);
}

public record MergeResult(String sourceTopic, String targetTopic, int messagesUpdated) {}
```

- [ ] **Step 4: Run unit tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TopicMergeTest -pl runtime`
Expected: all 7 tests PASS

- [ ] **Step 5: Write MCP tool integration test**

```java
package io.casehub.qhorus.runtime.mcp;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.message.TopicService;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.api.identity.ActorType;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class MergeTopicsToolTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject TopicService topicService;

    @Test
    @Transactional
    void mergeTopicsEmitsEventAndDeletesSource() {
        var ch = channelService.create(ChannelCreateRequest.builder("merge-test").build());
        topicService.ensureExists(ch.id(), "bugs", null);
        topicService.ensureExists(ch.id(), "issues", null);

        messageService.dispatch(MessageDispatch.builder()
            .channelId(ch.id()).sender("agent-a").type(MessageType.STATUS)
            .content("bug report").actorType(ActorType.AGENT).topic("bugs").build());

        TopicService.MergeResult result = tools.mergeTopics(
            ch.name(), "bugs", "issues", "admin");

        assertThat(result.messagesUpdated()).isEqualTo(1);
        assertThat(result.sourceTopic()).isEqualTo("bugs");
        assertThat(result.targetTopic()).isEqualTo("issues");
    }
}
```

- [ ] **Step 6: Implement merge_topics MCP tool**

Add to `QhorusMcpTools.java` after the `renameTopic` method:

```java
@Tool(name = "merge_topics", description = "Merge a source topic into a target topic — moves all messages and deletes the source topic. Emits an audit EVENT.")
@Transactional
public TopicService.MergeResult mergeTopics(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "source_topic", description = "Topic to merge from (will be deleted)") String sourceTopic,
        @ToolArg(name = "target_topic", description = "Topic to merge into (will receive messages)") String targetTopic,
        @ToolArg(name = "caller_instance_id", description = "Instance ID of the caller", required = false) String callerInstanceId) {
    Channel ch = resolveChannel(channel);
    String actorId = callerInstanceId != null ? callerInstanceId : "anonymous";
    TopicService.MergeResult result = topicService.merge(ch.id(), sourceTopic, targetTopic, actorId);
    messageService.dispatch(MessageDispatch.builder()
            .channelId(ch.id())
            .sender("system:topic-service")
            .type(MessageType.EVENT)
            .telemetry("{\"action\":\"topics-merged\",\"source_topic\":\"" + result.sourceTopic()
                    + "\",\"target_topic\":\"" + result.targetTopic()
                    + "\",\"messages_updated\":" + result.messagesUpdated() + "}")
            .actorType(ActorType.SYSTEM)
            .build());
    return result;
}
```

- [ ] **Step 7: Add reactive counterpart**

Add to `ReactiveQhorusMcpTools.java`:

```java
@Tool(name = "merge_topics", description = "Merge a source topic into a target topic — moves all messages and deletes the source topic. Emits an audit EVENT.")
@Blocking
public TopicService.MergeResult mergeTopics(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "source_topic", description = "Topic to merge from (will be deleted)") String sourceTopic,
        @ToolArg(name = "target_topic", description = "Topic to merge into (will receive messages)") String targetTopic,
        @ToolArg(name = "caller_instance_id", description = "Instance ID of the caller", required = false) String callerInstanceId) {
    Channel ch = resolveChannel(channel);
    String actorId = callerInstanceId != null ? callerInstanceId : "anonymous";
    TopicService.MergeResult result = topicService.merge(ch.id(), sourceTopic, targetTopic, actorId);
    messageService.dispatch(MessageDispatch.builder()
            .channelId(ch.id())
            .sender("system:topic-service")
            .type(MessageType.EVENT)
            .telemetry("{\"action\":\"topics-merged\",\"source_topic\":\"" + result.sourceTopic()
                    + "\",\"target_topic\":\"" + result.targetTopic()
                    + "\",\"messages_updated\":" + result.messagesUpdated() + "}")
            .actorType(ActorType.SYSTEM)
            .build());
    return result;
}
```

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="TopicMergeTest,MergeTopicsToolTest,ToolOverloadDiscoverabilityTest" -pl runtime`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicService.java runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java runtime/src/test/java/io/casehub/qhorus/runtime/message/TopicMergeTest.java runtime/src/test/java/io/casehub/qhorus/runtime/mcp/MergeTopicsToolTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#336): mergeTopics service + MCP tool with audit EVENT"
```

---

### Task 3: moveTopic — store methods + service + MCP tool (#335)

Cross-channel topic move with commitment gate, tenancy validation, and semantic check. Requires new store methods across all implementations.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/MessageStore.java` — add `updateChannelId()`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CommitmentStore.java` — add `findByIds()`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveMessageStore.java` — add `updateChannelId()` reactive
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveCommitmentStore.java` — add `findByIds()` reactive
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryMessageStore.java` — implement `updateChannelId()`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCommitmentStore.java` — implement `findByIds()`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveMessageStore.java` — implement `updateChannelId()`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveCommitmentStore.java` — implement `findByIds()`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java` — implement `updateChannelId()`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCommitmentStore.java` — implement `findByIds()`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicService.java` — add `move()` + `MoveResult`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add `moveTopic` `@Tool`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — add reactive `moveTopic`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/TopicMoveTest.java` — service unit tests
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/MoveTopicToolTest.java` — MCP integration test
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/MessageStoreUpdateChannelIdTest.java`

**Interfaces:**
- Consumes: `TopicStore`, `MessageStore`, `CommitmentStore`, `ChannelService.findById()` (all existing)
- Produces: `MessageStore.updateChannelId(UUID, String, UUID) → int`; `CommitmentStore.findByIds(Collection<UUID>) → List<Commitment>`; `TopicService.move(UUID, String, UUID, String) → MoveResult`; `TopicService.MoveResult(String, UUID, UUID, int)`

- [ ] **Step 1: Add store interface methods**

Add to `MessageStore.java`:
```java
int updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId);
```

Add to `CommitmentStore.java`:
```java
List<Commitment> findByIds(Collection<UUID> ids);
```

Add to `ReactiveMessageStore.java`:
```java
Uni<Integer> updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId);
```

Add to `ReactiveCommitmentStore.java`:
```java
Uni<List<Commitment>> findByIds(Collection<UUID> ids);
```

- [ ] **Step 2: Implement InMemory stores**

Add to `InMemoryMessageStore.java`:
```java
@Override
public int updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId) {
    int count = 0;
    for (Map.Entry<Long, Message> entry : store.entrySet()) {
        Message m = entry.getValue();
        if (sourceChannelId.equals(m.channelId())
                && topic.equalsIgnoreCase(m.topic())) {
            store.put(entry.getKey(), m.toBuilder().channelId(targetChannelId).build());
            count++;
        }
    }
    return count;
}
```

Add to `InMemoryCommitmentStore.java`:
```java
@Override
public List<Commitment> findByIds(Collection<UUID> ids) {
    return ids.stream()
        .map(byId::get)
        .filter(java.util.Objects::nonNull)
        .toList();
}
```

Add to `InMemoryReactiveMessageStore.java`:
```java
@Override
public Uni<Integer> updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId) {
    return Uni.createFrom().item(delegate.updateChannelId(sourceChannelId, topic, targetChannelId));
}
```

Add to `InMemoryReactiveCommitmentStore.java`:
```java
@Override
public Uni<List<Commitment>> findByIds(Collection<UUID> ids) {
    return Uni.createFrom().item(delegate.findByIds(ids));
}
```

- [ ] **Step 3: Implement JPA stores**

Add to `JpaMessageStore.java`:
```java
@Override
public int updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId) {
    return em.createQuery("UPDATE MessageEntity m SET m.channelId = :target WHERE m.channelId = :source AND LOWER(m.topic) = LOWER(:topic)")
        .setParameter("target", targetChannelId)
        .setParameter("source", sourceChannelId)
        .setParameter("topic", topic)
        .executeUpdate();
}
```

Add to `JpaCommitmentStore.java`:
```java
@Override
public List<Commitment> findByIds(Collection<UUID> ids) {
    if (ids == null || ids.isEmpty()) return List.of();
    return CommitmentEntity.find("id IN ?1", List.copyOf(ids))
        .stream().map(e -> ((CommitmentEntity) e).toDomain()).toList();
}
```

- [ ] **Step 4: Write store contract test**

```java
package io.casehub.qhorus.persistence.memory.contract;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class MessageStoreUpdateChannelIdTest {

    private InMemoryMessageStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryMessageStore();
    }

    @Test
    void updatesMatchingTopicMessages() {
        UUID src = UUID.randomUUID(), tgt = UUID.randomUUID();
        store.put(msg(src, "bugs"));
        store.put(msg(src, "bugs"));
        store.put(msg(src, "other"));

        int updated = store.updateChannelId(src, "bugs", tgt);

        assertThat(updated).isEqualTo(2);
        var all = store.scan(io.casehub.qhorus.api.store.query.MessageQuery.builder().channelId(tgt).build());
        assertThat(all).hasSize(2);
    }

    @Test
    void noMatchReturnsZero() {
        assertThat(store.updateChannelId(UUID.randomUUID(), "nonexistent", UUID.randomUUID())).isZero();
    }

    private Message msg(UUID channelId, String topic) {
        return Message.builder().channelId(channelId).sender("a").messageType(MessageType.STATUS)
            .actorType(ActorType.AGENT).content("x").topic(topic).build();
    }
}
```

- [ ] **Step 5: Run store tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageStoreUpdateChannelIdTest -pl persistence-memory`
Expected: PASS

- [ ] **Step 6: Write TopicService.move() unit tests**

```java
package io.casehub.qhorus.runtime.message;

import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.qhorus.persistence.memory.InMemoryTopicStore;
import io.casehub.qhorus.persistence.memory.InMemoryCommitmentStore;
import io.casehub.qhorus.api.message.*;
import io.casehub.qhorus.api.store.CommitmentStore;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class TopicMoveTest {

    private TopicService topicService;
    private InMemoryTopicStore topicStore;
    private InMemoryMessageStore messageStore;
    private InMemoryCommitmentStore commitmentStore;

    private UUID srcChannel = UUID.randomUUID();
    private UUID tgtChannel = UUID.randomUUID();

    @BeforeEach
    void setUp() {
        topicStore = new InMemoryTopicStore();
        messageStore = new InMemoryMessageStore();
        commitmentStore = new InMemoryCommitmentStore();
        topicService = new TopicService();
        topicService.topicStore = topicStore;
        topicService.messageStore = messageStore;
        topicService.commitmentStore = commitmentStore;
    }

    private void addTopic(UUID ch, String name) {
        topicStore.put(new Topic(null, ch, name, false, null, null, Instant.now(), null));
    }

    private Message addMessage(UUID ch, String topic) {
        return messageStore.put(Message.builder()
            .channelId(ch).sender("agent").messageType(MessageType.STATUS)
            .actorType(ActorType.AGENT).content("msg").topic(topic).build());
    }

    private Message addMessageWithCommitment(UUID ch, String topic, UUID commitmentId) {
        return messageStore.put(Message.builder()
            .channelId(ch).sender("agent").messageType(MessageType.COMMAND)
            .actorType(ActorType.AGENT).content("cmd").topic(topic)
            .commitmentId(commitmentId).build());
    }

    @Test
    void moveUpdatesChannelIdAndCreatesTopicInTarget() {
        addTopic(srcChannel, "bugs");
        addMessage(srcChannel, "bugs");
        addMessage(srcChannel, "bugs");

        TopicService.MoveResult result = topicService.move(srcChannel, "bugs", tgtChannel, "admin");

        assertThat(result.messagesUpdated()).isEqualTo(2);
        assertThat(result.sourceChannelId()).isEqualTo(srcChannel);
        assertThat(result.targetChannelId()).isEqualTo(tgtChannel);
        assertThat(topicStore.find(srcChannel, "bugs")).isEmpty();
        assertThat(topicStore.find(tgtChannel, "bugs")).isPresent();
    }

    @Test
    void moveMergesIntoExistingTargetTopic() {
        addTopic(srcChannel, "bugs");
        addTopic(tgtChannel, "bugs");
        addMessage(srcChannel, "bugs");

        TopicService.MoveResult result = topicService.move(srcChannel, "bugs", tgtChannel, "admin");

        assertThat(result.messagesUpdated()).isEqualTo(1);
        assertThat(topicStore.find(srcChannel, "bugs")).isEmpty();
        assertThat(topicStore.find(tgtChannel, "bugs")).isPresent();
    }

    @Test
    void moveBlockedByOpenCommitment() {
        addTopic(srcChannel, "bugs");
        UUID cId = UUID.randomUUID();
        addMessageWithCommitment(srcChannel, "bugs", cId);
        commitmentStore.save(Commitment.builder()
            .correlationId("corr-1").channelId(srcChannel).requester("r")
            .state(CommitmentState.OPEN).createdAt(Instant.now()).build());
        // Need to set the id to match — InMemory auto-generates, so let's use the generated one
        // Actually, the commitment gate checks messages' commitmentId against commitmentStore.findByIds()
        // So we need the message's commitmentId to match a saved commitment that is OPEN
        var savedCommitment = commitmentStore.save(Commitment.builder()
            .id(cId).correlationId("corr-2").channelId(srcChannel).requester("r")
            .state(CommitmentState.OPEN).createdAt(Instant.now()).build());

        // Re-add the message with the saved commitment's id
        messageStore = new InMemoryMessageStore();
        topicService.messageStore = messageStore;
        addMessageWithCommitment(srcChannel, "bugs", savedCommitment.id());

        assertThatThrownBy(() -> topicService.move(srcChannel, "bugs", tgtChannel, "admin"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("commitment");
    }

    @Test
    void moveAllowedWithTerminalCommitments() {
        addTopic(srcChannel, "bugs");
        UUID cId = UUID.randomUUID();
        var saved = commitmentStore.save(Commitment.builder()
            .id(cId).correlationId("corr-3").channelId(srcChannel).requester("r")
            .state(CommitmentState.FULFILLED).createdAt(Instant.now()).build());
        messageStore.put(Message.builder()
            .channelId(srcChannel).sender("agent").messageType(MessageType.COMMAND)
            .actorType(ActorType.AGENT).content("cmd").topic("bugs")
            .commitmentId(saved.id()).build());

        TopicService.MoveResult result = topicService.move(srcChannel, "bugs", tgtChannel, "admin");
        assertThat(result.messagesUpdated()).isEqualTo(1);
    }

    @Test
    void moveGeneralRejected() {
        topicService.ensureExists(srcChannel, "general", null);

        assertThatThrownBy(() -> topicService.move(srcChannel, "general", tgtChannel, "admin"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("general");
    }

    @Test
    void moveSourceNotFoundRejected() {
        assertThatThrownBy(() -> topicService.move(srcChannel, "nonexistent", tgtChannel, "admin"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("not found");
    }
}
```

- [ ] **Step 7: Run unit test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TopicMoveTest -pl runtime`
Expected: compilation error — `move()`, `MoveResult` not defined; `commitmentStore` field missing on `TopicService`

- [ ] **Step 8: Implement TopicService.move()**

Add `commitmentStore` field to `TopicService.java`:
```java
@Inject
public CommitmentStore commitmentStore;
```

Add `move()` method and `MoveResult` record:
```java
public MoveResult move(UUID sourceChannelId, String topicName, UUID targetChannelId, String actorId) {
    String normalTopic = normalise(topicName);
    if (DEFAULT_TOPIC.equalsIgnoreCase(normalTopic)) {
        throw new IllegalArgumentException("Cannot move the default topic 'general'");
    }
    if (topicStore.find(sourceChannelId, normalTopic).isEmpty()) {
        throw new IllegalArgumentException("Topic '" + normalTopic + "' not found in source channel");
    }

    // Commitment gate: block if any message in this topic has an open commitment
    var messages = messageStore.scan(
        io.casehub.qhorus.api.store.query.MessageQuery.builder()
            .channelId(sourceChannelId).topic(normalTopic).build());
    var commitmentIds = messages.stream()
        .map(Message::commitmentId)
        .filter(java.util.Objects::nonNull)
        .distinct()
        .toList();
    if (!commitmentIds.isEmpty()) {
        var commitments = commitmentStore.findByIds(commitmentIds);
        var openCommitments = commitments.stream()
            .filter(c -> c.state().isActive())
            .toList();
        if (!openCommitments.isEmpty()) {
            String blocking = openCommitments.stream()
                .map(Commitment::correlationId)
                .collect(java.util.stream.Collectors.joining(", "));
            throw new IllegalStateException(
                "Cannot move topic — open commitments exist: " + blocking);
        }
    }

    int moved = messageStore.updateChannelId(sourceChannelId, normalTopic, targetChannelId);

    // Move or merge topic record
    if (topicStore.find(targetChannelId, normalTopic).isEmpty()) {
        topicStore.put(new Topic(null, targetChannelId, normalTopic, false, null, null, java.time.Instant.now(), null));
    }
    topicStore.delete(sourceChannelId, normalTopic);

    return new MoveResult(normalTopic, sourceChannelId, targetChannelId, moved);
}

public record MoveResult(String topicName, UUID sourceChannelId, UUID targetChannelId, int messagesUpdated) {}
```

- [ ] **Step 9: Run unit tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TopicMoveTest -pl runtime`
Expected: all PASS

- [ ] **Step 10: Implement move_topic MCP tool**

Add to `QhorusMcpTools.java`:

```java
@Tool(name = "move_topic", description = "Move all messages in a topic from one channel to another. Blocks if open commitments exist. Emits audit EVENTs in both channels.")
@Transactional
public TopicService.MoveResult moveTopic(
        @ToolArg(name = "source_channel", description = "Source channel name or UUID") String sourceChannel,
        @ToolArg(name = "topic_name", description = "Topic to move") String topicName,
        @ToolArg(name = "target_channel", description = "Target channel name or UUID") String targetChannel,
        @ToolArg(name = "caller_instance_id", description = "Instance ID of the caller", required = false) String callerInstanceId) {
    Channel src = resolveChannel(sourceChannel);
    Channel tgt = resolveChannel(targetChannel);
    if (src.id().equals(tgt.id())) {
        throw new IllegalArgumentException("Source and target channels must be different");
    }
    if (!java.util.Objects.equals(src.tenancyId(), tgt.tenancyId())) {
        throw new IllegalArgumentException("Source and target channels must share the same tenancy");
    }
    io.casehub.qhorus.runtime.channel.ChannelSemantic semantic = tgt.semantic();
    if (semantic != io.casehub.qhorus.runtime.channel.ChannelSemantic.APPEND
            && semantic != io.casehub.qhorus.runtime.channel.ChannelSemantic.COLLECT) {
        throw new IllegalArgumentException("Target channel semantic must be APPEND or COLLECT, not " + semantic);
    }
    String actorId = callerInstanceId != null ? callerInstanceId : "anonymous";
    TopicService.MoveResult result = topicService.move(src.id(), topicName, tgt.id(), actorId);

    messageService.dispatch(MessageDispatch.builder()
            .channelId(src.id()).sender("system:topic-service").type(MessageType.EVENT)
            .telemetry("{\"action\":\"topic-moved-out\",\"topic\":\"" + result.topicName()
                    + "\",\"target_channel\":\"" + tgt.name()
                    + "\",\"messages_moved\":" + result.messagesUpdated() + "}")
            .actorType(ActorType.SYSTEM).build());
    messageService.dispatch(MessageDispatch.builder()
            .channelId(tgt.id()).sender("system:topic-service").type(MessageType.EVENT)
            .telemetry("{\"action\":\"topic-moved-in\",\"topic\":\"" + result.topicName()
                    + "\",\"source_channel\":\"" + src.name()
                    + "\",\"messages_moved\":" + result.messagesUpdated() + "}")
            .actorType(ActorType.SYSTEM).build());
    return result;
}
```

- [ ] **Step 11: Add reactive counterpart**

Add to `ReactiveQhorusMcpTools.java` — same logic with `@Blocking`:

```java
@Tool(name = "move_topic", description = "Move all messages in a topic from one channel to another. Blocks if open commitments exist. Emits audit EVENTs in both channels.")
@Blocking
public TopicService.MoveResult moveTopic(
        @ToolArg(name = "source_channel", description = "Source channel name or UUID") String sourceChannel,
        @ToolArg(name = "topic_name", description = "Topic to move") String topicName,
        @ToolArg(name = "target_channel", description = "Target channel name or UUID") String targetChannel,
        @ToolArg(name = "caller_instance_id", description = "Instance ID of the caller", required = false) String callerInstanceId) {
    Channel src = resolveChannel(sourceChannel);
    Channel tgt = resolveChannel(targetChannel);
    if (src.id().equals(tgt.id())) {
        throw new IllegalArgumentException("Source and target channels must be different");
    }
    if (!java.util.Objects.equals(src.tenancyId(), tgt.tenancyId())) {
        throw new IllegalArgumentException("Source and target channels must share the same tenancy");
    }
    io.casehub.qhorus.runtime.channel.ChannelSemantic semantic = tgt.semantic();
    if (semantic != io.casehub.qhorus.runtime.channel.ChannelSemantic.APPEND
            && semantic != io.casehub.qhorus.runtime.channel.ChannelSemantic.COLLECT) {
        throw new IllegalArgumentException("Target channel semantic must be APPEND or COLLECT, not " + semantic);
    }
    String actorId = callerInstanceId != null ? callerInstanceId : "anonymous";
    TopicService.MoveResult result = topicService.move(src.id(), topicName, tgt.id(), actorId);

    messageService.dispatch(MessageDispatch.builder()
            .channelId(src.id()).sender("system:topic-service").type(MessageType.EVENT)
            .telemetry("{\"action\":\"topic-moved-out\",\"topic\":\"" + result.topicName()
                    + "\",\"target_channel\":\"" + tgt.name()
                    + "\",\"messages_moved\":" + result.messagesUpdated() + "}")
            .actorType(ActorType.SYSTEM).build());
    messageService.dispatch(MessageDispatch.builder()
            .channelId(tgt.id()).sender("system:topic-service").type(MessageType.EVENT)
            .telemetry("{\"action\":\"topic-moved-in\",\"topic\":\"" + result.topicName()
                    + "\",\"source_channel\":\"" + src.name()
                    + "\",\"messages_moved\":" + result.messagesUpdated() + "}")
            .actorType(ActorType.SYSTEM).build());
    return result;
}
```

- [ ] **Step 12: Run all tests including ToolOverloadDiscoverabilityTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="TopicMoveTest,MergeTopicsToolTest,ToolOverloadDiscoverabilityTest" -pl runtime`
Expected: all PASS

- [ ] **Step 13: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#335): moveTopic with commitment gate, tenancy + semantic validation

Adds MessageStore.updateChannelId() and CommitmentStore.findByIds() across
all store implementations (blocking, reactive, InMemory, JPA).
TopicService.move() blocks on open commitments, creates/merges topic in
target channel. MCP tool validates same-tenancy and APPEND/COLLECT semantic."
```

---

### Task 4: ArtefactType.DEBATE (#340)

Single enum addition.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/ArtefactType.java` — add DEBATE
- Test: `api/src/test/java/io/casehub/qhorus/api/message/ArtefactTypeDebateTest.java`

**Interfaces:**
- Produces: `ArtefactType.DEBATE` enum constant

- [ ] **Step 1: Write test**

```java
package io.casehub.qhorus.api.message;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ArtefactTypeDebateTest {

    @Test
    void debateEnumExists() {
        ArtefactType debate = ArtefactType.valueOf("DEBATE");
        assertThat(debate).isNotNull();
        assertThat(debate.name()).isEqualTo("DEBATE");
    }

    @Test
    void allExpectedValuesPresent() {
        assertThat(ArtefactType.values()).extracting(ArtefactType::name)
            .contains("DOCUMENT", "CODE", "CASE", "WORK_ITEM", "CHANNEL", "MESSAGE", "EXTERNAL", "DEBATE");
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ArtefactTypeDebateTest -pl api`
Expected: FAIL — `DEBATE` not found

- [ ] **Step 3: Add DEBATE to ArtefactType enum**

Add `DEBATE` after `EXTERNAL` in `ArtefactType.java`:
```java
public enum ArtefactType {
    DOCUMENT, CODE, CASE, WORK_ITEM, CHANNEL, MESSAGE, EXTERNAL, DEBATE
}
```

- [ ] **Step 4: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ArtefactTypeDebateTest -pl api`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/main/java/io/casehub/qhorus/api/message/ArtefactType.java api/src/test/java/io/casehub/qhorus/api/message/ArtefactTypeDebateTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#340): add ArtefactType.DEBATE for chat-demo alignment"
```

---

### Task 5: Presence — API types, config, service, MCP tools (#333)

Ephemeral presence via Caffeine cache with lazy timeout degradation. No database, no migration, no scheduler.

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/PresenceStatus.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/Presence.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/config/PresenceConfig.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/PresenceService.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ClockProducer.java` — CDI producer for `java.time.Clock`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add presence `@Tool` methods
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — add reactive presence tools
- Modify: `runtime/pom.xml` — add Caffeine dependency
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/PresenceServiceTest.java` — unit tests with controllable clock
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/PresenceToolTest.java` — MCP integration test

**Interfaces:**
- Consumes: `ChannelMembershipService.listMembers(UUID)` (existing)
- Produces: `PresenceStatus` enum; `Presence` record; `PresenceService.heartbeat()`, `getPresence()`, `getChannelPresence()`, `setOffline()`; MCP tools `set_presence`, `get_presence`, `get_channel_presence`

- [ ] **Step 1: Add Caffeine dependency to runtime/pom.xml**

Add to `<dependencies>` section:
```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

Caffeine is a Quarkus managed dependency — no version needed.

- [ ] **Step 2: Create API types**

Create `api/src/main/java/io/casehub/qhorus/api/channel/PresenceStatus.java`:
```java
package io.casehub.qhorus.api.channel;

public enum PresenceStatus {
    ONLINE,
    AVAILABLE,
    BUSY,
    AWAY,
    OFFLINE;

    public boolean isReportable() {
        return this == ONLINE || this == AVAILABLE || this == BUSY;
    }
}
```

Create `api/src/main/java/io/casehub/qhorus/api/channel/Presence.java`:
```java
package io.casehub.qhorus.api.channel;

import java.time.Instant;

public record Presence(
    String memberId,
    PresenceStatus status,
    PresenceStatus reportedStatus,
    Instant lastSeenAt,
    String statusMessage
) {}
```

- [ ] **Step 3: Run API module build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 4: Create PresenceConfig**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/config/PresenceConfig.java`:
```java
package io.casehub.qhorus.runtime.config;

import java.time.Duration;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.qhorus.presence")
public interface PresenceConfig {
    @WithDefault("PT2M")
    Duration awayTimeout();

    @WithDefault("PT10M")
    Duration offlineTimeout();

    @WithDefault("PT30S")
    Duration heartbeatInterval();
}
```

- [ ] **Step 5: Create ClockProducer**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ClockProducer.java`:
```java
package io.casehub.qhorus.runtime.channel;

import java.time.Clock;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class ClockProducer {

    @Produces
    @ApplicationScoped
    public Clock clock() {
        return Clock.systemUTC();
    }
}
```

- [ ] **Step 6: Write PresenceService unit tests**

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.Presence;
import io.casehub.qhorus.api.channel.PresenceStatus;
import io.casehub.qhorus.runtime.config.PresenceConfig;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class PresenceServiceTest {

    private PresenceService service;
    private Instant now = Instant.parse("2026-07-12T12:00:00Z");

    @BeforeEach
    void setUp() {
        var config = new PresenceConfig() {
            public Duration awayTimeout() { return Duration.ofMinutes(2); }
            public Duration offlineTimeout() { return Duration.ofMinutes(10); }
            public Duration heartbeatInterval() { return Duration.ofSeconds(30); }
        };
        service = new PresenceService(config, Clock.fixed(now, ZoneOffset.UTC));
    }

    private void advanceTime(Duration d) {
        now = now.plus(d);
        var config = new PresenceConfig() {
            public Duration awayTimeout() { return Duration.ofMinutes(2); }
            public Duration offlineTimeout() { return Duration.ofMinutes(10); }
            public Duration heartbeatInterval() { return Duration.ofSeconds(30); }
        };
        service = new PresenceService(service, config, Clock.fixed(now, ZoneOffset.UTC));
    }

    @Test
    void heartbeatSetsPresence() {
        service.heartbeat("agent-1", PresenceStatus.ONLINE, null);
        Presence p = service.getPresence("agent-1");
        assertThat(p.status()).isEqualTo(PresenceStatus.ONLINE);
        assertThat(p.reportedStatus()).isEqualTo(PresenceStatus.ONLINE);
        assertThat(p.lastSeenAt()).isEqualTo(now);
    }

    @Test
    void heartbeatWithStatusMessage() {
        service.heartbeat("agent-1", PresenceStatus.BUSY, "Processing case-456");
        Presence p = service.getPresence("agent-1");
        assertThat(p.status()).isEqualTo(PresenceStatus.BUSY);
        assertThat(p.statusMessage()).isEqualTo("Processing case-456");
    }

    @Test
    void unknownMemberReturnsOffline() {
        Presence p = service.getPresence("unknown");
        assertThat(p.status()).isEqualTo(PresenceStatus.OFFLINE);
        assertThat(p.reportedStatus()).isEqualTo(PresenceStatus.OFFLINE);
        assertThat(p.lastSeenAt()).isNull();
    }

    @Test
    void awayAfterTimeout() {
        service.heartbeat("agent-1", PresenceStatus.ONLINE, null);
        advanceTime(Duration.ofMinutes(3));
        Presence p = service.getPresence("agent-1");
        assertThat(p.status()).isEqualTo(PresenceStatus.AWAY);
        assertThat(p.reportedStatus()).isEqualTo(PresenceStatus.ONLINE);
    }

    @Test
    void setOfflineImmediately() {
        service.heartbeat("agent-1", PresenceStatus.ONLINE, null);
        service.setOffline("agent-1");
        Presence p = service.getPresence("agent-1");
        assertThat(p.status()).isEqualTo(PresenceStatus.OFFLINE);
    }

    @Test
    void heartbeatWithAwayRejected() {
        assertThatThrownBy(() -> service.heartbeat("agent-1", PresenceStatus.AWAY, null))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("reportable");
    }

    @Test
    void heartbeatWithOfflineRejected() {
        assertThatThrownBy(() -> service.heartbeat("agent-1", PresenceStatus.OFFLINE, null))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("reportable");
    }

    @Test
    void configInvariantViolation() {
        var badConfig = new PresenceConfig() {
            public Duration awayTimeout() { return Duration.ofMinutes(15); }
            public Duration offlineTimeout() { return Duration.ofMinutes(10); }
            public Duration heartbeatInterval() { return Duration.ofSeconds(30); }
        };
        assertThatThrownBy(() -> new PresenceService(badConfig, Clock.systemUTC()))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("awayTimeout");
    }

    @Test
    void heartbeatResetsAwayBackToReported() {
        service.heartbeat("agent-1", PresenceStatus.AVAILABLE, null);
        advanceTime(Duration.ofMinutes(3));
        assertThat(service.getPresence("agent-1").status()).isEqualTo(PresenceStatus.AWAY);

        // Re-heartbeat resets
        service.heartbeat("agent-1", PresenceStatus.AVAILABLE, null);
        Presence p = service.getPresence("agent-1");
        assertThat(p.status()).isEqualTo(PresenceStatus.AVAILABLE);
    }
}
```

- [ ] **Step 7: Run to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PresenceServiceTest -pl runtime`
Expected: compilation error — `PresenceService` does not exist

- [ ] **Step 8: Implement PresenceService**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/channel/PresenceService.java`:

```java
package io.casehub.qhorus.runtime.channel;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.channel.Presence;
import io.casehub.qhorus.api.channel.PresenceStatus;
import io.casehub.qhorus.runtime.config.PresenceConfig;

@ApplicationScoped
public class PresenceService {

    private final Cache<String, PresenceEntry> cache;
    private final PresenceConfig config;
    private final Clock clock;

    record PresenceEntry(PresenceStatus reportedStatus, Instant lastSeenAt, String statusMessage) {}

    @Inject
    public PresenceService(PresenceConfig config, Clock clock) {
        if (config.awayTimeout().compareTo(config.offlineTimeout()) >= 0) {
            throw new IllegalStateException(
                "awayTimeout (" + config.awayTimeout() + ") must be less than offlineTimeout (" + config.offlineTimeout() + ")");
        }
        this.config = config;
        this.clock = clock;
        this.cache = Caffeine.newBuilder()
            .expireAfterWrite(config.offlineTimeout())
            .build();
    }

    // Package-private constructor for tests that advance time — reuses existing cache
    PresenceService(PresenceService previous, PresenceConfig config, Clock clock) {
        this.config = config;
        this.clock = clock;
        this.cache = previous.cache;
    }

    public void heartbeat(String memberId, PresenceStatus status, String statusMessage) {
        if (!status.isReportable()) {
            throw new IllegalArgumentException(
                "Only reportable statuses (ONLINE, AVAILABLE, BUSY) are accepted; got " + status);
        }
        cache.put(memberId, new PresenceEntry(status, clock.instant(), statusMessage));
    }

    public Presence getPresence(String memberId) {
        PresenceEntry entry = cache.getIfPresent(memberId);
        if (entry == null) {
            return new Presence(memberId, PresenceStatus.OFFLINE, PresenceStatus.OFFLINE, null, null);
        }
        PresenceStatus effective = computeEffectiveStatus(entry);
        return new Presence(memberId, effective, entry.reportedStatus(), entry.lastSeenAt(), entry.statusMessage());
    }

    public List<Presence> getChannelPresence(UUID channelId) {
        return membershipService.listMembers(channelId).stream()
            .map(m -> getPresence(m.memberId()))
            .toList();
    }

    public void setOffline(String memberId) {
        cache.invalidate(memberId);
    }

    private PresenceStatus computeEffectiveStatus(PresenceEntry entry) {
        Duration elapsed = Duration.between(entry.lastSeenAt(), clock.instant());
        if (elapsed.compareTo(config.awayTimeout()) >= 0) {
            return PresenceStatus.AWAY;
        }
        return entry.reportedStatus();
    }

    // Injected separately — not constructor-injected to allow CDI-free unit tests
    @Inject
    public ChannelMembershipService membershipService;
}
```

- [ ] **Step 9: Run unit tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PresenceServiceTest -pl runtime`
Expected: all PASS (unit tests don't use `getChannelPresence` — no CDI needed)

- [ ] **Step 10: Implement MCP tools**

Add to `QhorusMcpTools.java`:

```java
@Tool(name = "set_presence", description = "Report presence status (heartbeat). Accepted statuses: ONLINE, AVAILABLE, BUSY. AWAY and OFFLINE are computed from heartbeat absence.")
public Presence setPresence(
        @ToolArg(name = "status", description = "Presence status: ONLINE, AVAILABLE, or BUSY") String status,
        @ToolArg(name = "status_message", description = "Optional status message", required = false) String statusMessage,
        @ToolArg(name = "member_id", description = "Member ID. Defaults to caller identity.", required = false) String memberId) {
    String member = memberId != null ? memberId : currentPrincipal.actorId();
    PresenceStatus ps = PresenceStatus.valueOf(status.toUpperCase());
    presenceService.heartbeat(member, ps, statusMessage);
    return presenceService.getPresence(member);
}

@Tool(name = "get_presence", description = "Get presence status for a member")
public Presence getPresenceTool(
        @ToolArg(name = "member_id", description = "Member ID to query") String memberId) {
    return presenceService.getPresence(memberId);
}

@Tool(name = "get_channel_presence", description = "Get presence status for all members of a channel")
public java.util.List<Presence> getChannelPresence(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel) {
    Channel ch = resolveChannel(channel);
    return presenceService.getChannelPresence(ch.id());
}
```

Add `presenceService` injection to `QhorusMcpTools`:
```java
@Inject
PresenceService presenceService;
```

Add the same tools and injection to `ReactiveQhorusMcpTools.java` with `@Blocking`.

- [ ] **Step 11: Write MCP integration test**

```java
package io.casehub.qhorus.runtime.mcp;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.channel.Presence;
import io.casehub.qhorus.api.channel.PresenceStatus;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;
import io.casehub.qhorus.runtime.channel.ChannelMembershipService;
import io.casehub.qhorus.api.channel.MemberRole;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class PresenceToolTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelService channelService;
    @Inject ChannelMembershipService membershipService;

    @Test
    void setAndGetPresence() {
        Presence p = tools.setPresence("ONLINE", "Hello", "presence-test-1");
        assertThat(p.status()).isEqualTo(PresenceStatus.ONLINE);
        assertThat(p.statusMessage()).isEqualTo("Hello");

        Presence fetched = tools.getPresenceTool("presence-test-1");
        assertThat(fetched.status()).isEqualTo(PresenceStatus.ONLINE);
    }

    @Test
    void getChannelPresenceReturnsMemberStatuses() {
        var ch = channelService.create(ChannelCreateRequest.builder("presence-ch-test").build());
        membershipService.join(ch.id(), "member-a", MemberRole.PARTICIPANT, null);
        membershipService.join(ch.id(), "member-b", MemberRole.PARTICIPANT, null);

        tools.setPresence("AVAILABLE", null, "member-a");

        java.util.List<Presence> list = tools.getChannelPresence(ch.name());
        assertThat(list).hasSize(2);
        assertThat(list).extracting(Presence::memberId).containsExactlyInAnyOrder("member-a", "member-b");
        assertThat(list).filteredOn(p -> p.memberId().equals("member-a"))
            .extracting(Presence::status).containsExactly(PresenceStatus.AVAILABLE);
        assertThat(list).filteredOn(p -> p.memberId().equals("member-b"))
            .extracting(Presence::status).containsExactly(PresenceStatus.OFFLINE);
    }

    @Test
    void awayAndOfflineRejected() {
        assertThatThrownBy(() -> tools.setPresence("AWAY", null, "x"))
            .isInstanceOf(io.quarkiverse.mcp.server.ToolCallException.class);
        assertThatThrownBy(() -> tools.setPresence("OFFLINE", null, "x"))
            .isInstanceOf(io.quarkiverse.mcp.server.ToolCallException.class);
    }
}
```

- [ ] **Step 12: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="PresenceServiceTest,PresenceToolTest,ToolOverloadDiscoverabilityTest" -pl runtime`
Expected: all PASS

- [ ] **Step 13: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#333): Presence with Caffeine cache, heartbeat degradation, MCP tools

Adds PresenceStatus enum, Presence record in api/channel/.
PresenceService with Caffeine cache — lazy degradation on read
(ONLINE→AWAY after awayTimeout, cache eviction→OFFLINE after offlineTimeout).
Config invariant: awayTimeout < offlineTimeout validated at startup.
MCP tools: set_presence, get_presence, get_channel_presence.
Clock injection for deterministic time tests."
```

---

### Task 6: Update CLAUDE.md + close issues

Update CLAUDE.md with new conventions and close GitHub issues.

- [ ] **Step 1: Update CLAUDE.md**

Add to the testing conventions section:
- `PresenceService` uses `java.time.Clock` injection for deterministic time tests. Create service with `Clock.fixed()` and reconstruct with package-private constructor to advance time.
- `TopicService.move()` requires `CommitmentStore` injection. CDI-free unit tests must set `topicService.commitmentStore`.
- `MessageStore.updateChannelId(sourceChannelId, topic, targetChannelId)` — updates `message.channelId` for all messages matching source channel + topic.
- `CommitmentStore.findByIds(Collection<UUID>)` — batch lookup for commitment gate checks.

Add `DEBATE` to the ArtefactType enum listing.

Update the Flyway next V comment if needed.

- [ ] **Step 2: Close issues**

```bash
gh issue close 339 --repo casehubio/qhorus --comment "Implemented get_reactions_batch MCP tool with 200-entry limit."
gh issue close 336 --repo casehubio/qhorus --comment "Implemented mergeTopics service + MCP tool with audit EVENT."
gh issue close 335 --repo casehubio/qhorus --comment "Implemented moveTopic with commitment gate, tenancy validation, semantic check."
gh issue close 340 --repo casehubio/qhorus --comment "Added ArtefactType.DEBATE. Gap 1 (displayName) closed as design-intentional."
gh issue close 333 --repo casehubio/qhorus --comment "Implemented Presence with Caffeine cache, heartbeat degradation, MCP tools."
```

- [ ] **Step 3: Commit CLAUDE.md**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs: update CLAUDE.md with presence, moveTopic, mergeTopics conventions

Refs #339, #336, #335, #340, #333"
```

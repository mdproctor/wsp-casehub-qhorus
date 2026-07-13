# Topic-Aware Projections Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #338 — feat: topic-aware projections
**Issue group:** #343, #338

**Goal:** Expose topic-scoped projection folding via the `project_channel` MCP tool and add topic visibility to `MessageView` for internal grouping.

**Architecture:** Two orthogonal layers. Layer 1 adds a `topic` parameter to `project_channel` that filters the fold via `MessageQuery.topic()` — requires fixing `MessageQueryJpql` which currently silently ignores the topic field. Layer 2 adds `String topic` to `MessageView` so projections can see topic membership in their `apply()`.

**Tech Stack:** Java 21, Quarkus 3.32.2, H2 (tests), JPQL

## Global Constraints

- Pre-release — breaking changes to `MessageView` constructor are acceptable
- `MessageQuery.topic()` already exists — do not modify `MessageQuery`
- `ProjectionService` overloads are unchanged — topic filtering is injected via `MessageQuery` scope
- Topic matching is case-insensitive: `LOWER(topic) = LOWER(?n)` in JPQL, `equalsIgnoreCase` in `MessageQuery.matches()`
- Blank/empty topic strings normalized to null at the MCP tool boundary (consistent with `splitCsv()` pattern)
- Use `ide_edit_member` / `ide_insert_member` for all source file edits

---

### Task 1: Fix MessageQueryJpql topic predicate

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageQueryJpql.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/store/jpa/MessageQueryJpqlTopicTest.java`

**Interfaces:**
- Consumes: `MessageQuery.topic()` (existing)
- Produces: `MessageQueryJpql.from(MessageQuery)` and `MessageQueryJpql.from(MessageQuery, String)` now emit `LOWER(topic) = LOWER(?n)` when topic is set

- [ ] **Step 1: Write failing test for tenant-unaware topic predicate**

```java
package io.casehub.qhorus.store.jpa;

import io.casehub.qhorus.api.store.query.MessageQuery;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MessageQueryJpqlTopicTest {

    @Test
    void from_withTopic_addsCaseInsensitivePredicate() {
        MessageQuery q = MessageQuery.builder().topic("design").build();
        MessageQueryJpql jpql = MessageQueryJpql.from(q);

        assertThat(jpql.where()).contains("LOWER(topic) = LOWER(");
        assertThat(jpql.params()).contains("design");
    }

    @Test
    void from_withoutTopic_noTopicPredicate() {
        MessageQuery q = MessageQuery.builder().build();
        MessageQueryJpql jpql = MessageQueryJpql.from(q);

        assertThat(jpql.where()).doesNotContain("topic");
    }

    @Test
    void fromTenanted_withTopic_addsCaseInsensitivePredicate() {
        MessageQuery q = MessageQuery.builder().topic("Review").build();
        MessageQueryJpql jpql = MessageQueryJpql.from(q, "tenant-1");

        assertThat(jpql.where()).contains("LOWER(topic) = LOWER(");
        assertThat(jpql.params()).contains("Review");
    }

    @Test
    void fromTenanted_withoutTopic_noTopicPredicate() {
        MessageQuery q = MessageQuery.builder().build();
        MessageQueryJpql jpql = MessageQueryJpql.from(q, "tenant-1");

        assertThat(jpql.where()).doesNotContain("topic");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="MessageQueryJpqlTopicTest" -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: `from_withTopic_addsCaseInsensitivePredicate` and `fromTenanted_withTopic_addsCaseInsensitivePredicate` FAIL

- [ ] **Step 3: Add topic predicate to both `from()` variants**

In `MessageQueryJpql.from(MessageQuery)`, after the `contentPattern` block (before the return), add:

```java
if (q.topic() != null) {
    where.append(" AND LOWER(topic) = LOWER(?").append(idx++).append(")");
    params.add(q.topic());
}
```

In `MessageQueryJpql.from(MessageQuery, String)`, add the identical block after its `contentPattern` block.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="MessageQueryJpqlTopicTest" -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: All 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageQueryJpql.java \
       runtime/src/test/java/io/casehub/qhorus/store/jpa/MessageQueryJpqlTopicTest.java
git commit -m "fix(#338): MessageQueryJpql emits LOWER(topic) predicate — JPA topic filtering was silently broken"
```

---

### Task 2: Add topic field to MessageView

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageView.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java` (method `toMessageView`)
- Modify: `runtime/src/test/java/io/casehub/qhorus/projection/RenderableProjectionTest.java` (method `msg`)
- Modify: `runtime/src/test/java/io/casehub/qhorus/projection/ProjectionFoldLogicTest.java` (method `view`)

**Interfaces:**
- Consumes: `Message.topic()` (existing)
- Produces: `MessageView.topic()` — nullable String, positioned after `target`

- [ ] **Step 1: Write failing test that constructs MessageView with topic**

Add to `ProjectionFoldLogicTest`:

```java
@Test
void messageView_carriesTopicField() {
    MessageView v = new MessageView(
            1L, UUID.randomUUID(), "agent-a", MessageType.STATUS,
            "content", null, null, null, "design",
            null, ActorType.AGENT, Instant.now(), null, 0);
    assertThat(v.topic()).isEqualTo("design");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ProjectionFoldLogicTest#messageView_carriesTopicField" -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — constructor argument count mismatch (14 args, record has 13 components)

- [ ] **Step 3: Add `topic` field to `MessageView` record**

Use `ide_edit_member` to replace the `MessageView` record declaration. New field `String topic` positioned after `target`:

```java
public record MessageView(
        Long id,
        UUID channelId,
        String sender,
        MessageType type,
        String content,
        String correlationId,
        Long inReplyTo,
        String target,
        String topic,
        java.util.List<ArtefactRef> artefactRefs,
        ActorType actorType,
        Instant createdAt,
        Instant deadline,
        int replyCount) {}
```

- [ ] **Step 4: Update `QhorusEntityMapper.toMessageView()` to map topic**

Use `ide_replace_member` on `toMessageView` in `QhorusEntityMapper`:

```java
return new MessageView(
        msg.id(),
        msg.channelId(),
        msg.sender(),
        msg.messageType(),
        msg.content(),
        msg.correlationId(),
        msg.inReplyTo(),
        msg.target(),
        msg.topic(),
        msg.artefactRefs(),
        msg.actorType(),
        msg.createdAt(),
        msg.deadline(),
        msg.replyCount());
```

- [ ] **Step 5: Fix `RenderableProjectionTest.msg()` helper**

Use `ide_replace_member` on `msg` in `RenderableProjectionTest`:

```java
private static MessageView msg(MessageType type, String sender, String content) {
    return new MessageView(1L, UUID.randomUUID(), sender, type, content,
            null, null, null, null, null, ActorType.AGENT, Instant.now(), null, 0);
}
```

- [ ] **Step 6: Fix `ProjectionFoldLogicTest.view()` helper**

Use `ide_replace_member` on `view` in `ProjectionFoldLogicTest`:

```java
private static MessageView view(final MessageType type) {
    return new MessageView(
            1L, UUID.randomUUID(), "agent-a", type,
            "content", null, null, null, null,
            null, ActorType.AGENT, Instant.now(), null, 0);
}
```

- [ ] **Step 7: Find and fix any other MessageView constructor call sites**

Use `ide_find_references` on `MessageView` constructor to find all call sites. Fix any remaining ones that break (adding `null` for topic after `target`).

- [ ] **Step 8: Run all affected tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ProjectionFoldLogicTest,RenderableProjectionTest" -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: All tests PASS including `messageView_carriesTopicField`

- [ ] **Step 9: Run full build to catch any other breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS — all modules compile and tests pass

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#338): add topic field to MessageView, map in QhorusEntityMapper"
```

---

### Task 3: Add topic parameter to project_channel MCP tool

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` (method `projectAndRender`)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` (method `projectChannel`)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` (method `projectChannel`)
- Create: `runtime/src/test/java/io/casehub/qhorus/mcp/ProjectChannelTopicTest.java`

**Interfaces:**
- Consumes: `ProjectionService.project(UUID, MessageQuery, ChannelProjection)` (existing scoped overload), `MessageQuery.builder().topic(String)` (existing)
- Produces: `QhorusMcpTools.projectChannel(channel, projectionName, maxMessages, topic)` — new `topic` parameter

- [ ] **Step 1: Write failing integration tests**

```java
package io.casehub.qhorus.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.MessageView;
import io.casehub.qhorus.api.spi.ProjectionResult;
import io.casehub.qhorus.api.spi.RenderableProjection;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class ProjectChannelTopicTest {

    @Inject QhorusMcpTools tools;

    @ApplicationScoped
    static class TopicCounterProjection implements RenderableProjection<Integer> {
        @Override public String projectionName() { return "topic-counter"; }
        @Override public Integer identity() { return 0; }
        @Override public Integer apply(Integer state, MessageView msg) { return state + 1; }
        @Override public String render(ProjectionResult<Integer> result) {
            return result.isEmpty() ? "empty" : "count=" + result.state();
        }
    }

    @Test
    @TestTransaction
    void topicFilter_foldsOnlyMatchingTopic() {
        String ch = "proj-topic-" + System.nanoTime();
        tools.createChannel(ch, "test", null, null, null, null, null, null, null, null, null, null, null, null, null);
        tools.sendMessage(ch, "alice", "status", "a", null, null, null, null, null, null, null, "design");
        tools.sendMessage(ch, "bob", "status", "b", null, null, null, null, null, null, null, "testing");
        tools.sendMessage(ch, "carol", "status", "c", null, null, null, null, null, null, null, "design");

        String result = tools.projectChannel(ch, "topic-counter", null, "design");

        assertThat(result).isEqualTo("count=2");
    }

    @Test
    @TestTransaction
    void nullTopic_foldsAllMessages() {
        String ch = "proj-notopic-" + System.nanoTime();
        tools.createChannel(ch, "test", null, null, null, null, null, null, null, null, null, null, null, null, null);
        tools.sendMessage(ch, "alice", "status", "a", null, null, null, null, null, null, null, "design");
        tools.sendMessage(ch, "bob", "status", "b", null, null, null, null, null, null, null, "testing");

        String result = tools.projectChannel(ch, "topic-counter", null, null);

        assertThat(result).isEqualTo("count=2");
    }

    @Test
    @TestTransaction
    void blankTopic_normalizedToNull_foldsAll() {
        String ch = "proj-blank-" + System.nanoTime();
        tools.createChannel(ch, "test", null, null, null, null, null, null, null, null, null, null, null, null, null);
        tools.sendMessage(ch, "alice", "status", "a", null, null, null, null, null, null, null, "design");
        tools.sendMessage(ch, "bob", "status", "b", null, null, null, null, null, null, null, "testing");

        String result = tools.projectChannel(ch, "topic-counter", null, "  ");

        assertThat(result).isEqualTo("count=2");
    }

    @Test
    @TestTransaction
    void topicFilter_caseInsensitive() {
        String ch = "proj-case-" + System.nanoTime();
        tools.createChannel(ch, "test", null, null, null, null, null, null, null, null, null, null, null, null, null);
        tools.sendMessage(ch, "alice", "status", "a", null, null, null, null, null, null, null, "Design");
        tools.sendMessage(ch, "bob", "status", "b", null, null, null, null, null, null, null, "testing");

        String result = tools.projectChannel(ch, "topic-counter", null, "design");

        assertThat(result).isEqualTo("count=1");
    }

    @Test
    @TestTransaction
    void topicFilter_noMatchingMessages_returnsEmpty() {
        String ch = "proj-nomatch-" + System.nanoTime();
        tools.createChannel(ch, "test", null, null, null, null, null, null, null, null, null, null, null, null, null);
        tools.sendMessage(ch, "alice", "status", "a", null, null, null, null, null, null, null, "design");

        String result = tools.projectChannel(ch, "topic-counter", null, "nonexistent");

        assertThat(result).isEqualTo("empty");
    }

    @Test
    @TestTransaction
    void topicAndMaxMessages_bothApplied() {
        String ch = "proj-combined-" + System.nanoTime();
        tools.createChannel(ch, "test", null, null, null, null, null, null, null, null, null, null, null, null, null);
        tools.sendMessage(ch, "alice", "status", "a", null, null, null, null, null, null, null, "design");
        tools.sendMessage(ch, "bob", "status", "b", null, null, null, null, null, null, null, "design");
        tools.sendMessage(ch, "carol", "status", "c", null, null, null, null, null, null, null, "design");

        String result = tools.projectChannel(ch, "topic-counter", 2, "design");

        assertThat(result).isEqualTo("count=2");
    }

    @Test
    @TestTransaction
    void topicOnly_noMaxMessages_scopedPathTaken() {
        String ch = "proj-topiconly-" + System.nanoTime();
        tools.createChannel(ch, "test", null, null, null, null, null, null, null, null, null, null, null, null, null);
        tools.sendMessage(ch, "alice", "status", "a", null, null, null, null, null, null, null, "design");
        tools.sendMessage(ch, "bob", "status", "b", null, null, null, null, null, null, null, "testing");
        tools.sendMessage(ch, "carol", "status", "c", null, null, null, null, null, null, null, "design");
        tools.sendMessage(ch, "dave", "status", "d", null, null, null, null, null, null, null, "design");

        String result = tools.projectChannel(ch, "topic-counter", null, "design");

        assertThat(result).isEqualTo("count=3");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ProjectChannelTopicTest" -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — `projectChannel` method does not accept 4 arguments

- [ ] **Step 3: Restructure `projectAndRender` in `QhorusMcpToolsBase`**

Use `ide_edit_member` to replace the 3-arg `projectAndRender` overload with one that accepts `topic`:

```java
<S> String projectAndRender(final UUID channelId, final RenderableProjection<S> projection,
                             final Integer maxMessages, final String topic) {
    final String normalizedTopic = (topic != null && !topic.isBlank()) ? topic : null;
    final boolean hasLimit = maxMessages != null && maxMessages > 0;
    if (normalizedTopic != null || hasLimit) {
        final var qb = MessageQuery.builder();
        if (normalizedTopic != null) qb.topic(normalizedTopic);
        if (hasLimit) qb.limit(maxMessages);
        final ProjectionResult<S> result = projectionService.project(channelId, qb.build(), projection);
        return projection.render(result);
    }
    return projectAndRender(channelId, projection);
}
```

- [ ] **Step 4: Update `QhorusMcpTools.projectChannel` to accept topic**

Use `ide_edit_member` to replace `projectChannel` in `QhorusMcpTools`:

```java
@Tool(name = "project_channel",
        description = "Project a channel's message history through a named RenderableProjection "
                + "and return the rendered result as a String. "
                + "The projection folds messages in ascending insertion order (oldest first). "
                + "max_messages bounds the fold depth — use it to limit output size on busy channels. "
                + "Null or non-positive max_messages folds the full history. "
                + "topic scopes the fold to messages in a single topic — null folds all topics. "
                + "On LAST_WRITE channels the fold sees only the current snapshot (one message per sender, "
                + "not full history) — projections that assume a complete history will produce incorrect "
                + "results on LAST_WRITE channels. "
                + "Reads proceed on paused channels — projection is a read-only operation.")
public String projectChannel(
        @ToolArg(name = "channel",
                 description = "Channel name or UUID") String channel,
        @ToolArg(name = "projection_name",
                 description = "Name matching RenderableProjection.projectionName() "
                         + "(e.g. 'channel-summary')") String projectionName,
        @ToolArg(name = "max_messages",
                 description = "Maximum number of messages to fold, in insertion order (oldest first). "
                         + "Null or non-positive = fold full history. Default: null (unlimited).",
                 required = false) Integer maxMessages,
        @ToolArg(name = "topic",
                 description = "Scope the projection to messages in this topic only. "
                         + "Null = fold all topics. Case-insensitive matching.",
                 required = false) String topic) {
    return projectAndRender(resolveChannel(channel).id(), projectionRegistry.get(projectionName), maxMessages, topic);
}
```

- [ ] **Step 5: Update `ReactiveQhorusMcpTools.projectChannel` identically**

Use `ide_edit_member` to replace `projectChannel` in `ReactiveQhorusMcpTools` with the same signature and body as Step 4, keeping the `@Blocking` annotation.

- [ ] **Step 6: Fix existing callers of the old 3-arg `projectChannel`**

Use `ide_find_references` on `projectChannel` to find all callers. Update `ProjectChannelMaxMessagesTest` calls from `tools.projectChannel(ch, name, max)` to `tools.projectChannel(ch, name, max, null)`. Same for `ProjectChannelToolIT`.

- [ ] **Step 7: Run all projection tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ProjectChannelTopicTest,ProjectChannelMaxMessagesTest,ProjectChannelToolIT" -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: All tests PASS

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#338): topic parameter on project_channel MCP tool with blank normalization

Refs #338"
```

---

### Task 4: Update CLAUDE.md and verify coherence

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add MessageView.topic documentation to CLAUDE.md**

In the `MessageView` description (if it exists) or in the projection section, note:
- `MessageView.topic()` — nullable, positioned after `target`
- `project_channel` gains optional `topic` parameter — case-insensitive, blank normalized to null
- `MessageQueryJpql` now emits `LOWER(topic) = LOWER(?n)` predicate

- [ ] **Step 2: Run full test suite one final time**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#338): document MessageView.topic and project_channel topic parameter

Refs #338"
```

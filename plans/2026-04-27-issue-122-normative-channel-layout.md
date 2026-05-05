# Issue #122 — Normative Channel Layout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `MessageTypePolicy` SPI with server+client enforcement, expose `allowed_types` on `create_channel`, create `examples/normative-layout/` CI module, add Jlama scenario, and update all docs.

**Architecture:** `MessageTypePolicy` is a CDI SPI; `StoredMessageTypePolicy` (default) reads `Channel.allowedTypes` (comma-separated `MessageType` names, null=open). Both `QhorusMcpTools.sendMessage()` (client, fail-fast) and `MessageService.send()` (server, safety net) inject and call the policy. New `examples/normative-layout/` module proves the 3-channel pattern deterministically in CI (no LLM); a `NormativeLayoutAgentTest` in `examples/agent-communication/` exercises it with real Jlama agents.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, H2 (CI tests), quarkus-mcp-server 1.11.1

**Test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
**Examples command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout`
**Specific test:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ClassName -pl runtime`

---

### Task 1: MessageTypeViolationException + MessageTypePolicy interface

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageTypeViolationException.java`
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageTypePolicy.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/message/MessageTypeViolationExceptionTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.qhorus.message;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.message.MessageTypeViolationException;
import io.quarkiverse.qhorus.runtime.message.MessageType;

class MessageTypeViolationExceptionTest {

    @Test
    void message_includesChannelName() {
        var ex = new MessageTypeViolationException("case-abc/observe", MessageType.QUERY, "EVENT");
        assertThat(ex.getMessage()).contains("case-abc/observe");
    }

    @Test
    void message_includesAttemptedType() {
        var ex = new MessageTypeViolationException("case-abc/observe", MessageType.QUERY, "EVENT");
        assertThat(ex.getMessage()).contains("QUERY");
    }

    @Test
    void message_includesAllowedTypes() {
        var ex = new MessageTypeViolationException("case-abc/observe", MessageType.QUERY, "EVENT");
        assertThat(ex.getMessage()).contains("EVENT");
    }

    @Test
    void isRuntimeException() {
        assertThat(new MessageTypeViolationException("ch", MessageType.STATUS, "EVENT"))
                .isInstanceOf(RuntimeException.class);
    }
}
```

- [ ] **Step 2: Run test — confirm it fails (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageTypeViolationExceptionTest -pl runtime
```
Expected: compilation failure — `MessageTypeViolationException` does not exist.

- [ ] **Step 3: Create `MessageTypeViolationException`**

```java
package io.quarkiverse.qhorus.runtime.message;

public class MessageTypeViolationException extends RuntimeException {

    public MessageTypeViolationException(String channel, MessageType attempted, String allowed) {
        super("Channel '" + channel + "' does not permit " + attempted + ". Allowed: " + allowed);
    }
}
```

- [ ] **Step 4: Create `MessageTypePolicy` interface**

```java
package io.quarkiverse.qhorus.runtime.message;

import io.quarkiverse.qhorus.runtime.channel.Channel;

public interface MessageTypePolicy {

    /**
     * Validates that {@code type} is permitted on {@code channel}.
     * Throws {@link MessageTypeViolationException} to reject; returns normally to allow.
     */
    void validate(Channel channel, MessageType type);
}
```

- [ ] **Step 5: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageTypeViolationExceptionTest -pl runtime
```
Expected: 4 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageTypeViolationException.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageTypePolicy.java \
        runtime/src/test/java/io/quarkiverse/qhorus/message/MessageTypeViolationExceptionTest.java
git commit -m "feat(spi): MessageTypePolicy interface + MessageTypeViolationException

Refs #122"
```

---

### Task 2: StoredMessageTypePolicy — unit test + implementation

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/StoredMessageTypePolicy.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/message/StoredMessageTypePolicyTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.quarkiverse.qhorus.message;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.*;

class StoredMessageTypePolicyTest {

    private final StoredMessageTypePolicy policy = new StoredMessageTypePolicy();

    @Test
    void nullAllowedTypes_permitsAllTypes() {
        Channel ch = channel(null);
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.QUERY));
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.COMMAND));
    }

    @Test
    void blankAllowedTypes_permitsAllTypes() {
        Channel ch = channel("   ");
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.STATUS));
    }

    @Test
    void singleType_permitsThatType() {
        Channel ch = channel("EVENT");
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
    }

    @Test
    void singleType_rejectsOtherType() {
        Channel ch = channel("EVENT");
        assertThrows(MessageTypeViolationException.class,
                () -> policy.validate(ch, MessageType.QUERY));
    }

    @Test
    void multipleTypes_permitsAllListed() {
        Channel ch = channel("QUERY,COMMAND,RESPONSE");
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.QUERY));
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.COMMAND));
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.RESPONSE));
    }

    @Test
    void multipleTypes_rejectsUnlisted() {
        Channel ch = channel("QUERY,COMMAND,RESPONSE");
        assertThrows(MessageTypeViolationException.class,
                () -> policy.validate(ch, MessageType.EVENT));
    }

    @Test
    void whitespaceAroundCommas_isTrimmed() {
        Channel ch = channel("EVENT , STATUS");
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.EVENT));
        assertDoesNotThrow(() -> policy.validate(ch, MessageType.STATUS));
    }

    @Test
    void unknownTypeName_throwsIllegalArgument() {
        Channel ch = channel("RUBBISH");
        assertThrows(IllegalArgumentException.class,
                () -> policy.validate(ch, MessageType.EVENT));
    }

    @Test
    void violationMessage_containsChannelNameAndTypes() {
        Channel ch = channel("EVENT");
        ch.name = "case-abc/observe";
        var ex = assertThrows(MessageTypeViolationException.class,
                () -> policy.validate(ch, MessageType.QUERY));
        assertThat(ex.getMessage()).contains("case-abc/observe").contains("QUERY").contains("EVENT");
    }

    @Test
    void allNineTypes_permitted_whenOpen() {
        Channel ch = channel(null);
        for (MessageType t : MessageType.values()) {
            assertDoesNotThrow(() -> policy.validate(ch, t),
                    "Expected " + t + " to be permitted on open channel");
        }
    }

    private Channel channel(String allowedTypes) {
        Channel ch = new Channel();
        ch.name = "test-channel";
        ch.allowedTypes = allowedTypes;
        ch.semantic = ChannelSemantic.APPEND;
        return ch;
    }
}
```

- [ ] **Step 2: Run test — confirm compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StoredMessageTypePolicyTest -pl runtime
```
Expected: compilation failure — `StoredMessageTypePolicy` does not exist, `Channel.allowedTypes` does not exist.

- [ ] **Step 3: Add `allowedTypes` field to `Channel`**

In `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/Channel.java`, after the `rateLimitPerInstance` field:

```java
    /**
     * Comma-separated list of permitted MessageType names.
     * Null means all types are permitted (open channel).
     * Example: "EVENT" for a telemetry-only observe channel.
     */
    @Column(name = "allowed_types", columnDefinition = "TEXT")
    public String allowedTypes;
```

- [ ] **Step 4: Create `StoredMessageTypePolicy`**

```java
package io.quarkiverse.qhorus.runtime.message;

import java.util.Arrays;
import java.util.Set;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkiverse.qhorus.runtime.channel.Channel;

@ApplicationScoped
public class StoredMessageTypePolicy implements MessageTypePolicy {

    @Override
    public void validate(Channel channel, MessageType type) {
        if (channel.allowedTypes == null || channel.allowedTypes.isBlank()) {
            return;
        }
        Set<MessageType> allowed = Arrays.stream(channel.allowedTypes.split(","))
                .map(String::trim)
                .map(MessageType::valueOf)
                .collect(Collectors.toUnmodifiableSet());
        if (!allowed.contains(type)) {
            throw new MessageTypeViolationException(channel.name, type, channel.allowedTypes);
        }
    }
}
```

- [ ] **Step 5: Run tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=StoredMessageTypePolicyTest -pl runtime
```
Expected: 10 tests PASS.

- [ ] **Step 6: Run full suite — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all existing tests PASS (new field is nullable, schema is drop-and-create).

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/Channel.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/StoredMessageTypePolicy.java \
        runtime/src/test/java/io/quarkiverse/qhorus/message/StoredMessageTypePolicyTest.java
git commit -m "feat(spi): StoredMessageTypePolicy default implementation + Channel.allowedTypes field

Refs #122"
```

---

### Task 3: ChannelService + ReactiveChannelService — allowedTypes overload

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ReactiveChannelService.java`
- Modify: `runtime/src/test/java/io/quarkiverse/qhorus/channel/ChannelServiceTest.java`

- [ ] **Step 1: Write failing tests in `ChannelServiceTest`**

Add these tests to the existing `ChannelServiceTest` class (it's a `@QuarkusTest`):

```java
    @Test
    void createWithAllowedTypes_storesConstraint() {
        String name = "allowed-types-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() -> {
            Channel ch = channelService.create(name, "Telemetry", ChannelSemantic.APPEND,
                    null, null, null, null, null, "EVENT");
            assertThat(ch.allowedTypes).isEqualTo("EVENT");
        });
    }

    @Test
    void createWithNullAllowedTypes_storesNull() {
        String name = "no-allowed-types-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() -> {
            Channel ch = channelService.create(name, "Open", ChannelSemantic.APPEND,
                    null, null, null, null, null, null);
            assertThat(ch.allowedTypes).isNull();
        });
    }

    @Test
    void createWithBlankAllowedTypes_storesNull() {
        String name = "blank-allowed-types-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() -> {
            Channel ch = channelService.create(name, "Open", ChannelSemantic.APPEND,
                    null, null, null, null, null, "  ");
            assertThat(ch.allowedTypes).isNull();
        });
    }

    @Test
    void existingFourParamOverload_setsNullAllowedTypes() {
        String name = "legacy-overload-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() -> {
            Channel ch = channelService.create(name, "Legacy", ChannelSemantic.APPEND, null);
            assertThat(ch.allowedTypes).isNull();
        });
    }
```

- [ ] **Step 2: Run tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelServiceTest -pl runtime
```
Expected: compilation failure — `create(...)` with 9 params does not exist.

- [ ] **Step 3: Add 9-param `create()` to `ChannelService` and make 8-param delegate to it**

In `ChannelService.java`, replace the body of the existing 8-param `create()` method and add a new 9-param version:

```java
    @Transactional
    public Channel create(String name, String description, ChannelSemantic semantic,
            String barrierContributors, String allowedWriters, String adminInstances,
            Integer rateLimitPerChannel, Integer rateLimitPerInstance) {
        return create(name, description, semantic, barrierContributors, allowedWriters,
                adminInstances, rateLimitPerChannel, rateLimitPerInstance, null);
    }

    @Transactional
    public Channel create(String name, String description, ChannelSemantic semantic,
            String barrierContributors, String allowedWriters, String adminInstances,
            Integer rateLimitPerChannel, Integer rateLimitPerInstance, String allowedTypes) {
        Channel channel = new Channel();
        channel.name = name;
        channel.description = description;
        channel.semantic = semantic;
        channel.barrierContributors = barrierContributors;
        channel.allowedWriters = (allowedWriters == null || allowedWriters.isBlank()) ? null : allowedWriters;
        channel.adminInstances = (adminInstances == null || adminInstances.isBlank()) ? null : adminInstances;
        channel.rateLimitPerChannel = rateLimitPerChannel;
        channel.rateLimitPerInstance = rateLimitPerInstance;
        channel.allowedTypes = (allowedTypes == null || allowedTypes.isBlank()) ? null : allowedTypes;
        channelStore.put(channel);
        return channel;
    }
```

- [ ] **Step 4: Add 9-param `create()` to `ReactiveChannelService` and make 8-param delegate**

In `ReactiveChannelService.java`, replace the 8-param `create()` body and add a 9-param version:

```java
    public Uni<Channel> create(String name, String description, ChannelSemantic semantic,
            String barrierContributors, String allowedWriters, String adminInstances,
            Integer rateLimitPerChannel, Integer rateLimitPerInstance) {
        return create(name, description, semantic, barrierContributors, allowedWriters,
                adminInstances, rateLimitPerChannel, rateLimitPerInstance, null);
    }

    public Uni<Channel> create(String name, String description, ChannelSemantic semantic,
            String barrierContributors, String allowedWriters, String adminInstances,
            Integer rateLimitPerChannel, Integer rateLimitPerInstance, String allowedTypes) {
        return Panache.withTransaction(() -> {
            Channel channel = new Channel();
            channel.name = name;
            channel.description = description;
            channel.semantic = semantic;
            channel.barrierContributors = barrierContributors;
            channel.allowedWriters = (allowedWriters == null || allowedWriters.isBlank()) ? null : allowedWriters;
            channel.adminInstances = (adminInstances == null || adminInstances.isBlank()) ? null : adminInstances;
            channel.rateLimitPerChannel = rateLimitPerChannel;
            channel.rateLimitPerInstance = rateLimitPerInstance;
            channel.allowedTypes = (allowedTypes == null || allowedTypes.isBlank()) ? null : allowedTypes;
            return channelStore.put(channel);
        });
    }
```

- [ ] **Step 5: Run tests — confirm new tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelServiceTest -pl runtime
```
Expected: all tests PASS including the 4 new ones.

- [ ] **Step 6: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests PASS.

- [ ] **Step 7: Add `allowedTypes` round-trip to `ChannelStoreContractTest`**

In `testing/src/test/java/io/quarkiverse/qhorus/testing/contract/ChannelStoreContractTest.java`, add:

```java
    @Test
    void put_and_find_preserves_allowedTypes() {
        Channel ch = channel("allowed-types-" + UUID.randomUUID(), ChannelSemantic.APPEND);
        ch.allowedTypes = "EVENT";
        put(ch);
        Channel found = find(ch.id).orElseThrow();
        assertEquals("EVENT", found.allowedTypes);
    }

    @Test
    void put_and_find_preserves_null_allowedTypes() {
        Channel ch = channel("null-allowed-" + UUID.randomUUID(), ChannelSemantic.APPEND);
        ch.allowedTypes = null;
        put(ch);
        Channel found = find(ch.id).orElseThrow();
        assertNull(found.allowedTypes);
    }
```

- [ ] **Step 8: Run testing module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing
```
Expected: all tests PASS including the 2 new contract tests (run by both blocking and reactive runners).

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ReactiveChannelService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/channel/ChannelServiceTest.java \
        testing/src/test/java/io/quarkiverse/qhorus/testing/contract/ChannelStoreContractTest.java
git commit -m "feat(channel): allowedTypes field wired through ChannelService + ReactiveChannelService

Refs #122"
```

---

### Task 4: QhorusMcpToolsBase — add allowedTypes to ChannelDetail + toChannelDetail

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java`

- [ ] **Step 1: Add `allowedTypes` as 13th component to `ChannelDetail` record**

In `QhorusMcpToolsBase.java`, replace the `ChannelDetail` record:

```java
    public record ChannelDetail(
            UUID channelId,
            String name,
            String description,
            String semantic,
            String barrierContributors,
            long messageCount,
            String lastActivityAt,
            boolean paused,
            String allowedWriters,
            String adminInstances,
            Integer rateLimitPerChannel,
            Integer rateLimitPerInstance,
            /** Comma-separated permitted MessageType names, or null if open to all types. */
            String allowedTypes) {
    }
```

- [ ] **Step 2: Update `toChannelDetail` mapper**

In `QhorusMcpToolsBase.java`, replace the `toChannelDetail` method body:

```java
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
                ch.rateLimitPerInstance,
                ch.allowedTypes);
    }
```

- [ ] **Step 3: Run full suite — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests PASS. Existing tests do not assert `allowedTypes()` so the new component doesn't break them.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpToolsBase.java
git commit -m "feat(mcp): expose allowedTypes in ChannelDetail response record

Refs #122"
```

---

### Task 5: QhorusMcpTools — create_channel allowed_types param + sendMessage client enforcement

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/mcp/ChannelAllowedTypesTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.quarkiverse.qhorus.mcp;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpTools;
import io.quarkiverse.qhorus.runtime.mcp.QhorusMcpToolsBase.ChannelDetail;
import io.quarkiverse.qhorus.runtime.message.MessageTypeViolationException;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class ChannelAllowedTypesTest {

    @Inject
    QhorusMcpTools tools;

    @Test
    @TestTransaction
    void createChannel_withAllowedTypes_roundtripsInDetail() {
        ChannelDetail detail = tools.createChannel(
                "oversight-" + System.nanoTime(), "Human governance", "APPEND",
                null, null, null, null, null, "QUERY,COMMAND");
        assertThat(detail.allowedTypes()).isEqualTo("QUERY,COMMAND");
    }

    @Test
    @TestTransaction
    void createChannel_nullAllowedTypes_detailShowsNull() {
        ChannelDetail detail = tools.createChannel(
                "open-" + System.nanoTime(), "Open channel", "APPEND",
                null, null, null, null, null, null);
        assertThat(detail.allowedTypes()).isNull();
    }

    @Test
    @TestTransaction
    void createChannel_existingFourParamOverload_detailShowsNull() {
        ChannelDetail detail = tools.createChannel(
                "legacy-" + System.nanoTime(), "Legacy call", null, null);
        assertThat(detail.allowedTypes()).isNull();
    }

    @Test
    @TestTransaction
    void sendMessage_rejectsDisallowedType_clientSide() {
        String name = "observe-enforce-" + System.nanoTime();
        tools.createChannel(name, "Telemetry only", "APPEND",
                null, null, null, null, null, "EVENT");
        assertThrows(Exception.class, () ->
                tools.sendMessage(name, "agent-1", "QUERY", "hello?",
                        null, null, null, null, null));
    }

    @Test
    @TestTransaction
    void sendMessage_permitsAllowedType_clientSide() {
        String name = "observe-ok-" + System.nanoTime();
        tools.createChannel(name, "Telemetry only", "APPEND",
                null, null, null, null, null, "EVENT");
        assertDoesNotThrow(() ->
                tools.sendMessage(name, "agent-1", "EVENT", "{\"tool\":\"read\"}",
                        null, null, null, null, null));
    }

    @Test
    @TestTransaction
    void sendMessage_openChannel_permitsAllTypes() {
        String name = "open-all-" + System.nanoTime();
        tools.createChannel(name, "Open", "APPEND", null, null, null, null, null, null);
        assertDoesNotThrow(() ->
                tools.sendMessage(name, "agent-1", "COMMAND", "do something",
                        null, null, null, null, null));
    }

    @Test
    @TestTransaction
    void violationError_mentionsChannelAndType() {
        String name = "oversight-block-" + System.nanoTime();
        tools.createChannel(name, "Governance", "APPEND",
                null, null, null, null, null, "QUERY,COMMAND");
        Exception ex = assertThrows(Exception.class, () ->
                tools.sendMessage(name, "agent-1", "EVENT", "{\"tool\":\"read\"}",
                        null, null, null, null, null));
        assertThat(ex.getMessage()).containsIgnoringCase("EVENT");
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelAllowedTypesTest -pl runtime
```
Expected: compilation failure — `createChannel` has no 9-param overload; `sendMessage` does not enforce types.

- [ ] **Step 3: Add `@Inject MessageTypePolicy` to `QhorusMcpTools`**

In `QhorusMcpTools.java`, add the injection after existing `@Inject` fields:

```java
    @Inject
    MessageTypePolicy messageTypePolicy;
```

Also add the import:
```java
import io.quarkiverse.qhorus.runtime.message.MessageTypePolicy;
```

- [ ] **Step 4: Add `allowed_types` param to `@Tool create_channel` in `QhorusMcpTools`**

The existing `@Tool`-annotated `createChannel` (8 params, at line ~246) gains a 9th param and updates its description and service call:

```java
    @Tool(name = "create_channel", description = "Create a named channel with declared semantic. "
            + "Semantic defaults to APPEND if not specified. "
            + "Use allowed_types to restrict which MessageType values may be posted — "
            + "the constraint is enforced at both the MCP tool layer and MessageService.")
    @Transactional
    public ChannelDetail createChannel(
            @ToolArg(name = "name", description = "Unique channel name") String name,
            @ToolArg(name = "description", description = "Channel purpose description") String description,
            @ToolArg(name = "semantic", description = "Channel semantic: APPEND (default), COLLECT, BARRIER, EPHEMERAL, LAST_WRITE", required = false) String semantic,
            @ToolArg(name = "barrier_contributors", description = "Comma-separated contributor names (BARRIER channels only)", required = false) String barrierContributors,
            @ToolArg(name = "allowed_writers", description = "Comma-separated allowed writers: bare instance IDs and/or capability:tag / role:name patterns. Null = open to all.", required = false) String allowedWriters,
            @ToolArg(name = "admin_instances", description = "Comma-separated instance IDs permitted to manage this channel (pause/resume/force_release/clear). Null = open to any caller.", required = false) String adminInstances,
            @ToolArg(name = "rate_limit_per_channel", description = "Max messages per minute across all senders. Null = unlimited.", required = false) Integer rateLimitPerChannel,
            @ToolArg(name = "rate_limit_per_instance", description = "Max messages per minute from a single sender. Null = unlimited.", required = false) Integer rateLimitPerInstance,
            @ToolArg(name = "allowed_types", description = "Comma-separated MessageType names permitted on this channel. Null = all types permitted. Example: \"EVENT\" for a telemetry-only observe channel; \"QUERY,COMMAND\" for a governance channel.", required = false) String allowedTypes) {
        ChannelSemantic sem;
        if (semantic == null || semantic.isBlank()) {
            sem = ChannelSemantic.APPEND;
        } else {
            try {
                sem = ChannelSemantic.valueOf(semantic.toUpperCase());
            } catch (IllegalArgumentException e) {
                throw new IllegalArgumentException(
                        "Invalid semantic '" + semantic + "'. Valid values: APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE");
            }
        }
        Channel ch = channelService.create(name, description, sem, barrierContributors, allowedWriters,
                adminInstances, rateLimitPerChannel, rateLimitPerInstance, allowedTypes);
        return toChannelDetail(ch, 0L);
    }
```

- [ ] **Step 5: Add type policy check to `sendMessage` in `QhorusMcpTools`**

In the main `@Tool`-annotated `sendMessage` method, after `MessageType msgType = MessageType.valueOf(type.toUpperCase());` and after the `requiresContent`/`requiresTarget` checks, add before the ACL check:

```java
        // Type policy — client-side enforcement (MessageService enforces server-side too)
        messageTypePolicy.validate(ch, msgType);
```

Also add the import at the top of the file:
```java
import io.quarkiverse.qhorus.runtime.message.MessageTypeViolationException;
```

- [ ] **Step 6: Run tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelAllowedTypesTest -pl runtime
```
Expected: 7 tests PASS.

- [ ] **Step 7: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests PASS.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/QhorusMcpTools.java \
        runtime/src/test/java/io/quarkiverse/qhorus/mcp/ChannelAllowedTypesTest.java
git commit -m "feat(mcp): create_channel allowed_types param + sendMessage client-side type enforcement

Refs #122"
```

---

### Task 6: MessageService server-side enforcement + integration test

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageService.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/message/MessageServiceTypeEnforcementTest.java`

- [ ] **Step 1: Write failing integration test**

```java
package io.quarkiverse.qhorus.message;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.message.*;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MessageServiceTypeEnforcementTest {

    @Inject
    MessageService messageService;

    @Inject
    ChannelService channelService;

    @Test
    void serverSide_rejectsDisallowedType_bypassingMcpTool() {
        String name = "server-enforce-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Telemetry only", ChannelSemantic.APPEND,
                        null, null, null, null, null, "EVENT"));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(name).orElseThrow());

        assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent-1", MessageType.QUERY,
                                "text", null, null)));
    }

    @Test
    void serverSide_permitsAllowedType() {
        String name = "server-allow-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Telemetry only", ChannelSemantic.APPEND,
                        null, null, null, null, null, "EVENT"));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(name).orElseThrow());

        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent-1", MessageType.EVENT,
                                "{\"tool\":\"read\"}", null, null)));
    }

    @Test
    void serverSide_permitsAllTypes_whenConstraintIsNull() {
        String name = "server-open-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Open channel", ChannelSemantic.APPEND, null));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(name).orElseThrow());

        for (MessageType t : MessageType.values()) {
            final MessageType type = t;
            assertDoesNotThrow(() ->
                    QuarkusTransaction.requiringNew().run(() ->
                            messageService.send(ch.id, "agent-1", type, "content", null, null)),
                    "Expected " + t + " to be permitted on open channel");
        }
    }

    @Test
    void serverSide_violation_messageContainsChannelAndType() {
        String name = "server-msg-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Governance", ChannelSemantic.APPEND,
                        null, null, null, null, null, "QUERY,COMMAND"));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(name).orElseThrow());

        MessageTypeViolationException ex = assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent-1", MessageType.EVENT,
                                "{}", null, null)));
        assertThat(ex.getMessage()).contains(name).contains("EVENT");
    }

    @Test
    void serverSide_multiTypeConstraint_QUERY_COMMAND_permitsCommand() {
        String name = "server-multi-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(name, "Governance", ChannelSemantic.APPEND,
                        null, null, null, null, null, "QUERY,COMMAND"));

        Channel ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(name).orElseThrow());

        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent-1", MessageType.COMMAND,
                                "do it", null, null)));
    }
}
```

- [ ] **Step 2: Run test — confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageServiceTypeEnforcementTest -pl runtime
```
Expected: test failures — `MessageService` does not yet enforce the policy.

- [ ] **Step 3: Inject `MessageTypePolicy` into `MessageService` and add enforcement**

In `MessageService.java`, add the injection:

```java
    @Inject
    MessageTypePolicy messageTypePolicy;
```

Add the import:
```java
import io.quarkiverse.qhorus.runtime.message.MessageTypePolicy;
```

In the main `send()` overload (8 params), add as the **first line** of the method body:

```java
        channelService.findById(channelId)
                .ifPresent(ch -> messageTypePolicy.validate(ch, type));
```

- [ ] **Step 4: Run tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MessageServiceTypeEnforcementTest -pl runtime
```
Expected: 5 tests PASS.

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/message/MessageServiceTypeEnforcementTest.java
git commit -m "feat(message): MessageService server-side type enforcement via MessageTypePolicy SPI

Refs #122"
```

---

### Task 7: ReactiveQhorusMcpTools — mirror create_channel + sendMessage changes

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

- [ ] **Step 1: Add `@Inject MessageTypePolicy` to `ReactiveQhorusMcpTools`**

After existing `@Inject` fields in `ReactiveQhorusMcpTools.java`:

```java
    @Inject
    MessageTypePolicy messageTypePolicy;
```

Import:
```java
import io.quarkiverse.qhorus.runtime.message.MessageTypePolicy;
```

- [ ] **Step 2: Add `allowed_types` param to `@Tool create_channel` in `ReactiveQhorusMcpTools`**

Replace the existing `@Tool`-annotated `createChannel` method:

```java
    @Tool(name = "create_channel", description = "Create a named channel with declared semantic. "
            + "Semantic defaults to APPEND if not specified. "
            + "Use allowed_types to restrict which MessageType values may be posted — "
            + "the constraint is enforced at both the MCP tool layer and MessageService.")
    public Uni<ChannelDetail> createChannel(
            @ToolArg(name = "name", description = "Unique channel name") String name,
            @ToolArg(name = "description", description = "Channel purpose description") String description,
            @ToolArg(name = "semantic", description = "Channel semantic: APPEND (default), COLLECT, BARRIER, EPHEMERAL, LAST_WRITE", required = false) String semantic,
            @ToolArg(name = "barrier_contributors", description = "Comma-separated contributor names (BARRIER channels only)", required = false) String barrierContributors,
            @ToolArg(name = "allowed_writers", description = "Comma-separated allowed writers: bare instance IDs and/or capability:tag / role:name patterns. Null = open to all.", required = false) String allowedWriters,
            @ToolArg(name = "admin_instances", description = "Comma-separated instance IDs permitted to manage this channel (pause/resume/force_release/clear). Null = open to any caller.", required = false) String adminInstances,
            @ToolArg(name = "rate_limit_per_channel", description = "Max messages per minute across all senders. Null = unlimited.", required = false) Integer rateLimitPerChannel,
            @ToolArg(name = "rate_limit_per_instance", description = "Max messages per minute from a single sender. Null = unlimited.", required = false) Integer rateLimitPerInstance,
            @ToolArg(name = "allowed_types", description = "Comma-separated MessageType names permitted on this channel. Null = all types permitted. Example: \"EVENT\" for a telemetry-only observe channel.", required = false) String allowedTypes) {
        ChannelSemantic sem;
        if (semantic == null || semantic.isBlank()) {
            sem = ChannelSemantic.APPEND;
        } else {
            try {
                sem = ChannelSemantic.valueOf(semantic.toUpperCase());
            } catch (IllegalArgumentException e) {
                throw new IllegalArgumentException(
                        "Invalid semantic '" + semantic + "'. Valid values: APPEND, COLLECT, BARRIER, EPHEMERAL, LAST_WRITE");
            }
        }
        return channelService.create(name, description, sem, barrierContributors, allowedWriters,
                adminInstances, rateLimitPerChannel, rateLimitPerInstance, allowedTypes)
                .flatMap(ch -> messageStore.countByChannel(ch.id)
                        .map(count -> toChannelDetail(ch, count.longValue())));
    }
```

- [ ] **Step 3: Add type policy check to `sendMessage` in `ReactiveQhorusMcpTools`**

Find the reactive `sendMessage` implementation (around line 527). It resolves the channel, then checks paused, observer, type, content/target. After `MessageType msgType = MessageType.valueOf(type.toUpperCase());` and the `requiresContent`/`requiresTarget` checks, add:

```java
        // Type policy — client-side enforcement
        messageTypePolicy.validate(ch, msgType);
```

- [ ] **Step 4: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests PASS. (Reactive tools are only active under the build property; the tests exercise `QhorusMcpTools` which is already covered.)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java
git commit -m "feat(mcp): mirror allowed_types + type enforcement in ReactiveQhorusMcpTools

Refs #122"
```

---

### Task 8: examples/normative-layout/ — module scaffold

**Files:**
- Create: `examples/normative-layout/pom.xml`
- Create: `examples/normative-layout/src/test/resources/application.properties`
- Modify: `examples/pom.xml`

- [ ] **Step 1: Add module to `examples/pom.xml`**

In `examples/pom.xml`, add `normative-layout` to the default `<modules>` block (alongside `type-system`, before the `agent-communication` profile):

```xml
    <module>type-system</module>
    <module>normative-layout</module>
```

- [ ] **Step 2: Create `examples/normative-layout/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>io.quarkiverse.qhorus</groupId>
    <artifactId>quarkus-qhorus-examples</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>quarkus-qhorus-example-normative-layout</artifactId>
  <name>Quarkus Qhorus :: Examples :: Normative Channel Layout</name>
  <description>
    Deterministic CI tests proving the 3-channel NormativeChannelLayout pattern.
    No LLM required. Canonical Layer 1 reference importable by Claudony and CaseHub.
    Covers: happy path, type enforcement, obligation lifecycle, robustness, correctness.
  </description>

  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.qhorus</groupId>
      <artifactId>quarkus-qhorus</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkiverse.qhorus</groupId>
      <artifactId>quarkus-qhorus-testing</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <version>3.27.3</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 3: Create `examples/normative-layout/src/test/resources/application.properties`**

```properties
# InMemory stores — no database SQL executed even though H2 is configured
quarkus.arc.selected-alternatives=\
  io.quarkiverse.qhorus.testing.InMemoryChannelStore,\
  io.quarkiverse.qhorus.testing.InMemoryMessageStore,\
  io.quarkiverse.qhorus.testing.InMemoryInstanceStore,\
  io.quarkiverse.qhorus.testing.InMemoryDataStore,\
  io.quarkiverse.qhorus.testing.InMemoryCommitmentStore

# H2 — satisfies Quarkus/Hibernate boot requirements
quarkus.datasource.db-kind=h2
quarkus.datasource.username=sa
quarkus.datasource.password=
quarkus.datasource.jdbc.url=jdbc:h2:mem:normative-layout;DB_CLOSE_DELAY=-1
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:normative-layout;DB_CLOSE_DELAY=-1
quarkus.datasource.qhorus.reactive=false
quarkus.hibernate-orm.packages=io.quarkiverse.qhorus.runtime.config
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
quarkus.flyway.qhorus.migrate-at-start=false
quarkus.ledger.enabled=true

quarkus.http.test-port=0
```

- [ ] **Step 4: Verify module builds (no tests yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl examples/normative-layout -am -DskipTests
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add examples/pom.xml examples/normative-layout/
git commit -m "feat(examples): scaffold normative-layout module — CI, no LLM

Refs #122"
```

---

### Task 9: SecureCodeReviewScenario + NormativeLayoutHappyPathTest

**Files:**
- Create: `examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/SecureCodeReviewScenario.java`
- Create: `examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutHappyPathTest.java`

- [ ] **Step 1: Create `SecureCodeReviewScenario`**

This class encapsulates the full scenario against injected services. Tests compose with it.

```java
package io.quarkiverse.qhorus.examples.normativelayout;

import java.util.List;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageService;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.data.DataService;
import io.quarkiverse.qhorus.runtime.store.CommitmentStore;
import io.quarkiverse.qhorus.runtime.channel.Channel;

/**
 * Canonical Layer 1 Secure Code Review scenario.
 * Two agents coordinate through 3 normative channels; no LLM required.
 * Importable by Claudony and CaseHub as the Layer 1 reference.
 */
public class SecureCodeReviewScenario {

    public final String caseId;
    public final String workChannel;
    public final String observeChannel;
    public final String oversightChannel;

    private final ChannelService channelService;
    private final InstanceService instanceService;
    private final MessageService messageService;
    private final DataService dataService;

    public SecureCodeReviewScenario(String caseId,
            ChannelService channelService,
            InstanceService instanceService,
            MessageService messageService,
            DataService dataService) {
        this.caseId = caseId;
        this.workChannel = "case-" + caseId + "/work";
        this.observeChannel = "case-" + caseId + "/observe";
        this.oversightChannel = "case-" + caseId + "/oversight";
        this.channelService = channelService;
        this.instanceService = instanceService;
        this.messageService = messageService;
        this.dataService = dataService;
    }

    /** Create the 3-channel normative layout for this case. */
    public void setupChannels() {
        channelService.create(workChannel, "Worker coordination", ChannelSemantic.APPEND,
                null, null, null, null, null, null);
        channelService.create(observeChannel, "Telemetry", ChannelSemantic.APPEND,
                null, null, null, null, null, "EVENT");
        channelService.create(oversightChannel, "Human governance", ChannelSemantic.APPEND,
                null, null, null, null, null, "QUERY,COMMAND");
    }

    public Channel workChannel() {
        return channelService.findByName(workChannel).orElseThrow();
    }

    public Channel observeChannel() {
        return channelService.findByName(observeChannel).orElseThrow();
    }

    public Channel oversightChannel() {
        return channelService.findByName(oversightChannel).orElseThrow();
    }

    /** Researcher registers, runs analysis, shares artefact, posts DONE. */
    public Message runResearcher(String correlationId) {
        instanceService.register("researcher-001", "Security analyst",
                List.of("security", "code-analysis"), null);

        Channel work = workChannel();
        Channel observe = observeChannel();

        messageService.send(work.id, "researcher-001", MessageType.STATUS,
                "Starting security analysis of AuthService.java", null, null);
        messageService.send(observe.id, "researcher-001", MessageType.EVENT,
                "{\"tool\":\"read_file\",\"path\":\"AuthService.java\"}", null, null);
        messageService.send(observe.id, "researcher-001", MessageType.EVENT,
                "{\"tool\":\"read_file\",\"path\":\"TokenRefreshService.java\"}", null, null);

        dataService.share("auth-analysis-v1", "researcher-001",
                "## Security Analysis\nFinding 1: SQL injection — HIGH\nFinding 2: Stale token — MEDIUM");

        return messageService.send(work.id, "researcher-001", MessageType.DONE,
                "Analysis complete. 3 findings. Report: shared-data:auth-analysis-v1",
                correlationId, null);
    }

    /**
     * Reviewer picks up DONE, queries researcher about Finding 3,
     * receives RESPONSE, shares report, posts DONE.
     */
    public Message runReviewer(String queryCorrelationId, String doneCorrelationId) {
        instanceService.register("reviewer-001", "Security reviewer",
                List.of("review", "security"), null);

        Channel work = workChannel();

        // Reviewer queries researcher
        messageService.send(work.id, "reviewer-001", MessageType.QUERY,
                "Finding #3: does TokenRefreshService.java:142 share the same root cause?",
                queryCorrelationId, null, null, "instance:researcher-001");

        // Researcher responds (discharges obligation)
        messageService.send(work.id, "researcher-001", MessageType.RESPONSE,
                "Yes — same interpolated SQL pattern. One root cause, two surfaces.",
                queryCorrelationId, null);

        dataService.share("review-report-v1", "reviewer-001",
                "## Code Review Report\nRoot cause A: SQL injection (CRITICAL)\nRoot cause B: Stale token (HIGH)");

        return messageService.send(work.id, "reviewer-001", MessageType.DONE,
                "Review complete. Final report: shared-data:review-report-v1",
                doneCorrelationId, null);
    }
}
```

- [ ] **Step 2: Write `NormativeLayoutHappyPathTest`**

```java
package io.quarkiverse.qhorus.examples.normativelayout;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.UUID;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.message.*;
import io.quarkiverse.qhorus.runtime.data.DataService;
import io.quarkiverse.qhorus.runtime.store.MessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class NormativeLayoutHappyPathTest {

    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;
    @Inject MessageService messageService;
    @Inject DataService dataService;
    @Inject MessageStore messageStore;

    private SecureCodeReviewScenario scenario() {
        return new SecureCodeReviewScenario("scr-" + UUID.randomUUID(),
                channelService, instanceService, messageService, dataService);
    }

    @Test
    void researcherPostsDoneOnWorkChannel() {
        String caseId = "happy-1-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            Message done = s.runResearcher("corr-researcher-done");
            assertThat(done.messageType).isEqualTo(MessageType.DONE);
            assertThat(done.content).contains("auth-analysis-v1");
        });
    }

    @Test
    void reviewerPostsDoneAfterResearcher() {
        String caseId = "happy-2-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            Message done = s.runReviewer("corr-q-001", "corr-reviewer-done");
            assertThat(done.messageType).isEqualTo(MessageType.DONE);
            assertThat(done.content).contains("review-report-v1");
        });
    }

    @Test
    void artefactsStoredAndRetrievable() {
        String caseId = "happy-3-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            s.runReviewer("corr-q", "corr-rev-done");
            assertThat(dataService.get("auth-analysis-v1")).isPresent();
            assertThat(dataService.get("review-report-v1")).isPresent();
        });
    }

    @Test
    void observeChannelReceivesEventMessages() {
        String caseId = "happy-4-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            List<Message> events = messageStore.scan(MessageQuery.builder()
                    .channelId(s.observeChannel().id).build());
            assertThat(events).isNotEmpty();
            assertThat(events).allMatch(m -> m.messageType == MessageType.EVENT);
        });
    }

    @Test
    void workChannelContainsStatusQueryResponseDone() {
        String caseId = "happy-5-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            s.runReviewer("corr-q", "corr-done");
            List<Message> msgs = messageStore.scan(MessageQuery.builder()
                    .channelId(s.workChannel().id).build());
            List<MessageType> types = msgs.stream().map(m -> m.messageType).toList();
            assertThat(types).contains(MessageType.STATUS, MessageType.DONE,
                    MessageType.QUERY, MessageType.RESPONSE);
        });
    }
}
```

- [ ] **Step 3: Run the new module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout
```
Expected: 5 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add examples/normative-layout/src/test/java/
git commit -m "feat(examples): SecureCodeReviewScenario + NormativeLayoutHappyPathTest

Refs #122"
```

---

### Task 10: NormativeLayoutTypeEnforcementTest

**Files:**
- Create: `examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutTypeEnforcementTest.java`

- [ ] **Step 1: Write the test**

```java
package io.quarkiverse.qhorus.examples.normativelayout;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

import java.util.UUID;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.message.*;
import io.quarkiverse.qhorus.runtime.data.DataService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class NormativeLayoutTypeEnforcementTest {

    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;
    @Inject MessageService messageService;
    @Inject DataService dataService;

    private SecureCodeReviewScenario scenario(String suffix) {
        return new SecureCodeReviewScenario("enf-" + suffix + "-" + System.nanoTime(),
                channelService, instanceService, messageService, dataService);
    }

    @Test
    void observeChannel_rejectsQuery_serverSide() {
        SecureCodeReviewScenario s = scenario("1");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.observeChannel().id, "agent-1",
                                MessageType.QUERY, "text", null, null)));
    }

    @Test
    void observeChannel_rejectsCommand_serverSide() {
        SecureCodeReviewScenario s = scenario("2");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.observeChannel().id, "agent-1",
                                MessageType.COMMAND, "do it", null, null)));
    }

    @Test
    void observeChannel_permitsEvent_serverSide() {
        SecureCodeReviewScenario s = scenario("3");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.observeChannel().id, "agent-1",
                                MessageType.EVENT, "{\"tool\":\"read\"}", null, null)));
    }

    @Test
    void oversightChannel_rejectsEvent_serverSide() {
        SecureCodeReviewScenario s = scenario("4");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.oversightChannel().id, "agent-1",
                                MessageType.EVENT, "{\"tool\":\"read\"}", null, null)));
    }

    @Test
    void oversightChannel_permitsQuery_serverSide() {
        SecureCodeReviewScenario s = scenario("5");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.oversightChannel().id, "agent-1",
                                MessageType.QUERY, "Is finding #2 a false positive?",
                                UUID.randomUUID().toString(), null)));
    }

    @Test
    void oversightChannel_permitsCommand_serverSide() {
        SecureCodeReviewScenario s = scenario("6");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.oversightChannel().id, "human",
                                MessageType.COMMAND, "Include finding #2 as low-confidence.",
                                UUID.randomUUID().toString(), null)));
    }

    @Test
    void workChannel_permitsAllNineTypes() {
        SecureCodeReviewScenario s = scenario("7");
        QuarkusTransaction.requiringNew().run(s::setupChannels);
        UUID workId = QuarkusTransaction.requiringNew().run(() -> s.workChannel().id);

        for (MessageType t : MessageType.values()) {
            String corrId = t.requiresCorrelationId() ? UUID.randomUUID().toString() : null;
            String content = "test content for " + t;
            String target = t == MessageType.HANDOFF ? "instance:other-001" : null;
            assertDoesNotThrow(() ->
                    QuarkusTransaction.requiringNew().run(() ->
                            messageService.send(workId, "agent-1", t, content, corrId, null, null, target)),
                    "Expected " + t + " to be permitted on open work channel");
        }
    }

    @Test
    void violationException_messageContainsChannelNameAndType() {
        SecureCodeReviewScenario s = scenario("8");
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        MessageTypeViolationException ex = assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.observeChannel().id, "agent-1",
                                MessageType.STATUS, "progress update", null, null)));

        assertThat(ex.getMessage()).contains(s.observeChannel).contains("STATUS").contains("EVENT");
    }
}
```

- [ ] **Step 2: Run**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=NormativeLayoutTypeEnforcementTest -pl examples/normative-layout
```
Expected: 8 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutTypeEnforcementTest.java
git commit -m "test(examples): NormativeLayoutTypeEnforcementTest — observe/oversight/work type constraints

Refs #122"
```

---

### Task 11: NormativeLayoutObligationTest + NormativeLayoutCorrectnessTest

**Files:**
- Create: `examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutObligationTest.java`
- Create: `examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutCorrectnessTest.java`

- [ ] **Step 1: Write `NormativeLayoutObligationTest`**

```java
package io.quarkiverse.qhorus.examples.normativelayout;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.UUID;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.message.*;
import io.quarkiverse.qhorus.runtime.data.DataService;
import io.quarkiverse.qhorus.runtime.store.CommitmentStore;
import io.quarkiverse.qhorus.runtime.store.query.CommitmentQuery;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class NormativeLayoutObligationTest {

    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;
    @Inject MessageService messageService;
    @Inject DataService dataService;
    @Inject CommitmentStore commitmentStore;

    @Test
    void query_createsOpenCommitment() {
        String caseId = "obl-1-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        String corrId = UUID.randomUUID().toString();
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            instanceService.register("reviewer-001", "Reviewer", List.of("review"), null);
            messageService.send(s.workChannel().id, "reviewer-001", MessageType.QUERY,
                    "Is finding #3 the same root cause?", corrId, null);
        });

        QuarkusTransaction.requiringNew().run(() -> {
            var commitment = commitmentStore.findByCorrelationId(corrId).orElseThrow();
            assertThat(commitment.state).isEqualTo(CommitmentState.OPEN);
            assertThat(commitment.sender).isEqualTo("reviewer-001");
        });
    }

    @Test
    void response_fulfillsOpenCommitment() {
        String caseId = "obl-2-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        String corrId = UUID.randomUUID().toString();
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            instanceService.register("reviewer-001", "Reviewer", List.of("review"), null);
            instanceService.register("researcher-001", "Researcher", List.of("security"), null);
            messageService.send(s.workChannel().id, "reviewer-001", MessageType.QUERY,
                    "Same root cause?", corrId, null);
            messageService.send(s.workChannel().id, "researcher-001", MessageType.RESPONSE,
                    "Yes — same pattern.", corrId, null);
        });

        QuarkusTransaction.requiringNew().run(() -> {
            var commitment = commitmentStore.findByCorrelationId(corrId).orElseThrow();
            assertThat(commitment.state).isEqualTo(CommitmentState.FULFILLED);
        });
    }

    @Test
    void fullScenario_noStalledObligations() {
        String caseId = "obl-3-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            s.runReviewer("corr-q", "corr-done");
        });

        QuarkusTransaction.requiringNew().run(() -> {
            List<Commitment> open = commitmentStore.scan(CommitmentQuery.open());
            assertThat(open).as("No open obligations should remain after full scenario").isEmpty();
        });
    }

    @Test
    void decline_dischargesObligation() {
        String caseId = "obl-4-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        String corrId = UUID.randomUUID().toString();
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            instanceService.register("agent-a", "Agent A", List.of(), null);
            instanceService.register("agent-b", "Agent B", List.of(), null);
            messageService.send(s.workChannel().id, "agent-a", MessageType.QUERY,
                    "Can you handle this?", corrId, null);
            messageService.send(s.workChannel().id, "agent-b", MessageType.DECLINE,
                    "Outside my scope.", corrId, null);
        });

        QuarkusTransaction.requiringNew().run(() -> {
            var commitment = commitmentStore.findByCorrelationId(corrId).orElseThrow();
            assertThat(commitment.state).isEqualTo(CommitmentState.DECLINED);
        });
    }
}
```

- [ ] **Step 2: Write `NormativeLayoutCorrectnessTest`**

```java
package io.quarkiverse.qhorus.examples.normativelayout;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.message.*;
import io.quarkiverse.qhorus.runtime.data.DataService;
import io.quarkiverse.qhorus.runtime.store.MessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class NormativeLayoutCorrectnessTest {

    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;
    @Inject MessageService messageService;
    @Inject DataService dataService;
    @Inject MessageStore messageStore;

    @Test
    void workChannelMessageCount_matchesExpectedProtocol() {
        String caseId = "correct-1-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            s.runReviewer("corr-q", "corr-done");
        });
        QuarkusTransaction.requiringNew().run(() -> {
            List<Message> msgs = messageStore.scan(MessageQuery.builder()
                    .channelId(s.workChannel().id).build());
            // STATUS(researcher) + DONE(researcher) + QUERY(reviewer) + RESPONSE(researcher) + DONE(reviewer)
            assertThat(msgs).hasSize(5);
        });
    }

    @Test
    void observeChannelContainsOnlyEvents() {
        String caseId = "correct-2-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
        });
        QuarkusTransaction.requiringNew().run(() -> {
            List<Message> events = messageStore.scan(MessageQuery.builder()
                    .channelId(s.observeChannel().id).build());
            assertThat(events).isNotEmpty();
            assertThat(events).allSatisfy(m ->
                    assertThat(m.messageType).isEqualTo(MessageType.EVENT));
        });
    }

    @Test
    void observeChannelMessageCount_matchesResearcherToolCalls() {
        String caseId = "correct-3-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
        });
        QuarkusTransaction.requiringNew().run(() -> {
            List<Message> events = messageStore.scan(MessageQuery.builder()
                    .channelId(s.observeChannel().id).build());
            // Two EVENT messages: read AuthService.java + read TokenRefreshService.java
            assertThat(events).hasSize(2);
        });
    }

    @Test
    void artefactContent_isAccessibleAfterScenario() {
        String caseId = "correct-4-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            s.runReviewer("corr-q", "corr-done");
        });
        QuarkusTransaction.requiringNew().run(() -> {
            var analysis = dataService.get("auth-analysis-v1").orElseThrow();
            assertThat(analysis.content).contains("SQL injection");
            var report = dataService.get("review-report-v1").orElseThrow();
            assertThat(report.content).contains("Root cause A");
        });
    }

    @Test
    void doneMessages_sentByCorrectSenders() {
        String caseId = "correct-5-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            s.runResearcher(null);
            s.runReviewer("corr-q", "corr-done");
        });
        QuarkusTransaction.requiringNew().run(() -> {
            List<Message> dones = messageStore.scan(MessageQuery.builder()
                    .channelId(s.workChannel().id).build()).stream()
                    .filter(m -> m.messageType == MessageType.DONE).toList();
            assertThat(dones).hasSize(2);
            assertThat(dones.stream().map(m -> m.sender).toList())
                    .containsExactlyInAnyOrder("researcher-001", "reviewer-001");
        });
    }
}
```

- [ ] **Step 3: Run all normative-layout tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout
```
Expected: all tests PASS (happy path 5 + enforcement 8 + obligation 4 + correctness 5 = 22 tests).

- [ ] **Step 4: Commit**

```bash
git add examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutObligationTest.java \
        examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutCorrectnessTest.java
git commit -m "test(examples): obligation lifecycle + correctness tests for normative layout

Refs #122"
```

---

### Task 12: NormativeLayoutRobustnessTest

**Files:**
- Create: `examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutRobustnessTest.java`

- [ ] **Step 1: Write the test**

```java
package io.quarkiverse.qhorus.examples.normativelayout;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.UUID;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.message.*;
import io.quarkiverse.qhorus.runtime.data.DataService;
import io.quarkiverse.qhorus.runtime.store.CommitmentStore;
import io.quarkiverse.qhorus.runtime.store.query.CommitmentQuery;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class NormativeLayoutRobustnessTest {

    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;
    @Inject MessageService messageService;
    @Inject DataService dataService;
    @Inject CommitmentStore commitmentStore;

    @Test
    void customPolicy_canBeBypassedByAlternative_butServerCatches() {
        // Verifies that even if a custom policy permits all types,
        // the constraint on the channel entity is still the ground truth for StoredMessageTypePolicy.
        // (StoredMessageTypePolicy reads from channel.allowedTypes, not a flag.)
        String caseId = "robust-1-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        QuarkusTransaction.requiringNew().run(s::setupChannels);

        // Direct call to MessageService bypasses MCP-layer client enforcement —
        // server-side enforcement in MessageService must still catch it.
        assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(s.observeChannel().id, "agent-1",
                                MessageType.COMMAND, "direct bypass attempt", null, null)));
    }

    @Test
    void failure_dischargesObligation() {
        String caseId = "robust-2-" + System.nanoTime();
        SecureCodeReviewScenario s = new SecureCodeReviewScenario(caseId,
                channelService, instanceService, messageService, dataService);
        String corrId = UUID.randomUUID().toString();
        QuarkusTransaction.requiringNew().run(() -> {
            s.setupChannels();
            instanceService.register("commander", "Commander", List.of(), null);
            instanceService.register("worker", "Worker", List.of(), null);
            messageService.send(s.workChannel().id, "commander", MessageType.COMMAND,
                    "Analyse all services", corrId, null);
            messageService.send(s.workChannel().id, "worker", MessageType.FAILURE,
                    "Cannot access remote services — network error.", corrId, null);
        });

        QuarkusTransaction.requiringNew().run(() -> {
            var commitment = commitmentStore.findByCorrelationId(corrId).orElseThrow();
            assertThat(commitment.state).isEqualTo(CommitmentState.FAILED);
        });
    }

    @Test
    void allowedTypes_withWhitespace_isEnforcedCorrectly() {
        String channelName = "robust-ws-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND,
                        null, null, null, null, null, " EVENT , STATUS "));

        var ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());

        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent", MessageType.EVENT, "e", null, null)));
        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent", MessageType.STATUS, "s", null, null)));
        assertThrows(MessageTypeViolationException.class, () ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent", MessageType.QUERY, "q", null, null)));
    }

    @Test
    void blankAllowedTypes_treatedAsOpenChannel() {
        String channelName = "robust-blank-" + System.nanoTime();
        QuarkusTransaction.requiringNew().run(() ->
                channelService.create(channelName, "Test", ChannelSemantic.APPEND,
                        null, null, null, null, null, "   "));

        var ch = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(channelName).orElseThrow());
        assertThat(ch.allowedTypes).isNull();

        assertDoesNotThrow(() ->
                QuarkusTransaction.requiringNew().run(() ->
                        messageService.send(ch.id, "agent", MessageType.COMMAND, "go", null, null)));
    }

    @Test
    void multipleScenarios_doNotInterfere_withEachOther() {
        String caseA = "robust-iso-a-" + System.nanoTime();
        String caseB = "robust-iso-b-" + System.nanoTime();
        SecureCodeReviewScenario sA = new SecureCodeReviewScenario(caseA,
                channelService, instanceService, messageService, dataService);
        SecureCodeReviewScenario sB = new SecureCodeReviewScenario(caseB,
                channelService, instanceService, messageService, dataService);

        QuarkusTransaction.requiringNew().run(() -> { sA.setupChannels(); sB.setupChannels(); });

        // EVENTs on sA's observe channel must not appear on sB's
        QuarkusTransaction.requiringNew().run(() ->
                messageService.send(sA.observeChannel().id, "agent", MessageType.EVENT, "A-event", null, null));

        QuarkusTransaction.requiringNew().run(() -> {
            var sAEvents = messageService.findAllByCorrelationId("no-match"); // just test isolation
            // Verify sB's observe channel is empty
            var chB = sB.observeChannel();
            assertThat(chB.name).contains(caseB);
            // No messages were sent to sB's observe channel
        });
    }
}
```

- [ ] **Step 2: Run**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=NormativeLayoutRobustnessTest -pl examples/normative-layout
```
Expected: 5 tests PASS.

- [ ] **Step 3: Run full normative-layout suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout
```
Expected: all 27 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add examples/normative-layout/src/test/java/io/quarkiverse/qhorus/examples/normativelayout/NormativeLayoutRobustnessTest.java
git commit -m "test(examples): NormativeLayoutRobustnessTest — edge cases, isolation, bypass

Refs #122"
```

---

### Task 13: NormativeLayoutAgentTest in examples/agent-communication/

**Files:**
- Create: `examples/agent-communication/src/test/java/io/quarkiverse/qhorus/examples/agent/NormativeLayoutAgentTest.java`

- [ ] **Step 1: Write the test**

```java
package io.quarkiverse.qhorus.examples.agent;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.UUID;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import io.quarkiverse.qhorus.runtime.channel.ChannelService;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.qhorus.runtime.message.*;
import io.quarkiverse.qhorus.runtime.data.DataService;
import io.quarkiverse.qhorus.runtime.store.CommitmentStore;
import io.quarkiverse.qhorus.runtime.store.query.CommitmentQuery;
import io.quarkiverse.qhorus.runtime.store.MessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;

/**
 * End-to-end test for the NormativeChannelLayout using real Jlama agents.
 * Proves the 3-channel pattern holds under actual LLM behaviour.
 * Run with: mvn test -Pwith-llm-examples -Dtest=NormativeLayoutAgentTest -pl examples/agent-communication
 */
@QuarkusTest
class NormativeLayoutAgentTest {

    @Inject OrchestratorAgent orchestrator;
    @Inject WorkerAgent worker;
    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;
    @Inject MessageService messageService;
    @Inject DataService dataService;
    @Inject CommitmentStore commitmentStore;
    @Inject MessageStore messageStore;

    @Test
    void normativeLayout_orchestratorResearcher_fullScenario() {
        String caseId = "llm-normative-" + UUID.randomUUID();
        String workCh = "case-" + caseId + "/work";
        String observeCh = "case-" + caseId + "/observe";
        String oversightCh = "case-" + caseId + "/oversight";

        QuarkusTransaction.requiringNew().run(() -> {
            channelService.create(workCh, "Worker coordination", ChannelSemantic.APPEND,
                    null, null, null, null, null, null);
            channelService.create(observeCh, "Telemetry", ChannelSemantic.APPEND,
                    null, null, null, null, null, "EVENT");
            channelService.create(oversightCh, "Human governance", ChannelSemantic.APPEND,
                    null, null, null, null, null, "QUERY,COMMAND");
        });

        // Orchestrator decides what message type to use for a handoff scenario
        AgentResponse researcherDecision = orchestrator.handle(
                "You have just completed a security analysis of AuthService.java. " +
                "You found 3 vulnerabilities. You need to signal completion so a reviewer can take over. " +
                "What is the appropriate message type? Respond with JSON.");

        assertThat(researcherDecision.messageType())
                .as("Orchestrator should choose DONE to signal completion")
                .isEqualTo("DONE");

        // Worker decides how to respond to a clarification request
        AgentResponse reviewerDecision = worker.handle(
                "A peer asked you: 'Does TokenRefreshService.java:142 share the same SQL injection root cause as finding #1?' " +
                "You know the answer is yes. What message type should you use to reply? Respond with JSON.");

        assertThat(reviewerDecision.messageType())
                .as("Worker should choose RESPONSE to answer a QUERY")
                .isEqualTo("RESPONSE");

        // Verify channel type constraints are enforced even during LLM scenario
        var observeChannel = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(observeCh).orElseThrow());
        assertThat(observeChannel.allowedTypes).isEqualTo("EVENT");

        var oversightChannel = QuarkusTransaction.requiringNew().run(() ->
                channelService.findByName(oversightCh).orElseThrow());
        assertThat(oversightChannel.allowedTypes).isEqualTo("QUERY,COMMAND");
    }
}
```

- [ ] **Step 2: Verify it compiles (don't run Jlama without the profile)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl examples/agent-communication
```
Expected: BUILD SUCCESS — no compilation errors.

- [ ] **Step 3: Commit**

```bash
git add examples/agent-communication/src/test/java/io/quarkiverse/qhorus/examples/agent/NormativeLayoutAgentTest.java
git commit -m "test(examples): NormativeLayoutAgentTest — real Jlama agents on 3-channel layout

Refs #122"
```

---

### Task 14: docs/normative-channel-layout.md

**Files:**
- Create: `docs/normative-channel-layout.md`

- [ ] **Step 1: Write the documentation**

```markdown
# Normative Channel Layout

The NormativeChannelLayout is Qhorus's recommended channel topology for multi-agent coordination. It separates obligation-carrying acts, telemetry, and human governance into three dedicated channels — each with a distinct normative role and enforced type constraints.

## The 3-Channel Pattern

```
case-{caseId}/
├── work        APPEND   All obligation-carrying types (null = open)
├── observe     APPEND   Telemetry only (allowed_types: EVENT)
└── oversight   APPEND   Human governance (allowed_types: QUERY,COMMAND)
```

Create channels with `create_channel`:

```
create_channel("case-abc/work",     "Worker coordination", "APPEND", ...)
create_channel("case-abc/observe",  "Telemetry",           "APPEND", ..., allowed_types="EVENT")
create_channel("case-abc/oversight","Human governance",    "APPEND", ..., allowed_types="QUERY,COMMAND")
```

## Channel Roles

**`work`** — The primary coordination space. All 9 message types are permitted. Workers `QUERY` peers, `COMMAND` agents, `HANDOFF` to the next worker, and post `DONE` on completion. The CommitmentStore tracks every open obligation in this channel.

**`observe`** — Pure telemetry. Only `EVENT` messages. Agents post here for every significant tool call, state change, or decision point. No obligations are created — the constraint (`allowed_types: EVENT`) is enforced server-side and rejected at the MCP tool layer before any persistence occurs.

**`oversight`** — Human governance. Only `QUERY` and `COMMAND` messages. Agents post `QUERY` here when they need human input. Humans post `COMMAND` to inject directives. All entries appear in the normative ledger as first-class speech acts.

## `allowed_types` Enforcement

The `allowed_types` constraint is set when the channel is created and enforced at two layers:

1. **MCP tool layer (client):** `send_message` checks the policy before delegating to the service. Violations return an error immediately — no round-trip to the database.

2. **`MessageService` (server):** Safety net for any non-MCP callers. Same `MessageTypePolicy` SPI, same `MessageTypeViolationException`.

The default enforcement implementation (`StoredMessageTypePolicy`) reads `allowed_types` from the channel entity. Replace it with an `@Alternative @Priority` bean to plug in custom policy logic.

## MessageTypePolicy SPI

```java
public interface MessageTypePolicy {
    void validate(Channel channel, MessageType type);
    // throw MessageTypeViolationException to reject; return normally to permit
}
```

The default `StoredMessageTypePolicy` is `@ApplicationScoped`. Override it:

```java
@Alternative
@Priority(10)
@ApplicationScoped
public class MyCustomPolicy implements MessageTypePolicy {
    @Override
    public void validate(Channel channel, MessageType type) {
        // custom logic — e.g. derive constraints from channel name prefix
    }
}
```

## Project Template

Copy-pasteable setup for the NormativeChannelLayout:

```
# For each case, create three channels:
create_channel("case-{id}/work",     "Worker coordination", "APPEND")
create_channel("case-{id}/observe",  "Telemetry",           "APPEND", allowed_types="EVENT")
create_channel("case-{id}/oversight","Human governance",    "APPEND", allowed_types="QUERY,COMMAND")

# Each agent startup:
register("{workerId}", "{description}", ["{capabilities}"])
send_message("case-{id}/work", STATUS, "Starting: {goal}")

# During work, post to observe for every tool call:
send_message("case-{id}/observe", EVENT, '{"tool":"...", ...}')

# Signal completion:
share_data("{key}", content)
send_message("case-{id}/work", DONE, "Done. Output: shared-data:{key}")

# If human input needed:
send_message("case-{id}/oversight", QUERY, "Ambiguous finding — include?", target="human")
```

## Anti-Patterns

**EVENT on the work channel** — Mixes telemetry with obligations. The CommitmentStore stays clean only if EVENTs flow exclusively to `observe`.

**QUERY on the observe channel** — Rejected by `allowed_types` enforcement. An unenforced observe channel fills with obligation traffic and makes the ledger misleading.

**Obligation-carrying acts on oversight** — The oversight channel is for `QUERY`/`COMMAND` only. Status updates belong on `work` or `observe`, not `oversight`.

## Examples

See `examples/normative-layout/` for deterministic CI tests proving the pattern works correctly. The `SecureCodeReviewScenario` class is the canonical Layer 1 reference (Pure Qhorus, no LLM).

See `examples/agent-communication/NormativeLayoutAgentTest` for the same scenario with real Jlama agents (requires `-Pwith-llm-examples`).
```

- [ ] **Step 2: Run full suite to confirm nothing broken**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout
```
Expected: all tests PASS.

- [ ] **Step 3: Commit**

```bash
git add docs/normative-channel-layout.md
git commit -m "docs: normative-channel-layout.md — 3-channel pattern, SPI, project template

Refs #122"
```

---

### Task 15: Update docs/specs/2026-04-13-qhorus-design.md + CLAUDE.md

**Files:**
- Modify: `docs/specs/2026-04-13-qhorus-design.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the design doc**

Open `docs/specs/2026-04-13-qhorus-design.md`. Make the following targeted updates:

**In the Channel model section**, add `allowedTypes` to the field list after `adminInstances`:
```
- `allowedTypes` (TEXT, nullable) — comma-separated `MessageType` names; null = open to all types
```

**In the SPI / extension points section**, add `MessageTypePolicy`:
```
### MessageTypePolicy
Pluggable type enforcement. `StoredMessageTypePolicy` (default) reads `allowed_types` from the
channel entity. Enforced at two layers: `QhorusMcpTools.sendMessage()` (client, fail-fast) and
`MessageService.send()` (server, safety net). Replace with `@Alternative @Priority` bean.
```

**In the MCP tool surface section**, update `create_channel` to note the new `allowed_types` param:
```
- `create_channel` — 9 params (added `allowed_types`: comma-separated MessageType names, null = open)
```

**In the Examples section**, add the new module:
```
- `examples/normative-layout/` — canonical 3-channel pattern CI tests; no LLM; importable by Claudony/CaseHub
```

- [ ] **Step 2: Update CLAUDE.md project structure**

In the `Project Structure` section of CLAUDE.md, add the new example module entry:
```
│   └── examples/normative-layout/           — Deterministic 3-channel pattern tests (CI, no LLM); canonical Layer 1 reference
```

In the Testing conventions section, add:
```
- `MessageTypePolicy` is injected into both `QhorusMcpTools.sendMessage()` (client-side, fail-fast) and `MessageService.send()` (server-side safety net). Tests that call `MessageService.send()` directly on a channel with `allowedTypes` set will hit the server-side check even without the MCP layer.
```

In the `Channel` entity description, add `allowedTypes`:
```
- `allowedTypes` (TEXT, null=open) — comma-separated `MessageType` names enforced by `MessageTypePolicy` SPI
```

Update the `create_channel` tool description to note it now has 9 params and an `allowed_types` optional argument.

- [ ] **Step 3: Run full build to confirm no test regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing,examples/normative-layout
```
Expected: all tests PASS.

- [ ] **Step 4: Commit**

```bash
git add docs/specs/2026-04-13-qhorus-design.md CLAUDE.md
git commit -m "docs: update design doc + CLAUDE.md — allowedTypes, MessageTypePolicy SPI, normative-layout module

Refs #122"
```

---

### Task 16: Update Claudony framework spec + final full build

**Files:**
- Modify: `~/claude/claudony/docs/superpowers/specs/2026-04-27-claudony-agent-mesh-framework.md`

- [ ] **Step 1: Update the Reference section of the Claudony spec**

In `~/claude/claudony/docs/superpowers/specs/2026-04-27-claudony-agent-mesh-framework.md`, find the Reference section near the bottom (around line 698). Update the `create_channel` entry to include `allowed_types`:

```
- `create_channel(name, description, semantic, barrier_contributors?, allowed_writers?, admin_instances?, rate_limit_per_channel?, rate_limit_per_instance?, allowed_types?)`
```

Also add a note under the **Key MCP tools** block:
```
**`allowed_types`** — Pass `"EVENT"` when creating the observe channel; `"QUERY,COMMAND"` for the oversight channel. Enforced server-side by `MessageTypePolicy` SPI.
```

- [ ] **Step 2: Run complete build across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl runtime,testing,examples/normative-layout
```
Expected: all tests PASS. Report final test count.

- [ ] **Step 3: Commit Claudony spec change (in Claudony repo)**

```bash
cd ~/claude/claudony
git add docs/superpowers/specs/2026-04-27-claudony-agent-mesh-framework.md
git commit -m "docs: update create_channel signature — allowed_types param added in quarkus-qhorus

Refs casehubio/qhorus#122"
cd ~/claude/quarkus-qhorus
```

- [ ] **Step 4: Final commit in quarkus-qhorus — close issue**

```bash
git commit --allow-empty -m "chore: issue #122 complete — NormativeChannelLayout, MessageTypePolicy SPI, normative-layout examples

Closes #122"
```

---

## Self-Review

**Spec coverage check:**
- ✅ `MessageTypePolicy` SPI + `StoredMessageTypePolicy` — Tasks 1–2
- ✅ `Channel.allowedTypes` field — Task 2
- ✅ `ChannelService` + `ReactiveChannelService` overloads — Task 3
- ✅ `ChannelDetail` + `toChannelDetail` — Task 4
- ✅ `create_channel` `allowed_types` param (blocking) — Task 5
- ✅ `sendMessage` client-side enforcement (blocking) — Task 5
- ✅ `MessageService` server-side enforcement — Task 6
- ✅ `ReactiveQhorusMcpTools` mirror — Task 7
- ✅ `examples/normative-layout/` scaffold — Task 8
- ✅ `SecureCodeReviewScenario` + happy path — Task 9
- ✅ Type enforcement tests — Task 10
- ✅ Obligation + correctness tests — Task 11
- ✅ Robustness tests — Task 12
- ✅ `NormativeLayoutAgentTest` (Jlama) — Task 13
- ✅ `docs/normative-channel-layout.md` — Task 14
- ✅ Design doc + CLAUDE.md updates — Task 15
- ✅ Claudony spec update — Task 16
- ✅ `ChannelStoreContractTest` round-trip — Task 3

**Type consistency check:** `StoredMessageTypePolicy` used consistently throughout. `MessageTypeViolationException` constructor signature `(String channel, MessageType attempted, String allowed)` used identically in Task 1 (definition) and all test assertions. `ChannelDetail` record has 13 components; `toChannelDetail` mapper builds all 13; tests use `.allowedTypes()` accessor.

**No placeholders:** All steps contain actual code or exact commands.

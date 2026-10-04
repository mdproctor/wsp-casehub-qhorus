# AgentProvider Bridge + MeshApi Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #465 — qhorus-mesh
**Issue group:** #465

**Goal:** Bridge AgentProvider-managed agents (Claude, OpenAI, Gemini, langchain4j) into qhorus channels as first-class participants, and migrate MeshMcpTools from @Tool to @McpDomain.

**Architecture:** New `agent-bridge/` module implements `ChannelBackend` (AT_LEAST_ONCE) for delivering channel messages to `AgentSession.query()`, with sender-based loop guard, commitment-aware speech-act mapping, and cache-aware prompt structuring. MeshApi interface with `@McpDomain` replaces `@Tool` annotations on MeshMcpTools.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, casehub-platform-agent-api 0.2-SNAPSHOT

## Global Constraints

- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- After API visibility changes, always run `mvn install` from the project root
- Named `qhorus` datasource for tests (`quarkus.datasource.qhorus.*`)
- All commits must reference an issue (`Refs #465`)
- Use `ide_insert_member` for new methods, `ide_replace_text_in_file` for mechanical substitutions
- CDI-free unit tests where possible; `@QuarkusTest` only for integration tests needing the full CDI container

---

## Batch 1: Agent Binding Record + Speech-Act Mapper

### Task 1: AgentChannelBinding record + SpeechActMapper

**Files:**
- Create: `agent-bridge/pom.xml`
- Create: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentChannelBinding.java`
- Create: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/SpeechActMapper.java`
- Modify: `pom.xml` (parent — add `<module>agent-bridge</module>`)
- Test: `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/SpeechActMapperTest.java`

**Interfaces:**
- Produces: `AgentChannelBinding` record (id, channelId, agentInstanceId, backendKey, agentBriefing, mcpServers, persistent, maxConcurrency, contextWindowSize, metadata, tenancyId); `SpeechActMapper.mapToDispatch(OutboundMessage inbound, String agentInstanceId, String responseText, AgentEvent.InvocationComplete stats)` → `MessageDispatch`; `SpeechActMapper.mapToolStatus(OutboundMessage inbound, String agentInstanceId, AgentEvent.ToolCallComplete tool)` → `MessageDispatch`; `SpeechActMapper.mapFailure(OutboundMessage inbound, String agentInstanceId, Throwable error)` → `MessageDispatch`

- [ ] **Step 1: Create agent-bridge/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-qhorus-agent-bridge</artifactId>
  <name>Quarkus Qhorus - Agent Provider Bridge</name>
  <description>Optional bridge — routes channel messages to AgentProvider-managed agents via AT_LEAST_ONCE delivery. Activates by classpath presence.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-agent-api</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>

    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>

    <dependency>
      <groupId>org.jboss.logging</groupId>
      <artifactId>jboss-logging</artifactId>
      <scope>provided</scope>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-persistence-memory</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform</artifactId>
      <version>0.2-SNAPSHOT</version>
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
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5-mockito</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 2: Add module to parent pom**

In root `pom.xml`, add `<module>agent-bridge</module>` after `testing`, before `mesh`:

```xml
    <module>testing</module>
    <module>agent-bridge</module>
    <module>mesh</module>
```

- [ ] **Step 3: Create directory structure**

```bash
mkdir -p agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge
mkdir -p agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge
```

- [ ] **Step 4: Write the failing test for SpeechActMapper**

Create `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/SpeechActMapperTest.java`:

```java
package io.casehub.qhorus.agent.bridge;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.agent.AgentEvent;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class SpeechActMapperTest {

    private static final UUID CHANNEL_ID = UUID.randomUUID();
    private static final String AGENT_ID = "code-reviewer-1";
    private static final String CORRELATION_ID = UUID.randomUUID().toString();

    private OutboundMessage inbound(MessageType type) {
        return new OutboundMessage(UUID.randomUUID(), null, "requester",
                type, "Review this code", null, CORRELATION_ID, null,
                ActorType.AGENT, List.of(), AGENT_ID, null);
    }

    @Test
    void mapToDispatch_commandCommitment_producesResponseOnly() {
        var stats = new AgentEvent.InvocationComplete(100, 50, 10, 0, 0, 0.01, 2000L, 1800L, "sess-1", 1, false);
        MessageDispatch dispatch = SpeechActMapper.mapToDispatch(
                CHANNEL_ID, inbound(MessageType.COMMAND), AGENT_ID, "LGTM", stats);

        assertThat(dispatch.type()).isEqualTo(MessageType.RESPONSE);
        assertThat(dispatch.content()).isEqualTo("LGTM");
        assertThat(dispatch.sender()).isEqualTo(AGENT_ID);
        assertThat(dispatch.channelId()).isEqualTo(CHANNEL_ID);
        assertThat(dispatch.correlationId()).isEqualTo(CORRELATION_ID);
        assertThat(dispatch.inReplyTo()).isNotNull();
    }

    @Test
    void mapToDispatch_queryCommitment_producesResponseOnly() {
        var stats = new AgentEvent.InvocationComplete(80, 40, 0, 0, 0, null, 1500L, 1200L, "sess-1", 1, false);
        MessageDispatch dispatch = SpeechActMapper.mapToDispatch(
                CHANNEL_ID, inbound(MessageType.QUERY), AGENT_ID, "The answer is 42", stats);

        assertThat(dispatch.type()).isEqualTo(MessageType.RESPONSE);
        assertThat(dispatch.content()).isEqualTo("The answer is 42");
    }

    @Test
    void mapToolStatus_producesStatusMessage() {
        var tool = new AgentEvent.ToolCallComplete(0, "call-1", "grep", "{\"pattern\":\"TODO\"}");
        MessageDispatch dispatch = SpeechActMapper.mapToolStatus(
                CHANNEL_ID, inbound(MessageType.COMMAND), AGENT_ID, tool);

        assertThat(dispatch.type()).isEqualTo(MessageType.STATUS);
        assertThat(dispatch.sender()).isEqualTo(AGENT_ID);
        assertThat(dispatch.content()).contains("grep");
    }

    @Test
    void mapFailure_producesFailureMessage() {
        MessageDispatch dispatch = SpeechActMapper.mapFailure(
                CHANNEL_ID, inbound(MessageType.COMMAND), AGENT_ID,
                new RuntimeException("API timeout"));

        assertThat(dispatch.type()).isEqualTo(MessageType.FAILURE);
        assertThat(dispatch.content()).contains("API timeout");
        assertThat(dispatch.correlationId()).isEqualTo(CORRELATION_ID);
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SpeechActMapperTest -pl agent-bridge -DfailIfNoTests=false`
Expected: FAIL — `SpeechActMapper` does not exist.

- [ ] **Step 6: Create AgentChannelBinding record**

Create `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentChannelBinding.java`:

```java
package io.casehub.qhorus.agent.bridge;

import java.util.List;
import java.util.Map;
import java.util.UUID;

public record AgentChannelBinding(
        UUID id,
        UUID channelId,
        String agentInstanceId,
        String backendKey,
        String agentBriefing,
        List<String> mcpServers,
        boolean persistent,
        int maxConcurrency,
        int contextWindowSize,
        Map<String, String> metadata,
        String tenancyId) {

    public AgentChannelBinding {
        mcpServers = mcpServers != null ? List.copyOf(mcpServers) : List.of();
        metadata = metadata != null ? Map.copyOf(metadata) : null;
        if (persistent && maxConcurrency != 1) {
            throw new IllegalArgumentException(
                    "Persistent sessions require maxConcurrency=1, got " + maxConcurrency);
        }
        if (maxConcurrency < 1) {
            throw new IllegalArgumentException(
                    "maxConcurrency must be >= 1, got " + maxConcurrency);
        }
        if (contextWindowSize < 0) {
            throw new IllegalArgumentException(
                    "contextWindowSize must be >= 0, got " + contextWindowSize);
        }
    }

    public static Builder builder(UUID channelId, String agentInstanceId, String backendKey) {
        return new Builder(channelId, agentInstanceId, backendKey);
    }

    public static final class Builder {
        private final UUID channelId;
        private final String agentInstanceId;
        private final String backendKey;
        private UUID id = UUID.randomUUID();
        private String agentBriefing = "";
        private List<String> mcpServers = List.of();
        private boolean persistent = true;
        private int maxConcurrency = 1;
        private int contextWindowSize = 20;
        private Map<String, String> metadata;
        private String tenancyId;

        Builder(UUID channelId, String agentInstanceId, String backendKey) {
            this.channelId = channelId;
            this.agentInstanceId = agentInstanceId;
            this.backendKey = backendKey;
        }

        public Builder id(UUID v) { this.id = v; return this; }
        public Builder agentBriefing(String v) { this.agentBriefing = v; return this; }
        public Builder mcpServers(List<String> v) { this.mcpServers = v; return this; }
        public Builder persistent(boolean v) { this.persistent = v; return this; }
        public Builder maxConcurrency(int v) { this.maxConcurrency = v; return this; }
        public Builder contextWindowSize(int v) { this.contextWindowSize = v; return this; }
        public Builder metadata(Map<String, String> v) { this.metadata = v; return this; }
        public Builder tenancyId(String v) { this.tenancyId = v; return this; }

        public AgentChannelBinding build() {
            return new AgentChannelBinding(id, channelId, agentInstanceId, backendKey,
                    agentBriefing, mcpServers, persistent, maxConcurrency,
                    contextWindowSize, metadata, tenancyId);
        }
    }
}
```

- [ ] **Step 7: Create SpeechActMapper**

Create `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/SpeechActMapper.java`:

```java
package io.casehub.qhorus.agent.bridge;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;

import java.util.UUID;

public final class SpeechActMapper {

    private SpeechActMapper() {}

    public static MessageDispatch mapToDispatch(UUID channelId, OutboundMessage inbound,
                                                 String agentInstanceId, String responseText,
                                                 AgentEvent.InvocationComplete stats) {
        String telemetry = formatTelemetry(stats);
        return MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentInstanceId)
                .type(MessageType.RESPONSE)
                .content(responseText)
                .correlationId(inbound.correlationId())
                .inReplyTo(inbound.sequenceId())
                .actorType(ActorType.AGENT)
                .telemetry(telemetry)
                .build();
    }

    public static MessageDispatch mapToolStatus(UUID channelId, OutboundMessage inbound,
                                                 String agentInstanceId,
                                                 AgentEvent.ToolCallComplete tool) {
        String content = "Tool: " + tool.name() + " (id=" + tool.id() + ")";
        return MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentInstanceId)
                .type(MessageType.STATUS)
                .content(content)
                .correlationId(inbound.correlationId())
                .actorType(ActorType.AGENT)
                .build();
    }

    public static MessageDispatch mapFailure(UUID channelId, OutboundMessage inbound,
                                              String agentInstanceId, Throwable error) {
        String content = "Agent invocation failed: " + error.getMessage();
        return MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentInstanceId)
                .type(MessageType.FAILURE)
                .content(content)
                .correlationId(inbound.correlationId())
                .inReplyTo(inbound.sequenceId())
                .actorType(ActorType.AGENT)
                .build();
    }

    private static String formatTelemetry(AgentEvent.InvocationComplete stats) {
        if (stats == null) return null;
        return "{\"input_tokens\":" + stats.inputTokens()
                + ",\"output_tokens\":" + stats.outputTokens()
                + ",\"thinking_tokens\":" + stats.thinkingTokens()
                + ",\"duration_ms\":" + stats.durationMs()
                + (stats.totalCostUsd() != null ? ",\"total_cost_usd\":" + stats.totalCostUsd() : "")
                + "}";
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SpeechActMapperTest -pl agent-bridge`
Expected: PASS (4 tests)

- [ ] **Step 9: Run full build to verify module integration**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add agent-bridge/ pom.xml
git commit -m "feat(#465): scaffold agent-bridge module with AgentChannelBinding + SpeechActMapper

Refs #465"
```

---

## Batch 2: AgentProviderBackend — ChannelBackend with Delivery

### Task 2: AgentProviderBackend core + loop guard + async delivery

**Files:**
- Create: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentProviderBackend.java`
- Create: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentInvocationRunner.java`
- Test: `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/AgentProviderBackendTest.java`

**Interfaces:**
- Consumes: `AgentChannelBinding` (Task 1); `SpeechActMapper` (Task 1); `ChannelBackend` SPI; `AgentBackend.invoke(AgentSessionConfig)` and `AgentSession.query(String)` from `casehub-platform-agent-api`; `MessageDispatcher.dispatch(MessageDispatch)` from `casehub-qhorus-api`
- Produces: `AgentProviderBackend` implementing `ChannelBackend` with `AT_LEAST_ONCE` guarantee; `backendId()` → `"agent-bridge-" + binding.agentInstanceId()`; `postTracked(ChannelRef, OutboundMessage)` → accepts COMMAND/QUERY/PROPOSE, skips others, sender-based loop guard, async invocation via virtual thread; `AgentInvocationRunner` handles the actual agent call (persistent session `query()` or ephemeral `invoke()`) and dispatches responses via `MessageDispatcher`

- [ ] **Step 1: Write the failing test**

Create `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/AgentProviderBackendTest.java`:

```java
package io.casehub.qhorus.agent.bridge;

import io.casehub.platform.agent.AgentBackend;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.agent.AgentSessionInit;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.gateway.PostResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.message.MessageType;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class AgentProviderBackendTest {

    private static final UUID CHANNEL_ID = UUID.randomUUID();
    private static final String AGENT_ID = "code-reviewer";
    private static final String BACKEND_KEY = "claude";

    private final CopyOnWriteArrayList<MessageDispatch> dispatched = new CopyOnWriteArrayList<>();
    private final MessageDispatcher dispatcher = dispatch -> {
        dispatched.add(dispatch);
        return null;
    };

    private CountDownLatch invocationLatch;
    private AgentBackend stubBackend;
    private AgentProviderBackend backend;
    private AgentChannelBinding binding;

    @BeforeEach
    void setUp() {
        dispatched.clear();
        invocationLatch = new CountDownLatch(1);
        stubBackend = new AgentBackend() {
            @Override public String key() { return BACKEND_KEY; }
            @Override public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                return Multi.createFrom().items(
                        new AgentEvent.TextDelta("LGTM"),
                        new AgentEvent.InvocationComplete(100, 50, 0, 0, 0, null, 1000L, 900L, "s1", 1, false)
                ).onCompletion().invoke(invocationLatch::countDown);
            }
            @Override public AgentSession openSession(AgentSessionInit init) { return null; }
        };
        binding = AgentChannelBinding.builder(CHANNEL_ID, AGENT_ID, BACKEND_KEY)
                .persistent(false).maxConcurrency(3).build();
        backend = new AgentProviderBackend(binding, stubBackend, dispatcher);
    }

    private OutboundMessage message(MessageType type, String sender) {
        return new OutboundMessage(UUID.randomUUID(), 1L, sender, type,
                "Review this", null, UUID.randomUUID().toString(), null,
                ActorType.AGENT, List.of(), AGENT_ID, null);
    }

    @Test
    void postTracked_commandTriggers_invocation() throws Exception {
        PostResult result = backend.postTracked(
                new ChannelRef(CHANNEL_ID, "review-channel"),
                message(MessageType.COMMAND, "requester"));
        assertThat(result).isEqualTo(PostResult.ALL_DELIVERED);
        invocationLatch.await(5, TimeUnit.SECONDS);
        assertThat(dispatched).anyMatch(d -> d.type() == MessageType.RESPONSE);
    }

    @Test
    void postTracked_statusSkips_invocation() throws Exception {
        backend.postTracked(
                new ChannelRef(CHANNEL_ID, "review-channel"),
                message(MessageType.STATUS, "someone"));
        Thread.sleep(200);
        assertThat(dispatched).isEmpty();
    }

    @Test
    void postTracked_senderLoopGuard_skips() throws Exception {
        backend.postTracked(
                new ChannelRef(CHANNEL_ID, "review-channel"),
                message(MessageType.COMMAND, AGENT_ID));
        Thread.sleep(200);
        assertThat(dispatched).isEmpty();
    }

    @Test
    void postTracked_targetMismatch_skips() throws Exception {
        var msg = new OutboundMessage(UUID.randomUUID(), 1L, "requester",
                MessageType.COMMAND, "content", null, UUID.randomUUID().toString(), null,
                ActorType.AGENT, List.of(), "other-agent", null);
        backend.postTracked(new ChannelRef(CHANNEL_ID, "ch"), msg);
        Thread.sleep(200);
        assertThat(dispatched).isEmpty();
    }

    @Test
    void deliveryGuarantee_isAtLeastOnce() {
        assertThat(backend.deliveryGuarantee())
                .isEqualTo(io.casehub.qhorus.api.gateway.DeliveryGuarantee.AT_LEAST_ONCE);
    }

    @Test
    void backendId_includesAgentInstanceId() {
        assertThat(backend.backendId()).isEqualTo("agent-bridge-code-reviewer");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AgentProviderBackendTest -pl agent-bridge -DfailIfNoTests=false`
Expected: FAIL — `AgentProviderBackend` does not exist.

- [ ] **Step 3: Create AgentInvocationRunner**

Create `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentInvocationRunner.java`:

```java
package io.casehub.qhorus.agent.bridge;

import io.casehub.platform.agent.AgentBackend;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import org.jboss.logging.Logger;

import java.util.UUID;
import java.util.concurrent.Semaphore;

public class AgentInvocationRunner implements Runnable {

    private static final Logger LOG = Logger.getLogger(AgentInvocationRunner.class);

    private final UUID channelId;
    private final OutboundMessage inbound;
    private final AgentChannelBinding binding;
    private final AgentBackend agentBackend;
    private final AgentSession session;
    private final MessageDispatcher dispatcher;
    private final Semaphore concurrencyGuard;

    public AgentInvocationRunner(UUID channelId, OutboundMessage inbound,
                                  AgentChannelBinding binding, AgentBackend agentBackend,
                                  AgentSession session, MessageDispatcher dispatcher,
                                  Semaphore concurrencyGuard) {
        this.channelId = channelId;
        this.inbound = inbound;
        this.binding = binding;
        this.agentBackend = agentBackend;
        this.session = session;
        this.dispatcher = dispatcher;
        this.concurrencyGuard = concurrencyGuard;
    }

    @Override
    public void run() {
        try {
            concurrencyGuard.acquire();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return;
        }
        try {
            StringBuilder text = new StringBuilder();
            AgentEvent.InvocationComplete stats = null;

            var events = session != null
                    ? session.query(inbound.content())
                    : agentBackend.invoke(AgentSessionConfig.of(
                            binding.agentBriefing(), inbound.content()));

            for (AgentEvent event : events.subscribe().asIterable()) {
                switch (event) {
                    case AgentEvent.TextDelta td -> text.append(td.text());
                    case AgentEvent.ToolCallComplete tc -> {
                        MessageDispatch statusMsg = SpeechActMapper.mapToolStatus(
                                channelId, inbound, binding.agentInstanceId(), tc);
                        dispatcher.dispatch(statusMsg);
                    }
                    case AgentEvent.InvocationComplete ic -> stats = ic;
                    case AgentEvent.ThinkingDelta ignored -> {}
                    case AgentEvent.ToolCallDelta ignored -> {}
                    case AgentEvent.ToolResult ignored -> {}
                }
            }

            if (stats != null && stats.isError()) {
                MessageDispatch failure = SpeechActMapper.mapFailure(
                        channelId, inbound, binding.agentInstanceId(),
                        new RuntimeException("Agent invocation completed with error"));
                dispatcher.dispatch(failure);
            } else if (!text.isEmpty()) {
                MessageDispatch response = SpeechActMapper.mapToDispatch(
                        channelId, inbound, binding.agentInstanceId(),
                        text.toString(), stats);
                dispatcher.dispatch(response);
            }
        } catch (Exception e) {
            LOG.warnf(e, "Agent invocation failed for %s on channel %s",
                    binding.agentInstanceId(), channelId);
            MessageDispatch failure = SpeechActMapper.mapFailure(
                    channelId, inbound, binding.agentInstanceId(), e);
            dispatcher.dispatch(failure);
        } finally {
            concurrencyGuard.release();
        }
    }
}
```

- [ ] **Step 4: Create AgentProviderBackend**

Create `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentProviderBackend.java`:

```java
package io.casehub.qhorus.agent.bridge;

import io.casehub.platform.agent.AgentBackend;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.ChannelBackend;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.DeliveryGuarantee;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.gateway.PostResult;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.message.MessageType;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.Set;
import java.util.concurrent.Semaphore;

public class AgentProviderBackend implements ChannelBackend {

    private static final Logger LOG = Logger.getLogger(AgentProviderBackend.class);

    private static final Set<MessageType> INVOCATION_TYPES =
            Set.of(MessageType.COMMAND, MessageType.QUERY, MessageType.PROPOSE);

    private final AgentChannelBinding binding;
    private final AgentBackend agentBackend;
    private final MessageDispatcher dispatcher;
    private final Semaphore concurrencyGuard;
    private volatile AgentSession session;

    public AgentProviderBackend(AgentChannelBinding binding, AgentBackend agentBackend,
                                 MessageDispatcher dispatcher) {
        this.binding = binding;
        this.agentBackend = agentBackend;
        this.dispatcher = dispatcher;
        this.concurrencyGuard = new Semaphore(binding.maxConcurrency());
    }

    void setSession(AgentSession session) {
        this.session = session;
    }

    @Override
    public String backendId() {
        return "agent-bridge-" + binding.agentInstanceId();
    }

    @Override
    public ActorType actorType() {
        return ActorType.AGENT;
    }

    @Override
    public DeliveryGuarantee deliveryGuarantee() {
        return DeliveryGuarantee.AT_LEAST_ONCE;
    }

    @Override
    public void open(ChannelRef channel, Map<String, String> metadata) {}

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        postTracked(channel, message);
    }

    @Override
    public PostResult postTracked(ChannelRef channel, OutboundMessage message) {
        if (message.sender().equals(binding.agentInstanceId())) {
            return PostResult.ALL_DELIVERED;
        }

        if (!INVOCATION_TYPES.contains(message.type())) {
            return PostResult.ALL_DELIVERED;
        }

        String target = message.target();
        if (target != null && !target.isBlank()
                && !target.equals(binding.agentInstanceId())) {
            return PostResult.ALL_DELIVERED;
        }

        Thread.ofVirtual().name("agent-bridge-" + binding.agentInstanceId())
                .start(new AgentInvocationRunner(
                        channel.id(), message, binding, agentBackend,
                        session, dispatcher, concurrencyGuard));

        return PostResult.ALL_DELIVERED;
    }

    @Override
    public void close(ChannelRef channel) {
        if (session != null) {
            session.close();
            session = null;
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AgentProviderBackendTest -pl agent-bridge`
Expected: PASS (6 tests)

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add agent-bridge/
git commit -m "feat(#465): AgentProviderBackend — ChannelBackend with loop guard + async delivery

AT_LEAST_ONCE delivery, sender-based loop guard, target-based routing,
virtual thread async invocation, semaphore concurrency control.

Refs #465"
```

---

## Batch 3: Binding Lifecycle SPI

### Task 3: AgentBridgeService — create/update/destroy bindings

**Files:**
- Create: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentBridgeService.java`
- Test: `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/AgentBridgeServiceTest.java`

**Interfaces:**
- Consumes: `AgentProviderBackend` (Task 2); `AgentChannelBinding` (Task 1); `AgentBackend` via CDI `Instance<AgentBackend>`; `MessageDispatcher`; `BackendRegistry` (from `casehub-qhorus-api`) for registering/deregistering backends; `InstanceService` for agent instance registration
- Produces: `AgentBridgeService.createBinding(AgentChannelBinding)` → registers instance, opens session if persistent, registers ChannelBackend; `AgentBridgeService.destroyBinding(UUID bindingId)` → closes session, deregisters backend, deregisters instance; `AgentBridgeService.getBinding(UUID)` → `Optional<AgentChannelBinding>`; `AgentBridgeService.listBindings()` → `List<AgentChannelBinding>`

- [ ] **Step 1: Write the failing test**

Create `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/AgentBridgeServiceTest.java`:

```java
package io.casehub.qhorus.agent.bridge;

import io.casehub.platform.agent.AgentBackend;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.agent.AgentSessionInit;
import io.casehub.qhorus.api.gateway.BackendRegistry;
import io.casehub.qhorus.api.gateway.ChannelBackend;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AgentBridgeServiceTest {

    private static final UUID CHANNEL_ID = UUID.randomUUID();

    private final ConcurrentHashMap<String, ChannelBackend> registeredBackends = new ConcurrentHashMap<>();
    private final BackendRegistry backendRegistry = new BackendRegistry() {
        @Override public void register(UUID channelId, ChannelBackend backend) {
            registeredBackends.put(backend.backendId(), backend);
        }
        @Override public void deregister(UUID channelId, String backendId) {
            registeredBackends.remove(backendId);
        }
    };

    private final MessageDispatcher dispatcher = d -> null;
    private AgentBackend stubBackend;
    private AgentSession stubSession;
    private AgentBridgeService service;

    @BeforeEach
    void setUp() {
        registeredBackends.clear();
        stubSession = new AgentSession() {
            @Override public Multi<AgentEvent> query(String prompt) {
                return Multi.createFrom().items(new AgentEvent.TextDelta("ok"),
                        new AgentEvent.InvocationComplete(0,0,0,0,0,null,0,0,null,0,false));
            }
            @Override public Uni<Void> interrupt() { return Uni.createFrom().voidItem(); }
            @Override public void close(Duration maxWait) {}
        };
        stubBackend = new AgentBackend() {
            @Override public String key() { return "claude"; }
            @Override public Multi<AgentEvent> invoke(AgentSessionConfig c) {
                return Multi.createFrom().empty();
            }
            @Override public AgentSession openSession(AgentSessionInit init) {
                return stubSession;
            }
        };
        service = new AgentBridgeService(List.of(stubBackend), backendRegistry, dispatcher);
    }

    @Test
    void createBinding_registersBackend() {
        var binding = AgentChannelBinding.builder(CHANNEL_ID, "agent-1", "claude")
                .persistent(false).maxConcurrency(1).build();
        service.createBinding(binding);

        assertThat(registeredBackends).containsKey("agent-bridge-agent-1");
        assertThat(service.getBinding(binding.id())).isPresent();
    }

    @Test
    void createBinding_persistent_opensSession() {
        var binding = AgentChannelBinding.builder(CHANNEL_ID, "agent-2", "claude")
                .persistent(true).maxConcurrency(1).build();
        service.createBinding(binding);

        assertThat(registeredBackends).containsKey("agent-bridge-agent-2");
    }

    @Test
    void destroyBinding_deregistersBackend() {
        var binding = AgentChannelBinding.builder(CHANNEL_ID, "agent-3", "claude")
                .persistent(false).maxConcurrency(1).build();
        service.createBinding(binding);
        service.destroyBinding(binding.id());

        assertThat(registeredBackends).doesNotContainKey("agent-bridge-agent-3");
        assertThat(service.getBinding(binding.id())).isEmpty();
    }

    @Test
    void createBinding_unknownBackendKey_throws() {
        var binding = AgentChannelBinding.builder(CHANNEL_ID, "agent-4", "unknown-provider")
                .persistent(false).maxConcurrency(1).build();
        assertThatThrownBy(() -> service.createBinding(binding))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("unknown-provider");
    }

    @Test
    void listBindings_returnsAll() {
        var b1 = AgentChannelBinding.builder(CHANNEL_ID, "a1", "claude")
                .persistent(false).maxConcurrency(1).build();
        var b2 = AgentChannelBinding.builder(CHANNEL_ID, "a2", "claude")
                .persistent(false).maxConcurrency(1).build();
        service.createBinding(b1);
        service.createBinding(b2);

        assertThat(service.listBindings()).hasSize(2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AgentBridgeServiceTest -pl agent-bridge -DfailIfNoTests=false`
Expected: FAIL — `AgentBridgeService` does not exist.

- [ ] **Step 3: Create AgentBridgeService**

Create `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentBridgeService.java`:

```java
package io.casehub.qhorus.agent.bridge;

import io.casehub.platform.agent.AgentBackend;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionInit;
import io.casehub.qhorus.api.gateway.BackendRegistry;
import io.casehub.qhorus.api.message.MessageDispatcher;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class AgentBridgeService {

    private static final Logger LOG = Logger.getLogger(AgentBridgeService.class);

    private final Map<String, AgentBackend> backends;
    private final BackendRegistry backendRegistry;
    private final MessageDispatcher dispatcher;
    private final ConcurrentHashMap<UUID, ManagedBinding> managedBindings = new ConcurrentHashMap<>();

    record ManagedBinding(AgentChannelBinding binding, AgentProviderBackend backend,
                          AgentSession session) {}

    public AgentBridgeService(List<AgentBackend> backends,
                               BackendRegistry backendRegistry,
                               MessageDispatcher dispatcher) {
        this.backends = new ConcurrentHashMap<>();
        for (AgentBackend b : backends) {
            this.backends.put(b.key(), b);
        }
        this.backendRegistry = backendRegistry;
        this.dispatcher = dispatcher;
    }

    public void createBinding(AgentChannelBinding binding) {
        AgentBackend agentBackend = backends.get(binding.backendKey());
        if (agentBackend == null) {
            throw new IllegalArgumentException(
                    "No AgentBackend found for key: " + binding.backendKey()
                    + ". Available: " + backends.keySet());
        }

        var providerBackend = new AgentProviderBackend(binding, agentBackend, dispatcher);

        AgentSession session = null;
        if (binding.persistent()) {
            AgentSessionInit init = new AgentSessionInit(
                    binding.agentBriefing(), List.of(), null, null, null, null);
            session = agentBackend.openSession(init);
            providerBackend.setSession(session);
        }

        backendRegistry.register(binding.channelId(), providerBackend);
        managedBindings.put(binding.id(), new ManagedBinding(binding, providerBackend, session));

        LOG.infof("Agent binding created: %s on channel %s (backend=%s, persistent=%s)",
                binding.agentInstanceId(), binding.channelId(), binding.backendKey(),
                binding.persistent());
    }

    public void destroyBinding(UUID bindingId) {
        ManagedBinding managed = managedBindings.remove(bindingId);
        if (managed == null) {
            LOG.warnf("Binding not found for destroy: %s", bindingId);
            return;
        }
        backendRegistry.deregister(managed.binding.channelId(),
                managed.backend.backendId());
        if (managed.session != null) {
            managed.session.close();
        }
        LOG.infof("Agent binding destroyed: %s", managed.binding.agentInstanceId());
    }

    public Optional<AgentChannelBinding> getBinding(UUID bindingId) {
        ManagedBinding managed = managedBindings.get(bindingId);
        return managed != null ? Optional.of(managed.binding) : Optional.empty();
    }

    public List<AgentChannelBinding> listBindings() {
        return managedBindings.values().stream()
                .map(ManagedBinding::binding)
                .toList();
    }
}
```

- [ ] **Step 4: Check BackendRegistry interface exists**

Use `ide_find_class` to verify `BackendRegistry` exists in the API module and has `register(UUID, ChannelBackend)` and `deregister(UUID, String)` methods. If it doesn't exist, check `ChannelGateway` for the registration mechanism and adjust accordingly.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AgentBridgeServiceTest -pl agent-bridge`
Expected: PASS (5 tests)

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add agent-bridge/
git commit -m "feat(#465): AgentBridgeService — binding lifecycle create/destroy/list

Ephemeral ConcurrentHashMap storage, AgentBackend key resolution,
persistent session open/close, BackendRegistry integration.

Refs #465"
```

---

## Batch 4: MeshApi Migration

### Task 4: Migrate MeshMcpTools @Tool → MeshApi @McpDomain

**Files:**
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/MeshApi.java`
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/MeshRegistration.java`
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/MessageResult.java`
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/PeerInfo.java`
- Modify: `mesh/src/main/java/io/casehub/qhorus/mesh/MeshMcpTools.java` → rename to `MeshService.java`, implement `MeshApi`, change return types
- Modify: `mesh/src/test/java/io/casehub/qhorus/mesh/MeshMcpToolsTest.java` → rename to `MeshServiceTest.java`, update for new return types
- Modify: `mesh/pom.xml` — add `casehub-platform-api` dependency for `@McpDomain`

**Interfaces:**
- Consumes: `InstanceService`, `ChannelService`, `MessageDispatcher`, `MessageStore`, `InstanceStore` (all existing)
- Produces: `MeshApi` interface with `@McpDomain(value = "qhorus/mesh", app = "qhorus-mesh")`; `MeshService` implementation; typed return records `MeshRegistration`, `MessageResult`, `PeerInfo`

- [ ] **Step 1: Add casehub-platform-api dependency to mesh pom**

Add to `mesh/pom.xml` dependencies:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>
```

- [ ] **Step 2: Create return type records**

Create `mesh/src/main/java/io/casehub/qhorus/mesh/MeshRegistration.java`:

```java
package io.casehub.qhorus.mesh;

import java.util.UUID;

public record MeshRegistration(UUID id, String instanceId, String description) {}
```

Create `mesh/src/main/java/io/casehub/qhorus/mesh/MessageResult.java`:

```java
package io.casehub.qhorus.mesh;

import io.casehub.qhorus.api.message.MessageType;

public record MessageResult(Long messageId, String channel, MessageType type) {}
```

Create `mesh/src/main/java/io/casehub/qhorus/mesh/PeerInfo.java`:

```java
package io.casehub.qhorus.mesh;

import java.util.Map;

public record PeerInfo(String instanceId, String description, Map<String, String> metadata) {}
```

- [ ] **Step 3: Create MeshApi interface**

Create `mesh/src/main/java/io/casehub/qhorus/mesh/MeshApi.java`:

```java
package io.casehub.qhorus.mesh;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.qhorus.api.channel.ChannelDetail;

import java.util.List;
import java.util.Map;

@McpDomain(value = "qhorus/mesh", app = "qhorus-mesh",
           summary = "Mesh relay — register, discover peers, send messages")
public interface MeshApi {

    @PlatformMutation("Register a session with the mesh relay")
    MeshRegistration meshRegister(String instanceId, String description,
                                  Map<String, String> metadata);

    @PlatformMutation("Deregister a session from the mesh relay")
    String meshDeregister(String instanceId);

    @PlatformMutation("Send a message to a channel")
    MessageResult meshSendMessage(String channel, String sender,
                                   String type, String content);

    @PlatformQuery("Check messages in a channel")
    String meshCheckMessages(String channel, Long afterId);

    @PlatformMutation("Create a channel with metadata")
    String meshCreateChannel(String name, String metadataJson);

    @PlatformQuery("List channels filtered by metadata")
    String meshListChannels(String metadataKey, String metadataValue);

    @PlatformQuery("Discover peers by metadata")
    List<PeerInfo> meshDiscoverPeers(String metadataKey, String metadataValue);
}
```

- [ ] **Step 4: Rename MeshMcpTools → MeshService and implement MeshApi**

Use `ide_refactor_rename` to rename the class from `MeshService` to `MeshService`. Then modify the class to implement `MeshApi`, change return types, and remove `@Tool`/`@ToolArg` annotations. The business logic stays the same — only the annotations and return types change.

Key changes:
- Class declaration: `public class MeshService implements MeshApi`
- Remove all `@Tool` and `@ToolArg` annotations
- `mesh_register` → `meshRegister`, returns `MeshRegistration` instead of `String`
- `mesh_deregister` → `meshDeregister`
- `mesh_send_message` → `meshSendMessage`, returns `MessageResult`
- `mesh_discover_peers` → `meshDiscoverPeers`, returns `List<PeerInfo>`
- Other methods keep String returns for now (complex formatting)

- [ ] **Step 5: Rename MeshMcpToolsTest → MeshServiceTest**

Use `ide_refactor_rename` to rename the test class. Update method calls to match the new method names.

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mesh`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add mesh/
git commit -m "feat(#465): migrate MeshMcpTools → MeshApi @McpDomain + MeshService

Replaces @Tool annotations with @McpDomain pattern. Typed return records
for register, send, and discover operations.

Refs #465"
```

---

## Batch 5: Wire mesh to agent-bridge + Integration Test

### Task 5: Add agent-bridge dependency to mesh + end-to-end test

**Files:**
- Modify: `mesh/pom.xml` — add `casehub-qhorus-agent-bridge` dependency
- Test: `mesh/src/test/java/io/casehub/qhorus/mesh/AgentBridgeIntegrationTest.java`

**Interfaces:**
- Consumes: `AgentBridgeService` (Task 3); `ChannelService`; `MessageDispatcher`; `MessageStore`
- Produces: End-to-end verification that a COMMAND dispatched to a channel reaches a bridged agent and the RESPONSE appears in the channel

- [ ] **Step 1: Add agent-bridge dependency to mesh pom**

Add to `mesh/pom.xml`:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-agent-bridge</artifactId>
      <version>${project.version}</version>
    </dependency>
```

- [ ] **Step 2: Write the integration test**

Create `mesh/src/test/java/io/casehub/qhorus/mesh/AgentBridgeIntegrationTest.java`:

```java
package io.casehub.qhorus.mesh;

import io.casehub.platform.agent.AgentBackend;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.agent.AgentSessionInit;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.agent.bridge.AgentBridgeService;
import io.casehub.qhorus.agent.bridge.AgentChannelBinding;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.quarkus.test.junit.QuarkusTest;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class AgentBridgeIntegrationTest {

    @Inject ChannelService channelService;
    @Inject MessageDispatcher messageDispatcher;
    @Inject MessageStore messageStore;

    @Test
    @Transactional
    void bridgedAgent_receivesCommand_respondsToChannel() throws Exception {
        Channel ch = channelService.create(
                ChannelCreateRequest.builder("bridge-test-" + UUID.randomUUID()).build());

        AgentBackend echoBackend = new AgentBackend() {
            @Override public String key() { return "echo-test"; }
            @Override public Multi<AgentEvent> invoke(AgentSessionConfig c) {
                return Multi.createFrom().items(
                        new AgentEvent.TextDelta("Echo: " + c.userPrompt()),
                        new AgentEvent.InvocationComplete(10, 5, 0, 0, 0, null, 100L, 90L, "s", 1, false));
            }
            @Override public AgentSession openSession(AgentSessionInit init) { return null; }
        };

        // Manually wire the bridge (in production, CDI does this)
        var bridgeService = new AgentBridgeService(
                List.of(echoBackend),
                // BackendRegistry from ChannelGateway — inject or mock
                null, // TODO: wire to real BackendRegistry in integration
                messageDispatcher);

        var binding = AgentChannelBinding.builder(ch.id(), "echo-agent", "echo-test")
                .persistent(false).maxConcurrency(1).build();
        bridgeService.createBinding(binding);

        // Send a COMMAND to the channel
        messageDispatcher.dispatch(MessageDispatch.builder()
                .channelId(ch.id())
                .sender("human-user")
                .type(MessageType.COMMAND)
                .content("Hello agent")
                .actorType(ActorType.HUMAN)
                .build());

        // Wait for async agent processing
        Thread.sleep(2000);

        // Verify the agent's RESPONSE is in the channel
        List<Message> messages = messageStore.scan(MessageQuery.forChannel(ch.id()));
        assertThat(messages).anyMatch(m ->
                m.messageType() == MessageType.RESPONSE
                && m.content().contains("Echo: Hello agent"));
    }
}
```

Note: This test skeleton will need adjustment based on how `BackendRegistry` is exposed for testing. The actual integration may require `@InjectMock` or a test profile. The implementer should check how `ChannelGateway` exposes backend registration and adjust.

- [ ] **Step 3: Run the test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AgentBridgeIntegrationTest -pl mesh`
Expected: PASS (may need test adjustments for BackendRegistry wiring)

- [ ] **Step 4: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS (all modules, all tests)

- [ ] **Step 5: Commit**

```bash
git add mesh/
git commit -m "feat(#465): wire agent-bridge into mesh + integration test

End-to-end: COMMAND → bridge → echo agent → RESPONSE in channel.

Refs #465"
```

---

## References

- [2026-10-02-agent-provider-bridge-design.md](../specs/qhorus-mesh-agent-provider/2026-10-02-agent-provider-bridge-design.md) — design spec this plan implements
- `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelBackend.java` — ChannelBackend SPI
- `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java` — outbound message record
- `api/src/main/java/io/casehub/qhorus/api/gateway/PostResult.java` — delivery result
- `a2a-outbound/src/main/java/.../A2AOutboundBackend.java` — reference bridge pattern
- `casehub-platform/agent-api/src/main/java/.../AgentBackend.java` — provider backend SPI
- `casehub-platform/agent-api/src/main/java/.../AgentSession.java` — multi-turn session
- `casehub-platform/agent-api/src/main/java/.../AgentEvent.java` — sealed event hierarchy
- `casehub-platform/agent-api/src/main/java/.../AgentSessionInit.java` — session initialization
- `casehub-platform/agent-api/src/main/java/.../AgentSessionConfig.java` — ephemeral invocation config
- `mesh/src/main/java/io/casehub/qhorus/mesh/MeshMcpTools.java` — current @Tool pattern to migrate
- `api/src/main/java/io/casehub/qhorus/api/spi/channels/ChannelsApi.java` — @McpDomain pattern reference
- casehubio/qhorus#465 — focal issue
- casehubio/claudony#246 — fleet deployment (future consumer)
- casehubio/claudony#247 — fleet script runner (future consumer)

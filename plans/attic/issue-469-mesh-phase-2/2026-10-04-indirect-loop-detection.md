# Indirect Loop Detection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #468 — Indirect loop detection: cross-bridge invocation-context propagation
**Issue group:** #469 (Mesh Phase 2 epic)

**Goal:** Propagate a visited-set of agent instanceIds through the dispatch pipeline so the agent-bridge can detect indirect loops (A→B→A).

**Architecture:** Add `String invocationContext` (nullable JSON array) to MessageDispatch, Message, and OutboundMessage. The agent-bridge populates it before dispatching; the core pipeline preserves it through persistence and fanOut; the bridge reads it on the receiving end to detect loops. v1 ships advisory-only (EVENT alert, no invocation refusal).

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2

## Global Constraints

- `invocationContext` is nullable — null means no invocation context (default for all non-bridge dispatches)
- JSON format: `["agent-a","agent-b"]` — a JSON array of instanceId strings
- v1 enforcement: detect and alert only; do not refuse invocation
- Flyway migration V56 for the new column
- All existing OutboundMessage construction sites must be updated to pass through the field

---

## Batch 1: API + Pipeline Passthrough

### Task 1: Add invocationContext to API records + Message entity + pipeline

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java` — add 17th field
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/Message.java` — add 20th field
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java` — add 13th field
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageEntity.java` — add column + fromDomain/toDomain
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — pass invocationContext through Message builder (2 sites) and OutboundMessage construction (2 sites)
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` — pass invocationContext in OutboundMessage construction (1 site)
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java` — pass invocationContext in toOutbound() (1 site)
- Create: `runtime/src/main/resources/db/qhorus/migration/V56__invocation_context.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchBuilderTest.java` — verify builder passes field

**Interfaces:**
- Produces: `MessageDispatch.invocationContext()` — nullable String, JSON array
- Produces: `Message.invocationContext()` — nullable String, JSON array
- Produces: `OutboundMessage.invocationContext()` — nullable String, JSON array
- Produces: `MessageDispatch.Builder.invocationContext(String)` — builder setter

- [ ] **Step 1: Write failing test — MessageDispatch builder round-trips invocationContext**

Add to existing `MessageDispatchBuilderTest.java`:

```java
@Test
void invocationContext_roundTrips() {
    String context = "[\"agent-a\",\"agent-b\"]";
    MessageDispatch dispatch = MessageDispatch.builder()
            .channelId(UUID.randomUUID())
            .sender("test")
            .type(MessageType.COMMAND)
            .content("hello")
            .actorType(ActorType.AGENT)
            .invocationContext(context)
            .build();
    assertThat(dispatch.invocationContext()).isEqualTo(context);
}

@Test
void invocationContext_defaultsToNull() {
    MessageDispatch dispatch = MessageDispatch.builder()
            .channelId(UUID.randomUUID())
            .sender("test")
            .type(MessageType.COMMAND)
            .content("hello")
            .actorType(ActorType.AGENT)
            .build();
    assertThat(dispatch.invocationContext()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api,runtime -f pom.xml`
Expected: FAIL — `invocationContext` method/field does not exist

- [ ] **Step 3: Add invocationContext to MessageDispatch**

In `MessageDispatch.java`:
- Add `String invocationContext` as 17th record component (after `topic`)
- Add `Builder.invocationContext(String)` setter
- Update `Builder.build()` to pass `invocationContext` (no validation needed — nullable)
- Update `withTarget()` to pass `invocationContext` through

- [ ] **Step 4: Add invocationContext to Message**

In `Message.java`:
- Add `String invocationContext` as 20th record component (after `createdAt`)
- Add `Builder.invocationContext(String)` setter
- Update `Builder.build()` to pass `invocationContext`
- Update `toBuilder()` to copy `invocationContext`

- [ ] **Step 5: Add invocationContext to OutboundMessage**

In `OutboundMessage.java`:
- Add `String invocationContext` as 13th record component (after `topic`)
- Update backward-compatible constructors to delegate with `null`

- [ ] **Step 6: Add invocationContext to MessageEntity**

In `MessageEntity.java`:
- Add field: `@Column(name = "invocation_context", columnDefinition = "TEXT") public String invocationContext;`
- Update `fromDomain()`: `e.invocationContext = msg.invocationContext();`
- Update `toDomain()`: add `invocationContext` to the Message constructor call

- [ ] **Step 7: Create Flyway migration V56**

Create `runtime/src/main/resources/db/qhorus/migration/V56__invocation_context.sql`:
```sql
ALTER TABLE message ADD COLUMN invocation_context TEXT;
```

- [ ] **Step 8: Update MessageService — pass invocationContext through Message builder**

In `MessageService.java`, find the two Message.builder() blocks (around lines 360-381 and 440-448) and add `.invocationContext(dispatch.invocationContext())` to both.

- [ ] **Step 9: Update MessageService — pass invocationContext through OutboundMessage construction**

In `MessageService.java`, find the two `new OutboundMessage(...)` blocks (around lines 390-393 and 528-531) and add `dispatch.invocationContext()` as the 13th argument.

- [ ] **Step 10: Update ChannelGateway — pass invocationContext through OutboundMessage**

In `ChannelGateway.java` around line 361-364, add `msg.invocationContext()` as the 13th argument to `new OutboundMessage(...)`.

- [ ] **Step 11: Update DeliveryBatchExecutor — pass invocationContext through toOutbound()**

In `DeliveryBatchExecutor.java` around line 185-199, add `m.invocationContext()` as the 13th argument to `new OutboundMessage(...)`.

- [ ] **Step 12: Fix compilation across all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f pom.xml`

Fix any remaining compilation errors from OutboundMessage/Message/MessageDispatch constructor changes in other modules. Key sites:
- `InMemoryMessageStore` — `toDomain()`/`fromDomain()` methods
- `QhorusEntityMapper` — `toMessageView()` if it reads Message fields
- Any test helpers that construct Message or OutboundMessage directly

- [ ] **Step 13: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime -f pom.xml`
Expected: PASS (including the new builder tests)

- [ ] **Step 14: Commit**

```bash
git add api/ runtime/ runtime-core/
git commit -m "feat(#468): add invocationContext field to MessageDispatch, Message, and OutboundMessage

Nullable JSON string (visited-set) propagated through the dispatch
pipeline. MessageService persists it; ChannelGateway and DeliveryBatchExecutor
carry it through to OutboundMessage. V56 migration.

Refs #468"
```

---

## Batch 2: Agent-Bridge Loop Detection

### Task 2: Implement visited-set propagation and loop detection in agent-bridge

**Files:**
- Modify: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentInvocationRunner.java` — build augmented visited-set, set ThreadLocal, pass to all dispatches via SpeechActMapper
- Modify: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/AgentProviderBackend.java` — check visited-set in postTracked(), dispatch advisory EVENT on loop detection
- Modify: `agent-bridge/src/main/java/io/casehub/qhorus/agent/bridge/SpeechActMapper.java` — accept and pass invocationContext to all dispatch builders
- Test: `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/AgentProviderBackendTest.java` — add loop detection tests
- Test: `agent-bridge/src/test/java/io/casehub/qhorus/agent/bridge/SpeechActMapperTest.java` — verify invocationContext passthrough

**Interfaces:**
- Consumes: `OutboundMessage.invocationContext()` — nullable String from Task 1
- Consumes: `MessageDispatch.Builder.invocationContext(String)` — from Task 1

- [ ] **Step 1: Write failing test — postTracked detects indirect loop**

Add to `AgentProviderBackendTest.java`:

```java
@Test
void postTracked_indirectLoopDetected_skipsInvocation() throws Exception {
    // invocationContext contains this agent's ID — indirect loop
    String context = "[\"" + AGENT_ID + "\",\"other-agent\"]";
    var msg = new OutboundMessage(UUID.randomUUID(), 1L, "requester",
            MessageType.COMMAND, "content", null, UUID.randomUUID().toString(), null,
            ActorType.AGENT, List.of(), AGENT_ID, null, context);
    backend.postTracked(new ChannelRef(CHANNEL_ID, "ch"), msg);
    Thread.sleep(200);
    assertThat(dispatched).noneMatch(d -> d.type() == MessageType.RESPONSE);
    // Should dispatch an advisory EVENT instead
    assertThat(dispatched).anyMatch(d -> d.type() == MessageType.EVENT);
}

@Test
void postTracked_noLoop_proceedsNormally() throws Exception {
    // invocationContext does NOT contain this agent's ID
    String context = "[\"other-agent\"]";
    var msg = new OutboundMessage(UUID.randomUUID(), 1L, "requester",
            MessageType.COMMAND, "content", null, UUID.randomUUID().toString(), null,
            ActorType.AGENT, List.of(), AGENT_ID, null, context);
    backend.postTracked(new ChannelRef(CHANNEL_ID, "ch"), msg);
    invocationLatch.await(5, TimeUnit.SECONDS);
    assertThat(dispatched).anyMatch(d -> d.type() == MessageType.RESPONSE);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AgentProviderBackendTest -pl agent-bridge -f pom.xml`
Expected: FAIL — OutboundMessage constructor doesn't accept 13 args yet (or loop detection not implemented)

- [ ] **Step 3: Update SpeechActMapper to accept and pass invocationContext**

In `SpeechActMapper.java`, update all three mapping methods to accept `String invocationContext` parameter and pass it to `MessageDispatch.builder().invocationContext(invocationContext)`:

```java
public static MessageDispatch mapToDispatch(UUID channelId, OutboundMessage inbound,
                                             String agentInstanceId, String responseText,
                                             AgentEvent.InvocationComplete stats,
                                             String invocationContext) {
    // ... existing code ...
    return MessageDispatch.builder()
            .channelId(channelId)
            .sender(agentInstanceId)
            .type(MessageType.RESPONSE)
            .content(responseText)
            .correlationId(inbound.correlationId())
            .inReplyTo(inbound.sequenceId())
            .actorType(ActorType.AGENT)
            .telemetry(telemetry)
            .invocationContext(invocationContext)
            .build();
}
```

Same pattern for `mapToolStatus` and `mapFailure`. Keep backward-compatible overloads that pass `null`.

- [ ] **Step 4: Add ThreadLocal and visited-set logic to AgentInvocationRunner**

```java
public static final ThreadLocal<String> CURRENT_INVOCATION_CONTEXT = new ThreadLocal<>();

@Override
public void run() {
    // Build augmented visited-set
    String augmentedContext = buildAugmentedContext(
            inbound.invocationContext(), binding.agentInstanceId());
    CURRENT_INVOCATION_CONTEXT.set(augmentedContext);
    try {
        // ... existing concurrency guard + agent invocation ...
        // Update all SpeechActMapper calls to pass augmentedContext
    } finally {
        CURRENT_INVOCATION_CONTEXT.remove();
        concurrencyGuard.release();
    }
}

static String buildAugmentedContext(String existingContext, String agentId) {
    if (existingContext == null || existingContext.isBlank()) {
        return "[\"" + agentId + "\"]";
    }
    // Parse existing JSON array, add agentId, re-serialise
    String trimmed = existingContext.strip();
    if (trimmed.endsWith("]")) {
        return trimmed.substring(0, trimmed.length() - 1)
                + ",\"" + agentId + "\"]";
    }
    return "[\"" + agentId + "\"]";
}
```

- [ ] **Step 5: Add loop detection to AgentProviderBackend.postTracked()**

Before the existing sender loop guard, add:

```java
String context = message.invocationContext();
if (context != null && context.contains("\"" + binding.agentInstanceId() + "\"")) {
    LOG.warnf("Indirect loop detected for %s — visited: %s",
            binding.agentInstanceId(), context);
    dispatcher.dispatch(MessageDispatch.builder()
            .channelId(channel.id())
            .sender("system:agent-bridge")
            .type(MessageType.EVENT)
            .actorType(ActorType.SYSTEM)
            .telemetry("{\"tool_name\":\"loop_detection\",\"source_entity\":\"agent-bridge\""
                    + ",\"loop_agent\":\"" + binding.agentInstanceId() + "\""
                    + ",\"invocation_context\":" + context + "}")
            .build());
    return PostResult.ALL_DELIVERED;
}
```

- [ ] **Step 6: Update SpeechActMapper tests**

Add to `SpeechActMapperTest.java`:

```java
@Test
void mapToDispatch_passesInvocationContext() {
    String context = "[\"agent-a\"]";
    MessageDispatch dispatch = SpeechActMapper.mapToDispatch(
            UUID.randomUUID(), inbound, "agent-b", "response", null, context);
    assertThat(dispatch.invocationContext()).isEqualTo(context);
}
```

- [ ] **Step 7: Run all agent-bridge tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-bridge -f pom.xml`
Expected: PASS (all 24+ tests including new loop detection tests)

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl '!connector-backend' -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add agent-bridge/
git commit -m "feat(#468): indirect loop detection in agent-bridge via invocationContext visited-set

AgentInvocationRunner builds augmented visited-set (existing context +
current agent instanceId) and sets ThreadLocal for MCP tool access.
AgentProviderBackend.postTracked() checks the visited-set before spawning
invocation — if the agent is already in the chain, dispatches an advisory
EVENT and skips invocation (v1 enforcement).

SpeechActMapper gains invocationContext parameter for all mapping methods.

Closes #468"
```

---

## References

- [2026-10-04-indirect-loop-detection-design.md] — design spec this plan implements
- [AgentProviderBackend.java:69] — existing direct loop guard
- [AgentInvocationRunner.java:41-93] — agent event iteration and dispatch
- [MessageDispatch.java] — 16-field API record (becomes 17)
- [OutboundMessage.java] — 12-field API record (becomes 13)
- [Message.java] — 19-field API record (becomes 20)
- [MessageEntity.java:102-130] — fromDomain/toDomain mapping
- [MessageService.java:360-393,440-531] — dispatch pipeline construction sites
- [ChannelGateway.java:361-364] — OutboundMessage construction in deliverRemote
- [DeliveryBatchExecutor.java:185-199] — toOutbound() construction
- [GitHub #468] — indirect loop detection issue
- [GitHub #469] — Mesh Phase 2 epic

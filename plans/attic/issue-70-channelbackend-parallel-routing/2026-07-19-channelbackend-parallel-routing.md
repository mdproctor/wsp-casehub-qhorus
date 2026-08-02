# ChannelBackend Parallel COMMAND Routing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** openclaw#70 — feat: ChannelBackend parallel COMMAND routing for multi-agent cases
**Issue group:** openclaw#70

**Goal:** Propagate the `target` field from `MessageDispatch` through `OutboundMessage` to `ChannelBackend.post()` so backends can route COMMANDs to specific agents.

**Architecture:** Add `String target` to the `OutboundMessage` record (qhorus-api). Pass it through all 6 production construction sites. Add sender-based trust check exemption so engine-dispatched COMMANDs bypass `ObligorTrustPolicy`. Add `target` parameter to `CaseChannelProvider.postToChannel()` SPI (engine-api, breaking). Engine sets `target = worker.name()`. OpenClaw backend reads `message.target()` for direct agent lookup.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-qhorus, casehub-engine-api, casehub-engine, casehub-openclaw

## Global Constraints

- Pre-release platform — breaking SPI changes are fine, no backward-compat shims
- All edits via IntelliJ MCP (`ide_edit_member`, `ide_replace_member`, `ide_insert_member`)
- Verify after each edit with `ide_diagnostics`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Working in git worktree at `/Users/mdproctor/claude/casehub/worktrees/6/qhorus`
- Engine repo at `/Users/mdproctor/claude/casehub/engine`
- OpenClaw repo at `/Users/mdproctor/claude/casehub/openclaw`
- IntelliJ workspace must include all four repos — use `ide_open_workspace` or pass `project_path` per call

---

### Task 1: Add `target` to `OutboundMessage` record + fix all construction sites

This is the qhorus-only change. One record field addition, 6 production sites, ~33 test sites.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (lines 257, 382)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` (lines 388, 449)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` (line 433)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java` (line 168)
- Modify: ~33 test files constructing `OutboundMessage`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/DeliveryServiceTest.java` (existing `toOutbound_convertsAllFields`)

**Interfaces:**
- Produces: `OutboundMessage(UUID, String, MessageType, String, String, Long, ActorType, List<ArtefactRef>, String target)` — used by all `ChannelBackend.post()` callers

- [ ] **Step 1: Write failing test — OutboundMessage carries target**

Add a test in `DeliveryServiceTest.java` verifying `toOutbound()` passes `target` through:

```java
@Test
void toOutbound_carriesTarget() {
    Message m = Message.builder()
            .id(1L).channelId(UUID.randomUUID()).sender("agent-a")
            .messageType(MessageType.COMMAND).actorType(ActorType.AGENT)
            .content("do it").target("researcher").build();
    OutboundMessage out = DeliveryBatchExecutor.toOutbound(m);
    assertThat(out.target()).isEqualTo("researcher");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryServiceTest#toOutbound_carriesTarget -pl runtime`
Expected: FAIL — `OutboundMessage` has no `target()` accessor

- [ ] **Step 3: Add `target` field to `OutboundMessage` record**

Use `ide_edit_member` on `OutboundMessage` in `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java`:

```java
public record OutboundMessage(
        UUID messageId,
        String sender,
        MessageType type,
        String content,
        String correlationId,
        Long inReplyTo,
        ActorType senderActorType,
        java.util.List<io.casehub.qhorus.api.message.ArtefactRef> artefactRefs,
        String target) {}
```

- [ ] **Step 4: Fix all 6 production construction sites**

Each site gets `dispatch.target()` or `m.target()` as the 9th argument:

**MessageService.java line 257** (LAST_WRITE path) — add `dispatch.target()`:
```java
channelGateway.fanOut(ch.id(), ch.name(), new OutboundMessage(
        UUID.randomUUID(), dispatch.sender(), dispatch.type(), dispatch.content(),
        dispatch.correlationId(),
        dispatch.inReplyTo(),
        dispatch.actorType(),
        dispatch.artefactRefs(),
        dispatch.target()));
```

**MessageService.java line 382** (normal path) — same pattern.

**ReactiveMessageService.java line 388** (reactive LAST_WRITE) — same pattern.

**ReactiveMessageService.java line 449** (reactive normal) — same pattern.

**ChannelGateway.java line 433** (deliverRemote) — add `msg.target()`:
```java
OutboundMessage outbound = new OutboundMessage(
        UUID.randomUUID(),
        msg.sender(),
        msg.messageType(),
        msg.content(),
        msg.correlationId(),
        msg.inReplyTo(),
        msg.actorType(),
        msg.artefactRefs(),
        msg.target());
```

**DeliveryBatchExecutor.java line 168** (toOutbound) — add `m.target()`:
```java
static OutboundMessage toOutbound(Message m) {
    return new OutboundMessage(
            UUID.randomUUID(),
            m.sender(),
            m.messageType(),
            m.content(),
            m.correlationId(),
            m.inReplyTo(),
            ActorTypeResolver.resolve(m.sender()),
            m.artefactRefs(),
            m.target());
}
```

- [ ] **Step 5: Fix all test construction sites — append `null` as 9th arg**

Each `new OutboundMessage(...)` call in test files needs `null` appended. Files and lines:

| File | Lines |
|------|-------|
| `runtime/src/test/java/.../QhorusChannelBackendTest.java` | 41 |
| `runtime/src/test/java/.../A2AChannelBackendSseTest.java` | 42, 129 |
| `runtime/src/test/java/.../A2AChannelBackendIntegrationTest.java` | 119 |
| `runtime/src/test/java/.../ChannelGatewayTest.java` | 152, 164, 180, 194, 210 |
| `runtime/src/test/java/.../FanOutDeliveryGuaranteeTest.java` | 83 |
| `runtime/src/test/java/.../FanOutTracingTest.java` | 136, 176, 252, 289, 332 |
| `runtime/src/test/java/.../DebateChannelBackendFactoryTest.java` | 111, 161, 172 |
| `runtime/src/test/java/.../ReviewerChannelBackendTest.java` | 138, 271, 277 |
| `connector-backend/src/test/java/.../ConnectorChannelBackendTest.java` | 109, 168, 185, 199, 218, 237, 257 |
| `connector-backend/src/test/java/.../ConnectorChannelBackendIntegrationTest.java` | 134 |
| `connector-backend/src/test/java/.../ConnectorAutoChannelBackendTest.java` | 145 |
| `slack-channel/src/test/java/.../SlackChannelBackendTest.java` | 387 |

Use `ide_replace_text_in_file` with regex to append `,\n        null)` before the closing `)` on each multiline constructor, or fix individually with `ide_edit_member` for inline constructors.

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryServiceTest#toOutbound_carriesTarget -pl runtime`
Expected: PASS

- [ ] **Step 7: Run full build to verify no compilation errors**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS (all modules compile and tests pass)

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java
# + all test files
git commit -m "feat(#70): add target field to OutboundMessage — propagate through all 6 construction sites

Refs openclaw#70"
```

---

### Task 2: Add trust check sender exemption in MessageService + ReactiveMessageService

System senders (`casehub-engine:orchestrator`) bypass `ObligorTrustPolicy` when `target` is set.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (line 188)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` (line 261)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/MessageDispatchIntegrationTest.java` (or new test class)

**Interfaces:**
- Consumes: `OutboundMessage.target()` from Task 1
- Produces: Sender-based trust exemption — `dispatch.sender().contains(":")` bypasses trust check

- [ ] **Step 1: Write failing test — system sender bypasses trust check**

In `MessageDispatchIntegrationTest.java` (or a new test class), test that a system sender with `target` set does NOT trigger `ObligorTrustPolicy`:

```java
@Test
void dispatch_systemSenderWithTarget_bypassesTrustCheck() {
    // Setup: channel with trust enabled (minObligorTrust > 0)
    // Dispatch: sender="casehub-engine:orchestrator", target="researcher", type=COMMAND
    // Assert: dispatch succeeds without trust check (no IllegalStateException)
}
```

- [ ] **Step 2: Write failing test — agent sender still triggers trust check**

```java
@Test
void dispatch_agentSenderWithTarget_triggerssTrustCheck() {
    // Setup: channel with trust enabled, mock ObligorTrustPolicy returns false
    // Dispatch: sender="agent-b", target="researcher", type=COMMAND
    // Assert: IllegalStateException thrown (trust check NOT bypassed)
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run the tests. The system-sender test should fail because the current code triggers the trust check for any sender with a target.

- [ ] **Step 4: Add sender exemption to MessageService.java**

Change the trust gate condition at line 188 from:

```java
if (ch != null && dispatch.type() == MessageType.COMMAND
        && dispatch.target() != null
        && !dispatch.target().contains(":")) {
```

To:

```java
if (ch != null && dispatch.type() == MessageType.COMMAND
        && dispatch.target() != null
        && !dispatch.target().contains(":")
        && !dispatch.sender().contains(":")) {
```

- [ ] **Step 5: Add sender exemption to ReactiveMessageService.java**

Same change at line 261:

```java
if (ch != null && dispatch.type() == MessageType.COMMAND
        && dispatch.target() != null
        && !dispatch.target().contains(":")
        && !dispatch.sender().contains(":")) {
```

- [ ] **Step 6: Run tests to verify they pass**

Run both tests. Expected: PASS

- [ ] **Step 7: Run full qhorus test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests pass

- [ ] **Step 8: Commit**

```bash
git commit -m "feat(#70): sender-based trust check exemption for engine-dispatched COMMANDs

System senders (sender contains ':') bypass ObligorTrustPolicy when target
is set. Engine provisioning decisions ARE the trust authority.
Applied to both MessageService and ReactiveMessageService.

Refs openclaw#70"
```

---

### Task 3: Add `target` parameter to CaseChannelProvider SPI (engine-api)

Breaking SPI change — 7-param replaces 6-param.

**Files:**
- Modify: `engine/api/src/main/java/io/casehub/api/spi/CaseChannelProvider.java`
- Modify: `engine/api/src/main/java/io/casehub/api/spi/ReactiveCaseChannelProvider.java`
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/worker/NoOpCaseChannelProvider.java`
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveCaseChannelProvider.java`
- Modify: `engine/api/src/test/java/io/casehub/api/spi/CaseChannelProviderContractTest.java`
- Modify: `engine/api/src/test/java/io/casehub/api/spi/ReactiveCaseChannelProviderContractTest.java`

**Interfaces:**
- Produces: `CaseChannelProvider.postToChannel(CaseChannel, String, String, MessageType, String, String, String target)` — consumed by engine dispatch and all implementations

- [ ] **Step 1: Update CaseChannelProvider — 7-param abstract, 3-param default**

Replace the 6-param abstract `postToChannel` with 7-param. Update the 3-param default to delegate to 7-param with 4 nulls:

```java
void postToChannel(
    CaseChannel channel, String from, String content,
    MessageType type, String correlationId, String deadline,
    String target);

default void postToChannel(CaseChannel channel, String from, String content) {
    postToChannel(channel, from, content, null, null, null, null);
}
```

Remove the old 6-param method entirely.

- [ ] **Step 2: Update ReactiveCaseChannelProvider — same treatment**

```java
Uni<Void> postToChannel(
    CaseChannel channel, String from, String content,
    MessageType type, String correlationId, String deadline,
    String target);

default Uni<Void> postToChannel(CaseChannel channel, String from, String content) {
    return postToChannel(channel, from, content, null, null, null, null);
}
```

- [ ] **Step 3: Fix NoOpCaseChannelProvider — add 7th param**

The no-op implementation adds `String target` to its `postToChannel` signature and ignores it.

- [ ] **Step 4: Fix NoOpReactiveCaseChannelProvider — add 7th param**

Same treatment.

- [ ] **Step 5: Fix contract tests**

Update `CaseChannelProviderContractTest` and `ReactiveCaseChannelProviderContractTest`:
- Reflection-based signature checks now expect 7 params
- Direct calls use 7 args (pass `null` for target)

- [ ] **Step 6: Fix SpiWiringIntegrationTest**

The recording `CaseChannelProvider` in `SpiWiringIntegrationTest` needs the 7th param.

- [ ] **Step 7: Build engine to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine commit -am "feat(#70): add target param to CaseChannelProvider.postToChannel() — breaking SPI

7-param replaces 6-param. All implementations must update.
Pre-release platform — no backward-compat shim.

Refs openclaw#70"
```

---

### Task 4: Engine dispatch — set target on COMMANDs

**Files:**
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkerScheduleEventHandler.java` (line 295)
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandler.java` (line 98)
- Modify: `engine/runtime/src/test/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandlerTest.java`

**Interfaces:**
- Consumes: 7-param `CaseChannelProvider.postToChannel()` from Task 3
- Produces: Engine-dispatched COMMANDs carry `target = worker.name()`

- [ ] **Step 1: Update WorkerScheduleEventHandler.dispatchCommand()**

Line 295-301: add `worker.name()` as 7th argument:

```java
caseChannelProvider.postToChannel(
    channel,
    "casehub-engine:orchestrator",
    serialize(command),
    MessageType.COMMAND,
    String.valueOf(eventLogId),
    deadline,
    worker.name());
```

- [ ] **Step 2: Update AgentRoutingEscalationHandler.postQuery()**

Line 98-99: add `null` as 7th argument (QUERYs to oversight don't target agents):

```java
channelProvider.postToChannel(
    channel, "casehub-engine", message, MessageType.QUERY, null, null, null);
```

- [ ] **Step 3: Fix AgentRoutingEscalationHandlerTest**

Update mock verification calls to expect 7 args:
```java
verify(channelProvider, never()).postToChannel(any(), any(), any(), any(), any(), any(), any());
```

And the positive verification:
```java
.postToChannel(eq(channel), eq("casehub-engine"), contains("oversight"), eq(MessageType.QUERY), isNull(), isNull(), isNull());
```

- [ ] **Step 4: Build engine to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine commit -am "feat(#70): engine sets target=worker.name() on COMMAND dispatch

WorkerScheduleEventHandler passes worker.name() as target (= agentId by convention).
AgentRoutingEscalationHandler passes null (QUERYs not targeted).

Refs openclaw#70"
```

---

### Task 5: OpenClaw — CaseChannelProvider passes target + ChannelBackend routes by target

**Files:**
- Modify: `openclaw/casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java`
- Modify: `openclaw/casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawCaseChannelProvider.java`
- Modify: `openclaw/casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java`
- Modify: `openclaw/casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawChannelBackendTest.java`
- Test: new tests for target-based routing

**Interfaces:**
- Consumes: 7-param `CaseChannelProvider.postToChannel()` from Task 3, `OutboundMessage.target()` from Task 1
- Produces: Target-based agent routing in `OpenClawChannelBackend.post()`

- [ ] **Step 1: Write failing test — target routes to correct agent**

```java
@Test
void post_withTarget_routesToTargetAgent() {
    // Setup: register two agents for same caseId
    registry.register("agent-alpha", "t1", caseId, "key-alpha");
    registry.register("agent-beta", "t1", caseId, "key-beta");
    // Dispatch: COMMAND with target="agent-alpha"
    OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine",
            MessageType.COMMAND, "do it", null, null, ActorType.AGENT, null, "agent-alpha");
    backend.post(ref, msg);
    // Assert: hookClient.invoke() called with agentId="agent-alpha"
    verify(hookClient).invoke(eq("agent-alpha"), any(), any(), anyInt());
}
```

- [ ] **Step 2: Write failing test — target not registered, COMMAND dropped**

```java
@Test
void post_withTarget_agentNotRegistered_commandDropped() {
    // target="unknown-agent", no agent registered
    OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine",
            MessageType.COMMAND, "do it", null, null, ActorType.AGENT, null, "unknown-agent");
    backend.post(ref, msg);
    verify(hookClient, never()).invoke(any(), any(), any(), anyInt());
}
```

- [ ] **Step 3: Write failing test — target registered for different case, COMMAND dropped**

```java
@Test
void post_withTarget_agentOnDifferentCase_commandDropped() {
    UUID otherCase = UUID.randomUUID();
    registry.register("agent-alpha", "t1", otherCase, "key-alpha");
    OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine",
            MessageType.COMMAND, "do it", null, null, ActorType.AGENT, null, "agent-alpha");
    backend.post(ref, msg);
    verify(hookClient, never()).invoke(any(), any(), any(), anyInt());
}
```

- [ ] **Step 4: Write failing test — null target falls back to findAgentId**

```java
@Test
void post_withNullTarget_fallsBackToFindAgentId() {
    registry.register("agent-alpha", "t1", caseId, "key-alpha");
    OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine",
            MessageType.COMMAND, "do it", null, null, ActorType.AGENT, null, null);
    backend.post(ref, msg);
    verify(hookClient).invoke(eq("agent-alpha"), any(), any(), anyInt());
}
```

- [ ] **Step 5: Run tests to verify they fail**

Expected: FAIL — `OutboundMessage` constructor has 9 args now but backend doesn't read `target()`

- [ ] **Step 6: Update OpenClawCaseChannelProvider — 7-param override**

```java
@Override
public void postToChannel(CaseChannel channel, String from, String content,
                           MessageType type, String correlationId, String deadline,
                           String target) {
    MessageType effectiveType = type != null ? type : MessageType.STATUS;
    messageService.dispatch(MessageDispatch.builder()
            .channelId(UUID.fromString(channel.id()))
            .sender(from)
            .type(effectiveType)
            .content(content)
            .correlationId(correlationId)
            .deadline(deadline != null ? Instant.parse(deadline) : null)
            .target(target)
            .actorType(ActorType.AGENT)
            .build());
}
```

- [ ] **Step 7: Update ReactiveOpenClawCaseChannelProvider — same treatment**

Add `target` to the reactive override and pass through to `MessageDispatch`.

- [ ] **Step 8: Update OpenClawChannelBackend.post() — target-based routing**

```java
@Override
public void post(final ChannelRef channel, final OutboundMessage message) {
    if (message.type() != MessageType.COMMAND) return;

    final UUID caseId = extractCaseId(channel.name());
    if (caseId == null) return;

    final String agentId;
    if (message.target() != null && !message.target().isBlank()) {
        agentId = message.target();
        if (!registry.findCaseId(agentId).map(caseId::equals).orElse(false)) {
            if (registry.findCaseId(agentId).isPresent()) {
                log.warnf("Target agent %s registered for caseId=%s, not %s — routing bug, COMMAND dropped",
                        agentId, registry.findCaseId(agentId).orElse(null), caseId);
            } else {
                log.warnf("Target agent %s not registered — COMMAND dropped (agent may have crashed)", agentId);
            }
            return;
        }
    } else {
        agentId = registry.findAgentId(caseId).orElse(null);
    }
    if (agentId == null) {
        log.debugf("No OpenClaw agent for caseId=%s — ignoring COMMAND on %s", caseId, channel.name());
        return;
    }

    final String sessionKey = registry.findSessionKey(agentId).orElse(null);
    if (sessionKey == null) {
        log.warnf("No session key found for agentId=%s — ignoring COMMAND", agentId);
        return;
    }

    String webhookUrl = config.delivery().baseUrl() + "/channel/" + channel.id();
    webhookUrl = appendDeliveryToken(webhookUrl);
    hookClient.registerSession(agentId, sessionKey, webhookUrl);

    try {
        final UUID correlationId = message.correlationId() != null
                ? UUID.fromString(message.correlationId())
                : null;
        hookClient.invoke(agentId, buildPrompt(message.content(), agentId, correlationId),
                config.agent().defaultModel(), config.agent().defaultTimeoutSeconds());
        log.debugf("Invoked OpenClaw agent: agentId=%s caseId=%s", agentId, caseId);
    } catch (OpenClawInvocationException e) {
        log.errorf("OpenClaw invocation failed for agentId=%s: %s", agentId, e.getMessage());
    }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -f /Users/mdproctor/claude/casehub/openclaw/pom.xml`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw commit -am "feat(#70): ChannelBackend parallel COMMAND routing via target field

OpenClawCaseChannelProvider passes target through to MessageDispatch.
OpenClawChannelBackend routes COMMANDs by message.target() when set,
falls back to findAgentId() for single-agent compat.

Two distinct failure modes:
- Agent not registered → WARN, COMMAND dropped
- Agent on different case → WARN (routing bug), COMMAND dropped
No fallback when target is set — wrong-agent delivery is worse than drop.

Closes openclaw#70"
```

---

### Task 6: Full cross-repo integration test + CLAUDE.md update

**Files:**
- Verify: full build across all 4 repos
- Modify: qhorus `CLAUDE.md` — update `OutboundMessage` documentation

- [ ] **Step 1: Install qhorus SNAPSHOT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/worktrees/6/qhorus/pom.xml`
Expected: BUILD SUCCESS — publishes updated `casehub-qhorus-api` SNAPSHOT

- [ ] **Step 2: Build engine against updated qhorus**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 3: Build openclaw against updated qhorus + engine**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -f /Users/mdproctor/claude/casehub/openclaw/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Update CLAUDE.md — OutboundMessage documentation**

Update the `OutboundMessage` record description in CLAUDE.md to include `target`:

```
│       │   ├── OutboundMessage.java (messageId, sender, type, content, correlationId, inReplyTo, senderActorType, artefactRefs, target — last nullable)
```

Also add a testing convention note about `target`:
```
- `OutboundMessage` now carries `target` (nullable String). Test construction sites pass `null` unless testing target-based routing.
```

- [ ] **Step 5: Commit CLAUDE.md update**

```bash
git commit -m "docs(#70): update CLAUDE.md for OutboundMessage.target field

Refs openclaw#70"
```

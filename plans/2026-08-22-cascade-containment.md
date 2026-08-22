# Cascade Containment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #399 — E2: Cascade containment — watchdog-triggered quarantine
**Issue group:** #399

**Goal:** Connect existing watchdog detection to existing containment primitives so pathology detection triggers automatic response (pause, deregister, quarantine).

**Architecture:** New `WatchdogAction` enum on each watchdog registration. `WatchdogEvaluationService.fireAlert()` executes the configured action after alerting. DEREGISTER_AGENT marks instances offline (not gateway deregister). PAUSE_CHANNEL pauses the channel and expires its active commitments. QUARANTINE does both.

**Tech Stack:** Java 21, Quarkus 3.32.2, H2 (test), JPA/Hibernate

## Global Constraints

- No new containment mechanisms — uses existing `ChannelService.pause()` and `InstanceService`
- `CommitmentStore.findOpenByChannelId(UUID)` already exists — no new store method needed
- Notification channel excluded from containment scope (self-defeat prevention)
- Error isolation: containment failures must not roll back the evaluation cycle
- HANDOFF is non-terminal (carries through from #398 decisions)
- Flyway next version: V44
- `WatchdogEvaluationService` uses `"system:watchdog"` sender and `ActorType.SYSTEM`

---

## Batch 1: Foundation — enum, record, default method, service methods

### Task 1: WatchdogAction enum + Watchdog record + migration + entity

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAction.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/Watchdog.java` — add `action` field + compact constructor normalization + builder setter
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEntity.java` — add `action` column + toDomain/fromDomain mapping
- Create: `runtime/src/main/resources/db/qhorus/migration/V44__watchdog_action.sql`
- Test: `api/src/test/java/io/casehub/qhorus/api/watchdog/WatchdogActionTest.java` (if api has test dir, else unit test in runtime)

**Interfaces:**
- Produces: `WatchdogAction` enum (ALERT, PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE), `Watchdog.action()` accessor

- [ ] **Step 1: Create `WatchdogAction` enum**

Create `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAction.java`:

```java
package io.casehub.qhorus.api.watchdog;

public enum WatchdogAction {
    ALERT,
    PAUSE_CHANNEL,
    DEREGISTER_AGENT,
    QUARANTINE
}
```

- [ ] **Step 2: Add `action` field to `Watchdog` record**

Use `ide_edit_member` on `Watchdog` record — add `WatchdogAction action` as the last component (after `lastFiredAt`). Add null → ALERT normalization in compact constructor:

```java
public Watchdog {
    action = action != null ? action : WatchdogAction.ALERT;
}
```

Add `action(WatchdogAction v)` setter to `Builder`. Update `toBuilder()` to include `action`.

- [ ] **Step 3: Add `action` column to `WatchdogEntity`**

Use `ide_insert_member` to add field:

```java
@Column(name = "action")
@Enumerated(EnumType.STRING)
public WatchdogAction action;
```

Update `toDomain()` to pass `action` to `Watchdog` constructor. Update `fromDomain(Watchdog)` to set `entity.action = w.action()`.

- [ ] **Step 4: Create V44 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V44__watchdog_action.sql`:

```sql
ALTER TABLE watchdog ADD COLUMN action VARCHAR(32);
```

- [ ] **Step 5: Update `WatchdogSummary` DTO and `toWatchdogSummary` mapper**

Add `String action` field to `WatchdogSummary` record in `QhorusMcpToolsBase.java`. Update `toWatchdogSummary()` to include `w.action().name()`.

- [ ] **Step 6: Compile and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api,runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: compiles without errors.

- [ ] **Step 7: Commit**

```bash
git add api/ runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java runtime/src/main/resources/db/qhorus/migration/V44__watchdog_action.sql
git commit -m "feat(#399): WatchdogAction enum + Watchdog action field + V44 migration. Refs #399"
```

---

### Task 2: AlertContext.affectedAgentIds() default method + overrides

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java` — add default method
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/LoopDetectedContext.java` — override
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/AgentStaleContext.java` — override
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/ContextPressureContext.java` — override
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/EchoChamberContext.java` — override
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/AlertContextAgentIdsTest.java`

**Interfaces:**
- Produces: `AlertContext.affectedAgentIds()` → `List<String>`

- [ ] **Step 1: Write failing test**

Create CDI-free test verifying each context returns correct agent IDs:

```java
@Test
void loopDetected_returnsSender() {
    var ctx = new LoopDetectedContext(UUID.randomUUID(), "ch", "agent-a", 5, 0.95);
    assertThat(ctx.affectedAgentIds()).containsExactly("agent-a");
}

@Test
void agentStale_returnsStaleIds() {
    var ctx = new AgentStaleContext(2, List.of("agent-a", "agent-b"));
    assertThat(ctx.affectedAgentIds()).containsExactly("agent-a", "agent-b");
}

@Test
void barrierStuck_returnsEmpty() {
    var ctx = new BarrierStuckContext(...);
    assertThat(ctx.affectedAgentIds()).isEmpty();
}
```

- [ ] **Step 2: Add default method to `AlertContext`**

```java
default List<String> affectedAgentIds() { return List.of(); }
```

Add `import java.util.List;` to the file.

- [ ] **Step 3: Override in 4 agent-specific contexts**

- `LoopDetectedContext`: `@Override public List<String> affectedAgentIds() { return List.of(sender); }`
- `AgentStaleContext`: `@Override public List<String> affectedAgentIds() { return staleInstanceIds; }`
- `ContextPressureContext`: `@Override public List<String> affectedAgentIds() { return List.of(actorId); }`
- `EchoChamberContext`: `@Override public List<String> affectedAgentIds() { return participants; }`

- [ ] **Step 4: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AlertContextAgentIdsTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/watchdog/ runtime/src/test/
git commit -m "feat(#399): AlertContext.affectedAgentIds() with overrides for agent-specific contexts. Refs #399"
```

---

### Task 3: InstanceService.markOffline() + CommitmentService.expireByChannel()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java` — add `markOffline(String)`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java` — add `expireByChannel(UUID, String)`
- Test: `testing/src/test/java/io/casehub/qhorus/runtime/message/CommitmentServiceTest.java` — add expireByChannel tests
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/instance/InstanceServiceTest.java` (or add to existing)

**Interfaces:**
- Consumes: `CommitmentStore.findOpenByChannelId(UUID)` (existing), `InstanceEntity.update()` (Panache static, existing pattern from `markStaleOlderThan`)
- Produces: `InstanceService.markOffline(String instanceId)`, `CommitmentService.expireByChannel(UUID channelId, String tenancyId)`

- [ ] **Step 1: Write failing test for `expireByChannel`**

In `CommitmentServiceTest` (in `testing/` module):

```java
@Test
void expireByChannel_expiresOpenAndAcknowledged() {
    // Set up channel with OPEN and ACKNOWLEDGED commitments
    UUID channelId = UUID.randomUUID();
    Commitment open = createCommitment(channelId, CommitmentState.OPEN);
    Commitment acked = createCommitment(channelId, CommitmentState.ACKNOWLEDGED);
    Commitment fulfilled = createCommitment(channelId, CommitmentState.FULFILLED);
    store.save(open); store.save(acked); store.save(fulfilled);

    service.expireByChannel(channelId, TenancyConstants.DEFAULT_TENANT_ID);

    // OPEN and ACKNOWLEDGED should be EXPIRED
    assertThat(store.find(open.id()).get().state()).isEqualTo(CommitmentState.EXPIRED);
    assertThat(store.find(acked.id()).get().state()).isEqualTo(CommitmentState.EXPIRED);
    // FULFILLED unchanged
    assertThat(store.find(fulfilled.id()).get().state()).isEqualTo(CommitmentState.FULFILLED);
}
```

- [ ] **Step 2: Implement `CommitmentService.expireByChannel()`**

Follow the `expireOverdue()` pattern. Use `commitmentStore.findOpenByChannelId(channelId)`, filter to non-terminal, transition each to EXPIRED, fire `CommitmentExpiredEvent` for each:

```java
@Transactional
public void expireByChannel(UUID channelId, String tenancyId) {
    List<Commitment> active = commitmentStore.findOpenByChannelId(channelId);
    Instant now = Instant.now();
    for (Commitment c : active) {
        if (c.state().isTerminal()) continue;
        commitmentStore.updateState(c.id(), CommitmentState.EXPIRED);
        expiredEvents.fire(new CommitmentExpiredEvent(
                c.id(), c.correlationId(), channelId,
                c.obligor(), c.requester(), now));
    }
}
```

- [ ] **Step 3: Implement `InstanceService.markOffline()`**

Follow `markStaleOlderThan()` pattern:

```java
@Transactional
public void markOffline(String instanceId) {
    InstanceEntity.update("status = 'offline' WHERE instanceId = ?1", instanceId);
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentServiceTest -pl testing -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java testing/
git commit -m "feat(#399): InstanceService.markOffline + CommitmentService.expireByChannel. Refs #399"
```

---

## Batch 2: Execution + surface — containment logic, MCP, audit

### Task 4: WatchdogEvaluationService containment execution + audit EVENT

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java` — add `executeContainmentAction()` + `dispatchContainmentEvent()`, modify `fireAlert()` signature to accept `UUID channelId`, update all call sites
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogContainmentTest.java`

**Interfaces:**
- Consumes: `ChannelService.pause(UUID)` (existing), `InstanceService.markOffline(String)` (Task 3), `CommitmentService.expireByChannel(UUID, String)` (Task 3), `AlertContext.affectedAgentIds()` (Task 2), `Watchdog.action()` (Task 1)
- Produces: Containment execution triggered from `fireAlert()`

- [ ] **Step 1: Write failing tests — CDI-free with mocks**

Create `WatchdogContainmentTest.java`:

```java
@Test
void executeContainmentAction_alert_noOp() {
    // ALERT action → no calls to pause/deregister
}

@Test
void executeContainmentAction_pauseChannel_pausesAndExpires() {
    // PAUSE_CHANNEL → channelService.pause(channelId) + commitmentService.expireByChannel(channelId)
}

@Test
void executeContainmentAction_deregisterAgent_marksOffline() {
    // DEREGISTER_AGENT on LoopDetectedContext → instanceService.markOffline("agent-a")
}

@Test
void executeContainmentAction_deregisterAgent_noAgents_skips() {
    // DEREGISTER_AGENT on BarrierStuckContext → no call, log warning
}

@Test
void executeContainmentAction_quarantine_fullFlow() {
    // QUARANTINE → pause + markOffline + containment EVENT
}

@Test
void executeContainmentAction_nullChannelId_skipsPause() {
    // Cross-channel condition with null channelId → PAUSE skipped, DEREGISTER still works
}

@Test
void executeContainmentAction_notificationChannelExcluded() {
    // channelId == notification channel → skip entirely
}

@Test
void executeContainmentAction_failure_doesNotPropagate() {
    // channelService.pause() throws → exception caught, no propagation
}
```

- [ ] **Step 2: Add `executeContainmentAction` and `dispatchContainmentEvent` methods**

Add to `WatchdogEvaluationService`. Inject `InstanceService` and `ObjectMapper`. Follow the code in the spec §4 and §6.

- [ ] **Step 3: Modify `fireAlert()` — add `UUID channelId` parameter, call `executeContainmentAction`**

Update `fireAlert` signature from `(Watchdog w, String summary, AlertContext context, Instant now)` to `(Watchdog w, String summary, AlertContext context, Instant now, UUID channelId)`. Add `executeContainmentAction(w, context, channelId)` call at the end.

- [ ] **Step 4: Update all `fireAlert()` call sites**

Each `evaluate*()` method calls `fireAlert()`. Add the `channelId` argument at each call site. For channel-scoped conditions, pass the resolved `UUID`. For cross-channel conditions (AGENT_STALE, APPROVAL_PENDING), pass `null`.

- [ ] **Step 5: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WatchdogContainmentTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`

- [ ] **Step 6: Run all watchdog tests for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="WatchdogContainmentTest,WatchdogEvaluation*" -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/
git commit -m "feat(#399): containment execution in WatchdogEvaluationService + audit EVENT. Refs #399"
```

---

### Task 5: register_watchdog action param + CLAUDE.md update + full build

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add `action` param to `registerWatchdog`
- Modify: `CLAUDE.md` — add WatchdogAction, containment docs
- Test: Existing `@QuarkusTest` watchdog tool tests

**Interfaces:**
- Consumes: `WatchdogAction` (Task 1), `Watchdog.Builder.action()` (Task 1)
- Produces: `register_watchdog` MCP tool with `action` parameter

- [ ] **Step 1: Add `action` parameter to `registerWatchdog`**

Add `@ToolArg(name = "action", description = "Action to take: ALERT (default), PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE", required = false) String action` parameter. Parse and set on builder:

```java
WatchdogAction parsedAction = WatchdogAction.ALERT;
if (action != null && !action.isBlank()) {
    try { parsedAction = WatchdogAction.valueOf(action); }
    catch (IllegalArgumentException e) {
        throw new IllegalArgumentException("Unknown action '" + action + "'. Valid: " +
                java.util.Arrays.toString(WatchdogAction.values()));
    }
}
// ... builder.action(parsedAction) ...
```

- [ ] **Step 2: Update tool description**

Add `QUARANTINE` to the tool description string.

- [ ] **Step 3: Update CLAUDE.md**

Add WatchdogAction enum, containment execution docs, `expireByChannel`, `markOffline`, V44 migration to the project structure and conventions sections.

- [ ] **Step 4: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -nsu -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java CLAUDE.md
git commit -m "feat(#399): register_watchdog action param + CLAUDE.md update. Closes #399"
```

## References

- [2026-08-22-cascade-containment-design.md] — design spec
- `WatchdogEvaluationService.java:288-315` — existing `fireAlert()`
- `WatchdogEvaluationService.java:98-131` — existing `evaluateAll()` switch
- `Watchdog.java` — existing record
- `WatchdogEntity.java` — existing JPA entity
- `AlertContext.java` — sealed interface
- `InstanceService.java:91-97` — `markStaleOlderThan` pattern
- `CommitmentService.java:296` — `expireOverdue` pattern
- `QhorusMcpTools.java:2140-2162` — existing `register_watchdog`
- GitHub #399

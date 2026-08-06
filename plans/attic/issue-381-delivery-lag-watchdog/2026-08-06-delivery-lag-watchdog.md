# DELIVERY_LAG Watchdog Condition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #381 — feat: DELIVERY_LAG watchdog condition for observer health detection
**Issue group:** #381

**Goal:** Add a DELIVERY_LAG watchdog condition that fires when channel
members' delivery cursors fall behind the channel head by a configurable
message-count threshold.

**Architecture:** Follows the established watchdog condition pattern — enum
value in api, sealed AlertContext record in api, evaluation method in
WatchdogEvaluationService, switch case in ConnectorAlertBridge. Uses
existing `thresholdCount` on Watchdog (no schema changes).

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/H2 (tests)

## Global Constraints

- No Flyway migration — uses existing Watchdog fields
- No new dependencies
- Blocking stack only — reactive parity deferred
- IntelliJ MCP for all source file operations

---

### Task 1: API types — enum, context record, sealed permit

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogConditionType.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/DeliveryLagContext.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java`

**Interfaces:**
- Consumes: nothing
- Produces: `WatchdogConditionType.DELIVERY_LAG`, `DeliveryLagContext(UUID channelId, String channelName, List<LagDetail> laggingMembers, long latestMessageId)`, `DeliveryLagContext.LagDetail(String memberId, long lastDeliveredId, long lag)`

- [ ] **Step 1: Add DELIVERY_LAG to WatchdogConditionType**

Use `ide_replace_text_in_file` to add `DELIVERY_LAG` after `CIRCULAR_DELEGATION`:

```java
CIRCULAR_DELEGATION,
DELIVERY_LAG;
```

(Replace `CIRCULAR_DELEGATION;` with `CIRCULAR_DELEGATION,\n    DELIVERY_LAG;`)

- [ ] **Step 2: Create DeliveryLagContext record**

Use `ide_create_file` to create `api/src/main/java/io/casehub/qhorus/api/watchdog/DeliveryLagContext.java`:

```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;
import java.util.UUID;

public record DeliveryLagContext(
        UUID channelId,
        String channelName,
        List<LagDetail> laggingMembers,
        long latestMessageId
) implements AlertContext {

    public record LagDetail(String memberId, long lastDeliveredId, long lag) {}

    @Override
    public WatchdogConditionType conditionType() {
        return WatchdogConditionType.DELIVERY_LAG;
    }
}
```

- [ ] **Step 3: Add DeliveryLagContext to AlertContext sealed permits**

Use `ide_replace_text_in_file` on `AlertContext.java` to add `DeliveryLagContext` to the permits list:

```java
permits BarrierStuckContext, ApprovalPendingContext,
        AgentStaleContext, ChannelIdleContext, QueueDepthContext,
        ContextPressureContext,
        LoopDetectedContext, ObligationFanOutContext,
        ConversationStallContext, EchoChamberContext,
        CircularDelegationContext, DeliveryLagContext {
```

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api -q`
Expected: compiles clean

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogConditionType.java
git add api/src/main/java/io/casehub/qhorus/api/watchdog/DeliveryLagContext.java
git add api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java
git commit -m "feat(#381): add DELIVERY_LAG enum value and DeliveryLagContext record

Refs #381"
```

---

### Task 2: Evaluation logic — TDD in WatchdogEvaluationServiceTest

**Files:**
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`

**Interfaces:**
- Consumes: `WatchdogConditionType.DELIVERY_LAG`, `DeliveryLagContext`, `DeliveryLagContext.LagDetail`
- Produces: `WatchdogEvaluationService.evaluateDeliveryLag(Watchdog, Instant)` (private, called from `evaluateAll()` switch)

- [ ] **Step 1: Add ChannelMembershipStore injection to test class**

The test class already injects `ChannelStore`, `MessageStore`, `WatchdogStore`,
`CommitmentStore`, `InstanceStore`. Add `ChannelMembershipStore`:

```java
@Inject
io.casehub.qhorus.api.store.ChannelMembershipStore channelMembershipStore;
```

- [ ] **Step 2: Write failing test — happy path (one member lagging)**

```java
@Test
@TestTransaction
void evaluateDeliveryLag_fires_whenMemberBehindThreshold() {
    String suffix = UUID.randomUUID().toString().substring(0, 8);
    Channel ch = channelStore.put(Channel.builder("lag-fire-" + suffix)
            .semantic(ChannelSemantic.APPEND).trackDelivery(true).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-lag-" + suffix)
            .semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.DELIVERY_LAG, ch.name())
            .thresholdCount(5).notificationChannel(notifCh.name()).createdBy("test").build());

    // Add 10 messages
    for (int i = 0; i < 10; i++) {
        messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
                .messageType(MessageType.STATUS).content("msg-" + i).build());
    }

    // Member with cursor at 3 (lag = 7, above threshold of 5)
    channelMembershipStore.put(new io.casehub.qhorus.api.channel.ChannelMembership(
            null, ch.id(), "observer-1", io.casehub.qhorus.api.channel.MemberRole.PARTICIPANT,
            null, Instant.now(), null, 3L));

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertFalse(alerts.isEmpty(), "DELIVERY_LAG should fire when member is behind threshold");
    assertTrue(alerts.get(0).content().contains("DELIVERY_LAG"));
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WatchdogEvaluationServiceTest#evaluateDeliveryLag_fires_whenMemberBehindThreshold -pl runtime`
Expected: FAIL — no `DELIVERY_LAG` case in switch (compilation error or unmatched enum)

- [ ] **Step 4: Write failing test — no fire when all caught up**

```java
@Test
@TestTransaction
void evaluateDeliveryLag_noFire_whenAllCaughtUp() {
    String suffix = UUID.randomUUID().toString().substring(0, 8);
    Channel ch = channelStore.put(Channel.builder("lag-ok-" + suffix)
            .semantic(ChannelSemantic.APPEND).trackDelivery(true).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-lag-ok-" + suffix)
            .semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.DELIVERY_LAG, ch.name())
            .thresholdCount(5).notificationChannel(notifCh.name()).createdBy("test").build());

    for (int i = 0; i < 10; i++) {
        messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
                .messageType(MessageType.STATUS).content("msg-" + i).build());
    }

    Long latestId = messageStore.findLastMessage(ch.id()).map(Message::id).orElseThrow();
    channelMembershipStore.put(new io.casehub.qhorus.api.channel.ChannelMembership(
            null, ch.id(), "observer-1", io.casehub.qhorus.api.channel.MemberRole.PARTICIPANT,
            null, Instant.now(), null, latestId));

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "DELIVERY_LAG should not fire when member is caught up");
}
```

- [ ] **Step 5: Write failing test — skip when tracking disabled**

```java
@Test
@TestTransaction
void evaluateDeliveryLag_skips_whenTrackingDisabled() {
    String suffix = UUID.randomUUID().toString().substring(0, 8);
    Channel ch = channelStore.put(Channel.builder("lag-notrack-" + suffix)
            .semantic(ChannelSemantic.APPEND).build());  // trackDelivery defaults to null/off for APPEND
    Channel notifCh = channelStore.put(Channel.builder("notif-lag-notrack-" + suffix)
            .semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.DELIVERY_LAG, ch.name())
            .thresholdCount(5).notificationChannel(notifCh.name()).createdBy("test").build());

    for (int i = 0; i < 10; i++) {
        messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
                .messageType(MessageType.STATUS).content("msg-" + i).build());
    }

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "DELIVERY_LAG should skip channels without delivery tracking");
}
```

- [ ] **Step 6: Write failing test — null cursor treated as zero**

```java
@Test
@TestTransaction
void evaluateDeliveryLag_fires_whenCursorNull() {
    String suffix = UUID.randomUUID().toString().substring(0, 8);
    Channel ch = channelStore.put(Channel.builder("lag-null-" + suffix)
            .semantic(ChannelSemantic.APPEND).trackDelivery(true).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-lag-null-" + suffix)
            .semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.DELIVERY_LAG, ch.name())
            .thresholdCount(5).notificationChannel(notifCh.name()).createdBy("test").build());

    for (int i = 0; i < 10; i++) {
        messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
                .messageType(MessageType.STATUS).content("msg-" + i).build());
    }

    // Member with null cursor (never delivered)
    channelMembershipStore.put(new io.casehub.qhorus.api.channel.ChannelMembership(
            null, ch.id(), "observer-1", io.casehub.qhorus.api.channel.MemberRole.PARTICIPANT,
            null, Instant.now(), null, null));

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertFalse(alerts.isEmpty(), "DELIVERY_LAG should fire when cursor is null (never delivered)");
}
```

- [ ] **Step 7: Write failing test — empty channel does not fire**

```java
@Test
@TestTransaction
void evaluateDeliveryLag_noFire_whenChannelEmpty() {
    String suffix = UUID.randomUUID().toString().substring(0, 8);
    Channel ch = channelStore.put(Channel.builder("lag-empty-" + suffix)
            .semantic(ChannelSemantic.APPEND).trackDelivery(true).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-lag-empty-" + suffix)
            .semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.DELIVERY_LAG, ch.name())
            .thresholdCount(5).notificationChannel(notifCh.name()).createdBy("test").build());

    channelMembershipStore.put(new io.casehub.qhorus.api.channel.ChannelMembership(
            null, ch.id(), "observer-1", io.casehub.qhorus.api.channel.MemberRole.PARTICIPANT,
            null, Instant.now(), null, null));

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "DELIVERY_LAG should not fire on empty channel");
}
```

- [ ] **Step 8: Implement evaluateDeliveryLag()**

Add to `WatchdogEvaluationService`:

```java
private boolean evaluateDeliveryLag(Watchdog w, Instant now) {
    int threshold = w.thresholdCount() != null ? w.thresholdCount() : 50;

    List<Channel> channels = crossTenantChannelStore.listAll().stream()
            .filter(ch -> "*".equals(w.targetName()) || ch.name().equals(w.targetName()))
            .filter(ch -> io.casehub.qhorus.runtime.channel.ChannelService.isDeliveryTrackingEnabled(ch))
            .toList();

    boolean fired = false;
    for (Channel ch : channels) {
        Optional<Message> head = crossTenantMessageStore.findLastMessage(ch.id());
        if (head.isEmpty()) { continue; }
        long latestId = head.get().id();

        List<io.casehub.qhorus.api.channel.ChannelMembership> members =
                channelMembershipStore.findByChannel(ch.id());

        List<DeliveryLagContext.LagDetail> lagging = members.stream()
                .map(m -> {
                    long delivered = m.lastDeliveredMessageId() != null ? m.lastDeliveredMessageId() : 0L;
                    long lag = latestId - delivered;
                    return new DeliveryLagContext.LagDetail(m.memberId(), delivered, lag);
                })
                .filter(d -> d.lag() >= threshold)
                .toList();

        if (!lagging.isEmpty()) {
            String summary = "DELIVERY_LAG: " + lagging.size()
                    + " participant(s) lagging on '" + ch.name() + "'";
            fireAlert(w, summary,
                    new DeliveryLagContext(ch.id(), ch.name(), lagging, latestId), now);
            fired = true;
        }
    }
    return fired;
}
```

- [ ] **Step 9: Add DELIVERY_LAG case to evaluateAll() switch**

Use `ide_replace_text_in_file` to add the case:

```java
case CIRCULAR_DELEGATION -> evaluateCircularDelegation(w, now);
case DELIVERY_LAG -> evaluateDeliveryLag(w, now);
```

Add the import:

```java
import io.casehub.qhorus.api.watchdog.DeliveryLagContext;
```

- [ ] **Step 10: Run all five tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WatchdogEvaluationServiceTest -pl runtime`
Expected: all tests PASS (including existing tests)

- [ ] **Step 11: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java
git add runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java
git commit -m "feat(#381): evaluateDeliveryLag() with 5 TDD tests

Fires when any member's lastDeliveredMessageId cursor falls behind
the channel head by >= thresholdCount messages. Skips channels without
delivery tracking. Null cursor treated as 0.

Refs #381"
```

---

### Task 3: ConnectorAlertBridge, MCP tool description, CLAUDE.md

**Files:**
- Modify: `connectors/src/main/java/io/casehub/qhorus/connectors/ConnectorAlertBridge.java`
- Modify: `connectors/src/test/java/io/casehub/qhorus/connectors/ConnectorAlertBridgeTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: `DeliveryLagContext`, `DeliveryLagContext.LagDetail`
- Produces: formatted alert body string, updated MCP tool description

- [ ] **Step 1: Write failing test for ConnectorAlertBridge body formatting**

Read `ConnectorAlertBridgeTest.java` to understand the test pattern, then add:

```java
@Test
void buildBody_deliveryLag() {
    var context = new DeliveryLagContext(
            UUID.randomUUID(), "test-channel",
            List.of(new DeliveryLagContext.LagDetail("agent-a", 100L, 50L),
                    new DeliveryLagContext.LagDetail("agent-b", 80L, 70L)),
            150L);
    var event = new WatchdogAlertEvent(UUID.randomUUID(), "test-channel",
            "alerts", "DELIVERY_LAG: 2 participant(s) lagging on 'test-channel'",
            Instant.now(), context);
    String body = bridge.buildBody(event);
    assertTrue(body.contains("DELIVERY_LAG"));
    assertTrue(body.contains("agent-a (lag: 50)"));
    assertTrue(body.contains("agent-b (lag: 70)"));
    assertTrue(body.contains("Channel head: 150"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConnectorAlertBridgeTest#buildBody_deliveryLag -pl connectors`
Expected: FAIL — switch does not cover `DeliveryLagContext`

- [ ] **Step 3: Add DeliveryLagContext case to ConnectorAlertBridge.buildBody()**

Use `ide_replace_text_in_file` to add after the `CircularDelegationContext` case:

```java
case DeliveryLagContext c -> event.summary()
                              + "\nChannel: " + c.channelName()
                              + "\nLagging members: " + c.laggingMembers().stream()
                                      .map(d -> d.memberId() + " (lag: " + d.lag() + ")")
                                      .collect(java.util.stream.Collectors.joining(", "))
                              + "\nChannel head: " + c.latestMessageId();
```

Add import:

```java
import io.casehub.qhorus.api.watchdog.DeliveryLagContext;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ConnectorAlertBridgeTest -pl connectors`
Expected: PASS

- [ ] **Step 5: Update register_watchdog MCP tool description**

Use `ide_replace_text_in_file` in `QhorusMcpTools.java` to add `DELIVERY_LAG`
to the condition_type description string and the `@ToolArg` enum list.

In the `@Tool` description, replace:
```
CONTEXT_PRESSURE, LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER.
```
with:
```
CONTEXT_PRESSURE, LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER, DELIVERY_LAG.
```

In the `@ToolArg` for `condition_type`, replace:
```
CONVERSATION_STALL | ECHO_CHAMBER
```
with:
```
CONVERSATION_STALL | ECHO_CHAMBER | DELIVERY_LAG
```

- [ ] **Step 6: Update CLAUDE.md**

Add documentation for the new condition type in the watchdog conventions
section. Reference the threshold semantics (count-based, default 50),
tracking gate, and alert context fields.

- [ ] **Step 7: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all tests pass (including ToolOverloadDiscoverabilityTest)

- [ ] **Step 8: Commit**

```bash
git add connectors/src/main/java/io/casehub/qhorus/connectors/ConnectorAlertBridge.java
git add connectors/src/test/java/io/casehub/qhorus/connectors/ConnectorAlertBridgeTest.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java
git add CLAUDE.md
git commit -m "feat(#381): ConnectorAlertBridge, MCP tool, CLAUDE.md for DELIVERY_LAG

Adds buildBody() case for DeliveryLagContext, updates register_watchdog
condition_type list, documents the new condition in CLAUDE.md.

Closes #381"
```

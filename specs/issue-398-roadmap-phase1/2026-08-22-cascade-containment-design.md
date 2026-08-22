# Cascade Containment — Design Spec

**Issue:** #399 — E2: Cascade containment — watchdog-triggered quarantine
**Scale:** S | **Complexity:** Low
**Date:** 2026-08-22

---

## Problem

Watchdog conditions detect pathologies (loops, stalls, echo chambers, fan-out, circular delegation) but take no action beyond alerting. A compromised or malfunctioning agent can cascade failures across channels while the watchdog merely reports it. Detection without containment is insufficient.

## Solution

Add an action policy per watchdog registration. When a condition fires, the watchdog executes the configured action using existing containment primitives (channel pause, backend deregistration) — no new containment mechanisms needed.

### 1. `WatchdogAction` enum

New enum in `api/watchdog/`:

```java
public enum WatchdogAction {
    ALERT,             // current behavior — notify only
    PAUSE_CHANNEL,     // pause the affected channel
    DEREGISTER_AGENT,  // deregister affected agent(s) from the channel gateway
    QUARANTINE         // pause + deregister + containment EVENT
}
```

### 2. `Watchdog` record gains `action` field

Add `WatchdogAction action` to the `Watchdog` record (nullable — null treated as `ALERT`). The `WatchdogEntity` JPA entity gains a `VARCHAR(32)` column mapped via `@Enumerated(STRING)`.

**V42 migration:** `ALTER TABLE watchdog ADD COLUMN action VARCHAR(32)` — nullable, no default (existing rows are ALERT by semantic default).

**MCP tool:** `register_watchdog` gains an `action` parameter (optional, defaults to ALERT). `WatchdogSummary` DTO gains an `action` field.

### 3. `AlertContext.affectedAgentIds()` — agent extraction

New default method on the `AlertContext` sealed interface:

```java
default List<String> affectedAgentIds() { return List.of(); }
```

Overridden by agent-specific contexts:
- `LoopDetectedContext` → `List.of(sender)`
- `AgentStaleContext` → `staleInstanceIds`
- `ContextPressureContext` → `List.of(actorId)`
- `EchoChamberContext` → `participants`

Channel-level contexts (BARRIER_STUCK, CHANNEL_IDLE, QUEUE_DEPTH, OBLIGATION_FAN_OUT, CONVERSATION_STALL, CIRCULAR_DELEGATION, DELIVERY_LAG) return empty list. When DEREGISTER_AGENT is configured on a channel-level condition, the containment logs a warning and falls back to ALERT.

### 4. Containment execution in `WatchdogEvaluationService`

New private method called from `fireAlert()` after the alert dispatches:

```java
private void executeContainmentAction(Watchdog w, AlertContext context, UUID channelId) {
    WatchdogAction action = w.action() != null ? w.action() : WatchdogAction.ALERT;
    if (action == WatchdogAction.ALERT) return;

    // PAUSE_CHANNEL or QUARANTINE → pause the channel
    if (action == WatchdogAction.PAUSE_CHANNEL || action == WatchdogAction.QUARANTINE) {
        channelService.pause(channelId);
        commitmentService.expireByChannel(channelId);
    }

    // DEREGISTER_AGENT or QUARANTINE → deregister affected agents
    if (action == WatchdogAction.DEREGISTER_AGENT || action == WatchdogAction.QUARANTINE) {
        List<String> agents = context.affectedAgentIds();
        if (agents.isEmpty()) {
            // Channel-level condition — no agent to deregister
            LOG.warn("DEREGISTER_AGENT on channel-level condition {} — falling back to ALERT",
                     w.conditionType());
        } else {
            for (String agentId : agents) {
                channelGateway.deregisterBackend(channelId, agentId);
            }
        }
    }

    // Containment audit EVENT
    dispatchContainmentEvent(w, context, channelId, action);
}
```

**Channel ID resolution:** The existing `evaluate*()` methods already resolve the channel. For wildcard watchdogs (`targetName = "*"`), the evaluate method iterates all channels and fires per-channel. The `channelId` is passed through `fireAlert()` to `executeContainmentAction()`.

**`fireAlert()` signature change:** Add `UUID channelId` parameter. Currently `fireAlert` doesn't receive the channel UUID — it has the watchdog's `targetName` (a name or `*`) and `notificationChannel` (name). The evaluate methods already resolve the channel to its UUID for ledger queries. Thread the UUID through.

### 5. `CommitmentService.expireByChannel(UUID channelId)`

New method on `CommitmentService`:

```java
public void expireByChannel(UUID channelId) {
    List<Commitment> active = commitmentStore.findOpenByChannelId(channelId);
    Instant now = Instant.now();
    for (Commitment c : active) {
        c.expiresAt = now;
        commitmentStore.update(c);
        expiredEvents.fire(new CommitmentExpiredEvent(
                c.id, c.correlationId, channelId, c.obligor, c.requester, now));
    }
}
```

Requires `CommitmentStore.findOpenByChannelId(UUID)` — new method returning all OPEN/ACKNOWLEDGED commitments for a channel. The `InMemoryCommitmentStore` also needs this method.

`CommitmentExpiredEvent` already flows through `NotificationBridgeObserver` → `QhorusObligationEvent(Kind.EXPIRED)` → subscription engine. No changes to the notification bridge.

### 6. Containment audit EVENT

After executing any non-ALERT action, dispatch an EVENT to the notification channel:

```java
private void dispatchContainmentEvent(Watchdog w, AlertContext context,
                                       UUID channelId, WatchdogAction action) {
    String telemetry = telemetryJson(
            "containment_action", action.name(),
            "condition_type", w.conditionType().name(),
            "affected_agents", context.affectedAgentIds().toString(),
            "channel_id", channelId.toString(),
            "watchdog_id", w.id().toString());
    messageService.dispatch(MessageDispatch.builder()
            .channelId(notifChannelId)  // resolved from w.notificationChannel()
            .sender("system:watchdog")
            .type(MessageType.EVENT)
            .telemetry(telemetry)
            .actorType(ActorType.SYSTEM)
            .tenancyId(w.tenancyId())
            .build());
}
```

The `telemetryJson` helper is inherited from `QhorusMcpToolsBase`. Since `WatchdogEvaluationService` doesn't extend that class, either inject `ObjectMapper` directly or use a static utility. The existing `ConnectorAlertBridge` pattern shows how non-MCP classes handle JSON — use `ObjectMapper` directly.

### 7. `ConnectorAlertBridge` update

The `ConnectorAlertBridge` switch on `WatchdogConditionType` needs no change — it observes `WatchdogAlertEvent`, which still fires for all actions (alerting happens before containment). The containment is transparent to existing alert observers.

## What doesn't change

- Watchdog condition evaluation logic — same detection, new response
- `WatchdogScheduler` — still calls `evaluateAll()` on schedule
- Alert dispatch — `WatchdogAlertEvent` CDI event still fires for all actions
- Notification bridge — `CommitmentExpiredEvent` already handled
- `ChannelService.pause()` — used as-is
- `ChannelGateway.deregisterBackend()` — used as-is

## Testing

**Unit tests (`WatchdogEvaluationServiceTest` — CDI-free with mocks):**
- `executeContainmentAction_alert_noOp` — ALERT action does nothing
- `executeContainmentAction_pauseChannel_pausesAndExpiresCommitments` — verifies pause + expire
- `executeContainmentAction_deregisterAgent_deregistersFromGateway` — verifies deregister for agent-specific context
- `executeContainmentAction_deregisterAgent_channelLevelCondition_fallsBackToAlert` — empty agent list, logs warning
- `executeContainmentAction_quarantine_pausesDeregistersAndEmitsEvent` — full quarantine flow
- `executeContainmentAction_quarantine_dispatchesContainmentEvent` — EVENT with telemetry

**CommitmentService tests (`CommitmentServiceTest`):**
- `expireByChannel_expiresOpenAndAcknowledged` — transitions to EXPIRED, fires events
- `expireByChannel_noActiveCommitments_noOp` — empty channel
- `expireByChannel_terminalCommitments_skipped` — already FULFILLED/FAILED not affected

**Integration tests:**
- `register_watchdog` with `action=QUARANTINE` — persists action field
- End-to-end: register LOOP_DETECTED watchdog with QUARANTINE action, trigger loop, verify channel paused + agent deregistered + commitments expired + containment EVENT in ledger

**AlertContext tests:**
- `affectedAgentIds` returns correct values for each subtype
- Default method returns empty list for channel-level contexts

## References

- `WatchdogEvaluationService.java:288-315` — existing `fireAlert()` method
- `WatchdogAlertEvent.java` — existing alert event
- `AlertContext.java` — sealed interface hierarchy
- `ChannelService.java:211` — existing `pause()` method
- `ChannelGateway.java:188` — existing `deregisterBackend()`
- `CommitmentService.java:296` — existing `expireOverdue()` pattern
- `CommitmentExpiredEvent.java` — existing CDI event
- `notification-bridge/` — `CommitmentEventNotifier` routes expired events
- `QhorusMcpToolsBase.telemetryJson()` — telemetry JSON pattern
- `docs/roadmap-epics-2026.md` — Epic 2 definition
- GE-20260529-bfa5d5 — WatchdogAlertEvent carries no correlationId
- Issue #399

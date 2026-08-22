# Cascade Containment — Design Spec

**Issue:** #399 — E2: Cascade containment — watchdog-triggered quarantine
**Scale:** S | **Complexity:** Low
**Date:** 2026-08-22

---

## Problem

Watchdog conditions detect pathologies (loops, stalls, echo chambers, fan-out, circular delegation) but take no action beyond alerting. A compromised or malfunctioning agent can cascade failures across channels while the watchdog merely reports it. Detection without containment is insufficient.

## Solution

Add an action policy per watchdog registration. When a condition fires, the watchdog executes the configured action using existing containment primitives (channel pause, instance deregistration) — no new containment mechanisms needed.

### 1. `WatchdogAction` enum

New enum in `api/watchdog/`:

```java
public enum WatchdogAction {
    ALERT,             // current behavior — notify only
    PAUSE_CHANNEL,     // pause the affected channel
    DEREGISTER_AGENT,  // deregister affected agent(s) from the instance registry (mark offline)
    QUARANTINE         // pause + deregister + containment EVENT
}
```

**DEREGISTER_AGENT semantics:** Deregisters affected agents from the **instance registry** via `InstanceService.updateStatus(instanceId, OFFLINE)` — NOT `ChannelGateway.deregisterBackend()`. Agent instance IDs (from AlertContext) are not backend IDs. Backend deregistration targets transport-level backends ("a2a", "slack"); agent containment targets the instance level. An offline instance cannot be routed to by capability, is excluded from AllowedWritersPolicy checks, and its capabilities are removed from discovery.

### 2. `Watchdog` record gains `action` field

Add `WatchdogAction action` to the `Watchdog` record. Non-nullable — defaults to `ALERT` in the compact constructor (same normalization pattern as list fields):

```java
public Watchdog {
    action = action != null ? action : WatchdogAction.ALERT;
}
```

The `WatchdogEntity` JPA entity gains a `VARCHAR(32)` column mapped via `@Enumerated(STRING)`. The `Watchdog.Builder` gains an `action(WatchdogAction)` setter. The `WatchdogSummary` DTO gains an `action` field. The `toBuilder()` method passes `action` through.

**Migration:** Next available Flyway version (verify against `db/qhorus/migration/` — may be V42 or later). `ALTER TABLE watchdog ADD COLUMN action VARCHAR(32)` — nullable in DB (existing rows read as null → normalized to ALERT by the compact constructor).

**MCP tool:** `register_watchdog` gains an `action` parameter (optional, defaults to ALERT). The `WatchdogEntity.toDomain()` mapping passes the column value through `WatchdogAction.valueOf()` with null → ALERT default.

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
- `CircularDelegationContext` → delegation chain participants (if available)
- `ObligationFanOutContext` → obligor IDs (if available)

Channel-level contexts without agent identification (BARRIER_STUCK, CHANNEL_IDLE, QUEUE_DEPTH, DELIVERY_LAG, APPROVAL_PENDING, CONVERSATION_STALL) return empty list.

**Sealed interface note:** The `AlertContext` sealed interface's `permits` clause controls which subtypes exist. A new context type that forgets to override `affectedAgentIds()` gets the safe default (empty list). The default method is intentional — not all contexts can identify specific agents.

### 4. Containment execution in `WatchdogEvaluationService`

New private method called from `fireAlert()` after the alert dispatches:

```java
private void executeContainmentAction(Watchdog w, AlertContext context, UUID channelId) {
    if (w.action() == WatchdogAction.ALERT) return;

    try {
        // PAUSE_CHANNEL or QUARANTINE → pause the channel
        if (w.action() == WatchdogAction.PAUSE_CHANNEL || w.action() == WatchdogAction.QUARANTINE) {
            if (channelId == null) {
                LOG.warn("PAUSE_CHANNEL on cross-channel condition {} — no channelId, skipping",
                         w.conditionType());
                return;
            }
            // Exclude notification channel from containment scope
            Optional<Channel> notifCh = crossTenantChannelStore
                    .findByNameAndTenancy(w.notificationChannel(), w.tenancyId());
            if (notifCh.isPresent() && notifCh.get().id().equals(channelId)) {
                LOG.warn("Skipping containment on notification channel {} — self-defeating",
                         w.notificationChannel());
                return;
            }
            channelService.pause(channelId);
            commitmentService.expireByChannel(channelId, w.tenancyId());
        }

        // DEREGISTER_AGENT or QUARANTINE → deregister affected agents
        if (w.action() == WatchdogAction.DEREGISTER_AGENT || w.action() == WatchdogAction.QUARANTINE) {
            List<String> agents = context.affectedAgentIds();
            if (agents.isEmpty()) {
                LOG.warn("DEREGISTER_AGENT on condition {} with no identified agents — skipping",
                         w.conditionType());
            } else {
                for (String agentId : agents) {
                    instanceService.updateStatus(agentId, InstanceStatus.OFFLINE);
                }
            }
        }

        // Containment audit EVENT
        dispatchContainmentEvent(w, context, channelId, w.action());

    } catch (Exception e) {
        LOG.error("Containment action {} failed for watchdog {} — alert was still sent",
                  w.action(), w.id(), e);
    }
}
```

**Key design decisions in the execution logic:**
- **Error isolation:** Containment is wrapped in try-catch. A failure does not suppress the alert (already fired) or roll back the evaluation cycle.
- **Notification channel exclusion:** A wildcard watchdog with PAUSE_CHANNEL would pause every channel including the notification channel, silencing future alerts. The notification channel is explicitly excluded.
- **Cross-channel conditions:** AGENT_STALE and APPROVAL_PENDING don't carry a `channelId`. If `channelId` is null, PAUSE_CHANNEL skips with a warning. DEREGISTER_AGENT still works (instance-level, not channel-scoped).

**`fireAlert()` signature change:** Add `UUID channelId` parameter. The `channelId` is already resolved in most `evaluate*()` methods for ledger queries. Thread it through. For cross-channel conditions (AGENT_STALE, APPROVAL_PENDING) where the watchdog fires once across channels, pass `null` — containment handles this with the null guard.

### 5. `CommitmentService.expireByChannel(UUID channelId, String tenancyId)`

New method on `CommitmentService`:

```java
public void expireByChannel(UUID channelId, String tenancyId) {
    List<Commitment> active = commitmentStore.findOpenByChannelId(channelId);
    Instant now = Instant.now();
    for (Commitment c : active) {
        if (c.state().isTerminal()) continue;
        Commitment expired = c.toBuilder().expiresAt(now).state(CommitmentState.EXPIRED).build();
        commitmentStore.save(expired);
        expiredEvents.fire(new CommitmentExpiredEvent(
                c.id(), c.correlationId(), channelId, c.obligor(), c.requester(), now));
    }
}
```

Uses `Commitment.toBuilder()` to create a modified copy (Commitment is a record — immutable). `commitmentStore.save()` persists the updated state, following the existing pattern in `CommitmentService.expireOverdue()`.

Requires `CommitmentStore.findOpenByChannelId(UUID)` — if it doesn't already exist, add it. Returns all OPEN/ACKNOWLEDGED commitments for a channel. The `InMemoryCommitmentStore` also needs this method.

`CommitmentExpiredEvent` already flows through `NotificationBridgeObserver` → `QhorusObligationEvent(Kind.EXPIRED)` → subscription engine. No changes to the notification bridge.

### 6. Containment audit EVENT

After executing any non-ALERT action, dispatch an EVENT to the notification channel with structured telemetry:

```java
private void dispatchContainmentEvent(Watchdog w, AlertContext context,
                                       UUID channelId, WatchdogAction action) {
    Optional<Channel> notifChannel = crossTenantChannelStore
            .findByNameAndTenancy(w.notificationChannel(), w.tenancyId());
    if (notifChannel.isEmpty()) return;

    ObjectMapper mapper = new ObjectMapper();
    ObjectNode telemetry = mapper.createObjectNode();
    telemetry.put("containment_action", action.name());
    telemetry.put("condition_type", w.conditionType().name());
    telemetry.put("watchdog_id", w.id().toString());
    if (channelId != null) telemetry.put("channel_id", channelId.toString());
    telemetry.set("affected_agents", mapper.valueToTree(context.affectedAgentIds()));

    messageService.dispatch(MessageDispatch.builder()
            .channelId(notifChannel.get().id())
            .sender("system:watchdog")
            .type(MessageType.EVENT)
            .telemetry(telemetry.toString())
            .actorType(ActorType.SYSTEM)
            .tenancyId(w.tenancyId())
            .build());
}
```

Uses injected `ObjectMapper` directly (not `telemetryJson()` which is on `QhorusMcpToolsBase`). Dispatches to the notification channel — this is the audit/oversight channel where all watchdog alerts go.

### 7. `ConnectorAlertBridge` update

The `ConnectorAlertBridge` switch on `WatchdogConditionType` needs no change — it observes `WatchdogAlertEvent`, which still fires for all actions (alerting happens before containment). The containment is transparent to existing alert observers.

## What doesn't change

- Watchdog condition evaluation logic — same detection, new response
- `WatchdogScheduler` — still calls `evaluateAll()` on schedule
- Alert dispatch — `WatchdogAlertEvent` CDI event still fires for all actions
- Notification bridge — `CommitmentExpiredEvent` already handled
- `ChannelService.pause()` — used as-is
- `ChannelGateway.deregisterBackend()` — NOT used; containment uses instance-level deregistration

## Testing

**Unit tests (`WatchdogContainmentTest` — CDI-free with mocks):**
- `executeContainmentAction_alert_noOp` — ALERT action does nothing
- `executeContainmentAction_pauseChannel_pausesAndExpiresCommitments` — verifies pause + expire
- `executeContainmentAction_deregisterAgent_marksInstanceOffline` — verifies instance deregistration
- `executeContainmentAction_deregisterAgent_noAgentIds_skips` — empty agent list, logs warning
- `executeContainmentAction_quarantine_pausesDeregistersAndEmitsEvent` — full quarantine flow
- `executeContainmentAction_notificationChannelExcluded` — notification channel not paused
- `executeContainmentAction_nullChannelId_skipsPause` — cross-channel condition, pause skipped
- `executeContainmentAction_containmentFailure_doesNotRollBack` — exception caught, alert was sent
- `executeContainmentAction_dispatchesContainmentEvent` — EVENT with structured telemetry

**CommitmentService tests (`CommitmentServiceTest`):**
- `expireByChannel_expiresOpenAndAcknowledged` — transitions to EXPIRED, fires events
- `expireByChannel_noActiveCommitments_noOp` — empty channel
- `expireByChannel_terminalCommitments_skipped` — already FULFILLED/FAILED not affected

**Integration tests:**
- `register_watchdog` with `action=QUARANTINE` — persists action field, round-trips through MCP
- End-to-end: register LOOP_DETECTED watchdog with QUARANTINE action, trigger loop, verify channel paused + agent offline + commitments expired + containment EVENT in ledger

**AlertContext tests:**
- `affectedAgentIds` returns correct values for each overriding subtype
- Default method returns empty list for channel-level contexts

## References

- `WatchdogEvaluationService.java:288-315` — existing `fireAlert()` method
- `WatchdogAlertEvent.java` — existing alert event
- `AlertContext.java` — sealed interface hierarchy
- `ChannelService.java:211` — existing `pause()` method
- `InstanceService.java` — existing instance registry management
- `CommitmentService.java:296` — existing `expireOverdue()` pattern
- `CommitmentExpiredEvent.java` — existing CDI event
- `notification-bridge/` — `CommitmentEventNotifier` routes expired events
- `docs/roadmap-epics-2026.md` — Epic 2 definition
- GE-20260529-bfa5d5 — WatchdogAlertEvent carries no correlationId
- Issue #399

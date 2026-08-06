# DELIVERY_LAG Watchdog Condition Design

**Issue:** casehubio/qhorus#381
**Date:** 2026-08-06
**Status:** Approved

## Problem

Qhorus has no early warning for delivery failures. When an agent's webhook
returns 503s or an SSE stream drops, nothing fires until the coordination
layer detects a symptom — BARRIER_STUCK (all contributors haven't responded)
or CONVERSATION_STALL (commitments stale). By that point the entire flow
is blocked and recovery is harder.

The gap: no condition detects "messages exist but aren't reaching
participants" as a distinct, earlier signal.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Threshold semantics | Count-only (`thresholdCount`) | Measures the delivery gap directly. Time-based conflates channel inactivity with delivery failure. Uses existing `Watchdog.thresholdCount` field — no schema change. |
| Alert granularity | Per-channel aggregate | One alert per channel with a list of lagging members. Matches BARRIER_STUCK/OBLIGATION_FAN_OUT pattern. Avoids alert storms when multiple participants lag simultaneously. |
| Tracking disabled | Skip silently | Consistent with BARRIER_STUCK delivery enrichment. No meta-alerts for configuration issues. `register_watchdog` docs note the prerequisite. |
| Cross-condition suppression | None | DELIVERY_LAG (cause) and BARRIER_STUCK (symptom) firing together is diagnostic signal, not noise. No other conditions suppress each other. Debounce handles repeated firing. |

## Architecture

### API layer — `api/src/main/java/io/casehub/qhorus/api/watchdog/`

**`WatchdogConditionType`** — add `DELIVERY_LAG` to the enum.

**`DeliveryLagContext`** — new sealed record implementing `AlertContext`:

```java
public record DeliveryLagContext(
        UUID channelId,
        String channelName,
        List<LagDetail> laggingMembers,
        int thresholdCount
) implements AlertContext {

    public record LagDetail(String memberId, long lagCount) {}

    @Override
    public WatchdogConditionType conditionType() {
        return WatchdogConditionType.DELIVERY_LAG;
    }
}
```

**`AlertContext`** — add `DeliveryLagContext` to the sealed permits list.

### Runtime layer — `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/`

**`WatchdogEvaluationService.evaluateDeliveryLag(Watchdog, Instant)`**:

1. Read `thresholdCount` from watchdog (default: 10).
2. List channels matching `targetName` (or all if `*`).
3. For each channel where `isDeliveryTrackingEnabled(ch)` is true:
   a. Get channel head: `crossTenantMessageStore.findLastMessage(channelId)` → `headId`.
   b. If no messages, skip (nothing to lag behind).
   c. Get all memberships: `channelMembershipStore.findByChannel(channelId)`.
   d. For each member: compute `lag = headId - lastDeliveredMessageId`.
      Null cursor = treat as full lag (`lag = headId`).
   e. Collect members where `lag >= thresholdCount`.
   f. If any: fire alert with `DeliveryLagContext` carrying the list.

Add `case DELIVERY_LAG -> evaluateDeliveryLag(w, now);` to the switch.

### MCP tool — `register_watchdog`

Add `DELIVERY_LAG` to the condition_type description and enum listing.
`thresholdCount` = message lag threshold (default 10 in evaluation).
`targetName` = channel name or `*`.

### Connector bridge — `ConnectorAlertBridge`

Add case:
```java
case DeliveryLagContext c -> event.summary()
    + "\nChannel: " + c.channelName()
    + "\nLagging members: " + c.laggingMembers().stream()
        .map(l -> l.memberId() + " (" + l.lagCount() + " behind)")
        .collect(Collectors.joining(", "))
    + "\nThreshold: " + c.thresholdCount();
```

### Notification bridge — `notification-bridge/`

`NotificationBridgeObserver` and `CommitmentEventNotifier` do not need
changes — they observe commitment lifecycle events, not watchdog alerts.
The watchdog alert path is `WatchdogAlertEvent` → `ConnectorAlertBridge`,
which is independent.

## Testing

CDI-free unit tests in `WatchdogEvaluationServiceTest` following the
existing pattern (mocked stores, direct method calls):

| Test | Asserts |
|------|---------|
| `deliveryLag_firesWhenLagExceedsThreshold` | Member 50 behind on threshold 10 → fires with correct LagDetail |
| `deliveryLag_doesNotFireBelowThreshold` | Member 5 behind on threshold 10 → no fire |
| `deliveryLag_skipsChannelWithoutTracking` | Channel without tracking → no fire |
| `deliveryLag_nullCursor_treatedAsFullLag` | New member with null cursor → lag = headId |
| `deliveryLag_aggregatesMultipleLaggingMembers` | 2 lagging, 1 current → fires with 2 in list |
| `deliveryLag_emptyChannel_skips` | No messages → no fire |

`ConnectorAlertBridgeTest` — add `DeliveryLagContext` body formatting test.

## Files Changed

| File | Change |
|------|--------|
| `api/.../watchdog/WatchdogConditionType.java` | Add `DELIVERY_LAG` |
| `api/.../watchdog/AlertContext.java` | Add `DeliveryLagContext` to permits |
| `api/.../watchdog/DeliveryLagContext.java` | New file |
| `runtime/.../watchdog/WatchdogEvaluationService.java` | Add `evaluateDeliveryLag()` + switch case |
| `runtime/.../mcp/QhorusMcpTools.java` | Update `register_watchdog` description |
| `connectors/.../ConnectorAlertBridge.java` | Add `DeliveryLagContext` case |
| `connectors/.../ConnectorAlertBridgeTest.java` | Add body formatting test |
| `runtime/.../watchdog/WatchdogEvaluationServiceTest.java` | Add 6 unit tests |
| `CLAUDE.md` | Document DELIVERY_LAG condition |

No Flyway migration — `Watchdog` record already has `thresholdCount`.
No new dependencies. No SPI changes.

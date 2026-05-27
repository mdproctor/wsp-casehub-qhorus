# WatchdogAlertEvent + ConnectorAlertBridge — Design Spec

**Issue:** casehubio/qhorus#200  
**Date:** 2026-05-27  
**Status:** Approved (rev 3 — incorporates round-2 spec review)

---

## Problem

`WatchdogEvaluationService.fireAlert()` dispatches a STATUS message to an internal Qhorus channel only. Human operators have no external notification path — stalled-obligation alerts, stuck barrier warnings, and approval-pending notices fire into channels only agents can see.

---

## Design

### Layer 1 — `WatchdogConditionType` enum and `AlertContext` sealed hierarchy (in `casehub-qhorus-api`)

**`WatchdogConditionType`** — closed enum replacing the open String on `Watchdog.conditionType`. Follows the `MessageType` precedent.

```java
package io.casehub.qhorus.api.watchdog;

public enum WatchdogConditionType {
    BARRIER_STUCK, APPROVAL_PENDING, AGENT_STALE, CHANNEL_IDLE, QUEUE_DEPTH
}
```

**`AlertContext`** — sealed hierarchy replacing `Map<String, String>`. Misspelled keys and missing required fields are compile-time errors, not silent runtime failures. Pattern-matching switches in observers are exhaustive.

```java
package io.casehub.qhorus.api.watchdog;

public sealed interface AlertContext
    permits BarrierStuckContext, ApprovalPendingContext,
            AgentStaleContext, ChannelIdleContext, QueueDepthContext {

    WatchdogConditionType conditionType();
}

public record BarrierStuckContext(
    UUID channelId,
    String channelName,
    List<String> missingContributors,
    long elapsedSeconds
) implements AlertContext {
    public WatchdogConditionType conditionType() { return WatchdogConditionType.BARRIER_STUCK; }
}

public record ApprovalPendingContext(
    long pendingCount,
    Instant oldestExpiryAt   // null if no expiresAt set on any pending commitment
) implements AlertContext {
    public WatchdogConditionType conditionType() { return WatchdogConditionType.APPROVAL_PENDING; }
}

public record AgentStaleContext(
    long staleCount,
    List<String> staleInstanceIds   // up to 10 instance IDs; may be shorter than staleCount
) implements AlertContext {
    public WatchdogConditionType conditionType() { return WatchdogConditionType.AGENT_STALE; }
}

public record ChannelIdleContext(
    List<String> channelNames,   // up to 3; may be fewer than total idle channels
    long thresholdSeconds        // configured idle threshold — not per-channel elapsed time
) implements AlertContext {
    public WatchdogConditionType conditionType() { return WatchdogConditionType.CHANNEL_IDLE; }
}

public record QueueDepthContext(
    String channelName,
    long messageCount,
    int threshold
) implements AlertContext {
    public WatchdogConditionType conditionType() { return WatchdogConditionType.QUEUE_DEPTH; }
}
```

The `Watchdog` entity retains `String conditionType` for database storage. `WatchdogEvaluationService` parses the string before constructing `WatchdogConditionType` in the evaluate methods.

---

### Layer 1 — `WatchdogAlertEvent` (in `casehub-qhorus-api`)

Plain record, follows `MessageReceivedEvent` / `ChannelInitialisedEvent` pattern: no CDI annotations, no framework dependencies, safe to import from any module.

```java
package io.casehub.qhorus.api.watchdog;

public record WatchdogAlertEvent(
    UUID watchdogId,
    String targetName,           // channel/instance being monitored ("*" = all)
    String notificationChannel,  // internal Qhorus channel name — enables topology-aware
                                 // routing in custom WatchdogAlertRouter implementations
    String summary,              // pre-formatted human-readable line for notification body
    Instant firedAt,             // copied from evaluateAll()'s outer `now` — consistent with
                                 // w.lastFiredAt and cutoff calculations in the same cycle
    AlertContext context          // sealed — type carries conditionType, no separate field needed
) {
    /** Convenience accessor — delegates to context.conditionType(). */
    public WatchdogConditionType conditionType() { return context.conditionType(); }
}
```

---

### Layer 2 — `WatchdogAlertRouter` SPI (in `casehub-qhorus-api`)

The router returns delivery targets, not `ConnectorMessage` — keeping `casehub-qhorus-api` free of any `casehub-connectors` dependency. The `ConnectorAlertBridge` (in the `connectors/` module, which depends on both) builds the `ConnectorMessage`.

```java
package io.casehub.qhorus.api.watchdog;

/** Delivery target: which connector to use and at what address. */
public record AlertDeliveryTarget(String connectorId, String destination) {}

public interface WatchdogAlertRouter {
    List<AlertDeliveryTarget> route(WatchdogAlertEvent event);
}
```

**`ConfiguredWatchdogAlertRouter @DefaultBean`** (in `casehub-qhorus` runtime): reads `casehub.qhorus.watchdog.alert.endpoints[*]` config and returns one `AlertDeliveryTarget` per configured endpoint. If no endpoints are configured, returns empty list — no delivery, no silent failure.

**Config:**
```yaml
casehub.qhorus.watchdog.alert.endpoints[0].connector-id=slack
casehub.qhorus.watchdog.alert.endpoints[0].destination=https://hooks.slack.com/services/...
casehub.qhorus.watchdog.alert.endpoints[1].connector-id=email
casehub.qhorus.watchdog.alert.endpoints[1].destination=ops@example.com
```

**Required `QhorusConfig.Watchdog` additions:**
```java
interface Watchdog {
    boolean enabled();
    int checkIntervalSeconds();
    Alert alert();   // new

    interface Alert {
        @WithDefault("")
        List<AlertEndpoint> endpoints();   // empty list = no external delivery; @WithDefault("") required
                                           // by SmallRye Config for List<T> — omitting it causes
                                           // SRCFG00014 startup failure when no endpoints are configured

        interface AlertEndpoint {
            @WithName("connector-id")
            String connectorId();
            String destination();
        }
    }
}
```

**Overriding**: provide an `@ApplicationScoped` (without `@DefaultBean`) bean implementing `WatchdogAlertRouter`. Normal CDI resolution displaces the `@DefaultBean`. Use this for per-watchdog routing logic, severity-based fanout, or integration with Claudony's own schema.

Title formatted by bridge: `"[Qhorus Alert] {conditionType}: {targetName}"`. Body: `summary` plus a pattern-matched detail block from the `AlertContext`.

---

### Layer 3 — `ConnectorAlertBridge` (new `connectors/` submodule in qhorus repo)

New Maven module: `casehub-qhorus-connectors`. Depends on `casehub-qhorus-api` + `casehub-connectors-core`. Opt-in by classpath presence.

```java
@ApplicationScoped
public class ConnectorAlertBridge {

    @Inject ConnectorService connectorService;   // not Instance<Connector> — proper error behavior
    @Inject WatchdogAlertRouter router;

    void onAlert(@ObservesAsync WatchdogAlertEvent event) {
        String title = "[Qhorus Alert] " + event.conditionType() + ": " + event.targetName();
        String body = buildBody(event);
        for (AlertDeliveryTarget target : router.route(event)) {
            try {
                connectorService.send(target.connectorId(),
                    new ConnectorMessage(target.destination(), title, body));
            } catch (IllegalArgumentException e) {
                // Unknown connector id — config error; log and continue rather than
                // letting one misconfigured endpoint suppress all others.
                log.errorf("Unknown connector '%s' for watchdog alert — available: %s",
                    target.connectorId(), connectorService.ids());
            }
        }
    }

    private String buildBody(WatchdogAlertEvent event) {
        return switch (event.context()) {
            case BarrierStuckContext c ->
                event.summary() + "\nChannel: " + c.channelName()
                + "\nMissing: " + String.join(", ", c.missingContributors())
                + "\nElapsed: " + c.elapsedSeconds() + "s";
            case ApprovalPendingContext c ->
                event.summary() + "\nPending: " + c.pendingCount()
                + (c.oldestExpiryAt() != null ? "\nOldest expiry: " + c.oldestExpiryAt() : "");
            case AgentStaleContext c ->
                event.summary() + "\nStale count: " + c.staleCount()
                + (c.staleInstanceIds().isEmpty() ? "" : "\nIDs: " + String.join(", ", c.staleInstanceIds()));
            case ChannelIdleContext c ->
                event.summary() + "\nIdle channels: " + String.join(", ", c.channelNames())
                + "\nIdle > " + c.thresholdSeconds() + "s";
            case QueueDepthContext c ->
                event.summary() + "\nChannel: " + c.channelName()
                + "\nDepth: " + c.messageCount() + " (threshold: " + c.threshold() + ")";
        };
    }
}
```

**CDI validation fix** (GE-20260521-45e61c): `TwilioSmsConnector` and `WhatsAppConnector` fail CDI validation in JDBC-only test environments. Test `application.properties` must include:

```properties
quarkus.arc.exclude-types=io.casehub.connectors.twilio.TwilioSmsConnector,\
  io.casehub.connectors.whatsapp.WhatsAppConnector
```

---

### Changes to `WatchdogEvaluationService`

#### `fireAlert()` signature change

```java
@Inject Event<WatchdogAlertEvent> alertEvents;

// From:
private void fireAlert(Watchdog w, String alertContent)
// To:
private void fireAlert(Watchdog w, String summary, AlertContext context, Instant now)
```

`now` is passed from `evaluateAll()`'s outer `now` capture — ensures `firedAt` on the event is consistent with `w.lastFiredAt` and the cutoff calculations used in the same evaluation cycle.

#### `evaluate*()` method changes

Each method builds a typed `AlertContext` record:

- **`evaluateBarrierStuck`**: passes `new BarrierStuckContext(ch.id, ch.name, missingList, elapsedSeconds)` per stuck channel. Each stuck channel fires a **separate event** — the watchdog fires once per stuck barrier, not one aggregate per evaluation cycle.
- **`evaluateApprovalPending`**: accumulates `oldestExpiryAt` via `stream().map(c -> c.expiresAt).filter(Objects::nonNull).min(Comparator.naturalOrder()).orElse(null)`. The current count-only query must change to `list()` to support this.
- **`evaluateAgentStale`**: changes from `count()` to `list(...).stream().limit(10)` to gather instance IDs for the context.
- **`evaluateChannelIdle`**: passes `new ChannelIdleContext(names, threshold)` — `threshold` maps to `thresholdSeconds`.
- **`evaluateQueueDepth`**: passes `new QueueDepthContext(ch.name, count, threshold)`.
- **`evaluateAgentStale`** (latent bug fix): the pre-existing code runs two queries with inconsistent cutoff filters — first `count("status = 'stale' AND lastSeen < ?1", cutoff)` to decide whether to fire, then `count("status = 'stale'")` with no cutoff for the alert message. The new implementation collapses both into a single `list("status = 'stale' AND lastSeen < ?1", cutoff).stream().limit(10)` call, fixing the inconsistency: the count in the alert now matches the condition that triggered it.

#### `fireAlert()` ordering — critical

```java
private void fireAlert(Watchdog w, String summary, AlertContext context, Instant now) {
    // 1. Fire async event FIRST.
    //    Rationale: external delivery is not contingent on internal channel existence or
    //    dispatch success. A rate-limited, paused, or policy-rejected channel dispatch must
    //    not suppress the operator notification.
    //
    //    Reliability model (both directions must be accepted):
    //    - False-positive (ghost notification): outer @Transactional rolls back after
    //      fireAsync() dispatches. Observer fires for a state change that never committed.
    //      Narrow window in practice (evaluateAll() has no post-dispatch side effects
    //      that would cause rollback). Accepted — false-positive alert > missed alert.
    //    - False-negative (crash window): fireAsync() fires, transaction commits
    //      (persisting lastFiredAt), app crashes before the async observer delivers to
    //      the connector. Alert is lost AND the watchdog is debounced until the window
    //      expires. Accepted — at-most-once async delivery is the CDI event contract;
    //      an outbox pattern would be required for at-least-once.
    alertEvents.fireAsync(new WatchdogAlertEvent(
        w.id, w.targetName, w.notificationChannel, summary, now, context));

    // 2. Internal channel dispatch SECOND.
    //    Dispatch failure does not suppress the already-fired event above.
    Optional<Channel> notifChannel = channelService.findByName(w.notificationChannel);
    if (notifChannel.isEmpty()) {
        return;
    }
    messageService.dispatch(MessageDispatch.builder()
        .channelId(notifChannel.get().id)
        .sender("system:watchdog")
        .type(MessageType.STATUS)
        .content(summary)
        .actorType(ActorType.SYSTEM)
        .build());
}
```

---

### Module structure

```
casehub-qhorus/
├── api/
│   └── watchdog/              — WatchdogConditionType (enum), AlertContext (sealed + 5 records),
│                                WatchdogAlertEvent, WatchdogAlertRouter, AlertDeliveryTarget (new)
├── runtime/
│   ├── config/QhorusConfig    — Watchdog.Alert sub-interface added
│   └── watchdog/
│       ├── WatchdogEvaluationService  — fireAlert() signature + evaluate*() context builds
│       └── ConfiguredWatchdogAlertRouter @DefaultBean (new)
└── connectors/                — new optional submodule (casehub-qhorus-connectors)
    └── ConnectorAlertBridge   — @ObservesAsync → ConnectorService.send()
```

`connectors/` follows the `casehub-engine-ledger` / `casehub-engine-work-adapter` precedent: an integration module within the repo that bridges to a sibling foundation module. `casehub-qhorus` runtime does not depend on connectors; only `casehub-qhorus-connectors` does.

---

### Platform coherence

- **No dependency added to `casehub-qhorus` runtime** — bridge is separate artifact; core consumers pay no classpath cost unless they opt in.
- **`casehub-connectors-core` dependency** — registered in PLATFORM.md dependency table (PP-20260523-605b90): consuming module = `casehub-qhorus / connectors`, nature = `optional — WatchdogAlertEvent → Connector.send() bridge`.
- **parent#5 coupling** — `ConnectorAlertBridge` targets current `Connector` SPI. When parent#5 consolidates, the bridge `connectorService.send()` call updates mechanically.

---

### Testing

| Layer | Test location | Approach |
|---|---|---|
| `WatchdogConditionType` / `AlertContext` records | `runtime/src/test/` | Unit — construct directly, assert fields |
| `WatchdogEvaluationService` — event fired on alert | `runtime/src/test/` | `@QuarkusTest @TestTransaction` — call `evaluateAll()` directly; use `@ApplicationScoped` capture bean with `CountDownLatch.await()` to wait for async delivery (GE-20260517-712fe5). This requires the capture bean to be an `@ObservesAsync` observer — the CountDownLatch ensures the test thread waits rather than asserting before delivery. |
| `ConfiguredWatchdogAlertRouter` | `runtime/src/test/` | `@QuarkusTest` with config overrides — assert `route()` returns expected `AlertDeliveryTarget` list |
| `ConnectorAlertBridge` | `connectors/src/test/` | `@QuarkusTest` — inject bridge, call `onAlert()` directly (GE-20260513-b15933 — no async delivery needed in bridge test); `@Mock TestSlackConnector` or stub `ConnectorService` captures sent messages |
| End-to-end (evaluateAll → event → bridge → connector mock) | `runtime/src/test/` | `@QuarkusTest` with `casehub-qhorus-connectors` on test classpath — call `evaluateAll()` directly; `ConnectorAlertBridge` picks up the async event via CountDownLatch capture; assert on mock connector. **Not in `connectors/src/test/`** — that module does not depend on the runtime artifact, so `WatchdogEvaluationService` is not available there. |

**Testing distinction**: Service-layer tests (event IS fired) use CountDownLatch + `evaluateAll()` in `runtime/src/test/`. Bridge tests (bridge calls connector) call `onAlert()` directly in `connectors/src/test/` — no async delivery needed. E2E lives in `runtime/src/test/` where the runtime classpath is already present.

---

### Out of scope

- Per-watchdog external routing — deferred; `WatchdogAlertRouter` SPI is the extension point.
- Connector SPI consolidation (parent#5) — bridge written against current `Connector` SPI.
- Reactive parity for `ConnectorAlertBridge` — `@ObservesAsync` is already async; no reactive variant needed.
- At-least-once delivery guarantee — requires an outbox pattern; accepted as out of scope for this issue.

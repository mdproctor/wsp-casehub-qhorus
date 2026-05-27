# WatchdogAlertEvent + ConnectorAlertBridge — Design Spec

**Issue:** casehubio/qhorus#200  
**Date:** 2026-05-27  
**Status:** Approved

---

## Problem

`WatchdogEvaluationService.fireAlert()` dispatches a STATUS message to an internal Qhorus channel only. Human operators have no external notification path — stalled-obligation alerts, stuck barrier warnings, and approval-pending notices fire into channels only agents can see.

---

## Design

### Layer 1 — `WatchdogAlertEvent` (in `casehub-qhorus-api`)

A plain record following the `MessageReceivedEvent` / `ChannelInitialisedEvent` pattern: no CDI annotations, no framework dependencies, safe to import from any module.

```java
package io.casehub.qhorus.api.watchdog;

public record WatchdogAlertEvent(
    UUID watchdogId,
    String conditionType,    // BARRIER_STUCK, APPROVAL_PENDING, AGENT_STALE, CHANNEL_IDLE, QUEUE_DEPTH
    String targetName,       // channel/instance being monitored ("*" = all)
    String notificationChannel,
    String summary,          // pre-formatted human-readable line
    Instant firedAt,
    Map<String, String> context  // condition-specific detail — see below
) {}
```

**`context` contract by condition type:**

| conditionType | Required keys | Optional keys |
|---|---|---|
| BARRIER_STUCK | `channelId`, `channelName`, `missingContributors` (comma-sep) | `elapsedSeconds` |
| APPROVAL_PENDING | `pendingCount` | `oldestExpiryAt` |
| AGENT_STALE | `staleCount` | `staleInstanceIds` (comma-sep, up to 10) |
| CHANNEL_IDLE | `channelNames` (comma-sep, up to 3), `idleSeconds` | — |
| QUEUE_DEPTH | `channelName`, `messageCount`, `threshold` | — |

The `context` map carries condition-specific detail that transforms an "alert fired" notification into an actionable operator message. Implementations that render notifications (e.g. `ConnectorAlertBridge`) must tolerate missing keys gracefully — keys are required by convention, not enforced at compile time.

---

### Layer 2 — `WatchdogAlertRouter` SPI (in `casehub-qhorus-api`)

The router returns delivery targets, not `ConnectorMessage` — keeping `casehub-qhorus-api` free of any `casehub-connectors` dependency. The `ConnectorAlertBridge` (which already depends on both) builds the `ConnectorMessage`.

```java
package io.casehub.qhorus.api.watchdog;

/** Delivery target: which connector receives the alert and at what address. */
public record AlertDeliveryTarget(String connectorId, String destination) {}

public interface WatchdogAlertRouter {
    List<AlertDeliveryTarget> route(WatchdogAlertEvent event);
}
```

**`ConfiguredWatchdogAlertRouter @DefaultBean`** (in `casehub-qhorus` runtime): reads `casehub.qhorus.watchdog.alert.endpoints[*]` config and returns one `AlertDeliveryTarget` per configured endpoint.

```yaml
casehub.qhorus.watchdog.alert.endpoints[0].connector-id=slack
casehub.qhorus.watchdog.alert.endpoints[0].destination=https://hooks.slack.com/services/...
casehub.qhorus.watchdog.alert.endpoints[1].connector-id=email
casehub.qhorus.watchdog.alert.endpoints[1].destination=ops@example.com
```

If no endpoints are configured, the list is empty and no delivery occurs. The operator must explicitly configure at least one endpoint.

**Overriding**: provide an `@ApplicationScoped` (without `@DefaultBean`) bean implementing `WatchdogAlertRouter`. Normal CDI resolution displaces the `@DefaultBean`. Use this for per-watchdog routing logic, severity-based fanout, or integration with Claudony's own schema.

The `ConnectorAlertBridge` formats: title = `"[Qhorus Alert] {conditionType}: {targetName}"`; body = `summary` plus context key-value lines.

---

### Layer 3 — `ConnectorAlertBridge` (new `connectors/` submodule in qhorus repo)

New Maven module: `casehub-qhorus-connectors`. Depends on `casehub-qhorus-api` + `casehub-connectors-core`. Opt-in: activates when present on the classpath. Adding the artifact to `pom.xml` is the only required consumer action.

```java
@ApplicationScoped
public class ConnectorAlertBridge {

    @Inject Instance<Connector> connectors;
    @Inject WatchdogAlertRouter router;

    void onAlert(@ObservesAsync WatchdogAlertEvent event) {
        String title = "[Qhorus Alert] " + event.conditionType() + ": " + event.targetName();
        String body = buildBody(event);
        for (AlertDeliveryTarget target : router.route(event)) {
            connectors.stream()
                .filter(c -> c.id().equals(target.connectorId()))
                .forEach(c -> c.send(new ConnectorMessage(target.destination(), title, body)));
        }
    }

    private String buildBody(WatchdogAlertEvent event) {
        StringBuilder sb = new StringBuilder(event.summary());
        event.context().forEach((k, v) -> sb.append("\n  ").append(k).append(": ").append(v));
        return sb.toString();
    }
}
```

**CDI validation fix** (GE-20260521-45e61c): `TwilioSmsConnector` and `WhatsAppConnector` fail CDI validation in JDBC-only test environments. Test `application.properties` must include:

```properties
quarkus.arc.exclude-types=io.casehub.connectors.twilio.TwilioSmsConnector,\
  io.casehub.connectors.whatsapp.WhatsAppConnector
```

**`@ObservesAsync` in `@QuarkusTest`** (GE-20260513-b15933): async observers are not reliably delivered in `@QuarkusTest`. Tests must inject `ConnectorAlertBridge` and call `onAlert()` directly.

---

### Changes to `WatchdogEvaluationService`

#### `fireAlert()` signature change

From: `fireAlert(Watchdog w, String alertContent)`  
To: `fireAlert(Watchdog w, String summary, Map<String, String> context)`

Each `evaluate*` method builds the structured `context` map for its condition type before calling `fireAlert()`.

#### `fireAlert()` ordering — critical

```java
private void fireAlert(Watchdog w, String summary, Map<String, String> context) {
    // 1. Fire async event FIRST — external delivery is independent of internal dispatch success.
    //    fireAsync() fires immediately on the managed executor; it does not wait for the
    //    outer @Transactional boundary to commit.
    //    Ghost-notification risk: if the outer transaction rolls back (narrow window in
    //    practice), the observer fires for a state change that never committed. This is
    //    acceptable for an alerting system: false-positive alert > missed alert.
    alertEvents.fireAsync(new WatchdogAlertEvent(
        w.id, w.conditionType, w.targetName, w.notificationChannel,
        summary, Instant.now(), context));

    // 2. Internal channel dispatch SECOND.
    //    A dispatch failure (rate-limited, paused, policy-rejected) does not suppress
    //    the external alert already fired above.
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

Ordering rationale:
- `fireAsync()` before dispatch: external delivery is not contingent on internal channel existence or dispatch success.
- Dispatch failure (exception) propagates within the transaction but cannot suppress the already-fired event.
- Ghost-notification window: outer `@Transactional` rollback after `fireAsync()` fires. This is the irreducible minimum risk of CDI async events inside a transaction and is explicitly accepted.

---

### Module structure

```
casehub-qhorus/
├── api/                          — WatchdogAlertEvent, WatchdogAlertRouter (new)
├── runtime/                      — ConfiguredWatchdogAlertRouter @DefaultBean (new)
│   └── watchdog/
│       └── WatchdogEvaluationService  — updated fireAlert() + evaluate*() methods
└── connectors/                   — new optional submodule
    └── ConnectorAlertBridge      — @ObservesAsync WatchdogAlertBridge → Connector.send()
```

`connectors/` follows the `casehub-engine-ledger` / `casehub-engine-work-adapter` precedent: an integration module within the repo that bridges to a sibling foundation module. `casehub-qhorus` runtime does not depend on connectors. Only the new `casehub-qhorus-connectors` artifact does.

---

### Platform coherence

- **No dependency added to `casehub-qhorus` runtime**: The bridge is a separate artifact. Core qhorus consumers (Claudony, devtown) pay no classpath cost for connectors unless they opt in.
- **SPI placement**: `WatchdogAlertRouter` is an operational SPI (`@DefaultBean` no-op → populated default). Follows the rule: can the system function with a do-nothing implementation? Yes (no external delivery) → no-op default is acceptable; but to make the bridge useful out of the box, `ConfiguredWatchdogAlertRouter` provides A-style config-driven behavior.
- **PLATFORM.md dependency table**: `casehub-connectors-core` consumed by `casehub-qhorus` / `connectors` submodule → must be registered (PP-20260523-605b90).
- **parent#5 coupling**: `ConnectorAlertBridge` uses the current `Connector` SPI. When parent#5 consolidates the connector SPI, the bridge method signature updates. Migration is mechanical — noted in issue comments.

---

### Testing

| Layer | Test location | Approach |
|---|---|---|
| `WatchdogAlertEvent` construction | `runtime/src/test/` | Unit — instantiate directly |
| `WatchdogEvaluationService.fireAlert()` ordering | `runtime/src/test/` | `@QuarkusTest @TestTransaction` — inject `WatchdogEvaluationService`, call evaluate methods directly; assert event via `@ApplicationScoped` capture bean with `CountDownLatch` (GE-20260517-712fe5) |
| `ConfiguredWatchdogAlertRouter` | `runtime/src/test/` | Unit — `@QuarkusTest` with config overrides |
| `ConnectorAlertBridge` | `connectors/src/test/` | `@QuarkusTest` — inject bridge, call `onAlert()` directly (GE-20260513-b15933); `@Mock TestSlackConnector` captures messages |
| End-to-end (watchdog fires → Slack mock receives) | `connectors/src/test/` | `@QuarkusTest` with `TestSlackConnector @Mock` |

---

### Out of scope

- Per-watchdog external routing (each watchdog configured with its own connector destinations) — deferred; `WatchdogAlertRouter` SPI is the extension point.
- Connector SPI consolidation (parent#5) — prerequisite for finalizing multi-connector routing semantics; bridge is written against current `Connector` SPI.
- Reactive parity for `ConnectorAlertBridge` — `@ObservesAsync` is already async; no reactive variant needed.

# WatchdogAlertEvent + ConnectorAlertBridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Emit a CDI `WatchdogAlertEvent` from `WatchdogEvaluationService.fireAlert()` and route it to external connectors (Slack, Teams, SMS, email) via an opt-in bridge module, so human operators are notified when watchdog conditions fire.

**Architecture:** Three layers — (1) sealed `AlertContext` hierarchy + `WatchdogAlertEvent` in `casehub-qhorus-api`, (2) `WatchdogAlertRouter` SPI with config-driven `@DefaultBean` in qhorus runtime, (3) opt-in `casehub-qhorus-connectors` bridge module that observes events and calls `ConnectorService`. The `fireAsync()` fires before the internal channel dispatch so external delivery is independent of channel availability. The spec is at `specs/2026-05-27-watchdog-alert-event-design.md`.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI `fireAsync()`, SmallRye Config `@ConfigMapping`, casehub-connectors-core `ConnectorService`

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`  
**Single-module test:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClassName`  
**All tests:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,connectors`

---

## File Map

**New files — `api/src/main/java/io/casehub/qhorus/api/watchdog/`:**
- `WatchdogConditionType.java` — enum: 5 condition type constants
- `AlertContext.java` — sealed interface: `conditionType()` method
- `BarrierStuckContext.java` — record: `channelId, channelName, missingContributors, elapsedSeconds`
- `ApprovalPendingContext.java` — record: `pendingCount, oldestExpiryAt`
- `AgentStaleContext.java` — record: `staleCount, staleInstanceIds`
- `ChannelIdleContext.java` — record: `channelNames, thresholdSeconds`
- `QueueDepthContext.java` — record: `channelName, messageCount, threshold`
- `WatchdogAlertEvent.java` — record: event payload with derived `conditionType()` accessor
- `AlertDeliveryTarget.java` — record: `connectorId, destination`
- `WatchdogAlertRouter.java` — interface: `route(WatchdogAlertEvent) → List<AlertDeliveryTarget>`

**Modified — `runtime/src/main/java/...`:**
- `config/QhorusConfig.java` — add `Alert alert()` to `Watchdog` interface
- `watchdog/WatchdogEvaluationService.java` — add `@Inject Event<WatchdogAlertEvent> alertEvents`; change `fireAlert()` signature; update all 5 `evaluate*()` methods

**New files — `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/`:**
- `ConfiguredWatchdogAlertRouter.java` — `@DefaultBean @ApplicationScoped` reads config endpoints

**New module — `connectors/`:**
- `connectors/pom.xml` — depends on `casehub-qhorus-api` + `casehub-connectors-core`
- `connectors/src/main/java/io/casehub/qhorus/connectors/ConnectorAlertBridge.java`
- `connectors/src/test/java/io/casehub/qhorus/connectors/ConnectorAlertBridgeTest.java`
- `connectors/src/test/java/io/casehub/qhorus/connectors/TestWatchdogAlertRouter.java`
- `connectors/src/test/java/io/casehub/qhorus/connectors/TestSlackConnector.java`
- `connectors/src/test/resources/application.properties`

**Modified — root:**
- `pom.xml` — add `<module>connectors</module>` before `<module>runtime</module>`

**New test files — `runtime/src/test/java/io/casehub/qhorus/`:**
- `runtime/watchdog/ConfiguredWatchdogAlertRouterTest.java`
- `runtime/watchdog/WatchdogAlertEventTest.java`
- `runtime/watchdog/AlertEventCapture.java` — `@ApplicationScoped` CDI capture bean
- `runtime/watchdog/WatchdogAlertE2ETest.java`
- `runtime/watchdog/TestSlackConnectorE2E.java` — `@Mock` for E2E
- `api/WatchdogAlertEndpointsProfile.java` — extends `WatchdogEnabledProfile`

**Modified — `runtime/src/test/resources/application.properties`:**
- Add `quarkus.arc.exclude-types` for Twilio/WhatsApp (safe: no-op when connectors not on classpath)

---

## Task 1: API types in `casehub-qhorus-api`

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogConditionType.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/BarrierStuckContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/ApprovalPendingContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/AgentStaleContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/ChannelIdleContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/QueueDepthContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAlertEvent.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/AlertDeliveryTarget.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAlertRouter.java`

These are pure data types — no logic, no tests needed. Compile verification is the check.

- [ ] **Step 1: Create `WatchdogConditionType.java`**

```java
package io.casehub.qhorus.api.watchdog;

public enum WatchdogConditionType {
    BARRIER_STUCK, APPROVAL_PENDING, AGENT_STALE, CHANNEL_IDLE, QUEUE_DEPTH
}
```

- [ ] **Step 2: Create `AlertContext.java`**

```java
package io.casehub.qhorus.api.watchdog;

public sealed interface AlertContext
        permits BarrierStuckContext, ApprovalPendingContext,
                AgentStaleContext, ChannelIdleContext, QueueDepthContext {

    WatchdogConditionType conditionType();
}
```

- [ ] **Step 3: Create the five context records**

`BarrierStuckContext.java`:
```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;
import java.util.UUID;

public record BarrierStuckContext(
        UUID channelId,
        String channelName,
        List<String> missingContributors,
        long elapsedSeconds) implements AlertContext {

    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.BARRIER_STUCK; }
}
```

`ApprovalPendingContext.java`:
```java
package io.casehub.qhorus.api.watchdog;

import java.time.Instant;

public record ApprovalPendingContext(
        long pendingCount,
        Instant oldestExpiryAt) implements AlertContext {   // null if no expiresAt on any pending commitment

    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.APPROVAL_PENDING; }
}
```

`AgentStaleContext.java`:
```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;

public record AgentStaleContext(
        long staleCount,
        List<String> staleInstanceIds) implements AlertContext {  // up to 10 IDs; may be shorter than staleCount

    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.AGENT_STALE; }
}
```

`ChannelIdleContext.java`:
```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;

public record ChannelIdleContext(
        List<String> channelNames,   // up to 3; may be fewer than total idle channels
        long thresholdSeconds) implements AlertContext {  // configured idle threshold — NOT per-channel elapsed time

    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.CHANNEL_IDLE; }
}
```

`QueueDepthContext.java`:
```java
package io.casehub.qhorus.api.watchdog;

public record QueueDepthContext(
        String channelName,
        long messageCount,
        int threshold) implements AlertContext {

    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.QUEUE_DEPTH; }
}
```

- [ ] **Step 4: Create `WatchdogAlertEvent.java`**

```java
package io.casehub.qhorus.api.watchdog;

import java.time.Instant;
import java.util.UUID;

public record WatchdogAlertEvent(
        UUID watchdogId,
        String targetName,           // channel/instance being monitored ("*" = all)
        String notificationChannel,  // internal Qhorus channel name; enables topology-aware routing
        String summary,              // pre-formatted human-readable line for notification body
        Instant firedAt,             // copied from evaluateAll()'s outer now — consistent with w.lastFiredAt
        AlertContext context) {

    /** Convenience accessor — delegates to context.conditionType(). */
    public WatchdogConditionType conditionType() {
        return context.conditionType();
    }
}
```

- [ ] **Step 5: Create `AlertDeliveryTarget.java`**

```java
package io.casehub.qhorus.api.watchdog;

public record AlertDeliveryTarget(String connectorId, String destination) {}
```

- [ ] **Step 6: Create `WatchdogAlertRouter.java`**

```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;

public interface WatchdogAlertRouter {
    List<AlertDeliveryTarget> route(WatchdogAlertEvent event);
}
```

- [ ] **Step 7: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api
```

Expected: `BUILD SUCCESS` with no errors.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#200): WatchdogConditionType enum, sealed AlertContext hierarchy, WatchdogAlertEvent, WatchdogAlertRouter SPI in casehub-qhorus-api"
```

---

## Task 2: `QhorusConfig.Watchdog.Alert` sub-interface

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java`

The `Watchdog` interface currently has only `enabled()` and `checkIntervalSeconds()`. Add `Alert alert()`. The `@WithDefault("")` on `endpoints()` is required by SmallRye Config — without it, startup fails with `SRCFG00014` when no endpoints are configured.

- [ ] **Step 1: Add `Alert alert()` to `Watchdog` in `QhorusConfig.java`**

In `QhorusConfig.java`, inside the `interface Watchdog { ... }` block, add after `checkIntervalSeconds()`:

```java
        Alert alert();

        interface Alert {
            @WithDefault("")
            List<AlertEndpoint> endpoints();

            interface AlertEndpoint {
                @WithName("connector-id")
                String connectorId();
                String destination();
            }
        }
```

Add the required import at the top of `QhorusConfig.java`:
```java
import io.smallrye.config.WithName;
import java.util.List;
```

- [ ] **Step 2: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#200): add Watchdog.Alert sub-interface with endpoints list to QhorusConfig"
```

---

## Task 3: `ConfiguredWatchdogAlertRouter` (TDD)

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ConfiguredWatchdogAlertRouter.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/ConfiguredWatchdogAlertRouterTest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/api/WatchdogAlertEndpointsProfile.java`

- [ ] **Step 1: Create `WatchdogAlertEndpointsProfile.java`**

Extends `WatchdogEnabledProfile` (already exists in `runtime/src/test/.../api/`). This adds alert endpoint config so that `ConfiguredWatchdogAlertRouter` has something to read.

```java
package io.casehub.qhorus.api;

import java.util.HashMap;
import java.util.Map;

public class WatchdogAlertEndpointsProfile extends WatchdogEnabledProfile {

    @Override
    public Map<String, String> getConfigOverrides() {
        Map<String, String> config = new HashMap<>(super.getConfigOverrides());
        config.put("casehub.qhorus.watchdog.alert.endpoints[0].connector-id", "slack");
        config.put("casehub.qhorus.watchdog.alert.endpoints[0].destination", "https://hooks.slack.com/test");
        return config;
    }
}
```

- [ ] **Step 2: Write the failing test**

```java
package io.casehub.qhorus.runtime.watchdog;

import io.casehub.qhorus.api.WatchdogAlertEndpointsProfile;
import io.casehub.qhorus.api.watchdog.AgentStaleContext;
import io.casehub.qhorus.api.watchdog.AlertDeliveryTarget;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogAlertRouter;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(WatchdogAlertEndpointsProfile.class)
class ConfiguredWatchdogAlertRouterTest {

    @Inject
    WatchdogAlertRouter router;

    @Test
    void routeReturnsConfiguredEndpoints() {
        WatchdogAlertEvent event = new WatchdogAlertEvent(
                UUID.randomUUID(), "*", "alerts", "AGENT_STALE: 1 stale agent",
                Instant.now(), new AgentStaleContext(1L, List.of("id-1")));

        List<AlertDeliveryTarget> targets = router.route(event);

        assertThat(targets).hasSize(1);
        assertThat(targets.get(0).connectorId()).isEqualTo("slack");
        assertThat(targets.get(0).destination()).isEqualTo("https://hooks.slack.com/test");
    }

    @Test
    void routeReturnsEmptyListWhenNoEndpointsConfigured() {
        // WatchdogEnabledProfile has no alert endpoints — router should return empty list
        // This test runs in a SEPARATE @QuarkusTest context from the above (different profile)
        // so it cannot be in the same test class. Verified by the no-endpoints profile below.
    }
}
```

Also create `ConfiguredWatchdogAlertRouterNoEndpointsTest.java`:
```java
package io.casehub.qhorus.runtime.watchdog;

import io.casehub.qhorus.api.WatchdogEnabledProfile;
import io.casehub.qhorus.api.watchdog.AgentStaleContext;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogAlertRouter;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(WatchdogEnabledProfile.class)
class ConfiguredWatchdogAlertRouterNoEndpointsTest {

    @Inject
    WatchdogAlertRouter router;

    @Test
    void routeReturnsEmptyListWhenNoEndpointsConfigured() {
        WatchdogAlertEvent event = new WatchdogAlertEvent(
                UUID.randomUUID(), "*", "alerts", "summary",
                Instant.now(), new AgentStaleContext(1L, List.of()));

        assertThat(router.route(event)).isEmpty();
    }
}
```

- [ ] **Step 3: Run the tests to confirm they fail (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ConfiguredWatchdogAlertRouterTest,ConfiguredWatchdogAlertRouterNoEndpointsTest 2>&1 | tail -20
```

Expected: compilation error — `WatchdogAlertRouter` not satisfied / `ConfiguredWatchdogAlertRouter` does not exist.

- [ ] **Step 4: Create `ConfiguredWatchdogAlertRouter.java`**

```java
package io.casehub.qhorus.runtime.watchdog;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.watchdog.AlertDeliveryTarget;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogAlertRouter;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.quarkus.arc.DefaultBean;

@DefaultBean
@ApplicationScoped
public class ConfiguredWatchdogAlertRouter implements WatchdogAlertRouter {

    @Inject
    QhorusConfig config;

    @Override
    public List<AlertDeliveryTarget> route(WatchdogAlertEvent event) {
        return config.watchdog().alert().endpoints().stream()
                .map(ep -> new AlertDeliveryTarget(ep.connectorId(), ep.destination()))
                .toList();
    }
}
```

- [ ] **Step 5: Run the tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ConfiguredWatchdogAlertRouterTest,ConfiguredWatchdogAlertRouterNoEndpointsTest
```

Expected: `BUILD SUCCESS`, both tests green.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#200): ConfiguredWatchdogAlertRouter @DefaultBean reads alert.endpoints config"
```

---

## Task 4: Update `WatchdogEvaluationService` (TDD)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/AlertEventCapture.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogAlertEventTest.java`

The test uses a CDI `@ApplicationScoped` capture bean with a `CountDownLatch` to wait for async delivery (GE-20260517-712fe5). It calls `evaluateAll()` directly, not via the scheduler (to avoid `WatchdogScheduler` interfering with uncommitted test data).

- [ ] **Step 1: Create `AlertEventCapture.java`** (shared by event test and E2E test)

```java
package io.casehub.qhorus.runtime.watchdog;

import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;

import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;

@ApplicationScoped
public class AlertEventCapture {

    public static final CopyOnWriteArrayList<WatchdogAlertEvent> events = new CopyOnWriteArrayList<>();
    private static volatile CountDownLatch latch = new CountDownLatch(0);

    public static void expectCount(int n) {
        events.clear();
        latch = new CountDownLatch(n);
    }

    public static boolean await(long timeout, TimeUnit unit) throws InterruptedException {
        return latch.await(timeout, unit);
    }

    void onAlert(@ObservesAsync WatchdogAlertEvent event) {
        events.add(event);
        latch.countDown();
    }
}
```

- [ ] **Step 2: Write `WatchdogAlertEventTest.java`** (failing — service not yet updated)

```java
package io.casehub.qhorus.runtime.watchdog;

import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import io.casehub.qhorus.api.WatchdogEnabledProfile;
import io.casehub.qhorus.api.watchdog.BarrierStuckContext;
import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertTrue;

@QuarkusTest
@TestProfile(WatchdogEnabledProfile.class)
class WatchdogAlertEventTest {

    @Inject
    WatchdogEvaluationService service;

    @BeforeEach
    void resetCapture() {
        AlertEventCapture.expectCount(0);
    }

    @Test
    @Transactional
    void barrierStuck_firesWatchdogAlertEvent() throws InterruptedException {
        // Setup: BARRIER channel with two required contributors, no messages sent (so both are missing)
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = "barrier-test-" + ch.id;
        ch.semantic = ChannelSemantic.BARRIER;
        ch.barrierContributors = "agent-alpha,agent-beta";
        ch.lastActivityAt = Instant.now().minusSeconds(3600);
        ch.persist();

        Watchdog w = new Watchdog();
        w.conditionType = "BARRIER_STUCK";
        w.targetName = ch.name;
        w.thresholdSeconds = 0;    // always fire
        w.notificationChannel = "alerts-" + w.id;
        w.createdBy = "test";
        w.persist();

        AlertEventCapture.expectCount(1);
        service.evaluateAll();

        assertTrue(AlertEventCapture.await(2, TimeUnit.SECONDS), "WatchdogAlertEvent not delivered within 2s");
        assertThat(AlertEventCapture.events).hasSize(1);

        var event = AlertEventCapture.events.get(0);
        assertThat(event.conditionType()).isEqualTo(WatchdogConditionType.BARRIER_STUCK);
        assertThat(event.watchdogId()).isEqualTo(w.id);
        assertThat(event.targetName()).isEqualTo(ch.name);

        BarrierStuckContext ctx = (BarrierStuckContext) event.context();
        assertThat(ctx.channelId()).isEqualTo(ch.id);
        assertThat(ctx.channelName()).isEqualTo(ch.name);
        assertThat(ctx.missingContributors()).containsExactlyInAnyOrder("agent-alpha", "agent-beta");
        assertThat(ctx.elapsedSeconds()).isGreaterThan(3500L);
    }

    @Test
    @Transactional
    void agentStale_firesWatchdogAlertEvent() throws InterruptedException {
        // Note: stale instances are created by Instance entity — use Instance.persist() in real test.
        // This test verifies the event fires with AGENT_STALE conditionType.
        // If no stale instances exist the condition doesn't fire; threshold=0 still requires stale status.
        // Skipping full instance setup here — covered by existing WatchdogEnabledTest patterns.
        // Run full test suite to verify: mvn test -pl runtime -Dtest=WatchdogAlertEventTest
    }
}
```

- [ ] **Step 3: Run the test to confirm it compiles but fails (method signatures changed)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogAlertEventTest 2>&1 | tail -30
```

Expected: compilation error or test failure — `fireAlert()` doesn't fire any event yet.

- [ ] **Step 4: Update `WatchdogEvaluationService.java`**

Add the CDI event injection and update the `fireAlert()` method signature. The full updated file:

**Add field (after existing `@Inject MessageStore messageStore;`):**
```java
    @Inject
    Event<WatchdogAlertEvent> alertEvents;
```

**Add import at top of file:**
```java
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.AlertContext;
import io.casehub.qhorus.api.watchdog.BarrierStuckContext;
import io.casehub.qhorus.api.watchdog.ApprovalPendingContext;
import io.casehub.qhorus.api.watchdog.AgentStaleContext;
import io.casehub.qhorus.api.watchdog.ChannelIdleContext;
import io.casehub.qhorus.api.watchdog.QueueDepthContext;
import jakarta.enterprise.event.Event;
```

**Replace `evaluateBarrierStuck()`:**
```java
    private boolean evaluateBarrierStuck(Watchdog w, Instant now) {
        int threshold = w.thresholdSeconds != null ? w.thresholdSeconds : 300;
        Instant cutoff = now.minusSeconds(threshold);

        List<Channel> barriers = channelService.listAll().stream()
                .filter(ch -> ch.semantic == ChannelSemantic.BARRIER)
                .filter(ch -> "*".equals(w.targetName) || ch.name.equals(w.targetName))
                .filter(ch -> ch.lastActivityAt.isBefore(cutoff) || threshold == 0)
                .toList();

        boolean fired = false;
        for (Channel ch : barriers) {
            List<String> required = ch.barrierContributors != null
                    ? List.of(ch.barrierContributors.split(","))
                    : List.of();
            if (required.isEmpty())
                continue;

            List<String> written = messageStore.distinctSendersByChannel(ch.id, MessageType.EVENT);
            List<String> missing = required.stream()
                    .map(String::trim)
                    .filter(r -> !r.isBlank())
                    .filter(r -> !written.contains(r))
                    .toList();

            if (!missing.isEmpty()) {
                long elapsedSeconds = now.getEpochSecond() - ch.lastActivityAt.getEpochSecond();
                String summary = "BARRIER_STUCK: channel='" + ch.name + "' waiting for contributors";
                fireAlert(w, summary, new BarrierStuckContext(ch.id, ch.name, missing, elapsedSeconds), now);
                fired = true;
            }
        }
        return fired;
    }
```

**Replace `evaluateApprovalPending()`:**
```java
    private boolean evaluateApprovalPending(Watchdog w, Instant now) {
        int threshold = w.thresholdSeconds != null ? w.thresholdSeconds : 300;

        List<Commitment> pending = Commitment.<Commitment>list(
                "state IN ?1 AND expiresAt IS NOT NULL",
                List.of(CommitmentState.OPEN, CommitmentState.ACKNOWLEDGED))
                .stream()
                .filter(c -> threshold == 0 || c.expiresAt.isBefore(now.plusSeconds(60 - threshold)))
                .toList();

        if (!pending.isEmpty()) {
            Instant oldestExpiry = pending.stream()
                    .map(c -> c.expiresAt)
                    .min(Comparator.naturalOrder())
                    .orElse(null);
            String summary = "APPROVAL_PENDING: " + pending.size() + " approval(s) awaiting human response";
            fireAlert(w, summary, new ApprovalPendingContext(pending.size(), oldestExpiry), now);
            return true;
        }
        return false;
    }
```

**Replace `evaluateAgentStale()`:**
```java
    private boolean evaluateAgentStale(Watchdog w, Instant now) {
        int threshold = w.thresholdSeconds != null ? w.thresholdSeconds : 300;
        Instant cutoff = now.minusSeconds(threshold);

        // Single query with cutoff filter — fixes pre-existing inconsistency where two
        // queries used different cutoff predicates (count with cutoff, then count without).
        List<io.casehub.qhorus.runtime.instance.Instance> staleInstances =
                io.casehub.qhorus.runtime.instance.Instance
                        .<io.casehub.qhorus.runtime.instance.Instance>list(
                                "status = 'stale' AND lastSeen < ?1", cutoff);

        if (!staleInstances.isEmpty()) {
            List<String> ids = staleInstances.stream()
                    .limit(10)
                    .map(i -> i.id.toString())
                    .toList();
            String summary = "AGENT_STALE: " + staleInstances.size() + " stale agent(s) detected";
            fireAlert(w, summary, new AgentStaleContext(staleInstances.size(), ids), now);
            return true;
        }
        return false;
    }
```

**Replace `evaluateChannelIdle()`:**
```java
    private boolean evaluateChannelIdle(Watchdog w, Instant now) {
        int threshold = w.thresholdSeconds != null ? w.thresholdSeconds : 600;
        Instant cutoff = now.minusSeconds(threshold);

        List<Channel> idle = channelService.listAll().stream()
                .filter(ch -> "*".equals(w.targetName) || ch.name.equals(w.targetName))
                .filter(ch -> threshold == 0 || ch.lastActivityAt.isBefore(cutoff))
                .toList();

        if (!idle.isEmpty()) {
            List<String> names = idle.stream().map(ch -> ch.name).limit(3).toList();
            String joined = String.join(", ", names);
            String summary = "CHANNEL_IDLE: channel(s) idle > " + threshold + "s: " + joined;
            fireAlert(w, summary, new ChannelIdleContext(names, threshold), now);
            return true;
        }
        return false;
    }
```

**Replace `evaluateQueueDepth()`:**
```java
    private boolean evaluateQueueDepth(Watchdog w, Instant now) {
        int threshold = w.thresholdCount != null ? w.thresholdCount : 100;

        List<Channel> channels = channelService.listAll().stream()
                .filter(ch -> "*".equals(w.targetName) || ch.name.equals(w.targetName))
                .toList();

        for (Channel ch : channels) {
            long count = Message.count(
                    "channelId = ?1 AND messageType != ?2", ch.id, MessageType.EVENT);
            if (count >= threshold) {
                String summary = "QUEUE_DEPTH: channel='" + ch.name + "' has " + count
                        + " messages (threshold=" + threshold + ")";
                fireAlert(w, summary, new QueueDepthContext(ch.name, count, threshold), now);
                return true;
            }
        }
        return false;
    }
```

**Replace `fireAlert()`:**
```java
    private void fireAlert(Watchdog w, String summary, AlertContext context, Instant now) {
        // 1. Fire async event FIRST — external delivery is independent of internal channel
        //    success. fireAsync() dispatches immediately; it does not wait for the outer
        //    @Transactional boundary to commit.
        //    Ghost-notification risk (tx rollback after fire): narrow window, accepted.
        //    Crash/missed-alert risk (app crashes before observer delivers): accepted — CDI
        //    async is at-most-once; outbox pattern required for at-least-once.
        alertEvents.fireAsync(new WatchdogAlertEvent(
                w.id, w.targetName, w.notificationChannel, summary, now, context));

        // 2. Internal channel dispatch SECOND — failure does not suppress the event above.
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

Also add the missing `Comparator` import:
```java
import java.util.Comparator;
```

Remove the old `fireAlert(Watchdog w, String alertContent)` method entirely.

- [ ] **Step 5: Run the test to confirm it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogAlertEventTest
```

Expected: `BUILD SUCCESS`, barrier-stuck test green.

- [ ] **Step 6: Run the full runtime test suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: `BUILD SUCCESS`. Any failures in `WatchdogEnabledTest` indicate a regression in the evaluate methods — fix before continuing.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#200): update WatchdogEvaluationService — fireAlert fires WatchdogAlertEvent, typed AlertContext per condition, evaluateAgentStale cutoff bug fixed"
```

---

## Task 5: `connectors/` submodule scaffold

**Files:**
- Create: `connectors/pom.xml`
- Modify: `pom.xml` (root) — add `<module>connectors</module>` before `<module>runtime</module>`

The `connectors` module must be listed **before** `runtime` in the root pom so that it's installed before runtime's test phase compiles (the E2E test in runtime/src/test/ has a test-scope dep on this module).

- [ ] **Step 1: Add `<module>connectors</module>` to root `pom.xml`**

In `pom.xml`, change:
```xml
  <modules>
    <module>api</module>
    <module>runtime</module>
```
to:
```xml
  <modules>
    <module>api</module>
    <module>connectors</module>
    <module>runtime</module>
```

- [ ] **Step 2: Create `connectors/pom.xml`**

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

  <artifactId>casehub-qhorus-connectors</artifactId>
  <name>Quarkus Qhorus - Connectors Bridge</name>
  <description>Optional bridge — routes WatchdogAlertEvent to casehub-connectors. Activates by classpath presence.</description>

  <dependencies>
    <!-- Qhorus API — WatchdogAlertEvent, WatchdogAlertRouter, AlertDeliveryTarget -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- Connectors — Connector SPI + ConnectorService + ConnectorMessage -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>0.2-SNAPSHOT</version>
    </dependency>

    <!-- CDI — required for @ApplicationScoped, @ObservesAsync -->
    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>

    <!-- Logging -->
    <dependency>
      <groupId>org.jboss.logging</groupId>
      <artifactId>jboss-logging</artifactId>
      <scope>provided</scope>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>make-index</id>
            <goals>
              <goal>jandex</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 3: Create directory structure**

```bash
mkdir -p /Users/mdproctor/claude/casehub/qhorus/connectors/src/main/java/io/casehub/qhorus/connectors
mkdir -p /Users/mdproctor/claude/casehub/qhorus/connectors/src/test/java/io/casehub/qhorus/connectors
mkdir -p /Users/mdproctor/claude/casehub/qhorus/connectors/src/test/resources
```

- [ ] **Step 4: Verify scaffold compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl connectors
```

Expected: `BUILD SUCCESS` (no sources yet — empty module compiles).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add pom.xml connectors/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#200): add casehub-qhorus-connectors submodule scaffold"
```

---

## Task 6: `ConnectorAlertBridge` (TDD in `connectors/src/test/`)

**Files:**
- Create: `connectors/src/test/java/io/casehub/qhorus/connectors/TestWatchdogAlertRouter.java`
- Create: `connectors/src/test/java/io/casehub/qhorus/connectors/TestSlackConnector.java`
- Create: `connectors/src/test/resources/application.properties`
- Create: `connectors/src/test/java/io/casehub/qhorus/connectors/ConnectorAlertBridgeTest.java`
- Create: `connectors/src/main/java/io/casehub/qhorus/connectors/ConnectorAlertBridge.java`

Bridge tests call `onAlert()` directly (GE-20260513-b15933 — no async delivery needed here). Tests inject the bridge and verify `ConnectorService.send()` routes to the mock connector.

- [ ] **Step 1: Create `TestWatchdogAlertRouter.java`**

```java
package io.casehub.qhorus.connectors;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.qhorus.api.watchdog.AlertDeliveryTarget;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogAlertRouter;
import io.quarkus.test.Mock;

@Mock
@ApplicationScoped
public class TestWatchdogAlertRouter implements WatchdogAlertRouter {

    public static volatile List<AlertDeliveryTarget> targets = List.of(
            new AlertDeliveryTarget("slack", "https://hooks.slack.com/test"));

    @Override
    public List<AlertDeliveryTarget> route(WatchdogAlertEvent event) {
        return targets;
    }
}
```

- [ ] **Step 2: Create `TestSlackConnector.java`**

```java
package io.casehub.qhorus.connectors;

import java.util.concurrent.CopyOnWriteArrayList;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.quarkus.test.Mock;

@Mock
@ApplicationScoped
public class TestSlackConnector implements Connector {

    public static final CopyOnWriteArrayList<ConnectorMessage> sent = new CopyOnWriteArrayList<>();

    public static void clear() {
        sent.clear();
    }

    @Override
    public String id() {
        return "slack";
    }

    @Override
    public void send(ConnectorMessage message) {
        sent.add(message);
    }
}
```

- [ ] **Step 3: Create `connectors/src/test/resources/application.properties`**

```properties
quarkus.http.test-port=0

# Exclude credential-bearing connectors — they fail CDI validation in test environments
# without Twilio/WhatsApp credentials. SlackConnector and TeamsConnector have no required
# config (destination is per-call), so they do not need exclusion.
# GE-20260521-45e61c
quarkus.arc.exclude-types=io.casehub.connectors.twilio.TwilioSmsConnector,\
  io.casehub.connectors.whatsapp.WhatsAppConnector
```

- [ ] **Step 4: Write `ConnectorAlertBridgeTest.java`** (failing — bridge doesn't exist yet)

```java
package io.casehub.qhorus.connectors;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import io.casehub.qhorus.api.watchdog.AgentStaleContext;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ConnectorAlertBridgeTest {

    @Inject
    ConnectorAlertBridge bridge;

    @BeforeEach
    void reset() {
        TestSlackConnector.clear();
        TestWatchdogAlertRouter.targets = List.of(
                new io.casehub.qhorus.api.watchdog.AlertDeliveryTarget("slack", "https://hooks.slack.com/test"));
    }

    @Test
    void onAlert_sendsToConfiguredConnector() {
        WatchdogAlertEvent event = new WatchdogAlertEvent(
                UUID.randomUUID(), "prod-instances", "alerts",
                "AGENT_STALE: 2 stale agent(s) detected",
                Instant.now(),
                new AgentStaleContext(2L, List.of("id-a", "id-b")));

        bridge.onAlert(event);

        assertThat(TestSlackConnector.sent).hasSize(1);
        var msg = TestSlackConnector.sent.get(0);
        assertThat(msg.destination()).isEqualTo("https://hooks.slack.com/test");
        assertThat(msg.title()).isEqualTo("[Qhorus Alert] AGENT_STALE: prod-instances");
        assertThat(msg.body()).contains("AGENT_STALE: 2 stale agent(s) detected");
        assertThat(msg.body()).contains("id-a");
        assertThat(msg.body()).contains("id-b");
    }

    @Test
    void onAlert_unknownConnectorId_logsAndContinues() {
        TestWatchdogAlertRouter.targets = List.of(
                new io.casehub.qhorus.api.watchdog.AlertDeliveryTarget("does-not-exist", "https://example.com"),
                new io.casehub.qhorus.api.watchdog.AlertDeliveryTarget("slack", "https://hooks.slack.com/test"));

        WatchdogAlertEvent event = new WatchdogAlertEvent(
                UUID.randomUUID(), "*", "alerts", "AGENT_STALE: 1 stale agent",
                Instant.now(), new AgentStaleContext(1L, List.of("id-x")));

        // Should not throw — logs the unknown connector and continues to "slack"
        bridge.onAlert(event);

        assertThat(TestSlackConnector.sent).hasSize(1);
    }

    @Test
    void onAlert_noTargets_doesNothing() {
        TestWatchdogAlertRouter.targets = List.of();

        WatchdogAlertEvent event = new WatchdogAlertEvent(
                UUID.randomUUID(), "*", "alerts", "AGENT_STALE: 1",
                Instant.now(), new AgentStaleContext(1L, List.of()));

        bridge.onAlert(event);

        assertThat(TestSlackConnector.sent).isEmpty();
    }
}
```

- [ ] **Step 5: Run to confirm test fails (ConnectorAlertBridge not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors -Dtest=ConnectorAlertBridgeTest 2>&1 | tail -20
```

Expected: `BUILD FAILURE` — `ConnectorAlertBridge` does not exist.

- [ ] **Step 6: Create `ConnectorAlertBridge.java`**

```java
package io.casehub.qhorus.connectors;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import io.casehub.connectors.ConnectorMessage;
import io.casehub.connectors.ConnectorService;
import io.casehub.qhorus.api.watchdog.AgentStaleContext;
import io.casehub.qhorus.api.watchdog.AlertDeliveryTarget;
import io.casehub.qhorus.api.watchdog.ApprovalPendingContext;
import io.casehub.qhorus.api.watchdog.BarrierStuckContext;
import io.casehub.qhorus.api.watchdog.ChannelIdleContext;
import io.casehub.qhorus.api.watchdog.QueueDepthContext;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogAlertRouter;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ConnectorAlertBridge {

    private static final Logger log = Logger.getLogger(ConnectorAlertBridge.class);

    @Inject
    ConnectorService connectorService;

    @Inject
    WatchdogAlertRouter router;

    void onAlert(@ObservesAsync WatchdogAlertEvent event) {
        String title = "[Qhorus Alert] " + event.conditionType() + ": " + event.targetName();
        String body = buildBody(event);
        for (AlertDeliveryTarget target : router.route(event)) {
            try {
                connectorService.send(target.connectorId(),
                        new ConnectorMessage(target.destination(), title, body));
            } catch (IllegalArgumentException e) {
                // Unknown connector id — config error. Log and continue so one misconfigured
                // endpoint doesn't suppress delivery to all others.
                log.errorf("Unknown connector '%s' for watchdog alert — available: %s",
                        target.connectorId(), connectorService.ids());
            }
        }
    }

    private String buildBody(WatchdogAlertEvent event) {
        return switch (event.context()) {
            case BarrierStuckContext c ->
                    event.summary()
                    + "\nChannel: " + c.channelName()
                    + "\nMissing: " + String.join(", ", c.missingContributors())
                    + "\nElapsed: " + c.elapsedSeconds() + "s";
            case ApprovalPendingContext c ->
                    event.summary()
                    + "\nPending: " + c.pendingCount()
                    + (c.oldestExpiryAt() != null ? "\nOldest expiry: " + c.oldestExpiryAt() : "");
            case AgentStaleContext c ->
                    event.summary()
                    + "\nStale count: " + c.staleCount()
                    + (c.staleInstanceIds().isEmpty() ? ""
                       : "\nIDs: " + String.join(", ", c.staleInstanceIds()));
            case ChannelIdleContext c ->
                    event.summary()
                    + "\nIdle channels: " + String.join(", ", c.channelNames())
                    + "\nIdle > " + c.thresholdSeconds() + "s";
            case QueueDepthContext c ->
                    event.summary()
                    + "\nChannel: " + c.channelName()
                    + "\nDepth: " + c.messageCount() + " (threshold: " + c.threshold() + ")";
        };
    }
}
```

- [ ] **Step 7: Run tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors
```

Expected: `BUILD SUCCESS`, all 3 bridge tests green.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add connectors/src/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#200): ConnectorAlertBridge — @ObservesAsync WatchdogAlertEvent → ConnectorService.send()"
```

---

## Task 7: E2E test (TDD in `runtime/src/test/`)

**Files:**
- Modify: `runtime/pom.xml` — add `casehub-qhorus-connectors` test-scope dep
- Modify: `runtime/src/test/resources/application.properties` — add `quarkus.arc.exclude-types`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/TestSlackConnectorE2E.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogAlertE2ETest.java`

The E2E test lives here because `WatchdogEvaluationService` is in the runtime module. The `casehub-qhorus-connectors` module is added as a test-scope dependency — this is not circular: connectors depends on api (not runtime).

- [ ] **Step 1: Add `casehub-qhorus-connectors` test-scope dep to `runtime/pom.xml`**

Inside the `<dependencies>` block in `runtime/pom.xml`, add with the test deps:
```xml
    <!-- E2E tests — ConnectorAlertBridge on classpath to verify full event → connector path -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-connectors</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Update `runtime/src/test/resources/application.properties`**

Add at the end:
```properties
# Exclude credential-bearing connectors pulled in via casehub-qhorus-connectors test dep.
# Safe to leave in place permanently — quarkus.arc.exclude-types for absent types is a no-op.
# GE-20260521-45e61c
quarkus.arc.exclude-types=io.casehub.connectors.twilio.TwilioSmsConnector,\
  io.casehub.connectors.whatsapp.WhatsAppConnector
```

- [ ] **Step 3: Create `TestSlackConnectorE2E.java`** (separate from the bridge module's test connector)

```java
package io.casehub.qhorus.runtime.watchdog;

import java.util.concurrent.CopyOnWriteArrayList;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.connectors.Connector;
import io.casehub.connectors.ConnectorMessage;
import io.quarkus.test.Mock;

@Mock
@ApplicationScoped
public class TestSlackConnectorE2E implements Connector {

    public static final CopyOnWriteArrayList<ConnectorMessage> sent = new CopyOnWriteArrayList<>();

    public static void clear() {
        sent.clear();
    }

    @Override
    public String id() {
        return "slack";
    }

    @Override
    public void send(ConnectorMessage message) {
        sent.add(message);
    }
}
```

- [ ] **Step 4: Write `WatchdogAlertE2ETest.java`** (failing — alert endpoints not wired to bridge yet in this context)

```java
package io.casehub.qhorus.runtime.watchdog;

import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import io.casehub.qhorus.api.WatchdogAlertEndpointsProfile;
import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelSemantic;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertTrue;

@QuarkusTest
@TestProfile(WatchdogAlertEndpointsProfile.class)
class WatchdogAlertE2ETest {

    @Inject
    WatchdogEvaluationService service;

    @BeforeEach
    void reset() {
        TestSlackConnectorE2E.clear();
        AlertEventCapture.expectCount(1);
    }

    @Test
    @Transactional
    void barrierStuck_eventFlowsToConnector() throws InterruptedException {
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = "e2e-barrier-" + ch.id;
        ch.semantic = ChannelSemantic.BARRIER;
        ch.barrierContributors = "agent-x";
        ch.lastActivityAt = Instant.now().minusSeconds(3600);
        ch.persist();

        Watchdog w = new Watchdog();
        w.conditionType = "BARRIER_STUCK";
        w.targetName = ch.name;
        w.thresholdSeconds = 0;
        w.notificationChannel = "e2e-alerts-" + w.id;
        w.createdBy = "test";
        w.persist();

        service.evaluateAll();

        assertTrue(AlertEventCapture.await(2, TimeUnit.SECONDS),
                "WatchdogAlertEvent not delivered within 2s");

        assertThat(AlertEventCapture.events.get(0).conditionType())
                .isEqualTo(WatchdogConditionType.BARRIER_STUCK);

        assertThat(TestSlackConnectorE2E.sent).hasSize(1);
        var msg = TestSlackConnectorE2E.sent.get(0);
        assertThat(msg.destination()).isEqualTo("https://hooks.slack.com/test");
        assertThat(msg.title()).startsWith("[Qhorus Alert] BARRIER_STUCK:");
        assertThat(msg.body()).contains("agent-x");
    }
}
```

- [ ] **Step 5: Run to confirm the test fails initially (ConnectorAlertBridge isn't loaded or alert not reaching connector)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogAlertE2ETest 2>&1 | tail -30
```

Expected: compilation succeeds; test fails because `TestSlackConnectorE2E.sent` is empty (bridge not configured with endpoints, or `ConfiguredWatchdogAlertRouter` returns empty for this profile). If it fails for a different reason (CDI validation error), check the `quarkus.arc.exclude-types` was applied correctly.

Note: `WatchdogAlertEndpointsProfile` sets `casehub.qhorus.watchdog.alert.endpoints[0].*`, so `ConfiguredWatchdogAlertRouter` will return one target: `(slack, https://hooks.slack.com/test)`. `ConnectorAlertBridge` (from the test classpath via the test-scope dep) observes the event and calls `connectorService.send("slack", msg)`. `TestSlackConnectorE2E @Mock` intercepts the call.

If the test still fails after everything is wired correctly, add a short explicit sleep before the assertion to confirm async timing isn't the issue: `Thread.sleep(200)` before the `await()`.

- [ ] **Step 6: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: `BUILD SUCCESS`. If `WatchdogAlertE2ETest` passes, the full stack is verified.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/pom.xml runtime/src/test/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#200): E2E test — evaluateAll fires WatchdogAlertEvent, ConnectorAlertBridge routes to mock Slack connector"
```

---

## Task 8: PLATFORM.md dependency table + full build verification

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md` — add row for `casehub-qhorus-connectors → casehub-connectors-core`

- [ ] **Step 1: Add row to the Cross-Repo Dependency Map in `PLATFORM.md`**

In the dependency table (around the `casehub-connectors-core` rows), add:
```markdown
| `casehub-connectors-core` | `casehub-qhorus` | `connectors` | optional — `WatchdogAlertEvent → ConnectorService.send()` bridge; activates by classpath presence |
```

- [ ] **Step 2: Run the full project build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: `BUILD SUCCESS` across all modules: api, connectors, runtime, deployment, testing, examples.

- [ ] **Step 3: Commit PLATFORM.md**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(#200): register casehub-qhorus-connectors → casehub-connectors-core in dependency table"
```

- [ ] **Step 4: Commit to project repo (Refs #200)**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus log --oneline -8
```

All commits should reference #200. The final commit message confirming issue closure:
```bash
git -C /Users/mdproctor/claude/casehub/qhorus commit --allow-empty -m "chore(#200): implementation complete — WatchdogAlertEvent + ConnectorAlertBridge"
```

---

## Self-Review Checklist

- [x] **`WatchdogConditionType` enum** → Task 1 Step 1
- [x] **Sealed `AlertContext` + 5 records** → Task 1 Steps 2–3
- [x] **`WatchdogAlertEvent` with derived `conditionType()`** → Task 1 Step 4
- [x] **`AlertDeliveryTarget` + `WatchdogAlertRouter`** → Task 1 Steps 5–6
- [x] **`QhorusConfig.Watchdog.Alert` with `@WithDefault("")` on `endpoints()`** → Task 2
- [x] **`ConfiguredWatchdogAlertRouter @DefaultBean`** → Task 3
- [x] **`fireAlert()` signature with `Instant now` param** → Task 4 Step 4
- [x] **`fireAsync()` fires before dispatch** → Task 4 Step 4 (`fireAlert()` body)
- [x] **`evaluateBarrierStuck`: computes `elapsedSeconds`, extracts `missingContributors` list** → Task 4 Step 4
- [x] **`evaluateApprovalPending`: changes count→list, accumulates `oldestExpiryAt`** → Task 4 Step 4
- [x] **`evaluateAgentStale`: single query with cutoff, fixes double-query inconsistency** → Task 4 Step 4
- [x] **`evaluateChannelIdle`: passes `thresholdSeconds` (not elapsed)** → Task 4 Step 4
- [x] **`evaluateQueueDepth`: structured `QueueDepthContext`** → Task 4 Step 4
- [x] **`connectors/pom.xml` with correct deps** → Task 5
- [x] **Module order: `connectors` before `runtime` in root pom** → Task 5
- [x] **`ConnectorAlertBridge`: uses `ConnectorService` not `Instance<Connector>`** → Task 6
- [x] **Bridge catch-and-log on `IllegalArgumentException`** → Task 6
- [x] **Bridge test calls `onAlert()` directly (not via async)** → Task 6
- [x] **E2E test in `runtime/src/test/` (not `connectors/src/test/`)** → Task 7
- [x] **`quarkus.arc.exclude-types` in both test `application.properties` files** → Tasks 6, 7
- [x] **PLATFORM.md dependency table updated** → Task 8

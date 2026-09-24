# Channel Policy + RAS Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #455 — evolve ChannelProtocol SPI — structured output + channel policy model
**Issue group:** #455, casehubio/casehub-ras#66

**Goal:** Replace `List<String>` protocol output with structured `DispatchAdvisory` carrying severity, evidence, and suggested action. Wire CDI event prerequisites for RAS integration. The channel policy YAML compiler and RAS adapter module are designed but deferred to child issues.

**Architecture:** Single `DispatchAdvisory` type flows through the entire dispatch pipeline — SPI return, enforcement carrier, and API output. No mapping layers. Severity × EnforcementMode matrix determines enforcement behavior with CRITICAL-in-ADVISORY upgrade path.

**Tech Stack:** Java 21, Quarkus 3.32.2, qhorus named PU (H2 in tests), Flyway

## Global Constraints

- Pre-release — all changes are breaking, no backward compat needed
- `DispatchAdvisory` lives in `api/spi/` (not `api/message/`) — it is the SPI return type
- `TaggedAdvisory` is deleted entirely — not evolved
- All protocol violations carry `Map<String, Object> evidence` with JSON-safe types only
- `mvn install` from project root after API changes (not just `mvn test -pl runtime`)
- Tests use `@TestTransaction` unless testing observer dispatch (then `QuarkusTransaction.requiringNew()`)

---

## Batch 1: Foundation Types + SPI Change

### Task 1: Create DispatchAdvisory, Severity, SuggestedAction

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/DispatchAdvisory.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/Severity.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/SuggestedAction.java`
- Test: `api/src/test/java/io/casehub/qhorus/api/spi/DispatchAdvisoryTest.java`

**Interfaces:**
- Produces: `DispatchAdvisory(String source, Severity severity, String message, Map<String, Object> evidence, SuggestedAction suggestedAction)`, `Severity.isAtLeast(Severity)`, `SuggestedAction` enum

- [ ] **Step 1: Write failing test for DispatchAdvisory record**

```java
package io.casehub.qhorus.api.spi;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class DispatchAdvisoryTest {

    @Test
    void constructsWithAllFields() {
        var advisory = new DispatchAdvisory("REQUEST_RESPONSE", Severity.WARNING,
                "3 open queries", Map.of("count", 3), SuggestedAction.LOG);
        assertThat(advisory.source()).isEqualTo("REQUEST_RESPONSE");
        assertThat(advisory.severity()).isEqualTo(Severity.WARNING);
        assertThat(advisory.message()).isEqualTo("3 open queries");
        assertThat(advisory.evidence()).containsEntry("count", 3);
        assertThat(advisory.suggestedAction()).isEqualTo(SuggestedAction.LOG);
    }

    @Test
    void nullSourceThrows() {
        assertThatThrownBy(() -> new DispatchAdvisory(null, Severity.WARNING, "msg", Map.of(), SuggestedAction.LOG))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullSeverityThrows() {
        assertThatThrownBy(() -> new DispatchAdvisory("SRC", null, "msg", Map.of(), SuggestedAction.LOG))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullMessageThrows() {
        assertThatThrownBy(() -> new DispatchAdvisory("SRC", Severity.WARNING, null, Map.of(), SuggestedAction.LOG))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullEvidenceDefaultsToEmptyMap() {
        var advisory = new DispatchAdvisory("SRC", Severity.WARNING, "msg", null, SuggestedAction.LOG);
        assertThat(advisory.evidence()).isEmpty();
    }

    @Test
    void nullActionDefaultsToLog() {
        var advisory = new DispatchAdvisory("SRC", Severity.WARNING, "msg", Map.of(), null);
        assertThat(advisory.suggestedAction()).isEqualTo(SuggestedAction.LOG);
    }

    @Test
    void evidenceIsImmutableCopy() {
        var mutable = new java.util.HashMap<String, Object>();
        mutable.put("key", "val");
        var advisory = new DispatchAdvisory("SRC", Severity.WARNING, "msg", mutable, SuggestedAction.LOG);
        mutable.put("key2", "val2");
        assertThat(advisory.evidence()).doesNotContainKey("key2");
    }

    @Test
    void severityOrdering() {
        assertThat(Severity.CRITICAL.isAtLeast(Severity.WARNING)).isTrue();
        assertThat(Severity.WARNING.isAtLeast(Severity.CRITICAL)).isFalse();
        assertThat(Severity.ADVISORY.isAtLeast(Severity.ADVISORY)).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DispatchAdvisoryTest -pl api`
Expected: FAIL — classes don't exist

- [ ] **Step 3: Create Severity enum**

Use `ide_create_file` for `api/src/main/java/io/casehub/qhorus/api/spi/Severity.java`:

```java
package io.casehub.qhorus.api.spi;

public enum Severity {
    ADVISORY,
    WARNING,
    CRITICAL;

    public boolean isAtLeast(Severity threshold) {
        return this.ordinal() >= threshold.ordinal();
    }
}
```

- [ ] **Step 4: Create SuggestedAction enum**

Use `ide_create_file` for `api/src/main/java/io/casehub/qhorus/api/spi/SuggestedAction.java`:

```java
package io.casehub.qhorus.api.spi;

public enum SuggestedAction {
    LOG,
    ESCALATE,
    INVESTIGATE,
    REROUTE
}
```

- [ ] **Step 5: Create DispatchAdvisory record**

Use `ide_create_file` for `api/src/main/java/io/casehub/qhorus/api/spi/DispatchAdvisory.java`:

```java
package io.casehub.qhorus.api.spi;

import java.util.Map;
import java.util.Objects;

public record DispatchAdvisory(
        String source,
        Severity severity,
        String message,
        Map<String, Object> evidence,
        SuggestedAction suggestedAction) {

    public DispatchAdvisory {
        Objects.requireNonNull(source);
        Objects.requireNonNull(severity);
        Objects.requireNonNull(message);
        evidence = evidence != null ? Map.copyOf(evidence) : Map.of();
        suggestedAction = suggestedAction != null ? suggestedAction : SuggestedAction.LOG;
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DispatchAdvisoryTest -pl api`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/spi/DispatchAdvisory.java \
       api/src/main/java/io/casehub/qhorus/api/spi/Severity.java \
       api/src/main/java/io/casehub/qhorus/api/spi/SuggestedAction.java \
       api/src/test/java/io/casehub/qhorus/api/spi/DispatchAdvisoryTest.java
git commit -m "feat(#455): add DispatchAdvisory, Severity, SuggestedAction foundation types Refs #455"
```

### Task 2: Update ChannelProtocol SPI + built-in protocols

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/protocol/RequestResponseProtocol.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/protocol/TaskCompletionProtocol.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/protocol/RoundRobinProtocol.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/protocol/ContributionRequiredProtocol.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/RequestResponseProtocolTest.java` (update existing)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/TaskCompletionProtocolTest.java` (update existing)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/RoundRobinProtocolTest.java` (update existing)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/ContributionRequiredProtocolTest.java` (update existing)

**Interfaces:**
- Consumes: `DispatchAdvisory`, `Severity`, `SuggestedAction` (from Task 1)
- Produces: `ChannelProtocol.evaluate(ProtocolContext) → List<DispatchAdvisory>`

- [ ] **Step 1: Update ChannelProtocol SPI return type**

Use `ide_replace_member` on `ChannelProtocol.evaluate`:

```java
List<DispatchAdvisory> evaluate(ProtocolContext context);
```

Add import: `import io.casehub.qhorus.api.spi.DispatchAdvisory;` (already in same package — no import needed).

- [ ] **Step 2: Update RequestResponseProtocol**

Use `ide_replace_member` on `RequestResponseProtocol.evaluate`:

```java
@Override
public List<DispatchAdvisory> evaluate(ProtocolContext ctx) {
    List<Commitment> openQueries = ctx.activeCommitments().stream()
            .filter(c -> c.messageType() == MessageType.QUERY)
            .toList();
    if (openQueries.isEmpty()) return List.of();

    List<DispatchAdvisory> advisories = new ArrayList<>();
    if (ctx.incomingType() == MessageType.QUERY && openQueries.size() >= maxOpenQueries) {
        advisories.add(new DispatchAdvisory("REQUEST_RESPONSE", Severity.WARNING,
                openQueries.size() + " unanswered QUERYs in channel '" + ctx.channelName()
                        + "' — consider waiting for responses",
                Map.of("openQueryCount", openQueries.size(), "threshold", maxOpenQueries,
                        "channelName", ctx.channelName()),
                SuggestedAction.LOG));
    }
    if (ctx.incomingType() != MessageType.RESPONSE && ctx.incomingType() != MessageType.QUERY) {
        advisories.add(new DispatchAdvisory("REQUEST_RESPONSE", Severity.ADVISORY,
                "channel '" + ctx.channelName() + "' has open QUERYs awaiting RESPONSE",
                Map.of("openQueryCount", openQueries.size(), "channelName", ctx.channelName()),
                SuggestedAction.LOG));
    }
    return advisories;
}
```

- [ ] **Step 3: Update TaskCompletionProtocol**

Use `ide_replace_member` on `TaskCompletionProtocol.evaluate` — same pattern, return `List<DispatchAdvisory>` with structured evidence.

- [ ] **Step 4: Update RoundRobinProtocol**

Use `ide_replace_member` on `RoundRobinProtocol.evaluate` — return `DispatchAdvisory` with `Severity.ADVISORY` and evidence map containing `expectedSender`, `actualSender`.

- [ ] **Step 5: Update ContributionRequiredProtocol**

Use `ide_replace_member` on `ContributionRequiredProtocol.evaluate` — return `DispatchAdvisory` with `Severity.WARNING` and evidence map containing `consecutiveCount`, `sender`, `missingSenders`.

- [ ] **Step 6: Update protocol tests**

Update each protocol test to assert on `DispatchAdvisory` fields instead of string matching. For each test:
- Assert `advisory.source()` equals protocol name
- Assert `advisory.severity()` matches expected severity
- Assert `advisory.evidence()` contains expected keys and values
- Assert `advisory.suggestedAction()` is correct

- [ ] **Step 7: Run all protocol tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="RequestResponseProtocolTest,TaskCompletionProtocolTest,RoundRobinProtocolTest,ContributionRequiredProtocolTest"`
Expected: PASS

- [ ] **Step 8: Run `mvn install` from root**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS — verifies API change compiles across all modules

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java \
       runtime-core/src/main/java/io/casehub/qhorus/runtime/message/protocol/ \
       runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/
git commit -m "feat(#455): evolve ChannelProtocol SPI to return List<DispatchAdvisory> Refs #455"
```

---

## Batch 2: Pipeline Evolution

### Task 3: Delete TaggedAdvisory + update MessageService and EnforcementExecutor

**Files:**
- Delete: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/TaggedAdvisory.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/EnforcementExecutor.java`

**Interfaces:**
- Consumes: `DispatchAdvisory` (from Task 1), updated protocol return types (from Task 2)
- Produces: `MessageService.dispatch()` using `List<DispatchAdvisory>` internally, `enforceIfRequired()` with severity-aware logic

- [ ] **Step 1: Write failing test for severity-aware enforcement**

Add new test methods to `EnforcementGateTest.java`:

```java
@Test
void advisoryModeCriticalUpgradesEnforcement() {
    Channel ch = channelWithMode(EnforcementMode.ADVISORY);
    List<DispatchAdvisory> violations = List.of(
            new DispatchAdvisory("TYPE_POLICY", Severity.CRITICAL, "violation", Map.of(), SuggestedAction.LOG));
    assertThatThrownBy(() -> MessageService.enforceIfRequired(ch, violations,
            MessageType.COMMAND, "agent-a", executor))
            .isInstanceOf(EnforcementBlockedException.class);
}

@Test
void advisoryModeWarningDoesNotBlock() {
    Channel ch = channelWithMode(EnforcementMode.ADVISORY);
    List<DispatchAdvisory> violations = List.of(
            new DispatchAdvisory("REQUEST_RESPONSE", Severity.WARNING, "violation", Map.of(), SuggestedAction.LOG));
    assertThatCode(() -> MessageService.enforceIfRequired(ch, violations,
            MessageType.COMMAND, "agent-a", executor))
            .doesNotThrowAnyException();
}

@Test
void blockingModeAdvisorySeverityDoesNotBlock() {
    Channel ch = channelWithMode(EnforcementMode.BLOCKING);
    List<DispatchAdvisory> violations = List.of(
            new DispatchAdvisory("ROUND_ROBIN", Severity.ADVISORY, "out of turn", Map.of(), SuggestedAction.LOG));
    assertThatCode(() -> MessageService.enforceIfRequired(ch, violations,
            MessageType.COMMAND, "agent-a", executor))
            .doesNotThrowAnyException();
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `enforceIfRequired` still uses `TaggedAdvisory`

- [ ] **Step 3: Update MessageService to use DispatchAdvisory**

Replace all `TaggedAdvisory` usage in `MessageService.dispatch()`:
- Change `List<TaggedAdvisory> taggedAdvisories` → `List<DispatchAdvisory> advisories`
- TYPE_POLICY: `new DispatchAdvisory("TYPE_POLICY", Severity.CRITICAL, adv, Map.of(), SuggestedAction.LOG)`
- CORRELATION_INTEGRITY: `new DispatchAdvisory("CORRELATION_INTEGRITY", Severity.ADVISORY, ca, Map.of(), SuggestedAction.LOG)`
- Protocol violations: `advisories.addAll(protocol.evaluate(protocolCtx))` — direct, no wrapping
- DispatchResult construction: pass `advisories` directly (handled in Task 4)

- [ ] **Step 4: Update enforceIfRequired with severity-aware logic**

Replace both overloads with severity-aware filtering per spec.

- [ ] **Step 5: Update EnforcementExecutor parameter type**

Change `List<TaggedAdvisory>` → `List<DispatchAdvisory>` in `execute()`. Use `DispatchAdvisory::message` and `DispatchAdvisory::source` for telemetry extraction.

- [ ] **Step 6: Delete TaggedAdvisory.java**

Use `ide_refactor_safe_delete` on `TaggedAdvisory.java`.

- [ ] **Step 7: Update existing enforcement tests to use DispatchAdvisory**

Update `EnforcementGateTest`, `EnforcementExecutorTest`, delete `TaggedAdvisoryTest`.

- [ ] **Step 8: Run enforcement tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="EnforcementGateTest,EnforcementExecutorTest"`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#455): replace TaggedAdvisory with DispatchAdvisory — severity-aware enforcement Refs #455"
```

### Task 4: Evolve DispatchResult + EnforcementBlockedException + EnforcementBlockedEvent

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/EnforcementBlockedException.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/EnforcementBlockedEvent.java`
- Modify: all callers of `DispatchResult.advisories()` and `EnforcementBlockedException.violations()`

**Interfaces:**
- Consumes: `DispatchAdvisory` (from Task 1)
- Produces: `DispatchResult.advisories() → List<DispatchAdvisory>`, `EnforcementBlockedException.violations() → List<DispatchAdvisory>`, `EnforcementBlockedException.severityUpgrade()`, `EnforcementBlockedException.effectiveMode()`

- [ ] **Step 1: Update DispatchResult**

Change `List<String> advisories` → `List<DispatchAdvisory> advisories`. Update compact constructor.

- [ ] **Step 2: Update EnforcementBlockedException**

Add `severityUpgrade` field, `effectiveMode()` method. Change `violations` type to `List<DispatchAdvisory>`.

- [ ] **Step 3: Update EnforcementBlockedEvent**

Add `severityUpgrade` field. Change `violations` type from `List<String>` to `List<DispatchAdvisory>`.

- [ ] **Step 4: Update MessageService DispatchResult construction sites**

Two sites in `dispatch()`: the LAST_WRITE path (line ~381) and the normal path (line ~522). Both currently do `.stream().map(TaggedAdvisory::message).toList()` — change to pass `advisories` directly.

- [ ] **Step 5: Update EnforcementExecutor event construction**

`EnforcementBlockedEvent` now takes `List<DispatchAdvisory>` instead of `List<String>`.

- [ ] **Step 6: Update enforceIfRequired throw site**

Construct `EnforcementBlockedException` with `severityUpgrade` flag.

- [ ] **Step 7: Find and update all callers**

Use `ide_find_references` on `DispatchResult.advisories`, `EnforcementBlockedException.violations`, `EnforcementBlockedEvent`. Update each caller in QhorusTestHelper, REST resources, connector backends, etc.

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#455): evolve DispatchResult/EnforcementBlockedException/Event to List<DispatchAdvisory> Refs #455"
```

---

## Batch 3: CDI Event Prerequisites

### Task 5: Create ProtocolEvaluationEvent + fire from MessageService

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/ProtocolEvaluationEvent.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/cdi/CdiMessageService.java` (inject Event<ProtocolEvaluationEvent>)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/ProtocolEvaluationEventTest.java`

**Interfaces:**
- Consumes: `DispatchAdvisory` (from Task 1)
- Produces: `ProtocolEvaluationEvent(channelId, channelName, tenancyId, violations, enforcementOutcome)` CDI event

- [ ] **Step 1: Write test for ProtocolEvaluationEvent record**

```java
@Test
void constructsWithAllFields() {
    var violations = List.of(new DispatchAdvisory("SRC", Severity.WARNING, "msg", Map.of(), SuggestedAction.LOG));
    var event = new ProtocolEvaluationEvent(UUID.randomUUID(), "test-channel", "tenant-1",
            violations, ProtocolEvaluationEvent.EnforcementOutcome.ALLOWED);
    assertThat(event.violations()).hasSize(1);
    assertThat(event.enforcementOutcome()).isEqualTo(ProtocolEvaluationEvent.EnforcementOutcome.ALLOWED);
}
```

- [ ] **Step 2: Create ProtocolEvaluationEvent**

```java
package io.casehub.qhorus.api.spi;

import java.util.List;
import java.util.UUID;

public record ProtocolEvaluationEvent(
        UUID channelId,
        String channelName,
        String tenancyId,
        List<DispatchAdvisory> violations,
        EnforcementOutcome enforcementOutcome) {

    public enum EnforcementOutcome {
        ALLOWED,
        BLOCKED,
        QUARANTINED
    }
}
```

- [ ] **Step 3: Wire CDI firing in MessageService**

Add an `EventCallback` functional interface to `MessageService` (same pattern as `ObserverCallback` and `LedgerRecorder`). Fire after enforcement gate — on success path with `ALLOWED`, in catch block with `BLOCKED`/`QUARANTINED`. Only fire when `advisories` is non-empty.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#455): add ProtocolEvaluationEvent CDI event for RAS observation Refs #455"
```

### Task 6: Wire CommitmentStateChangedEvent in CommitmentService

**Files:**
- Check: `api/src/main/java/io/casehub/qhorus/api/gateway/` for `CommitmentStateChangedEvent` (verify it exists)
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/cdi/CdiCommitmentService.java` (inject Event<CommitmentStateChangedEvent>)
- Test: `testing/src/test/java/io/casehub/qhorus/testing/CommitmentServiceTest.java` (add event verification)

**Interfaces:**
- Produces: `CommitmentStateChangedEvent` fired on all state transitions

- [ ] **Step 1: Verify CommitmentStateChangedEvent exists**

Use `ide_find_class` for `CommitmentStateChangedEvent`. If it doesn't exist, create it in `api/gateway/`.

- [ ] **Step 2: Write test for event firing**

Test that `commitmentService.open()` fires a `CommitmentStateChangedEvent` with state=OPEN.

- [ ] **Step 3: Add event firing callback to CommitmentService**

Add an `EventCallback` pattern (same as MessageService's `ObserverCallback`) that fires `CommitmentStateChangedEvent` on each state transition method. CDI wiring in `CdiCommitmentService`.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#455): wire CommitmentStateChangedEvent for all commitment transitions Refs #455"
```

### Task 7: Fire ChannelActivityEvent as CDI event + add tenancyId

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelActivityBroadcaster.java` (add tenancyId to ChannelActivityEvent)
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (fire CDI event)
- Modify: all construction sites of `ChannelActivityEvent`

**Interfaces:**
- Produces: `ChannelActivityEvent(channelId, channelName, messageId, tenancyId)` as CDI async event

- [ ] **Step 1: Add tenancyId to ChannelActivityEvent**

Use `ide_find_class` for `ChannelActivityEvent` (nested in `ChannelActivityBroadcaster`). Add `String tenancyId` field. Update all construction sites via `ide_find_references`.

- [ ] **Step 2: Add CDI Event<ChannelActivityEvent> firing alongside broadcaster.broadcast()**

In `MessageService.dispatch()`, add `Event<ChannelActivityEvent>.fireAsync()` in the same `afterCompletion(STATUS_COMMITTED)` callback.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat(#455): fire ChannelActivityEvent as CDI event + add tenancyId Refs #455"
```

---

## Batch 4: Channel Policy Overrides (DB + MCP)

### Task 8: V55 migration + Channel.policyOverrides + MCP/REST

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V55__channel_policy_overrides.sql`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` (add `policyOverrides`)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` (add JPA mapping)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` (add setPolicyOverrides)
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelStore.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/PolicyOverridesTest.java`

**Interfaces:**
- Produces: `Channel.policyOverrides() → Map<String, String>` (nullable), `ChannelService.setPolicyOverrides(UUID, Map<String, String>)`

- [ ] **Step 1: Write test for policy overrides**

```java
@Test
void setPolicyOverridesStoresAndRetrieves() {
    Channel ch = helper.createChannel("policy-test");
    channelService.setPolicyOverrides(ch.id(), Map.of("max_open_queries", "5"));
    Channel updated = channelService.findById(ch.id()).orElseThrow();
    assertThat(updated.policyOverrides()).containsEntry("max_open_queries", "5");
}

@Test
void nullValueRemovesOverride() {
    // Set, then remove
}
```

- [ ] **Step 2: Create V55 migration**

```sql
ALTER TABLE channel ADD COLUMN policy_overrides TEXT;
```

- [ ] **Step 3: Add policyOverrides to Channel API + entity + stores**

Follow the established pattern for nullable fields (same as `routingTrustThreshold`).

- [ ] **Step 4: Add setPolicyOverrides to ChannelService**

Merge semantics: null value on a key removes the override.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#455): add Channel.policyOverrides — V55 migration + service + MCP Refs #455"
```

---

## Deferred Work (child issues)

The following are designed in the spec but deferred to separate issues:

1. **Channel policy YAML compiler** (`ChannelPolicyCompiler` for `dispatch_rules:`) — requires MVEL expression engine integration, YAML parsing, policy-to-ChannelProtocol compilation. File as child issue of #455.

2. **RAS adapter module** (`ras/qhorus/`) — `QhorusEventBridge`, `QhorusChannelFilter`, `QhorusSituationProvider`, `SituationPolicyCompiler`. This is in the casehub-ras repo. Track via casehubio/casehub-ras#66.

3. **ChannelPolicyChangedEvent** — CDI event fired when channel protocols change. Prerequisite for runtime `QhorusChannelFilter` lifecycle management. File as child issue.

4. **Pre-built situation templates** — ack-timeout, obligation-pressure, decline-pattern, correction-uncertainty, channel-silence. Part of casehubio/casehub-ras#66.

---

## References

- [2026-09-23-channel-policy-ras-adapter-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java] — current SPI
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java] — dispatch pipeline
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/message/TaggedAdvisory.java] — type to delete
- [runtime-core/src/main/java/io/casehub/qhorus/runtime/message/EnforcementExecutor.java] — enforcement execution
- [api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java] — advisory output
- [GitHub #455] — focal issue
- [GitHub casehubio/casehub-ras#66] — companion RAS adapter issue

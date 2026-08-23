# Enforcement Modes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #400 — E3: Active governance policies — enforcement modes
**Issue group:** #398, #399, #400, #407

**Goal:** Add configurable enforcement modes (ADVISORY/BLOCKING/QUARANTINE) to channels so protocol and advisory violations can reject dispatches or trigger containment.

**Architecture:** New `EnforcementMode` enum on Channel determines what happens when advisory violations are detected during `MessageService.dispatch()`. A new enforcement gate after the three advisory sources (MessageTypePolicy, CorrelationIntegrityChecker, ChannelProtocol) checks the mode and either passes through (ADVISORY), throws (BLOCKING), or throws + contains (QUARANTINE). Containment and audit actions execute in a `REQUIRES_NEW` CDI bean (`EnforcementExecutor`) so they survive the outer transaction's rollback.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, Flyway, quarkus-mcp-server

## Global Constraints

- Next Flyway migration: V45
- Channel record already has 21 fields + 4 backward-compatible constructors — minimize constructor proliferation
- `DispatchResult` is unchanged — no new fields
- Terminal types (DONE, FAILURE, DECLINE, RESPONSE, HANDOFF) always exempt from enforcement (ADR-0016)
- System senders (containing `:`) always exempt from enforcement
- EVENT type always exempt from enforcement
- Use `telemetryJson()` from `QhorusMcpToolsBase` for audit EVENT telemetry
- `@WrapBusinessError` wraps `IllegalStateException` (and subclasses) for MCP callers

---

## Batch 1: Data model foundation

### Task 1: EnforcementMode enum + EnforcementBlockedException + EnforcementBlockedEvent

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/EnforcementMode.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/EnforcementBlockedException.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/EnforcementBlockedEvent.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/message/EnforcementBlockedExceptionTest.java`

**Interfaces:**
- Produces: `EnforcementMode { ADVISORY, BLOCKING, QUARANTINE }`
- Produces: `EnforcementBlockedException(EnforcementMode, List<String> violationSources, List<String> violations)`
- Produces: `EnforcementBlockedEvent(UUID channelId, String channelName, EnforcementMode mode, String blockedSender, MessageType blockedType, List<String> violations, List<String> violationSources)`

- [ ] **Step 1: Write the test for EnforcementBlockedException**

```java
package io.casehub.qhorus.message;

import io.casehub.qhorus.api.channel.EnforcementMode;
import io.casehub.qhorus.api.message.EnforcementBlockedException;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class EnforcementBlockedExceptionTest {

    @Test
    void extendsIllegalStateException() {
        var ex = new EnforcementBlockedException(
                EnforcementMode.BLOCKING,
                List.of("REQUEST_RESPONSE"),
                List.of("[REQUEST_RESPONSE] too many open queries"));
        assertThat(ex).isInstanceOf(IllegalStateException.class);
        assertThat(ex.mode()).isEqualTo(EnforcementMode.BLOCKING);
        assertThat(ex.violationSources()).containsExactly("REQUEST_RESPONSE");
        assertThat(ex.violations()).hasSize(1);
        assertThat(ex.getMessage()).contains("BLOCKING");
    }

    @Test
    void quarantineModeInMessage() {
        var ex = new EnforcementBlockedException(
                EnforcementMode.QUARANTINE,
                List.of("TYPE_POLICY", "CORRELATION_INTEGRITY"),
                List.of("violation 1", "violation 2"));
        assertThat(ex.getMessage()).contains("QUARANTINE");
        assertThat(ex.violationSources()).containsExactly("TYPE_POLICY", "CORRELATION_INTEGRITY");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EnforcementBlockedExceptionTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 3: Create EnforcementMode enum**

```java
package io.casehub.qhorus.api.channel;

public enum EnforcementMode {
    ADVISORY,
    BLOCKING,
    QUARANTINE
}
```

- [ ] **Step 4: Create EnforcementBlockedException**

```java
package io.casehub.qhorus.api.message;

import io.casehub.qhorus.api.channel.EnforcementMode;
import java.util.List;

public class EnforcementBlockedException extends IllegalStateException {

    private final EnforcementMode mode;
    private final List<String> violationSources;
    private final List<String> violations;

    public EnforcementBlockedException(EnforcementMode mode,
                                        List<String> violationSources,
                                        List<String> violations) {
        super("Enforcement " + mode.name() + ": " + violations.size()
              + " violation(s) from " + violationSources);
        this.mode = mode;
        this.violationSources = List.copyOf(violationSources);
        this.violations = List.copyOf(violations);
    }

    public EnforcementMode mode() { return mode; }
    public List<String> violationSources() { return violationSources; }
    public List<String> violations() { return violations; }
}
```

- [ ] **Step 5: Create EnforcementBlockedEvent**

```java
package io.casehub.qhorus.api.message;

import io.casehub.qhorus.api.channel.EnforcementMode;
import java.util.List;
import java.util.UUID;

public record EnforcementBlockedEvent(
        UUID channelId,
        String channelName,
        EnforcementMode mode,
        String blockedSender,
        MessageType blockedType,
        List<String> violations,
        List<String> violationSources) {

    public EnforcementBlockedEvent {
        violations = violations != null ? List.copyOf(violations) : List.of();
        violationSources = violationSources != null ? List.copyOf(violationSources) : List.of();
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EnforcementBlockedExceptionTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/channel/EnforcementMode.java api/src/main/java/io/casehub/qhorus/api/message/EnforcementBlockedException.java api/src/main/java/io/casehub/qhorus/api/message/EnforcementBlockedEvent.java runtime/src/test/java/io/casehub/qhorus/message/EnforcementBlockedExceptionTest.java
git commit -m "feat(#400): EnforcementMode enum + EnforcementBlockedException + EnforcementBlockedEvent Refs #400"
```

---

### Task 2: Channel + ChannelEntity + ChannelCreateRequest + V45 migration

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — add `enforcementMode`, `enforcementExclusions` fields
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java` — add fields + builder
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java` — add String fields
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` — add JPA columns + mapping
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java` — map enforcement fields in `toChannelDetail()`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java` — add fields + `from()` mapping
- Create: `runtime/src/main/resources/db/qhorus/migration/V45__enforcement_mode.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/channel/EnforcementModeChannelTest.java`

**Interfaces:**
- Consumes: `EnforcementMode` (from Task 1)
- Produces: `Channel.enforcementMode()`, `Channel.enforcementExclusions()`, `Channel.toBuilder().enforcementMode()`, `Channel.toBuilder().enforcementExclusions()`
- Produces: `ChannelCreateRequest.enforcementMode()`, `ChannelCreateRequest.builder("name").enforcementMode()`
- Produces: `ChannelDetail(... String enforcementMode, String enforcementExclusions, ...)`
- Produces: `ChannelResponse(... String enforcementMode, List<String> enforcementExclusions, ...)`

- [ ] **Step 1: Write test for Channel record enforcement fields**

```java
package io.casehub.qhorus.channel;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.channel.EnforcementMode;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class EnforcementModeChannelTest {

    @Test
    void channelDefaultsToAdvisory() {
        var ch = Channel.builder("test")
                .semantic(ChannelSemantic.APPEND)
                .tenancyId("default")
                .build();
        assertThat(ch.enforcementMode()).isNull();
        assertThat(ch.enforcementExclusions()).isEmpty();
    }

    @Test
    void channelBuilderSetsEnforcementMode() {
        var ch = Channel.builder("test")
                .semantic(ChannelSemantic.APPEND)
                .enforcementMode(EnforcementMode.BLOCKING)
                .enforcementExclusions(List.of("CORRELATION_INTEGRITY"))
                .tenancyId("default")
                .build();
        assertThat(ch.enforcementMode()).isEqualTo(EnforcementMode.BLOCKING);
        assertThat(ch.enforcementExclusions()).containsExactly("CORRELATION_INTEGRITY");
    }

    @Test
    void channelCreateRequestSetsEnforcementMode() {
        var req = ChannelCreateRequest.builder("test")
                .enforcementMode(EnforcementMode.QUARANTINE)
                .enforcementExclusions(List.of("TYPE_POLICY"))
                .build();
        assertThat(req.enforcementMode()).isEqualTo(EnforcementMode.QUARANTINE);
        assertThat(req.enforcementExclusions()).containsExactly("TYPE_POLICY");
    }

    @Test
    void channelFromRequestPropagatesEnforcementFields() {
        var req = ChannelCreateRequest.builder("test")
                .enforcementMode(EnforcementMode.BLOCKING)
                .enforcementExclusions(List.of("REQUEST_RESPONSE"))
                .build();
        var ch = Channel.fromRequest(req, "default");
        assertThat(ch.enforcementMode()).isEqualTo(EnforcementMode.BLOCKING);
        assertThat(ch.enforcementExclusions()).containsExactly("REQUEST_RESPONSE");
    }

    @Test
    void toBuilderPreservesEnforcementFields() {
        var ch = Channel.builder("test")
                .semantic(ChannelSemantic.APPEND)
                .enforcementMode(EnforcementMode.QUARANTINE)
                .enforcementExclusions(List.of("TYPE_POLICY", "CORRELATION_INTEGRITY"))
                .tenancyId("default")
                .build();
        var rebuilt = ch.toBuilder().build();
        assertThat(rebuilt.enforcementMode()).isEqualTo(EnforcementMode.QUARANTINE);
        assertThat(rebuilt.enforcementExclusions()).containsExactly("TYPE_POLICY", "CORRELATION_INTEGRITY");
    }

    @Test
    void backwardCompatConstructorDefaultsEnforcementToNull() {
        // The existing 20-arg constructor (pre-enforcement) should still compile and default
        var ch = Channel.builder("test").semantic(ChannelSemantic.APPEND).tenancyId("t").build();
        assertThat(ch.enforcementMode()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EnforcementModeChannelTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: FAIL — enforcementMode/enforcementExclusions methods not found

- [ ] **Step 3: Add fields to Channel record**

Add `EnforcementMode enforcementMode` and `List<String> enforcementExclusions` to the Channel record's canonical constructor. Add null normalization for `enforcementExclusions` in the compact constructor: `enforcementExclusions = enforcementExclusions != null ? List.copyOf(enforcementExclusions) : List.of()`. Add builder setters. Update `fromRequest()` and `toBuilder()`. Add backward-compatible constructor(s) defaulting both to null.

- [ ] **Step 4: Add fields to ChannelCreateRequest**

Add `EnforcementMode enforcementMode` (nullable) and `List<String> enforcementExclusions` (nullable) to ChannelCreateRequest. Add builder setters. Update backward-compatible constructors.

- [ ] **Step 5: Add JPA columns to ChannelEntity**

Add `enforcementMode` (String, nullable) and `enforcementExclusions` (String/TEXT, nullable) fields. Update `fromDomain()` to map: `entity.enforcementMode = channel.enforcementMode() != null ? channel.enforcementMode().name() : null` and `entity.enforcementExclusions = joinCsv(channel.enforcementExclusions())`. Update `toDomain()` to map: `enforcementMode != null ? EnforcementMode.valueOf(enforcementMode) : null` and `splitCsv(enforcementExclusions)`.

- [ ] **Step 6: Create V45 migration**

```sql
ALTER TABLE channel ADD COLUMN enforcement_mode VARCHAR(20);
ALTER TABLE channel ADD COLUMN enforcement_exclusions TEXT;
```

- [ ] **Step 7: Add fields to ChannelDetail**

Add `String enforcementMode` and `String enforcementExclusions` to the ChannelDetail record. Add backward-compatible constructors defaulting both to null.

- [ ] **Step 8: Update QhorusEntityMapper.toChannelDetail()**

Map from Channel: `ch.enforcementMode() != null && ch.enforcementMode() != EnforcementMode.ADVISORY ? ch.enforcementMode().name() : null` and `joinCsv(ch.enforcementExclusions())`.

- [ ] **Step 9: Add fields to ChannelResponse**

Add `String enforcementMode` and `List<String> enforcementExclusions` to the ChannelResponse record. Update `from()`: `ch.enforcementMode() != null && ch.enforcementMode() != EnforcementMode.ADVISORY ? ch.enforcementMode().name() : null` and `ch.enforcementExclusions()`.

- [ ] **Step 10: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EnforcementModeChannelTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 11: Run full build to catch cross-module breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: BUILD SUCCESS — backward-compatible constructors prevent breakage in consumers

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat(#400): Channel enforcement fields + ChannelEntity + V45 migration Refs #400"
```

---

## Batch 2: Enforcement pipeline

### Task 3: TaggedAdvisory + advisory tagging in dispatch pipeline

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/TaggedAdvisory.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — refactor advisory collection to use TaggedAdvisory
- Test: `runtime/src/test/java/io/casehub/qhorus/message/TaggedAdvisoryTest.java`

**Interfaces:**
- Produces: `TaggedAdvisory(String source, String message)` — package-private record
- Produces: advisory collection in `dispatch()` now builds `List<TaggedAdvisory>`, flattened to `List<String>` for DispatchResult

- [ ] **Step 1: Write test for TaggedAdvisory and advisory tagging**

```java
package io.casehub.qhorus.message;

import io.casehub.qhorus.runtime.message.TaggedAdvisory;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class TaggedAdvisoryTest {

    @Test
    void flattenExtractsMessages() {
        var advisories = List.of(
                new TaggedAdvisory("TYPE_POLICY", "type violation"),
                new TaggedAdvisory("REQUEST_RESPONSE", "[REQUEST_RESPONSE] too many queries"));
        List<String> flat = advisories.stream().map(TaggedAdvisory::message).toList();
        assertThat(flat).containsExactly("type violation", "[REQUEST_RESPONSE] too many queries");
    }

    @Test
    void distinctSources() {
        var advisories = List.of(
                new TaggedAdvisory("TYPE_POLICY", "v1"),
                new TaggedAdvisory("TYPE_POLICY", "v2"),
                new TaggedAdvisory("REQUEST_RESPONSE", "v3"));
        List<String> sources = advisories.stream().map(TaggedAdvisory::source).distinct().toList();
        assertThat(sources).containsExactly("TYPE_POLICY", "REQUEST_RESPONSE");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TaggedAdvisoryTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: FAIL — TaggedAdvisory not found

- [ ] **Step 3: Create TaggedAdvisory record**

```java
package io.casehub.qhorus.runtime.message;

record TaggedAdvisory(String source, String message) {}
```

- [ ] **Step 4: Refactor MessageService.dispatch() advisory collection**

In `dispatch()`, change the three advisory collection sites:

1. **MessageTypePolicy.advisory()** — wrap result: `new TaggedAdvisory("TYPE_POLICY", adv)`
2. **CorrelationIntegrityChecker.check()** — wrap each result: `new TaggedAdvisory("CORRELATION_INTEGRITY", ca)`
3. **ChannelProtocol.evaluate()** — extract source from `[PROTOCOL_NAME]` prefix, or use `protocol.protocolName()`: `new TaggedAdvisory(protocol.protocolName(), v)`

Replace the `List<String> advisories` variable with `List<TaggedAdvisory> taggedAdvisories`. At the DispatchResult construction sites, flatten: `taggedAdvisories.stream().map(TaggedAdvisory::message).toList()`.

- [ ] **Step 5: Run test to verify TaggedAdvisory passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TaggedAdvisoryTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 6: Run existing dispatch tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: All existing tests PASS — the refactoring is behavioral no-op for ADVISORY mode

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/message/TaggedAdvisory.java runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java runtime/src/test/java/io/casehub/qhorus/message/TaggedAdvisoryTest.java
git commit -m "feat(#400): TaggedAdvisory + advisory tagging in dispatch pipeline Refs #400"
```

---

### Task 4: EnforcementExecutor + enforcement gate in MessageService

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/EnforcementExecutor.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — add enforcement gate after advisory collection
- Test: `runtime/src/test/java/io/casehub/qhorus/message/EnforcementExecutorTest.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/message/EnforcementGateTest.java`

**Interfaces:**
- Consumes: `TaggedAdvisory` (from Task 3), `EnforcementMode`, `EnforcementBlockedException`, `EnforcementBlockedEvent` (from Task 1), `Channel.enforcementMode()`, `Channel.enforcementExclusions()` (from Task 2)
- Produces: `EnforcementExecutor.execute(Channel, MessageDispatch, List<TaggedAdvisory>, String tenancyId)` — `@Transactional(REQUIRES_NEW)`

- [ ] **Step 1: Write CDI-free test for EnforcementExecutor**

```java
package io.casehub.qhorus.message;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.channel.EnforcementMode;
import io.casehub.qhorus.api.message.EnforcementBlockedEvent;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.EnforcementExecutor;
import io.casehub.qhorus.runtime.message.TaggedAdvisory;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import java.util.List;
import java.util.UUID;
import static org.mockito.Mockito.*;

class EnforcementExecutorTest {

    EnforcementExecutor executor;
    // Mock dependencies — messageService, channelService, commitmentService, event

    @BeforeEach
    void setUp() {
        // Construct executor with mocked dependencies
        // Set tracingConfig to disabled
    }

    @Test
    void blockingModeDispatchesEventButDoesNotPause() {
        // Verify: messageService.dispatch() called with sender="system:enforcement" + type=EVENT
        // Verify: channelService.pause() NOT called
        // Verify: commitmentService.expireByChannel() NOT called
        // Verify: CDI event fired
    }

    @Test
    void quarantineModeDispatchesEventAndPausesAndExpires() {
        // Verify: messageService.dispatch() called with EVENT
        // Verify: channelService.pause() called
        // Verify: commitmentService.expireByChannel() called
        // Verify: CDI event fired
    }

    @Test
    void eventTelemetryContainsViolationDetails() {
        // Capture the MessageDispatch passed to messageService.dispatch()
        // Verify telemetry JSON contains enforcement_action, violations, blocked_sender, etc.
    }
}
```

- [ ] **Step 2: Write CDI-free test for enforcement gate logic**

```java
package io.casehub.qhorus.message;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.channel.EnforcementMode;
import io.casehub.qhorus.api.message.EnforcementBlockedException;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.TaggedAdvisory;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class EnforcementGateTest {

    @Test
    void advisoryModeNeverThrows() {
        // Channel with ADVISORY mode + violations → no exception
    }

    @Test
    void blockingModeThrowsOnViolation() {
        // Channel with BLOCKING mode + violations → EnforcementBlockedException
    }

    @Test
    void blockingModePassesWhenNoViolations() {
        // Channel with BLOCKING mode + empty violations → no exception
    }

    @Test
    void eventTypeExemptFromEnforcement() {
        // dispatch.type() == EVENT + BLOCKING mode + violations → no exception
    }

    @Test
    void systemSenderExemptFromEnforcement() {
        // sender "system:enforcement" + BLOCKING mode + violations → no exception
    }

    @Test
    void terminalTypesExemptFromEnforcement() {
        // DONE, FAILURE, DECLINE, RESPONSE, HANDOFF each exempt
    }

    @Test
    void exclusionsFilterOutSources() {
        // Channel with exclusions=["TYPE_POLICY"] + violation from TYPE_POLICY → no exception
        // Same channel + violation from REQUEST_RESPONSE → exception
    }

    @Test
    void mixedExcludedAndEnforceable() {
        // Some violations excluded, some not → exception with only enforceable violations
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="EnforcementExecutorTest,EnforcementGateTest" -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 4: Create EnforcementExecutor**

Package-private `@ApplicationScoped` CDI bean in `io.casehub.qhorus.runtime.message`. Injects `MessageService` (via `ConsumerMessaging`), `ChannelService` (via `ChannelManager`), `CommitmentService`, and `Event<EnforcementBlockedEvent>`. Method `execute()` annotated `@Transactional(REQUIRES_NEW)`.

The `execute()` method:
1. Builds telemetry JSON using `ObjectMapper` (inject) with enforcement_action, violations, violation_sources, blocked_sender, blocked_type, enforcement_mode
2. Dispatches EVENT via `messageService.dispatch()` with sender="system:enforcement", actorType=SYSTEM
3. If QUARANTINE: calls `channelService.pause()` and `commitmentService.expireByChannel()`
4. Fires `EnforcementBlockedEvent` async

Package-private constructor for CDI-free unit tests (same pattern as other services).

- [ ] **Step 5: Add enforcement gate to MessageService.dispatch()**

After the protocol evaluation block (after all three advisory sources produce `List<TaggedAdvisory>`), add:

```java
if (ch != null && ch.enforcementMode() != null
        && ch.enforcementMode() != EnforcementMode.ADVISORY
        && dispatch.type() != MessageType.EVENT
        && !dispatch.sender().contains(":")
        && !RESOLUTION_TYPES.contains(dispatch.type())) {
    List<TaggedAdvisory> enforceable = taggedAdvisories.stream()
            .filter(ta -> !ch.enforcementExclusions().contains(ta.source()))
            .toList();
    if (!enforceable.isEmpty()) {
        try {
            enforcementExecutor.execute(ch, dispatch, enforceable, effectiveTenancyId);
        } catch (Exception e) {
            LOG.warnf(e, "Enforcement execution failed for channel '%s'", ch.name());
        }
        throw new EnforcementBlockedException(
                ch.enforcementMode(),
                enforceable.stream().map(TaggedAdvisory::source).distinct().toList(),
                enforceable.stream().map(TaggedAdvisory::message).toList());
    }
}
```

Note: `MessageType.isTerminal()` exists but only covers DONE, FAILURE, HANDOFF — not DECLINE or RESPONSE. Use a static set for the exemption check:
```java
private static final Set<MessageType> RESOLUTION_TYPES = Set.of(
        MessageType.DONE, MessageType.FAILURE, MessageType.DECLINE,
        MessageType.RESPONSE, MessageType.HANDOFF);
```
Then: `RESOLUTION_TYPES.contains(dispatch.type())`.

Inject `EnforcementExecutor enforcementExecutor` into MessageService.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="EnforcementExecutorTest,EnforcementGateTest" -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 7: Run full runtime tests for regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: All existing tests PASS — ADVISORY mode behavior unchanged

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/message/EnforcementExecutor.java runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java runtime/src/test/java/io/casehub/qhorus/message/EnforcementExecutorTest.java runtime/src/test/java/io/casehub/qhorus/message/EnforcementGateTest.java
git commit -m "feat(#400): EnforcementExecutor (REQUIRES_NEW) + enforcement gate in dispatch Refs #400"
```

---

## Batch 3: API surface + CLAUDE.md

### Task 5: ChannelService methods + MCP tools + REST endpoint + CLAUDE.md

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` — add `setEnforcementMode()`, `setEnforcementExclusions()`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add `set_enforcement_mode`, `set_enforcement_exclusions`, `get_channel_enforcement` tools; update `create_channel`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java` — add `PUT /api/channels/{id}/enforcement-mode`
- Modify: `CLAUDE.md` — document new fields, tools, migration
- Test: `runtime/src/test/java/io/casehub/qhorus/message/EnforcementMcpToolTest.java`

**Interfaces:**
- Consumes: `Channel.enforcementMode()`, `Channel.enforcementExclusions()` (from Task 2), `EnforcementMode` (from Task 1)
- Produces: `ChannelService.setEnforcementMode(UUID, EnforcementMode)`, `ChannelService.setEnforcementExclusions(UUID, List<String>)`
- Produces: MCP tools: `set_enforcement_mode(channel, mode)`, `set_enforcement_exclusions(channel, exclusions)`, `get_channel_enforcement(channel)`
- Produces: REST: `PUT /api/channels/{id}/enforcement-mode`

- [ ] **Step 1: Write integration test for MCP enforcement tools**

```java
package io.casehub.qhorus.message;

import io.casehub.qhorus.api.channel.EnforcementMode;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class EnforcementMcpToolTest {

    @Inject
    io.casehub.qhorus.runtime.mcp.QhorusMcpTools tools;

    @Test
    @TestTransaction
    void setEnforcementModeAndRetrieve() {
        // Create channel via tools
        // Call set_enforcement_mode(channel, "BLOCKING")
        // Call get_channel_enforcement(channel)
        // Assert mode = BLOCKING, exclusions = empty
    }

    @Test
    @TestTransaction
    void setEnforcementExclusionsAndRetrieve() {
        // Create channel
        // set_enforcement_exclusions(channel, "TYPE_POLICY,CORRELATION_INTEGRITY")
        // get_channel_enforcement → assert exclusions present
    }

    @Test
    @TestTransaction
    void createChannelWithEnforcementMode() {
        // create_channel with enforcement_mode=QUARANTINE
        // verify channel has QUARANTINE mode
    }

    @Test
    @TestTransaction
    void invalidModeReturnsError() {
        // set_enforcement_mode(channel, "INVALID") → ToolCallException
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EnforcementMcpToolTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: FAIL — methods not found

- [ ] **Step 3: Add ChannelService.setEnforcementMode() and setEnforcementExclusions()**

Follow the existing pattern (e.g., `setProtocols()`):

```java
@Transactional
public Channel setEnforcementMode(UUID channelId, EnforcementMode mode) {
    Channel ch = channelStore.findById(channelId).orElseThrow(() ->
            new IllegalArgumentException("Channel not found: " + channelId));
    Channel updated = ch.toBuilder().enforcementMode(mode).build();
    return channelStore.put(updated);
}

@Transactional
public Channel setEnforcementExclusions(UUID channelId, List<String> exclusions) {
    Channel ch = channelStore.findById(channelId).orElseThrow(() ->
            new IllegalArgumentException("Channel not found: " + channelId));
    Channel updated = ch.toBuilder().enforcementExclusions(exclusions).build();
    return channelStore.put(updated);
}
```

- [ ] **Step 4: Add MCP tools to QhorusMcpTools**

Add three `@Tool` methods:

- `set_enforcement_mode(String channel, String mode)` — validates EnforcementMode.valueOf(), resolves channel, delegates to channelService
- `set_enforcement_exclusions(String channel, String exclusions)` — parses CSV, resolves channel, delegates
- `get_channel_enforcement(String channel)` — returns JSON with mode, exclusions, and available source tags

Update `create_channel` to accept optional `enforcement_mode` and `enforcement_exclusions` parameters. Pass through to ChannelCreateRequest builder.

- [ ] **Step 5: Add REST endpoint to ChannelResource**

```java
record EnforcementModeRequest(String mode, List<String> exclusions) {}

@PUT
@Path("{id}/enforcement-mode")
public Response setEnforcementMode(@PathParam("id") String id, EnforcementModeRequest req) {
    Channel ch = resolve(id);
    if (req.mode() != null) {
        EnforcementMode mode = EnforcementMode.valueOf(req.mode().toUpperCase());
        ch = channelService.setEnforcementMode(ch.id(), mode);
    }
    if (req.exclusions() != null) {
        ch = channelService.setEnforcementExclusions(ch.id(), req.exclusions());
    }
    return Response.ok(toResponse(ch)).build();
}
```

Update `CreateChannelRequest` to accept `enforcementMode` and `enforcementExclusions`.

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=EnforcementMcpToolTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 7: Run full build to verify everything compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Update CLAUDE.md**

Add to the Channel record documentation:
- `EnforcementMode enforcementMode` field description
- `List<String> enforcementExclusions` field description
- V45 migration (next = V46)
- MCP tools: `set_enforcement_mode`, `set_enforcement_exclusions`, `get_channel_enforcement`
- `EnforcementBlockedException extends IllegalStateException` — testing notes
- `EnforcementExecutor` — REQUIRES_NEW bean, CDI-free test pattern
- `EnforcementBlockedEvent` — CDI event for external notification
- Terminal type + system sender + EVENT exemptions
- REST endpoint: `PUT /api/channels/{id}/enforcement-mode`

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#400): enforcement MCP tools + REST endpoint + ChannelService methods + CLAUDE.md Refs #400"
```

---

## References

- `specs/issue-398-roadmap-phase1/2026-08-23-enforcement-modes-design.md` — design spec
- `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — dispatch pipeline
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` — JPA entity
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` — service mutations
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateHelper.java` — REQUIRES_NEW precedent
- `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — MCP tool definitions
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java` — REST API
- `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — channel record
- `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAlertEvent.java` — CDI event pattern
- `docs/adr/0016-hybrid-channel-type-enforcement.md` — ADR for type enforcement
- GitHub #400

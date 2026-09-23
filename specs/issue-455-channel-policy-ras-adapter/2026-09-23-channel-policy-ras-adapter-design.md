# Channel Policy Model + RAS Adapter Design

**Issue:** casehubio/qhorus#455, casehubio/casehub-ras#66
**Date:** 2026-09-23
**Status:** Draft

## Problem

The `ChannelProtocol` SPI returns `List<String>` — flat violation descriptions with no severity, evidence, or action recommendation. This limits protocol evaluation to simple pass/fail validation and prevents composition with RAS situation awareness. Channel operators cannot express temporal patterns ("if a COMMAND hasn't been acknowledged within 5 minutes, escalate") because those require asynchronous event observation that qhorus's synchronous dispatch pipeline cannot provide.

## Solution Overview

Three-layer evolution:

1. **Structured protocol output** — `ProtocolViolation` replaces `List<String>` as the SPI return type, carrying severity, structured evidence, human-readable message, and suggested action.

2. **Channel policy definition model** — YAML format that compiles into both dispatch-time `ChannelProtocol` implementations (synchronous, Layer 1) and RAS `SituationDefinition` registrations (temporal, Layer 2). Hybrid storage: base policies in YAML files at startup, per-channel DB overrides for thresholds.

3. **RAS adapter module** — `ras/qhorus/` module that bridges qhorus message CloudEvents to RAS situation detection. Extracts commitment lifecycle from message types. Selective forwarding: only RAS-active channels, only relevant event types.

---

## Layer 1: Structured Protocol Output

### ProtocolViolation (api/spi/)

New record in `io.casehub.qhorus.api.spi`:

```java
public record ProtocolViolation(
        String protocolName,
        Severity severity,
        String message,
        Map<String, Object> evidence,
        SuggestedAction suggestedAction) {

    public ProtocolViolation {
        Objects.requireNonNull(protocolName);
        Objects.requireNonNull(severity);
        Objects.requireNonNull(message);
        evidence = evidence != null ? Map.copyOf(evidence) : Map.of();
        suggestedAction = suggestedAction != null ? suggestedAction : SuggestedAction.WARN;
    }
}
```

### Severity (api/spi/)

```java
public enum Severity {
    ADVISORY,   // informational — always just logs
    WARNING,    // follows channel enforcement mode
    CRITICAL;   // upgrades enforcement — blocks even in ADVISORY-mode channels

    public boolean isAtLeast(Severity threshold) {
        return this.ordinal() >= threshold.ordinal();
    }
}
```

**Severity × enforcement mode interaction:**

| Channel Mode | CRITICAL | WARNING | ADVISORY |
|---|---|---|---|
| ADVISORY | **block** (upgrade) | log only | log only |
| BLOCKING | block | block | log only |
| QUARANTINE | quarantine | quarantine | log only |

CRITICAL violations override ADVISORY-mode channels — analogous to a circuit breaker. Channel operators control which protocols are active; severity controls the floor.

### SuggestedAction (api/spi/)

```java
public enum SuggestedAction {
    BLOCK,      // reject the message
    WARN,       // allow but surface as advisory
    ESCALATE,   // allow but fire CDI event for external handling
    REROUTE     // suggest redirection (metadata for RAS, not dispatch-time)
}
```

`suggestedAction` is advisory metadata — the enforcement gate uses `severity` for decisions, not `suggestedAction`. The action informs RAS situation responses and is recorded in enforcement event telemetry for operational analysis.

### ChannelProtocol SPI change

```java
public interface ChannelProtocol {
    String protocolName();
    List<ProtocolViolation> evaluate(ProtocolContext context);
}
```

**Breaking change** — return type changes from `List<String>` to `List<ProtocolViolation>`. All four built-in protocols updated. Pre-release: no backward compat needed.

### Built-in protocol updates

Each protocol gains structured evidence:

**REQUEST_RESPONSE:**
```java
new ProtocolViolation("REQUEST_RESPONSE", Severity.WARNING,
    "3 unanswered QUERYs in channel 'ops' — consider waiting for responses",
    Map.of("openQueryCount", 3, "threshold", maxOpenQueries, "channelName", ctx.channelName()),
    SuggestedAction.WARN)
```

**TASK_COMPLETION:**
```java
new ProtocolViolation("TASK_COMPLETION", Severity.WARNING,
    "2 open COMMANDs in channel 'ops' — consider resolving existing tasks",
    Map.of("openCommandCount", 2, "threshold", maxOpenCommands, "senderIsObligor", true),
    SuggestedAction.WARN)
```

**ROUND_ROBIN:**
```java
new ProtocolViolation("ROUND_ROBIN", Severity.ADVISORY,
    "expected 'agent-b' to speak next, got 'agent-a'",
    Map.of("expectedSender", "agent-b", "actualSender", "agent-a"),
    SuggestedAction.WARN)
```

**CONTRIBUTION_REQUIRED:**
```java
new ProtocolViolation("CONTRIBUTION_REQUIRED", Severity.WARNING,
    "agent-a has sent 3 consecutive messages without contributions from: agent-b",
    Map.of("consecutiveCount", 3, "sender", "agent-a", "missingSenders", List.of("agent-b")),
    SuggestedAction.WARN)
```

### TaggedAdvisory evolution (runtime-core, internal)

```java
record TaggedAdvisory(
        String source,
        String message,
        Severity severity,
        Map<String, Object> evidence,
        SuggestedAction suggestedAction) {

    // backward-compat factory for non-protocol sources
    static TaggedAdvisory of(String source, String message) {
        return new TaggedAdvisory(source, message, Severity.WARNING, Map.of(), SuggestedAction.WARN);
    }

    // factory from ProtocolViolation
    static TaggedAdvisory fromViolation(ProtocolViolation v) {
        return new TaggedAdvisory(v.protocolName(), v.message(), v.severity(), v.evidence(), v.suggestedAction());
    }
}
```

Non-protocol advisory sources get default severity:
- `TYPE_POLICY` — `Severity.CRITICAL` (these are already hard-enforced for COMMAND/QUERY)
- `CORRELATION_INTEGRITY` — `Severity.ADVISORY` (informational checks)

### Enforcement gate evolution

`enforceIfRequired()` gains severity-aware logic:

```java
static void enforceIfRequired(Channel ch, List<TaggedAdvisory> taggedAdvisories, ...) {
    // ADVISORY mode: only block if any CRITICAL severity present
    // BLOCKING mode: block all non-ADVISORY severities (existing behavior for WARNING+CRITICAL)
    // QUARANTINE mode: quarantine for WARNING+CRITICAL (existing behavior)

    if (ch.enforcementMode() == null || ch.enforcementMode() == EnforcementMode.ADVISORY) {
        // In ADVISORY mode, only CRITICAL can upgrade to blocking
        List<TaggedAdvisory> critical = taggedAdvisories.stream()
                .filter(ta -> ta.severity() == Severity.CRITICAL)
                .filter(ta -> !ch.enforcementExclusions().contains(ta.source()))
                .toList();
        if (critical.isEmpty()) return;
        // CRITICAL override path — treat as BLOCKING
        enforceable = critical;
    } else {
        // BLOCKING/QUARANTINE mode — filter out ADVISORY severity
        enforceable = taggedAdvisories.stream()
                .filter(ta -> ta.severity() != Severity.ADVISORY)
                .filter(ta -> !ch.enforcementExclusions().contains(ta.source()))
                .toList();
        if (enforceable.isEmpty()) return;
    }
    // ... existing enforcement execution
}
```

### DispatchResult evolution

`DispatchResult.advisories` evolves from `List<String>` to structured output:

```java
public record ProtocolAdvisory(
        String source,
        Severity severity,
        String message,
        Map<String, Object> evidence,
        SuggestedAction suggestedAction) {

    public ProtocolAdvisory {
        evidence = evidence != null ? Map.copyOf(evidence) : Map.of();
    }
}
```

`DispatchResult` gains `List<ProtocolAdvisory> advisories` replacing `List<String>`. Evidence included — callers (dashboards, HIL review) need structured data for display. **Breaking change** — pre-release, acceptable.

### EnforcementBlockedException evolution

Gains severity and structured violations:

```java
public class EnforcementBlockedException extends IllegalStateException {
    private final EnforcementMode mode;
    private final List<String> violationSources;
    private final List<ProtocolAdvisory> violations; // was List<String>
    private final boolean severityUpgrade; // true when CRITICAL overrode ADVISORY mode
}
```

### CDI event for RAS observation

New event fired after protocol evaluation, before enforcement:

```java
// in api/spi/
public record ProtocolEvaluationEvent(
        UUID channelId,
        String channelName,
        String tenancyId,
        List<ProtocolViolation> violations) {}
```

Fired from `MessageService.dispatch()` after protocol evaluation completes (even if violations are empty — RAS may care about "no violations detected" as a signal). The RAS adapter observes this for situation correlation.

---

## Layer 2: Channel Policy Definition Model

### YAML format

A channel policy defines a channel's complete behavioral expectations — both synchronous dispatch-time rules and asynchronous temporal patterns.

```yaml
channel-policy: strict-operations
version: 1

dispatch_rules:
  - name: command-requires-target
    when:
      type: COMMAND
      target: null
    severity: critical
    action: block
    message: "COMMAND dispatched without target"
    evidence:
      sender: "${sender}"

  - name: query-pressure
    when:
      type: QUERY
    condition: "open_queries >= ${max_open_queries}"
    severity: warning
    action: warn
    message: "${open_queries} unanswered QUERYs — consider waiting"
    evidence:
      openQueryCount: "${open_queries}"
      threshold: "${max_open_queries}"
    defaults:
      max_open_queries: 3

situations:
  # Reference a pre-built RAS situation with parameter overrides
  - ref: ack-timeout
    params:
      window: 5m
      target: "role:team-lead"

  # Inline situation definition
  - name: obligation-pressure
    trigger:
      event_type: "io.casehub.qhorus.message.command"
    detection:
      ganglion: expression-rules
      rules:
        - when: "open_commitments_for_obligor >= 5"
          signal: DETECTED
          confidence: 0.8
    action:
      type: advisory
      evidence: "Agent ${obligor} has ${count} open commitments"
```

### Policy compilation

At startup, the `ChannelPolicyCompiler` processes YAML files and produces:

1. **ChannelProtocol beans** — each `dispatch_rules` block compiles into a `ChannelProtocol` implementation registered in `ProtocolRegistry`. The protocol name is `policy:<policy-name>` (e.g., `policy:strict-operations`).

2. **SituationDefinition registrations** — each `situations` entry (both `ref:` and inline) compiles into a `SituationRegistration` passed to the RAS adapter's `SituationDefinitionProvider`.

### Policy → Channel binding

Channels reference policies via the existing `Channel.protocols` field:

```
protocols: ["policy:strict-operations", "REQUEST_RESPONSE"]
```

Both YAML-compiled policies and built-in Java protocols coexist in the `ProtocolRegistry`. A channel can mix them.

### Variable resolution

Variables in dispatch rules (`${var}`) resolve from three sources, in precedence order:

1. **Per-channel DB overrides** (highest) — `Channel.policyOverrides` map
2. **YAML defaults** — `defaults:` block in the dispatch rule
3. **Built-in context** — runtime values injected by the protocol engine: `${sender}`, `${channel_name}`, `${open_queries}`, `${open_commands}`, `${open_commitments_for_obligor}`

Context variables are computed from `ProtocolContext` at evaluation time. YAML defaults are set at compilation time. DB overrides are checked at evaluation time — the compiled protocol reads the channel's override map for parameter values, falling back to the YAML default.

### DB overrides

Per-channel threshold overrides via a new `Channel.policyOverrides` field (nullable `Map<String, String>`, JSON in DB):

```json
{"max_open_queries": "5", "ack-timeout.window": "10m"}
```

**Key format:** `<param>` for dispatch-rule defaults within the active policy, `<situation-ref>.<param>` for situation parameter overrides. Keys are scoped to the policies active on that channel — an override for a policy not in the channel's `protocols` list has no effect.

**Merge semantics:** Individual key override, not whole-map replacement. Setting a key overrides that specific default; unset keys retain the YAML default. `set_policy_overrides` merges into the existing map. Null value on a key removes the override (reverts to YAML default).

### Storage: Flyway migration

New column on `channel` table:

```sql
-- V55
ALTER TABLE channel ADD COLUMN policy_overrides TEXT;
```

Nullable TEXT column, JSON-encoded `Map<String, String>`. Null = no overrides.

### MCP tools

- `set_policy_overrides(channel, overrides)` — sets per-channel overrides
- `get_policy_overrides(channel)` — returns current overrides
- `list_policies` — lists all registered policies (YAML-compiled + built-in)

### REST API

- `PUT /api/channels/{id}/policy-overrides` with `PolicyOverridesRequest(Map<String, String>)`

---

## Layer 3: RAS Adapter Module

### Module placement

New module in the RAS repo: `ras/qhorus/`

```
ras/
  └── qhorus/
      ├── pom.xml                    (depends on casehub-qhorus-api, casehub-ras-api)
      ├── src/main/java/io/casehub/ras/qhorus/
      │   ├── QhorusEventBridge.java
      │   ├── QhorusChannelFilter.java
      │   ├── QhorusSituationProvider.java
      │   └── CommitmentStateMapper.java
      └── src/test/
```

**Dependency direction:** `ras/qhorus/` → `casehub-qhorus-api` + `casehub-ras-api`. Qhorus has no knowledge of RAS.

### QhorusEventBridge

Observes qhorus CloudEvents and routes to RAS:

```java
@ApplicationScoped
public class QhorusEventBridge {

    @Inject QhorusChannelFilter channelFilter;
    @Inject Event<CloudEvent> rasEventBus;

    void onQhorusMessage(@ObservesAsync CloudEvent event) {
        if (!event.getType().startsWith("io.casehub.qhorus.message.")) return;
        UUID channelId = extractChannelId(event);
        if (!channelFilter.isRasActive(channelId)) return;
        if (!channelFilter.isRelevantType(channelId, event.getType())) return;

        // Re-emit as a RAS-consumable CloudEvent
        // (may enrich with commitment state, derived fields)
        rasEventBus.fireAsync(event);
    }
}
```

### QhorusChannelFilter

Maintains the set of RAS-active channels and their relevant event types:

```java
@ApplicationScoped
public class QhorusChannelFilter {

    // channelId → set of CloudEvent types this channel's situations listen for
    private final ConcurrentHashMap<UUID, Set<String>> activeChannels = new ConcurrentHashMap<>();

    public boolean isRasActive(UUID channelId) {
        return activeChannels.containsKey(channelId);
    }

    public boolean isRelevantType(UUID channelId, String eventType) {
        Set<String> types = activeChannels.get(channelId);
        return types != null && types.contains(eventType);
    }

    public void register(UUID channelId, Set<String> eventTypes) {
        activeChannels.put(channelId, Set.copyOf(eventTypes));
    }

    public void deregister(UUID channelId) {
        activeChannels.remove(channelId);
    }
}
```

### CommitmentStateMapper

Pure function: maps message type to commitment state transition.

```java
public class CommitmentStateMapper {

    public enum CommitmentTransition {
        OPENED, ACKNOWLEDGED, FULFILLED, DECLINED, FAILED, DELEGATED, EXPIRED
    }

    public static Optional<CommitmentTransition> fromMessageType(MessageType type) {
        return switch (type) {
            case COMMAND, QUERY, PROPOSE -> Optional.of(CommitmentTransition.OPENED);
            case STATUS -> Optional.of(CommitmentTransition.ACKNOWLEDGED);
            case DONE -> Optional.of(CommitmentTransition.FULFILLED);
            case RESPONSE -> Optional.of(CommitmentTransition.FULFILLED); // for non-PROPOSE
            case DECLINE -> Optional.of(CommitmentTransition.DECLINED);
            case FAILURE -> Optional.of(CommitmentTransition.FAILED);
            case HANDOFF -> Optional.of(CommitmentTransition.DELEGATED);
            case EVENT -> Optional.empty();
        };
    }
}
```

### QhorusSituationProvider

Implements `SituationDefinitionProvider` — provides pre-built qhorus-aware situation templates:

```java
@ApplicationScoped
public class QhorusSituationProvider implements SituationDefinitionProvider {

    @Override
    public List<SituationRegistration> registrations() {
        return List.of(
            ackTimeout(),
            obligationPressure(),
            declinePattern(),
            channelSilence()
        );
    }

    @Override
    public List<GanglionDescriptor> ganglionDescriptors() {
        return List.of(
            ackTimeoutGanglion(),
            obligationPressureGanglion()
        );
    }
}
```

Pre-built situation templates (configurable via policy parameters):

| Template | Detects | Ganglion type |
|---|---|---|
| `ack-timeout` | COMMAND pending > window with no STATUS/DONE | ExpressionRules |
| `obligation-pressure` | Agent has N+ open commitments, new COMMAND arriving | ExpressionRules |
| `decline-pattern` | N consecutive DECLINEs from same obligor | Threshold (ChainMode) |
| `channel-silence` | Activity drops to zero after FAILURE | Rate (ChainMode) |

### ProtocolEvaluationEvent observation

The adapter also observes `ProtocolEvaluationEvent` from qhorus for protocol-level situation correlation:

```java
void onProtocolEvaluation(@ObservesAsync ProtocolEvaluationEvent event) {
    if (!channelFilter.isRasActive(event.channelId())) return;
    // Convert violations to CloudEvent and emit for RAS
    CloudEvent ce = toCloudEvent(event);
    rasEventBus.fireAsync(ce);
}
```

This enables RAS situations that correlate protocol violations with temporal patterns (e.g., "3 TASK_COMPLETION violations in 10 minutes → escalate").

---

## Cross-Cutting Concerns

### Testing strategy

**Layer 1 (structured output):**
- CDI-free unit tests for all 4 protocol implementations with structured assertions on evidence maps
- CDI-free unit tests for `enforceIfRequired()` with severity × mode matrix
- `@QuarkusTest` integration tests for full dispatch pipeline with ProtocolViolation flow
- `TaggedAdvisory` mapping tests

**Layer 2 (policy model):**
- YAML parsing unit tests for policy format
- Compilation unit tests (YAML → ChannelProtocol, YAML → SituationDefinition)
- Override application tests (base + channel overrides → effective config)

**Layer 3 (RAS adapter):**
- CDI-free unit tests for CommitmentStateMapper (pure function)
- CDI-free unit tests for QhorusChannelFilter
- `@QuarkusTest` integration tests for end-to-end bridge (mock RAS event bus)

### Migration path

All changes are breaking (pre-release). No backward compat shims needed.

1. `ChannelProtocol.evaluate()` return type: `List<String>` → `List<ProtocolViolation>`
2. `DispatchResult.advisories`: `List<String>` → `List<ProtocolAdvisory>`
3. `EnforcementBlockedException.violations`: `List<String>` → `List<ProtocolAdvisory>`
4. All callers of the above updated in the same commit

### Flyway migrations

- **V55:** `channel.policy_overrides` TEXT nullable

### Module dependencies

```
casehub-qhorus-api
  └── ProtocolViolation, Severity, SuggestedAction, ProtocolEvaluationEvent

casehub-qhorus (runtime-core)
  └── TaggedAdvisory (evolved), enforcement gate (severity-aware)

casehub-qhorus (runtime)
  └── ChannelPolicyCompiler, YAML loading

casehub-ras-api
  └── SituationDefinition, GanglionDescriptor (unchanged)

casehub-ras/qhorus (new adapter module)
  └── depends on: casehub-qhorus-api, casehub-ras-api
  └── provides: QhorusEventBridge, QhorusSituationProvider, QhorusChannelFilter
```

---

## Implementation Sequence

Recommended implementation order within a single branch:

1. **ProtocolViolation, Severity, SuggestedAction** in `api/spi/` — the foundation types
2. **ChannelProtocol SPI change** + update all 4 built-in protocols
3. **TaggedAdvisory evolution** + enforcement gate severity logic
4. **DispatchResult/EnforcementBlockedException** evolution
5. **ProtocolEvaluationEvent** CDI event
6. **Update all tests** for the above
7. **Channel policy YAML format** — parser + compiler
8. **V55 migration** + policy overrides on Channel
9. **RAS adapter module** (in casehub-ras repo, separate branch)

Steps 1-6 are the core SPI evolution (Layer 1). Steps 7-8 are the policy model (Layer 2). Step 9 is the RAS bridge (Layer 3).

---

## References

- `api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java` — current SPI
- `api/src/main/java/io/casehub/qhorus/api/spi/ProtocolContext.java` — evaluation context
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java:290-311` — protocol evaluation in dispatch pipeline
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/TaggedAdvisory.java` — current internal type
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/EnforcementExecutor.java` — enforcement execution
- `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java` — current advisory output
- `ras/api/src/main/java/io/casehub/ras/api/SituationDefinition.java` — RAS situation model
- `ras/api/src/main/java/io/casehub/ras/api/DetectionResult.java` — RAS detection output (Map evidence pattern)
- `ras/api/src/main/java/io/casehub/ras/api/GanglionDescriptor.java` — RAS ganglion descriptors
- `docs/platform/boundary-rules.md` — no violations
- `docs/platform/capability-ownership.md` — qhorus owns channel protocols, RAS owns situation awareness
- casehubio/qhorus#455 — issue
- casehubio/casehub-ras#66 — companion RAS issue

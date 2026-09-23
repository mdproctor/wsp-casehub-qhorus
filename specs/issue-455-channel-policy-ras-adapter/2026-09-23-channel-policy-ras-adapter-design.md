# Channel Policy Model + RAS Adapter Design

**Issue:** casehubio/qhorus#455, casehubio/casehub-ras#66
**Date:** 2026-09-23
**Status:** Draft

## Problem

The `ChannelProtocol` SPI returns `List<String>` — flat violation descriptions with no severity, evidence, or action recommendation. This limits protocol evaluation to simple pass/fail validation and prevents composition with RAS situation awareness. Channel operators cannot express temporal patterns ("if a COMMAND hasn't been acknowledged within 5 minutes, escalate") because those require asynchronous event observation that qhorus's synchronous dispatch pipeline cannot provide.

## Solution Overview

Three-layer evolution:

1. **Structured protocol output** — `DispatchAdvisory` replaces `List<String>` as the SPI return type, carrying severity, structured evidence, human-readable message, and suggested action. A single type used across the entire pipeline: SPI return, internal carrier, and API output.

2. **Channel policy definition model** — YAML format that compiles into both dispatch-time `ChannelProtocol` implementations (synchronous, Layer 1) and RAS `SituationDefinition` registrations (temporal, Layer 2). Hybrid storage: base policies in YAML files at startup, per-channel DB overrides for thresholds.

3. **RAS adapter module** — `ras/qhorus/` module that bridges qhorus CDI commitment lifecycle events to RAS situation detection. Observes `CommitmentStateChangedEvent` (all commitment transitions) and `ChannelActivityEvent` (channel activity). Selective forwarding: only RAS-active channels.

---

## Layer 1: Structured Protocol Output

### DispatchAdvisory (api/spi/)

New record in `io.casehub.qhorus.api.spi` — the single structured type for all advisory/violation output in the dispatch pipeline:

```java
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
    LOG,         // record for telemetry only — the common default
    ESCALATE,    // fire CDI event for external handling (human review, alerts)
    INVESTIGATE, // flag for deeper analysis (dashboards, RAS situation triggers)
    REROUTE      // suggest message redirection (metadata for RAS situation responses)
}
```

`suggestedAction` is downstream metadata for RAS situation responses, dashboards, and telemetry — the enforcement gate uses `severity` for decisions, not `suggestedAction`. The vocabulary reflects operational responses, not enforcement actions. Enforcement is controlled exclusively by `Severity × EnforcementMode`.

### ChannelProtocol SPI change

```java
public interface ChannelProtocol {
    String protocolName();
    List<DispatchAdvisory> evaluate(ProtocolContext context);
}
```

**Breaking change** — return type changes from `List<String>` to `List<DispatchAdvisory>`. All four built-in protocols updated. Pre-release: no backward compat needed.

### Built-in protocol updates

Each protocol gains structured evidence:

**REQUEST_RESPONSE:**
```java
new DispatchAdvisory("REQUEST_RESPONSE", Severity.WARNING,
    "3 unanswered QUERYs in channel 'ops' — consider waiting for responses",
    Map.of("openQueryCount", 3, "threshold", maxOpenQueries, "channelName", ctx.channelName()),
    SuggestedAction.LOG)
```

**TASK_COMPLETION:**
```java
new DispatchAdvisory("TASK_COMPLETION", Severity.WARNING,
    "2 open COMMANDs in channel 'ops' — consider resolving existing tasks",
    Map.of("openCommandCount", 2, "threshold", maxOpenCommands, "senderIsObligor", true),
    SuggestedAction.LOG)
```

**ROUND_ROBIN:**
```java
new DispatchAdvisory("ROUND_ROBIN", Severity.ADVISORY,
    "expected 'agent-b' to speak next, got 'agent-a'",
    Map.of("expectedSender", "agent-b", "actualSender", "agent-a"),
    SuggestedAction.LOG)
```

**CONTRIBUTION_REQUIRED:**
```java
new DispatchAdvisory("CONTRIBUTION_REQUIRED", Severity.WARNING,
    "agent-a has sent 3 consecutive messages without contributions from: agent-b",
    Map.of("consecutiveCount", 3, "sender", "agent-a", "missingSenders", List.of("agent-b")),
    SuggestedAction.LOG)
```

### TaggedAdvisory removal (runtime-core)

`TaggedAdvisory` is deleted. `DispatchAdvisory` replaces it as the single advisory type throughout the pipeline:

- Protocol evaluation returns `List<DispatchAdvisory>` directly — no mapping needed
- Non-protocol sources construct `DispatchAdvisory` with appropriate defaults:

```java
// TYPE_POLICY — hard-enforced for COMMAND/QUERY
new DispatchAdvisory("TYPE_POLICY", Severity.CRITICAL, message, Map.of(), SuggestedAction.LOG)

// CORRELATION_INTEGRITY — informational
new DispatchAdvisory("CORRELATION_INTEGRITY", Severity.ADVISORY, message, Map.of(), SuggestedAction.LOG)
```

This eliminates the `TaggedAdvisory.fromViolation()` mapping, the `ProtocolAdvisory.from(TaggedAdvisory)` conversion, and three types' worth of lockstep evolution.

### Enforcement gate evolution

`enforceIfRequired()` gains severity-aware logic. The existing structural safety guards are preserved — these prevent enforcement from blocking messages that must always flow:

```java
static void enforceIfRequired(Channel ch, List<DispatchAdvisory> advisories,
                              MessageType type, String sender, ...) {
    // --- Structural guards (unchanged from current implementation) ---
    if (type == MessageType.EVENT) { return; }
    if (sender.contains(":")) { return; }
    if (RESOLUTION_TYPES.contains(type)) { return; }
    if (advisories.isEmpty()) { return; }

    // --- Severity-aware enforcement (new) ---
    List<DispatchAdvisory> enforceable;

    if (ch.enforcementMode() == null || ch.enforcementMode() == EnforcementMode.ADVISORY) {
        // In ADVISORY mode, only CRITICAL can upgrade to blocking
        enforceable = advisories.stream()
                .filter(a -> a.severity() == Severity.CRITICAL)
                .filter(a -> !ch.enforcementExclusions().contains(a.source()))
                .toList();
        if (enforceable.isEmpty()) return;
        // CRITICAL override path — severityUpgrade = true
    } else {
        // BLOCKING/QUARANTINE mode — filter out ADVISORY severity
        enforceable = advisories.stream()
                .filter(a -> a.severity() != Severity.ADVISORY)
                .filter(a -> !ch.enforcementExclusions().contains(a.source()))
                .toList();
        if (enforceable.isEmpty()) return;
    }
    // ... existing enforcement execution with DispatchAdvisory directly
}
```

### DispatchResult evolution

`DispatchResult.advisories` evolves from `List<String>` to `List<DispatchAdvisory>`. The same `DispatchAdvisory` type used throughout the pipeline — no separate API-facing type, no conversion layer. **Breaking change** — pre-release, acceptable.

```java
public record DispatchResult(
        ...
        @JsonInclude(JsonInclude.Include.NON_EMPTY) List<DispatchAdvisory> advisories
) {}
```

Evidence included — callers (dashboards, HIL review) need structured data for display.

### EnforcementBlockedException evolution

Gains severity and structured violations:

```java
public class EnforcementBlockedException extends IllegalStateException {
    private final EnforcementMode mode;             // channel's configured mode
    private final List<String> violationSources;
    private final List<DispatchAdvisory> violations; // was List<String>
    private final boolean severityUpgrade;           // true when CRITICAL overrode ADVISORY mode

    /** The enforcement behavior actually applied — BLOCKING for severity upgrades. */
    public EnforcementMode effectiveMode() {
        return severityUpgrade ? EnforcementMode.BLOCKING : mode;
    }
}
```

`mode()` returns the channel's configured enforcement mode (unchanged contract). `effectiveMode()` returns the enforcement behavior that was actually applied — for CRITICAL-in-ADVISORY, this is `BLOCKING` even though `mode()` returns `ADVISORY`. Callers that need the actual enforcement behavior (REST error mapping, dashboards, telemetry) should use `effectiveMode()`. `severityUpgrade` remains available for callers that need to distinguish "configured BLOCKING" from "CRITICAL override."

### EnforcementBlockedEvent evolution

Evolves in parallel with the exception — structured violations replace flat strings, and `severityUpgrade` flag propagates to CDI observers:

```java
public record EnforcementBlockedEvent(
        UUID channelId,
        String channelName,
        EnforcementMode mode,
        String blockedSender,
        MessageType blockedType,
        List<DispatchAdvisory> violations,   // was List<String>
        List<String> violationSources,
        boolean severityUpgrade) {           // new — CRITICAL overrode ADVISORY mode
}
```

`EnforcementExecutor.execute()` updated to construct the event directly from `DispatchAdvisory` data — no mapping needed. Downstream CDI observers gain access to severity, evidence, and suggested action.

### CDI event for RAS observation

New event fired after protocol evaluation and enforcement:

```java
// in api/spi/
public record ProtocolEvaluationEvent(
        UUID channelId,
        String channelName,
        String tenancyId,
        List<DispatchAdvisory> violations,
        EnforcementOutcome enforcementOutcome) {

    public enum EnforcementOutcome {
        ALLOWED,    // message dispatched successfully
        BLOCKED,    // enforcement blocked the message
        QUARANTINED // enforcement quarantined the channel
    }
}
```

Fired from `MessageService.dispatch()` after the enforcement gate, **only when violations are non-empty**:

- **Success path** (after enforcement passes or in ADVISORY mode): fired with `EnforcementOutcome.ALLOWED`
- **Enforcement catch block** (before rethrowing `EnforcementBlockedException`): fired with `BLOCKED` or `QUARANTINED` depending on effective mode

This ensures RAS situations can distinguish "violations observed, message dispatched" from "violations observed, message blocked." Situations counting violation patterns (e.g., "3 TASK_COMPLETION violations in 10 minutes → escalate") should filter by `enforcementOutcome` to avoid inflating counts with properly-handled blocks.

Firing on every dispatch (empty violations) creates continuous event volume where >99% would carry empty lists — no concrete situation template needs the absence-of-violations signal. Absence detection (e.g. channel health scoring) should observe `ChannelActivityEvent` instead, which already fires on every dispatch.

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
    message: "COMMAND dispatched without target"
    evidence:
      sender: "${sender}"

  - name: query-pressure
    when:
      type: QUERY
    condition: "open_queries >= ${max_open_queries}"
    severity: warning
    suggested_action: escalate    # optional — defaults to log
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

The YAML file contains two independent sections compiled by different modules — each module reads only the sections it owns:

**In qhorus runtime — `ChannelPolicyCompiler`:**

At startup, the `ChannelPolicyCompiler` processes the `dispatch_rules:` section of each YAML policy file and produces `ChannelProtocol` beans registered in `ProtocolRegistry`. The protocol name is `policy:<policy-name>` (e.g., `policy:strict-operations`). The compiler ignores the `situations:` section — it has no knowledge of RAS types.

**In `ras/qhorus/` adapter — `SituationPolicyCompiler`:**

At startup, the `SituationPolicyCompiler` processes the `situations:` section of the same YAML policy files and compiles each entry (both `ref:` and inline) into `SituationRegistration` + `GanglionDescriptor` objects registered via `SituationRegistrar`. This compiler lives in the RAS adapter module, which depends on both `casehub-qhorus-api` and `casehub-ras-api`.

This split keeps the dependency direction clean: qhorus produces dispatch-time protocols, the RAS adapter produces situation definitions. Neither module depends on the other. Both read the same YAML files — the single policy file is the shared contract, not a shared compilation pipeline.

### Policy → Channel binding

Channels reference policies via the existing `Channel.protocols` field:

```
protocols: ["policy:strict-operations", "REQUEST_RESPONSE"]
```

Both YAML-compiled policies and built-in Java protocols coexist in the `ProtocolRegistry`. A channel can mix them.

### Condition evaluation

Dispatch rules use two distinct condition mechanisms:

1. **`when:` — structural match.** Field-level equality checks against the dispatch context. `type: COMMAND` matches `dispatch.type() == MessageType.COMMAND`. `target: null` matches `dispatch.target() == null`. These are compiled to direct field-access predicates at startup — no expression engine needed.

2. **`condition:` — MVEL expression.** Free-form boolean expressions evaluated against the dispatch context after `${var}` substitution. Uses the platform's `MvelExpressionEvaluator` (`io.casehub.platform.api.expression.MvelExpressionEvaluator`), consistent with RAS `expression-rules` ganglions. Example: `condition: "open_queries >= ${max_open_queries}"` → after substitution: `"open_queries >= 3"` → evaluated by MVEL against a `Map<String, Object>` built from `ProtocolContext`.

The `ChannelPolicyCompiler` pre-compiles `condition:` expressions at startup using `MvelExpressionEvaluator`. At evaluation time, the compiled expression receives the `ProtocolContext` data as a Map.

**Error handling:**

- **Unknown variable reference.** `${unknown_var}` in YAML → compilation-time error. The compiler validates all variable references against the union of YAML `defaults:` keys and known built-in context variables (`sender`, `channel_name`, `open_queries`, `open_commands`, `open_commitments_for_obligor`). Unknown variables fail startup with a clear error identifying the policy, rule, and variable name. This is a static check — no runtime surprises.

- **Expression compilation failure.** Malformed MVEL in `condition:` → compilation-time error at startup. The `MvelExpressionEvaluator` validates syntax at compile time.

- **Evaluation-time type mismatch.** If a DB override sets `max_open_queries` to a non-numeric string (e.g., `"unlimited"`), the substituted expression `open_queries >= unlimited` fails at evaluation time. Behavior: treat the evaluation failure as a violation (fail-closed) and log a warning with the policy name, rule name, raw expression, and exception message. Fail-closed is safer — a misconfigured threshold should surface as a visible problem, not silently disable the rule.

- **Evaluation-time exception.** Any uncaught exception from MVEL evaluation is caught at the `ChannelProtocol.evaluate()` boundary. The exception is logged, and a `DispatchAdvisory` with `Severity.WARNING` is emitted: `"Policy rule '<rule-name>' evaluation failed: <exception message>"`. The violation propagates through the normal advisory pipeline — it does not crash the dispatch.

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

**Lifecycle:** `policyOverrides` is post-creation only — not part of `ChannelCreateRequest`. Rationale: channels are created with policy bindings (`protocols` list), then thresholds are tuned via overrides. This separates channel creation from policy tuning.

**Propagation chain:** Adding `policyOverrides` to `Channel` requires mechanical updates across the stack: the `Channel` record (canonical constructor + `Builder.policyOverrides()` method + `toBuilder()`), `ChannelEntity` JPA mapping (`@Column` + JSON converter), and `ChannelStore` SPI. These are implementation details that follow the established pattern for adding nullable fields — the same pattern used for `routingTrustThreshold`, `redistributionCapacityThreshold`, etc.

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
      │   └── SituationPolicyCompiler.java
      └── src/test/
```

**Dependency direction:** `ras/qhorus/` → `casehub-qhorus-api` + `casehub-ras-api`. Qhorus has no knowledge of RAS.

### QhorusEventBridge

Observes qhorus CDI commitment lifecycle events and converts to RAS CloudEvents. Aligned with casehub-ras#66 — the bridge consumes commitment-level domain events, not raw message CloudEvents.

```java
@ApplicationScoped
public class QhorusEventBridge {

    @Inject QhorusChannelFilter channelFilter;
    @Inject Event<CloudEvent> rasEventBus;

    void onCommitmentStateChanged(@ObservesAsync CommitmentStateChangedEvent event) {
        if (!channelFilter.isRasActive(event.channelId())) return;
        rasEventBus.fireAsync(toCloudEvent(event, "io.casehub.qhorus.commitment.state-changed"));
    }

    void onChannelActivity(@ObservesAsync ChannelActivityEvent event) {
        if (!channelFilter.isRasActive(event.channelId())) return;
        rasEventBus.fireAsync(toCloudEvent(event, "io.casehub.qhorus.channel.activity"));
    }
}
```

**Input events:** The bridge observes exactly two event types:
- `CommitmentStateChangedEvent` — fires for ALL commitment state transitions (OPEN, ACKNOWLEDGED, FULFILLED, DECLINED, FAILED, DELEGATED, EXPIRED). A single CloudEvent type (`io.casehub.qhorus.commitment.state-changed`) carrying the full `Commitment` object with `previousState` gives RAS everything it needs to detect any commitment pattern.
- `ChannelActivityEvent` — channel-level activity signal for rate/silence detection.

The bridge does NOT observe `CommitmentDeclinedEvent` or `CommitmentExpiredEvent` separately. Those dedicated events continue to fire for their existing consumers (e.g. `CommitmentEventNotifier`) but would produce duplicate CloudEvents if the bridge also observed them. `CommitmentStateChangedEvent` already covers DECLINED and EXPIRED transitions.

**Prerequisite — wire up `CommitmentStateChangedEvent`:** This event is defined in `api/gateway/` but **not fired** by `CommitmentService` in production code (confirmed via `ide_find_references` — zero construction sites in runtime/runtime-core). `CommitmentService` must be updated to fire `CommitmentStateChangedEvent` on ALL state transitions:
- `open()` → state=OPEN, previousState=null
- `acknowledge()` → state=ACKNOWLEDGED, previousState=OPEN
- `fulfill()` → state=FULFILLED, previousState=(OPEN or ACKNOWLEDGED)
- `decline()` → state=DECLINED, previousState=(OPEN or ACKNOWLEDGED) — fires alongside existing `CommitmentDeclinedEvent`
- `fail()` → state=FAILED, previousState=(OPEN or ACKNOWLEDGED)
- `delegate()` → state=DELEGATED, previousState=(OPEN or ACKNOWLEDGED)
- `expireOverdue()` / `expireByChannel()` → state=EXPIRED — fires alongside existing `CommitmentExpiredEvent`

`CommitmentDeclinedEvent` and `CommitmentExpiredEvent` are preserved as-is (existing consumers depend on them). `CommitmentStateChangedEvent` is additive — it fires in addition to the dedicated events, not as a replacement. This is step 5 in the implementation sequence.

**Prerequisite — wire up `ChannelActivityEvent` as CDI event:** `ChannelActivityEvent` is currently a nested record inside `ChannelActivityBroadcaster` (`@FunctionalInterface`), broadcast via direct method call: `broadcaster.broadcast(new ChannelActivityEvent(...))` in `MessageService.dispatch()`'s `afterCompletion(STATUS_COMMITTED)` callback. This is NOT a CDI event fire — `@ObservesAsync ChannelActivityEvent` will not trigger. `MessageService` must also fire `ChannelActivityEvent` as a CDI async event (via `Event<ChannelActivityEvent>.fireAsync()`) in the same `afterCompletion` callback. The broadcaster call is preserved for its existing consumers (SSE, WebSocket, `pg_notify`); the CDI fire is additive. This is step 5 in the implementation sequence.

**API evolution — add `tenancyId` to `ChannelActivityEvent`:** The event currently carries `(UUID channelId, String channelName, Long messageId)`. The bridge's `toCloudEvent()` must set the `tenancyid` CloudEvent extension — `RasEngine.onCloudEvent()` unconditionally skips events without it (logs "CloudEvent without tenancyid extension — skipping" and returns). Add `String tenancyId` to the record. At the fire site in `MessageService.dispatch()`, `Channel.tenancyId()` is in scope.

**CloudEvent `tenancyid` resolution in `toCloudEvent()`:**

| Event type | `tenancyid` source |
|---|---|
| `CommitmentStateChangedEvent` | `event.commitment().tenancyId()` |
| `ChannelActivityEvent` | `event.tenancyId()` (after API evolution above) |

**Note:** `CommitmentCancelledEvent` (referenced in casehub-ras#66) does not exist in the qhorus codebase. `CommitmentState` has no `CANCELLED` value — cancellation is not modelled in qhorus's commitment lifecycle at all. The issue's reference is aspirational. If cancellation semantics are needed in future, they would require adding `CANCELLED` to `CommitmentState` and wiring a corresponding state transition — a separate concern from this spec.

**Event ordering:** CDI `@ObservesAsync` does not guarantee delivery order. Two rapid commitment state changes on the same channel could arrive at RAS out of order. This is acceptable because each commitment lifecycle event is self-contained — `CommitmentStateChangedEvent` carries the full `Commitment` object with `previousState`, so RAS can reconstruct the transition without depending on arrival order. Situation templates must be designed to tolerate out-of-order delivery: use the event's embedded state rather than inferring state from event sequence. The pre-built situations (`ack-timeout`, `decline-pattern`, etc.) use `CommitmentState` values from the event payload, not arrival order, for detection logic.

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

**Lifecycle management:**

1. **Startup initialization.** `QhorusChannelFilter` observes `@Initialized(ApplicationScoped.class)` and scans all `Channel` entities whose `protocols` list references situation-containing policies. For each, it calls `register(channelId, eventTypes)` where `eventTypes` is derived from the situations' `SituationDefinition.eventTypes()`. The `SituationPolicyCompiler` drives this — it knows which policies have `situations:` blocks and which channels bind those policies.

2. **Runtime policy changes.** When a channel's `protocols` list is updated (via REST API `PUT /api/channels/{id}` or `ChannelService.update()`), a new `ChannelPolicyChangedEvent` CDI event is fired. This event does not exist today and must be created as part of this work — it is distinct from `ChannelMutationEvent` (which covers UI-observable mutations: reactions, topics, members, spaces). Protocol changes are internal configuration, not UI mutations. `QhorusChannelFilter` observes `ChannelPolicyChangedEvent` and re-evaluates whether the channel is RAS-active:
   - If the new protocols include situation-containing policies → `register()` with updated event types
   - If the new protocols no longer include any situation-containing policies → `deregister()`

3. **Situation registration/deregistration.** When `QhorusSituationProvider` runtime-registers or deregisters a situation (via `SituationRegistrar.register()`/`deregister()`), the filter is updated to reflect the changed event type mappings. The provider calls `channelFilter.register()`/`deregister()` as part of the registration lifecycle.

4. **DB override changes.** `set_policy_overrides` changes threshold parameters, not which situations exist. The filter does not need updating for override changes — overrides affect situation behavior, not event routing.

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
            correctionUncertainty(),
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
| `correction-uncertainty` | Correction then retraction on same message, repeated | Sequence (ChainMode) |
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
- CDI-free unit tests for `enforceIfRequired()` with severity × mode matrix (all 9 cells of Severity × EnforcementMode)
- `@QuarkusTest` integration tests for full dispatch pipeline with `DispatchAdvisory` flow

**Layer 2 (policy model):**
- YAML parsing unit tests for policy format
- Compilation unit tests — `ChannelPolicyCompiler`: YAML `dispatch_rules:` → `ChannelProtocol`
- Compilation unit tests — `SituationPolicyCompiler`: YAML `situations:` → `SituationRegistration` (in `ras/qhorus/` test scope)
- Override application tests (base + channel overrides → effective config)

**Layer 3 (RAS adapter):**
- CDI-free unit tests for QhorusChannelFilter
- CDI-free unit tests for CloudEvent conversion (each CDI event type → CloudEvent)
- `@QuarkusTest` integration tests for end-to-end bridge (mock RAS event bus, verify CDI event → CloudEvent flow)

### Migration path

All changes are breaking (pre-release). No backward compat shims needed.

1. `ChannelProtocol.evaluate()` return type: `List<String>` → `List<DispatchAdvisory>`
2. `DispatchResult.advisories`: `List<String>` → `List<DispatchAdvisory>`
3. `EnforcementBlockedException.violations`: `List<String>` → `List<DispatchAdvisory>`
4. `EnforcementBlockedEvent.violations`: `List<String>` → `List<DispatchAdvisory>`
5. `TaggedAdvisory` deleted — all usages replaced with `DispatchAdvisory`
6. All callers of the above updated in the same commit

### Flyway migrations

- **V55:** `channel.policy_overrides` TEXT nullable

### Module dependencies

```
casehub-qhorus-api
  └── DispatchAdvisory, Severity, SuggestedAction, ProtocolEvaluationEvent

casehub-qhorus (runtime-core)
  └── enforcement gate (severity-aware), DispatchAdvisory used directly (no TaggedAdvisory)

casehub-qhorus (runtime)
  └── ChannelPolicyCompiler (dispatch_rules → ChannelProtocol only), YAML loading

casehub-ras-api
  └── SituationDefinition, GanglionDescriptor (unchanged)

casehub-ras/qhorus (new adapter module)
  └── depends on: casehub-qhorus-api, casehub-ras-api
  └── provides: QhorusEventBridge, QhorusSituationProvider, QhorusChannelFilter,
                SituationPolicyCompiler (situations → SituationRegistration)
  └── observes: CommitmentStateChangedEvent, ChannelActivityEvent, ProtocolEvaluationEvent
```

---

## Implementation Sequence

Recommended implementation order within a single branch:

1. **DispatchAdvisory, Severity, SuggestedAction** in `api/spi/` — the foundation types
2. **ChannelProtocol SPI change** + update all 4 built-in protocols to return `List<DispatchAdvisory>`
3. **Delete TaggedAdvisory** + update enforcement gate to use `DispatchAdvisory` directly + severity-aware logic
4. **DispatchResult/EnforcementBlockedException/EnforcementBlockedEvent** evolution to `List<DispatchAdvisory>`
5. **Wire up CDI event prerequisites** — fire `CommitmentStateChangedEvent` from `CommitmentService` on all state transitions + fire `ChannelActivityEvent` as CDI async event alongside `broadcaster.broadcast()` in `MessageService` + add `tenancyId` to `ChannelActivityEvent` record + create `ChannelPolicyChangedEvent` in `api/event/` (fired from `ChannelService.update()` when protocols change)
6. **ProtocolEvaluationEvent** CDI event
7. **Update all tests** for the above
8. **Channel policy YAML format** — `ChannelPolicyCompiler` for `dispatch_rules:` only
9. **V55 migration** + policy overrides on Channel
10. **RAS adapter module** (in casehub-ras repo, separate branch) — `SituationPolicyCompiler` for `situations:`, event bridge, channel filter

Steps 1-7 are the core SPI evolution (Layer 1). Steps 8-9 are the policy model (Layer 2). Step 10 is the RAS bridge (Layer 3).

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

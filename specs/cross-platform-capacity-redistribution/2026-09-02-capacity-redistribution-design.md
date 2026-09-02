# Context-Aware Work Redistribution — Cross-Platform Design Spec

**Focal issue:** casehubio/qhorus#405 (E8: Context-aware work redistribution)
**Scope:** casehub-platform, casehub-eidos, casehub-qhorus, casehub-engine
**Status:** draft

---

## Problem Statement

Three casehub systems independently detect actor overload using incompatible vocabularies:
- **Qhorus** measures `context_window_pct` (0-100) via CONTEXT_PRESSURE watchdog — fires alerts
- **Engine** measures `activeTaskCount` vs `maxActiveTaskCount` via `WorkloadConstraint` — filters candidates at pre-assignment only
- **Platform agent-gate** measures concurrent sessions vs semaphore — blocks callers with exceptions

No system can see another's load signals. Eidos agent selection (RAS — reputation-aware selection) picks agents by trust and behavioral health but is load-blind. No post-assignment redistribution exists anywhere — once work is assigned, it stays even when the actor is saturated.

## Design Principles

1. **Shared vocabulary, domain-specific signals** — all domains express load as pressure 0.0–1.0; each maps from its own units
2. **Single selection authority** — eidos RAS is enriched with load awareness, not bypassed
3. **Prevention before redistribution** — overloaded agents excluded from new work assignment (cheaper than moving existing work)
4. **Graceful degradation** — compression before redistribution, redistribution before escalation
5. **Domain-specific execution** — qhorus HANDOFFs obligations, engine reassigns work items; platform defines the vocabulary and policy, not the mechanism

## Architecture — Four Layers

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: EXECUTE (domain-specific)                          │
│   qhorus: compress → HANDOFF + commitment delegation        │
│   engine: WorkItem reassignment via WorkloadDataProvider    │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: DECIDE (platform-api)                              │
│   RedistributionPolicy SPI                                  │
│   → Redistribute / Compress / Hold / Escalate               │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: SELECT (eidos)                                     │
│   RAS gains Overloaded probe in CapabilityHealth chain      │
│   SelectionContext + ActorCapacityView                      │
├─────────────────────────────────────────────────────────────┤
│ Layer 0: OBSERVE (platform-api + domain implementations)    │
│   CapacitySignal — shared vocabulary for actor load         │
│   Sources: qhorus, engine, platform-gate                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 0: Observe — Shared Capacity Vocabulary

**Repo: platform-api** (`io.casehub.platform.api.capacity`)

### Types

```java
public record CapacitySignal(
    String actorId,
    String signalType,
    double pressure,       // 0.0 = idle, 1.0 = saturated, >1.0 = overloaded
    Instant observedAt,
    Map<String, String> metadata  // domain-specific context
) {}

public interface CapacitySignalSource {
    String signalType();
    Optional<CapacitySignal> observe(String actorId);
    List<CapacitySignal> observeOverloaded(double threshold);
}

public record ActorCapacity(
    String actorId,
    double aggregatePressure,
    Map<String, Double> pressureBySignalType,
    Instant observedAt
) {
    public boolean isOverloaded(double threshold) {
        return aggregatePressure > threshold;
    }
}

public interface ActorCapacityView {
    ActorCapacity getCapacity(String actorId);
    List<ActorCapacity> getOverloaded(double threshold);
}
```

### Signal Type Constants

```java
public final class CapacitySignalTypes {
    public static final String CONTEXT_PRESSURE = "context_pressure";
    public static final String TASK_COUNT = "task_count";
    public static final String SESSION_COUNT = "session_count";
}
```

### Domain Signal Sources

| Domain | Signal Type | Raw Metric | Pressure Formula |
|--------|-------------|------------|------------------|
| Qhorus | `context_pressure` | `context_window_pct` (0-100) from EVENT ledger entries | `pct / 100.0` |
| Engine | `task_count` | `activeTaskCount` / `maxActiveTaskCount` from `WorkloadDataProvider` | `count / max` (1.0+ when over limit) |
| Platform gate | `session_count` | `activeSessions` / `maxConcurrent` from `SessionRegistry` | `active / max` |

**Qhorus source** (`ContextPressureCapacitySource`): reads latest `context_window_pct` per agent from `MessageLedgerEntryRepository.findLatestContextPressure()` — same query the CONTEXT_PRESSURE watchdog uses. No new data collection needed.

**Engine source** (`WorkloadCapacitySource`): reads from `WorkloadDataProvider.getSnapshot(actorId)`. Existing SPI — already implemented by engine consumers.

**Platform gate source** (`SessionCapacitySource`): reads from `SessionRegistry`. Lower priority — defer to batch 3.

---

## Layer 0.5: Aggregate — Unified Actor Capacity

**Repo: platform** (`io.casehub.platform.capacity`)

```java
@ApplicationScoped
public class AggregatingActorCapacityView implements ActorCapacityView {

    @Any Instance<CapacitySignalSource> sources;

    @Override
    public ActorCapacity getCapacity(String actorId) {
        Map<String, Double> pressures = new LinkedHashMap<>();
        Instant latest = Instant.EPOCH;

        for (var source : sources) {
            source.observe(actorId).ifPresent(signal -> {
                pressures.put(signal.signalType(), signal.pressure());
            });
        }

        double aggregate = pressures.values().stream()
                .mapToDouble(Double::doubleValue)
                .max().orElse(0.0);  // max-pressure wins

        return new ActorCapacity(actorId, aggregate, pressures, Instant.now());
    }

    @Override
    public List<ActorCapacity> getOverloaded(double threshold) {
        // Union of all sources' overloaded actors, deduplicated
    }
}
```

**Aggregation strategy:** max-pressure across signal types. An agent at 0.9 context pressure and 0.3 task count is at 0.9 aggregate. Rationale: any single saturated dimension is a redistribution trigger — an agent with full context window can't take new work regardless of task count.

### CapacityPressureEvent

```java
// CDI event — fired when any actor crosses threshold
public record CapacityPressureEvent(
    String actorId,
    ActorCapacity capacity,
    double threshold,
    String triggerSignalType  // which signal type pushed over
) {}
```

Fired by a `@Scheduled` sweep in `CapacityPressureMonitor` (platform), not by individual signal sources. Single sweep prevents event storms from multiple sources crossing threshold simultaneously.

---

## Layer 1: Select — RAS Load Enrichment

**Repo: eidos-api + eidos runtime**

### SelectionContext Extension

```java
// eidos-api — backward-compat extension
public record SelectionContext(
    String tenancyId,
    String capabilityName,
    String taskDomain,
    ActorCapacityView capacityView  // nullable — null = load-unaware (backward-compat)
) {
    // Existing 3-arg constructor delegates with null capacityView
    public SelectionContext(String tenancyId, String capabilityName, String taskDomain) {
        this(tenancyId, capabilityName, taskDomain, null);
    }
}
```

### CapabilityHealth — Overloaded Probe Step

```java
// eidos runtime — new probe step in existing chain
// Positioned after BehavioralViolation, before Ready
case OVERLOADED -> {
    if (context.capacityView() != null) {
        ActorCapacity cap = context.capacityView().getCapacity(agent.instanceId());
        if (cap.isOverloaded(effectiveThreshold(channel))) {
            yield CapabilityHealth.OVERLOADED;
        }
    }
    yield CapabilityHealth.READY;
}
```

**Effect on RAS:** Overloaded agents are excluded from `AgentSelector.select()` candidate sets. This serves two purposes:
1. **Prevention** — new `role:X` COMMAND routing skips overloaded agents
2. **Target selection** — when redistribution fires, the source agent is naturally excluded from candidates

**Threshold source:** Per-channel `routingTrustThreshold` already exists; add per-channel `routingCapacityThreshold` (nullable, falls back to global `casehub.eidos.routing.default-capacity-threshold`, default 0.8).

---

## Layer 2: Decide — Redistribution Policy

**Repo: platform-api** (`io.casehub.platform.api.capacity`)

```java
public interface RedistributionPolicy {
    RedistributionDecision evaluate(RedistributionContext context);
}

public record RedistributionContext(
    String actorId,
    ActorCapacity capacity,
    String triggerSignalType,
    int openObligationCount,    // from domain — qhorus commitments, engine tasks
    Duration timeSinceLastActivity
) {}

public sealed interface RedistributionDecision {
    record Redistribute(String reason, Duration gracePeriod,
                         Set<String> excludeActors) implements RedistributionDecision {}
    record Compress(String reason) implements RedistributionDecision {}
    record Hold(String reason) implements RedistributionDecision {}
    record Escalate(String reason) implements RedistributionDecision {}
}
```

**Default policy** (`platform/capacity/DefaultRedistributionPolicy`):

| Pressure | Open Obligations | Decision |
|----------|-----------------|----------|
| < 0.7 | any | Hold — within tolerance |
| 0.7–0.85 | any | Compress — try context compression first |
| 0.85–0.95 | > 0 | Redistribute — grace period 30s |
| > 0.95 | > 0 | Redistribute — grace period 0 (immediate) |
| > 1.0 | 0 | Hold — overloaded but no movable work |
| any | any, inactive > 5m | Escalate — agent may be stuck |

Config: `casehub.capacity.redistribution.compress-threshold` (0.7), `casehub.capacity.redistribution.redistribute-threshold` (0.85), `casehub.capacity.redistribution.immediate-threshold` (0.95).

---

## Layer 3: Execute — Domain-Specific Redistribution

### Qhorus Executor

**Repo: qhorus** (`io.casehub.qhorus.runtime.capacity`)

Observes `CapacityPressureEvent`. Execution sequence:

1. **Compress** — `ChannelSummaryService.triggerUpdate()` for all channels where the overloaded agent has open commitments. Wait for grace period.
2. **Re-evaluate** — check `ActorCapacityView.getCapacity()` again after compression
3. **Redistribute** — for each OPEN/ACKNOWLEDGED commitment on the overloaded agent:
   - Select target via `RoutingBridge.resolve()` (RAS excludes the overloaded agent)
   - Send HANDOFF message (creates child commitment on delegate)
   - Record in ledger with `routing_original_target` = overloaded agent, `routing_strategy` = "redistribution"
4. **Notify** — fire `CommitmentDelegatedEvent` (existing CDI event) + `RedistributionExecutedEvent` (new, for audit/notification bridge)

**Interaction with existing watchdog:** CONTEXT_PRESSURE watchdog continues to fire alerts independently. The capacity redistribution executor handles the automated response. They coexist — the watchdog is the notification path, the executor is the action path.

### Engine Executor

**Repo: engine** (deferred — batch 3)

Observes `CapacityPressureEvent`. Execution sequence:

1. Query active WorkItems for the overloaded actor
2. Select target via eidos `AgentSelector` (or engine's `AgentRoutingStrategy`)
3. Reassign WorkItem to target — `TaskStatus.DELEGATED`
4. Record audit event

**Deferral rationale:** Engine's `WorkloadConstraint` already prevents over-assignment. The gap (post-assignment redistribution) is real but less urgent — engine tasks are typically shorter-lived than qhorus obligations. Batch 3 delivers engine integration after the platform+eidos+qhorus foundation is proven.

---

## Relationship to Existing Infrastructure

### What Gets Reused (no changes needed)

| Component | Repo | Role in Redistribution |
|-----------|------|----------------------|
| `CONTEXT_PRESSURE` watchdog | qhorus | Source data for `ContextPressureCapacitySource` |
| `RoutingBridge` | qhorus | Target selection for HANDOFF (already trust+capability-aware via RAS) |
| `HANDOFF` message type + commitment `DELEGATED` state | qhorus | Execution mechanism |
| `Channel.routingTrustThreshold` | qhorus | Trust floor for redistribution targets |
| `ChannelSummaryService.triggerUpdate()` | qhorus | Context compression before redistribution |
| `WorkloadConstraint` / `WorkloadDataProvider` | engine | Source data for `WorkloadCapacitySource` |
| `ActorStateAggregator` | engine | Cross-domain actor view (complementary, not replaced) |
| `AgentSelector` / `CapabilityHealth` probe chain | eidos | Selection with new Overloaded probe step |

### What Gets Extended

| Component | Change | Repo |
|-----------|--------|------|
| `SelectionContext` | Add `capacityView` field (nullable, backward-compat constructor) | eidos-api |
| `CapabilityHealth` | Add `OVERLOADED` probe step after `BehavioralViolation` | eidos |
| `Channel` | Add `routingCapacityThreshold` (nullable, V-next migration) | qhorus |
| `DssSigningConfig` | *(already done in #262-265 — no further changes)* | platform |

### What's New

| Component | Repo |
|-----------|------|
| `CapacitySignal`, `CapacitySignalSource`, `ActorCapacityView`, `CapacitySignalTypes` | platform-api |
| `ActorCapacity`, `RedistributionPolicy`, `RedistributionDecision`, `RedistributionContext` | platform-api |
| `CapacityPressureEvent` | platform-api |
| `AggregatingActorCapacityView`, `CapacityPressureMonitor`, `DefaultRedistributionPolicy` | platform |
| `ContextPressureCapacitySource` | qhorus |
| `QhorusRedistributionExecutor` | qhorus |
| `RedistributionExecutedEvent` | qhorus |
| `WorkloadCapacitySource` | engine (batch 3) |
| `SessionCapacitySource` | platform-gate (batch 3) |

---

## Migration & Compatibility

- All new SPIs have `@DefaultBean` no-op implementations — existing deployments see no change
- `SelectionContext` backward-compat 3-arg constructor — existing `AgentSelector` callers unaffected
- `CapabilityHealth.OVERLOADED` probe only fires when `capacityView` is non-null — null = load-unaware (original behavior)
- Qhorus `ContextPressureCapacitySource` reads existing ledger data — no new data collection, no migration
- Engine `WorkloadCapacitySource` reads existing `WorkloadDataProvider` — no new data
- New qhorus migration: `Channel.routing_capacity_threshold` (nullable Double)

---

## Batching

### Batch 1: Platform Foundation (platform-api + platform)
- `CapacitySignal`, `CapacitySignalSource`, `ActorCapacityView` SPIs
- `ActorCapacity`, `CapacitySignalTypes`
- `RedistributionPolicy`, `RedistributionDecision`, `RedistributionContext`
- `CapacityPressureEvent`
- `AggregatingActorCapacityView` implementation
- `DefaultRedistributionPolicy` implementation
- `CapacityPressureMonitor` (`@Scheduled` sweep)
- No-op defaults for all SPIs
- Tests: CDI-free unit tests for aggregation + policy

### Batch 2: Eidos Selection Enrichment (eidos-api + eidos)
- `SelectionContext` extension (capacityView field)
- `CapabilityHealth.OVERLOADED` probe step
- `casehub.eidos.routing.default-capacity-threshold` config
- Tests: selection with overloaded candidates excluded

### Batch 3: Qhorus Signal Source + Redistribution (qhorus)
- `ContextPressureCapacitySource` implements `CapacitySignalSource`
- `QhorusRedistributionExecutor` observes `CapacityPressureEvent`
- Compression-first flow (channel summary → re-evaluate → HANDOFF)
- `Channel.routingCapacityThreshold` + migration
- `RedistributionExecutedEvent` CDI event
- Tests: CDI-free executor tests, integration test with HANDOFF verification

### Batch 4: Engine Signal Source (engine) — deferrable
- `WorkloadCapacitySource` implements `CapacitySignalSource`
- Engine redistribution executor (WorkItem reassignment)
- Can ship independently after batches 1-3 prove the model

---

## Configuration Summary

| Key | Default | Where |
|-----|---------|-------|
| `casehub.capacity.sweep-interval` | `60s` | platform |
| `casehub.capacity.redistribution.compress-threshold` | `0.7` | platform |
| `casehub.capacity.redistribution.redistribute-threshold` | `0.85` | platform |
| `casehub.capacity.redistribution.immediate-threshold` | `0.95` | platform |
| `casehub.capacity.redistribution.grace-period` | `30s` | platform |
| `casehub.eidos.routing.default-capacity-threshold` | `0.8` | eidos |
| `casehub.qhorus.routing.capacity-threshold` | *(per-channel, nullable → eidos default)* | qhorus |

---

## Open Questions

1. **Aggregation strategy** — max-pressure is simple but aggressive. Should it be configurable (max / weighted-average / domain-priority)?
2. **Redistribution scope** — should redistribution move ALL obligations from an overloaded agent, or just enough to bring pressure below threshold?
3. **Circular redistribution guard** — if agent A hands off to B and B becomes overloaded, need a circuit breaker to prevent ping-pong. Depth limit on delegation chain (existing `CIRCULAR_DELEGATION` watchdog covers this)?
4. **Cross-tenant capacity** — should capacity view be tenant-scoped or global? An agent at 0.9 in tenant-A and 0.1 in tenant-B is 0.9 aggregate — but redistribution in tenant-A shouldn't move work to a tenant-B candidate.

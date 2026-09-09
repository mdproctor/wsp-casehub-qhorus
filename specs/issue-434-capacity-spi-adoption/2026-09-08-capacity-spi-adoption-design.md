# Capacity Redistribution SPI Adoption — Design Spec

**Focal issue:** casehubio/qhorus#434
**Upstream:** casehubio/platform#268 (capacity signal SPI + redistribution policy framework)
**Parent spec:** `docs/specs/cross-platform-capacity-redistribution/2026-09-02-capacity-redistribution-design.md`
**Status:** approved

---

## Problem

Qhorus implements `ContextPressureCapacitySource` and `QhorusRedistributionExecutor` (from #428) using platform-api SPI types, but three gaps remain:

1. **Platform's DefaultRedistributionPolicy** ignores `RedistributionContext.openObligationCount` and `timeSinceLastActivity` — maps >=0.95 to Escalate instead of immediate Redistribute, and never triggers inactivity-based escalation. The enrichment uses platform-generic concepts and belongs upstream.
2. **Commitment overload is invisible to prevention layers** — only available in Layer 2 (policy context), not Layers 0/0.5/1 (signal sources, aggregation, routing exclusion). An agent at its commitment limit still receives new work via routing.
3. **No per-channel threshold granularity** — all channels share global thresholds. High-urgency channels can't trigger earlier redistribution.

## Scope

### Cross-repo: casehub-platform (PR to platform)
- Enrich `DefaultRedistributionPolicy` with obligation-aware and inactivity-aware decisions (D1)
- Consolidate config keys under `casehub.capacity.redistribution.*` prefix — replaces `casehub.capacity.threshold.*` (aligns with parent spec and `CapacityPressureMonitor`)
- Annotate `DefaultRedistributionPolicy` with `@DefaultBean` (D6)

### This repo: casehub-qhorus
- `CommitmentCountCapacitySource` — new `CapacitySignalSource` implementation (D2)
- `Channel.redistributionCapacityThreshold` — per-channel nullable field (D4); `routingCapacityThreshold` deferred until eidos OVERLOADED probe is implemented (tracked: casehubio/qhorus#TBD)
- Executor channel-threshold filtering integrated into `RedistributionDelegate.redistribute()` (D5)
- `RedistributionResult` — extended with `attemptedCount` and `filteredCount` for channel-threshold awareness
- MCP tools: `get_actor_capacity`, `list_overloaded_actors`, `get_redistribution_history`, `set_channel_redistribution_threshold`, `get_channel_redistribution_threshold` (D3)
- V51 migration for the new channel column
- Integration test with HANDOFF verification, commitment-count-driven redistribution, and channel-threshold filtering

### Deferred (tracked as GitHub issues)
- `Channel.routingCapacityThreshold` + eidos OVERLOADED probe integration — the `CapabilityStatus.Overloaded` record type exists in eidos-api but `DefaultCapabilityHealth.probe()` never returns it and `ProbeContext` lacks the required `capacityView` field (casehubio/qhorus#TBD)
- Platform MCP surface for capacity tools — `get_actor_capacity` and `list_overloaded_actors` read platform-scoped `ActorCapacityView` data; temporarily placed in qhorus MCP surface (casehubio/platform#TBD)

---

## Platform Changes (cross-repo PR)

### Enrich DefaultRedistributionPolicy

The enriched policy uses the full `RedistributionContext`:

| Pressure | Obligations | Inactivity | Decision |
|----------|------------|------------|----------|
| < compress (0.7) | any | < 5m | Hold |
| >= compress, < redistribute | any | < 5m | Compress |
| >= redistribute (0.85), < immediate (0.95) | > 0 | < 5m | Redistribute (grace period 30s, excludeActors = {actorId}) |
| >= immediate (0.95) | > 0 | < 5m | Redistribute (grace period 0, excludeActors = {actorId}) |
| >= redistribute | 0 | < 5m | Hold — overloaded but no movable work |
| any | any | >= 5m | Escalate — agent may be stuck |

The `excludeActors` field on `Redistribute` is populated with the context's `actorId` to prevent the overloaded agent from being selected as a redistribution target. This is the only defense against self-routing until the eidos OVERLOADED probe is implemented (see Deferred items above).

Config keys (consolidated under `casehub.capacity.redistribution.*` — replaces old `casehub.capacity.threshold.*` prefix):
- `casehub.capacity.redistribution.compress-threshold` (0.7) — shared with `CapacityPressureMonitor`
- `casehub.capacity.redistribution.redistribute-threshold` (0.85)
- `casehub.capacity.redistribution.immediate-threshold` (0.95)
- `casehub.capacity.redistribution.grace-period` (30s) — new
- `casehub.capacity.redistribution.inactivity-escalation` (5m) — new

### @DefaultBean annotation

`DefaultRedistributionPolicy` gains `@DefaultBean` alongside `@ApplicationScoped`. Consumers override with a plain `@ApplicationScoped` implementation — no `@Alternative` needed. Follows the established pattern from the engine @DefaultBean SPI spec.

---

## Qhorus Changes

### 1. CommitmentCountCapacitySource

New `CapacitySignalSource` in `io.casehub.qhorus.runtime.capacity`.

```java
@ApplicationScoped
public class CommitmentCountCapacitySource implements CapacitySignalSource {

    @Override
    public List<CapacitySignal> observe(String actorId) {
        // Query CrossTenantCommitmentStore.findOpenByObligor(actorId)
        // pressure = count / maxObligations (config)
        // Clamp to 1.0
    }

    @Override
    public List<CapacitySignal> observeOverloaded(double threshold) {
        // Efficient query: CrossTenantCommitmentStore.findObligorsExceedingCount(minCount)
        //   where minCount = ceil(threshold * maxObligations)
        // Returns Map<String, Long> of obligor → count
        // Convert to CapacitySignal with pressure = count / maxObligations
    }
}
```

`observeOverloaded` requires a new efficient query method on `CrossTenantCommitmentStore`:

```java
/**
 * Find obligors with at least {@code minCount} open commitments.
 * Executes: SELECT obligor, COUNT(*) FROM commitment
 *           WHERE state IN ('OPEN','ACKNOWLEDGED') GROUP BY obligor HAVING COUNT(*) >= ?
 * Returns one row per qualifying obligor — O(qualifying-actors), not O(all-commitments).
 */
Map<String, Long> findObligorsExceedingCount(int minCount);
```

This mirrors `ContextPressureCapacitySource.observeOverloaded()` which uses the efficient `findLatestContextPressureGlobal()` (one row per actor).

Signal type: `CapacitySignalTypes.TASK_COUNT` (reuses existing constant — commitment is the qhorus analogue of a task).

Config: `casehub.qhorus.capacity.max-obligations` (default 20).

Aggregation effect: `AggregatingActorCapacityView` uses max-pressure (D9). An agent at 0.9 commitment pressure and 0.3 context pressure is at 0.9 aggregate.

### 2. Per-Channel Capacity Thresholds

One new nullable `Double` field on `Channel`:

| Field | Purpose | Fallback |
|-------|---------|----------|
| `redistributionCapacityThreshold` | Executor filtering — only redistribute obligations from channels where this threshold is exceeded | `casehub.capacity.redistribution.redistribute-threshold` (0.85) |

`routingCapacityThreshold` is deferred until the eidos OVERLOADED probe is implemented (tracked: casehubio/qhorus#TBD). The probe type (`CapabilityStatus.Overloaded`) exists in eidos-api but `DefaultCapabilityHealth.probe()` does not return it, and `ProbeContext` lacks the required `capacityView` field. Adding the routing threshold column now would create dead data with no consumer.

V51 migration:
```sql
ALTER TABLE channel ADD COLUMN redistribution_capacity_threshold DOUBLE PRECISION;
```

MCP tools:
- `set_channel_redistribution_threshold(channel, threshold?)` — nullable, null clears
- `get_channel_redistribution_threshold(channel)` — returns value with effective fallback

### 3. Executor Channel-Threshold Filtering

Channel-threshold filtering is integrated into the existing per-obligation loop in `RedistributionDelegate.redistribute()`, eliminating a separate pre-filter pass and avoiding double channel lookups:

```java
@Transactional
public RedistributionResult redistribute(String actorId, List<Commitment> obligations,
                                          RedistributionDecision.Redistribute decision,
                                          double aggregatePressure) {
    var redistributable = obligations.stream()
            .filter(c -> c.capabilityTag() != null)
            .toList();

    int successCount = 0;
    int filteredCount = 0;
    for (var commitment : redistributable) {
        try {
            inboundTenancyContext.set(commitment.tenancyId());

            // ... existing message lookup ...

            var channel = channelStore.findById(commitment.channelId()).orElse(null);
            if (channel == null) continue;

            // Channel-threshold filtering — single lookup, no pre-filter
            double threshold = channel.redistributionCapacityThreshold() != null
                ? channel.redistributionCapacityThreshold()
                : globalRedistributeThreshold;
            if (aggregatePressure < threshold) {
                filteredCount++;
                continue;
            }

            // ... existing HANDOFF dispatch logic ...
            successCount++;
        } catch (Exception e) {
            // ... existing error handling ...
        }
    }

    int attemptedCount = redistributable.size() - filteredCount;
    executedEvents.fireAsync(
            RedistributionExecutedEvent.redistributed(actorId, successCount, attemptedCount, filteredCount));
    return new RedistributionResult(successCount, attemptedCount, filteredCount);
}
```

`RedistributionResult` is extended to distinguish channel-filtered obligations from failed HANDOFF attempts:

```java
public record RedistributionResult(int successCount, int attemptedCount, int filteredCount) {}
```

The executor's escalation guard is updated to handle the three outcomes:

```java
case RedistributionDecision.Redistribute r -> {
    // ... grace period check ...
    RedistributionResult result = delegate.redistribute(actorId, obligations, r, pressure);
    if (result.successCount() == 0 && result.attemptedCount() > 0) {
        // HANDOFF attempted but all targets unavailable — escalate
        delegate.escalate(actorId, "redistribution requested but no targets available");
    } else if (result.attemptedCount() == 0 && result.filteredCount() > 0) {
        // All obligations filtered by channel thresholds — compress as fallback
        LOG.infof("All obligations filtered by channel thresholds for %s — compressing", actorId);
        delegate.compress(actorId, obligations);
    }
}
```

This correctly distinguishes:
- **All attempted, all failed** → escalate (target unavailability — operator attention needed)
- **All filtered** → compress fallback (channels have higher thresholds than aggregate pressure — try freeing context)
- **Some succeeded** → normal success (no further action)

### 4. MCP Tools

Five `@Tool` methods in `QhorusMcpTools`:

**`get_actor_capacity(actor_id)`**
- Delegates to `ActorCapacityView.getCapacity(actorId)`
- Returns: actorId, aggregatePressure, pressureBySignalType map, observedAt
- Boundary note: reads platform-scoped data; temporary placement until platform MCP surface exists (tracked: casehubio/platform#TBD)

**`list_overloaded_actors(threshold?)`**
- Delegates to `ActorCapacityView.getOverloaded(threshold)` — threshold defaults to compress threshold (0.7)
- Returns: list of ActorCapacity records
- Boundary note: same as get_actor_capacity (tracked: casehubio/platform#TBD)

**`get_redistribution_history(actor_id?, channel?, limit?)`**
- Queries ledger for HANDOFF entries from sender `system:redistribution`
- Filters by actorId (as routing_original_target) and/or channel
- Returns: list of ledger entries with redistribution metadata
- Qhorus-scoped — queries MessageLedgerEntryRepository

**`set_channel_redistribution_threshold(channel, threshold?)`**
- Sets `redistributionCapacityThreshold` on the channel; null clears
- Qhorus-scoped — updates Channel record

**`get_channel_redistribution_threshold(channel)`**
- Returns the channel's `redistributionCapacityThreshold` with effective value (including fallback to global `casehub.capacity.redistribution.redistribute-threshold`)
- Qhorus-scoped — reads Channel record

### 5. Watchdog Interaction (D7)

No coordination mechanism between CONTEXT_PRESSURE watchdog and the redistribution executor. They coexist via natural severity ordering:

- `CapacityPressureMonitor` sweeps at compress threshold (0.7) — fires `CapacityPressureEvent` for automated redistribution
- Watchdog fires at its configured threshold (typically higher) — ALERT/PAUSE_CHANNEL/QUARANTINE

Normal flow: redistribution fires first (lower threshold), reduces pressure, watchdog never triggers. If pressure is extreme: watchdog containment fires, executor's per-obligation try-catch handles HANDOFF failures from paused channels gracefully.

### 6. Integration Test

`@QuarkusTest` in `runtime/src/test/` with `@TestProfile` enabling capacity:

**Scenario 1: Context-pressure-driven redistribution with channel-threshold filtering**
1. Register two agents with capabilities
2. Create channel-A with `redistributionCapacityThreshold = 0.80`
3. Create channel-B with `redistributionCapacityThreshold = 0.95`
4. Dispatch COMMAND to agent-1 in both channels (creates OPEN commitments)
5. Dispatch EVENT with `context_window_pct: 90` for agent-1 (simulates 0.90 pressure)
6. Fire `CapacityPressureEvent` directly (bypass scheduler)
7. Verify: HANDOFF message dispatched for channel-A obligation (0.90 >= 0.80)
8. Verify: NO HANDOFF for channel-B obligation (0.90 < 0.95 — filtered)
9. Verify: commitment for agent-1 in channel-A is DELEGATED
10. Verify: new OPEN commitment exists for agent-2 in channel-A
11. Verify: commitment for agent-1 in channel-B remains OPEN
12. Verify: ledger entry recorded with `routing_original_target`, `routing_strategy = "redistribution"`

**Scenario 2: Commitment-count-driven redistribution**
1. Register two agents with capabilities
2. Create channel with default thresholds
3. Create 20 OPEN commitments for agent-1 (max-obligations = 20, pressure = 1.0)
4. Context pressure is LOW (0.1)
5. Fire `CapacityPressureEvent` — aggregate pressure should be max(1.0, 0.1) = 1.0
6. Verify: redistribution triggers from commitment pressure alone
7. Verify: HANDOFF message dispatched

**Scenario 3: Compress-fallback when all obligations are channel-filtered**
1. Register two agents with capabilities
2. Create channel with `redistributionCapacityThreshold = 0.90`
3. Dispatch COMMAND to agent-1 (creates OPEN commitment)
4. Dispatch EVENT with `context_window_pct: 86` for agent-1 (simulates 0.86 pressure)
5. Fire `CapacityPressureEvent` directly (bypass scheduler)
6. Verify: NO HANDOFF dispatched (0.86 < 0.90 — all obligations filtered)
7. Verify: compress triggered as fallback (channel summary update attempted)
8. Verify: `RedistributionExecutedEvent` with outcome=REDISTRIBUTED, attemptedCount=0, filteredCount>0

Uses `QuarkusTransaction.requiringNew()` for setup and verification (per observer test conventions).

---

## Configuration Summary

| Key | Default | Where |
|-----|---------|-------|
| `casehub.capacity.redistribution.compress-threshold` | `0.7` | platform (existing in CapacityPressureMonitor, adopted by DefaultRedistributionPolicy — replaces `casehub.capacity.threshold.compress`) |
| `casehub.capacity.redistribution.redistribute-threshold` | `0.85` | platform (replaces `casehub.capacity.threshold.redistribute`) |
| `casehub.capacity.redistribution.immediate-threshold` | `0.95` | platform (replaces `casehub.capacity.threshold.escalate` — name now matches semantics) |
| `casehub.capacity.redistribution.grace-period` | `30s` | platform (new) |
| `casehub.capacity.redistribution.inactivity-escalation` | `5m` | platform (new) |
| `casehub.qhorus.capacity.max-obligations` | `20` | qhorus (new) |

---

## What's NOT Changing

- `ContextPressureCapacitySource` — already correctly implements `CapacitySignalSource`, no changes
- `QhorusRedistributionExecutor` — event observation, policy delegation unchanged; escalation guard updated for channel-threshold awareness (see §3)
- `RedistributionDelegate.compress()` and `escalate()` — unchanged
- `RedistributionExecutedEvent` — extended: `totalCount` renamed to `attemptedCount`, `filteredCount` added; factory method `redistributed()` gains `filteredCount` parameter (see §3)
- Existing unit tests — unchanged (new tests added alongside)

---

## Migration & Compatibility

- Platform `DefaultRedistributionPolicy` enrichment includes config key rename from `casehub.capacity.threshold.*` to `casehub.capacity.redistribution.*` — deployments with custom values for the old keys must update their config. This is deliberate: the old naming was inconsistent with `CapacityPressureMonitor` and the parent spec.
- `@DefaultBean` annotation is backward-compatible — no consumer changes unless they want to override
- `CommitmentCountCapacitySource` is additive — existing deployments gain a second signal source transparently via CDI discovery
- Per-channel threshold column is nullable — null = use global default (existing behavior)
- `RedistributionResult` gains `attemptedCount` and `filteredCount` fields (replacing the previous `totalCount`) — existing callers break at compile time, which is the point: they must handle the new semantics explicitly
- `RedistributionExecutedEvent` renames `totalCount` → `attemptedCount` and adds `filteredCount`; factory method `redistributed()` gains a `filteredCount` parameter — same compile-time break rationale
- MCP tools are additive — no existing tool signatures change

---

## References

- `docs/specs/cross-platform-capacity-redistribution/2026-09-02-capacity-redistribution-design.md` — parent design spec (Layers 0-3)
- `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/` — existing implementation from #428
- `platform/platform-api/src/main/java/io/casehub/platform/api/capacity/` — platform SPI types
- `platform/platform/src/main/java/io/casehub/platform/capacity/DefaultRedistributionPolicy.java` — platform default policy (to be enriched)
- `platform/platform/src/main/java/io/casehub/platform/capacity/AggregatingActorCapacityView.java` — max-pressure aggregation
- `docs/specs/2026-05-14-defaultbean-spi-noops-design.md` — @DefaultBean pattern precedent
- casehubio/platform#268 — platform capacity SPI PR
- casehubio/qhorus#428 — initial capacity implementation (closed)

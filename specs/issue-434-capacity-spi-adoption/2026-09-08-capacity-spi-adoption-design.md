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
3. **No per-channel threshold granularity** — all channels share global thresholds. High-urgency channels can't trigger earlier routing exclusion or redistribution.

## Scope

### Cross-repo: casehub-platform (PR to platform)
- Enrich `DefaultRedistributionPolicy` with obligation-aware and inactivity-aware decisions (D1)
- Annotate `DefaultRedistributionPolicy` with `@DefaultBean` (D6)

### This repo: casehub-qhorus
- `CommitmentCountCapacitySource` — new `CapacitySignalSource` implementation (D2)
- `Channel.routingCapacityThreshold` + `Channel.redistributionCapacityThreshold` — two per-channel nullable fields (D4)
- Executor channel-threshold filtering in `RedistributionDelegate.redistribute()` (D5)
- MCP tools: `get_actor_capacity`, `list_overloaded_actors`, `get_redistribution_history` (D3)
- V52 migration for the two new channel columns
- Integration test with HANDOFF verification

---

## Platform Changes (cross-repo PR)

### Enrich DefaultRedistributionPolicy

The enriched policy uses the full `RedistributionContext`:

| Pressure | Obligations | Inactivity | Decision |
|----------|------------|------------|----------|
| < compress (0.7) | any | < 5m | Hold |
| >= compress, < redistribute | any | < 5m | Compress |
| >= redistribute (0.85), < immediate (0.95) | > 0 | < 5m | Redistribute (grace period 30s) |
| >= immediate (0.95) | > 0 | < 5m | Redistribute (grace period 0) |
| >= redistribute | 0 | < 5m | Hold — overloaded but no movable work |
| any | any | >= 5m | Escalate — agent may be stuck |

Config keys (existing names, enriched semantics):
- `casehub.capacity.threshold.compress` (0.7)
- `casehub.capacity.threshold.redistribute` (0.85)
- `casehub.capacity.threshold.escalate` (0.95) — renamed semantics: now the "immediate redistribute" threshold, not escalation
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
        // Query all actors with open obligations
        // Filter by count/max >= threshold
    }
}
```

Signal type: `CapacitySignalTypes.TASK_COUNT` (reuses existing constant — commitment is the qhorus analogue of a task).

Config: `casehub.qhorus.capacity.max-obligations` (default 20).

Aggregation effect: `AggregatingActorCapacityView` uses max-pressure (D9). An agent at 0.9 commitment pressure and 0.3 context pressure is at 0.9 aggregate — correctly excluded from new routing via the OVERLOADED probe.

### 2. Per-Channel Capacity Thresholds

Two new nullable `Double` fields on `Channel`:

| Field | Purpose | Fallback |
|-------|---------|----------|
| `routingCapacityThreshold` | Eidos OVERLOADED probe — excludes agent from new `role:X` routing in this channel | `casehub.eidos.routing.default-capacity-threshold` (0.8) |
| `redistributionCapacityThreshold` | Executor filtering — only redistribute obligations from channels where this threshold is exceeded | `casehub.capacity.threshold.redistribute` (0.85) |

Separation rationale: routing exclusion (prevention) is near-zero cost — exclude early. Redistribution (HANDOFF) has real cost — trigger only when genuinely necessary. A high-urgency channel might set routing exclusion at 0.6 but redistribution at 0.8.

V52 migration:
```sql
ALTER TABLE channel ADD COLUMN routing_capacity_threshold DOUBLE PRECISION;
ALTER TABLE channel ADD COLUMN redistribution_capacity_threshold DOUBLE PRECISION;
```

MCP tools:
- `set_channel_capacity_thresholds(channel, routing_threshold?, redistribution_threshold?)` — both nullable, null clears
- `get_channel_capacity_thresholds(channel)` — returns both with effective values (including fallback)

### 3. Executor Channel-Threshold Filtering

In `RedistributionDelegate.redistribute()`, after the policy returns Redistribute, filter obligations by channel threshold:

```java
// Existing: var redistributable = obligations.stream()
//     .filter(c -> c.capabilityTag() != null).toList();

// Added: per-channel threshold filter
var qualifying = redistributable.stream()
    .filter(c -> {
        var ch = channelStore.findById(c.channelId()).orElse(null);
        if (ch == null) return false;
        double threshold = ch.redistributionCapacityThreshold() != null
            ? ch.redistributionCapacityThreshold()
            : globalRedistributeThreshold;
        return aggregatePressure >= threshold;
    })
    .toList();

if (qualifying.isEmpty() && !redistributable.isEmpty()) {
    LOG.debugf("No obligations qualify after channel-threshold filtering for %s", actorId);
    return new RedistributionResult(0, redistributable.size());
}
```

The executor iterates `qualifying` (not `redistributable`) for the HANDOFF loop. When no obligations qualify, the caller (`QhorusRedistributionExecutor`) triggers escalation via the existing `successCount == 0 && totalCount > 0` guard.

### 4. MCP Tools

Three new `@Tool` methods in `QhorusMcpTools`:

**`get_actor_capacity(actor_id)`**
- Delegates to `ActorCapacityView.getCapacity(actorId)`
- Returns: actorId, aggregatePressure, pressureBySignalType map, observedAt
- Boundary note: reads platform-scoped data; temporary placement until platform MCP surface exists

**`list_overloaded_actors(threshold?)`**
- Delegates to `ActorCapacityView.getOverloaded(threshold)` — threshold defaults to compress threshold (0.7)
- Returns: list of ActorCapacity records
- Boundary note: same as get_actor_capacity

**`get_redistribution_history(actor_id?, channel?, limit?)`**
- Queries ledger for HANDOFF entries from sender `system:redistribution`
- Filters by actorId (as routing_original_target) and/or channel
- Returns: list of ledger entries with redistribution metadata
- Qhorus-scoped — queries MessageLedgerEntryRepository

### 5. Watchdog Interaction (D7)

No coordination mechanism between CONTEXT_PRESSURE watchdog and the redistribution executor. They coexist via natural severity ordering:

- `CapacityPressureMonitor` sweeps at compress threshold (0.7) — fires `CapacityPressureEvent` for automated redistribution
- Watchdog fires at its configured threshold (typically higher) — ALERT/PAUSE_CHANNEL/QUARANTINE

Normal flow: redistribution fires first (lower threshold), reduces pressure, watchdog never triggers. If pressure is extreme: watchdog containment fires, executor's per-obligation try-catch handles HANDOFF failures from paused channels gracefully.

### 6. Integration Test

`@QuarkusTest` in `runtime/src/test/` with `@TestProfile` enabling capacity:

1. Register two agents with capabilities
2. Create channel with `routingCapacityThreshold` and `redistributionCapacityThreshold`
3. Dispatch COMMAND to agent-1 (creates OPEN commitment)
4. Dispatch EVENT with `context_window_pct: 90` for agent-1 (simulates pressure)
5. Fire `CapacityPressureEvent` directly (bypass scheduler)
6. Verify: HANDOFF message dispatched with sender `system:redistribution`
7. Verify: commitment for agent-1 is DELEGATED
8. Verify: new OPEN commitment exists for agent-2
9. Verify: ledger entry recorded with `routing_original_target`, `routing_strategy = "redistribution"`

Uses `QuarkusTransaction.requiringNew()` for setup and verification (per observer test conventions).

---

## Configuration Summary

| Key | Default | Where |
|-----|---------|-------|
| `casehub.capacity.threshold.compress` | `0.7` | platform (existing, unchanged) |
| `casehub.capacity.threshold.redistribute` | `0.85` | platform (existing, enriched semantics) |
| `casehub.capacity.threshold.escalate` | `0.95` | platform (existing, renamed semantics → immediate redistribute) |
| `casehub.capacity.redistribution.grace-period` | `30s` | platform (new) |
| `casehub.capacity.redistribution.inactivity-escalation` | `5m` | platform (new) |
| `casehub.qhorus.capacity.max-obligations` | `20` | qhorus (new) |

---

## What's NOT Changing

- `ContextPressureCapacitySource` — already correctly implements `CapacitySignalSource`, no changes
- `QhorusRedistributionExecutor` — event observation, policy delegation, grace-period logic unchanged
- `RedistributionDelegate.compress()` and `escalate()` — unchanged
- `RedistributionExecutedEvent` — unchanged
- `RedistributionResult` — unchanged
- Existing unit tests — unchanged (new tests added alongside)

---

## Migration & Compatibility

- Platform `DefaultRedistributionPolicy` enrichment is backward-compatible — same config keys, additional branching uses fields already present in `RedistributionContext`
- `@DefaultBean` annotation is backward-compatible — no consumer changes unless they want to override
- `CommitmentCountCapacitySource` is additive — existing deployments gain a second signal source transparently via CDI discovery
- Per-channel threshold columns are nullable — null = use global default (existing behavior)
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

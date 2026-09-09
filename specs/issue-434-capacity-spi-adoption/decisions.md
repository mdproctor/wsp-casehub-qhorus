## D1: Enrich platform's DefaultRedistributionPolicy upstream

**Choice:** Enrich DefaultRedistributionPolicy to use full RedistributionContext: obligation-aware branching (obligations > 0 for Redistribute vs Hold at >= 1.0), inactivity-triggered escalation (inactive > 5m → Escalate), grace period differentiation (>= 0.95 immediate, 0.85–0.95 configurable grace period). Cross-repo PR to casehub-platform.
**Alternatives:**
- QhorusRedistributionPolicy as @Alternative @Priority(1) (original quick-pick) — works but treats a platform gap as a domain-specific need
- Configure platform defaults only — accepts the gap in the default policy
**Rationale:** RedistributionContext carries openObligationCount and timeSinceLastActivity — the platform API was designed for these signals. DefaultRedistributionPolicy ignoring them is a gap in the platform default, not evidence that qhorus needs domain-specific logic. Every entry in the design spec's richer decision table uses platform-generic concepts (pressure thresholds, obligation count, inactivity duration) — none are qhorus-specific. FixedCurrentPrincipal is not valid precedent for production @Alternative overrides (test utility in casehub-platform-testing, @Priority(200)). The engine's @DefaultBean SPI spec (2026-05-14) establishes the pattern for platform fallback implementations.
**Trade-offs:** Requires cross-repo PR to casehub-platform. Design philosophy: cost is always worth paying when the design is right.
**Sources:** DefaultRedistributionPolicy.java (platform), RedistributionContext.java (platform-api), 2026-05-14-defaultbean-spi-noops-design.md, FixedCurrentPrincipal.java (casehub-platform-testing)
**Exploration:** quick → revised by reviewer challenge
**Status:** revised (R1-03: upstream enrichment replaces qhorus @Alternative override)

## D2: CommitmentCountCapacitySource alongside context pressure

**Choice:** Add CommitmentCountCapacitySource mapping openObligations / configuredMax to pressure 0.0–1.0. Platform-wide default via casehub.qhorus.capacity.max-obligations (default 20). Alongside existing ContextPressureCapacitySource.
**Alternatives:**
- Context pressure only (original) — commitment overload invisible to Layer 1 routing exclusion
- Defer to follow-up — leaves prevention gap until addressed
**Rationale:** Design spec principle #3: prevention before redistribution applies to ALL overload dimensions. Commitment count in RedistributionContext is Layer 2 only — invisible to AggregatingActorCapacityView (Layer 0.5), CapabilityHealth.OVERLOADED probe (Layer 1), and CapacityPressureMonitor sweep (Layer 0.5). A CommitmentCountCapacitySource surfaces commitment overload at all layers.
**Trade-offs:** One additional CapacitySignalSource, one config key. Max-pressure aggregation means commitment overload triggers the same prevention/redistribution path as context pressure.
**Sources:** CapacitySignalSource SPI, AggregatingActorCapacityView (max-pressure), design spec principle #3
**Exploration:** quick → revised by reviewer challenge
**Status:** revised (R1-01: commitment overload now visible at all capacity layers)

## D3: Add minimal MCP tool set for capacity observability

**Choice:** get_actor_capacity, list_overloaded_actors, get_redistribution_history (ledger query for sender system:redistribution). Placed in QhorusMcpTools. get_actor_capacity and list_overloaded_actors read platform-scoped data via ActorCapacityView — boundary crossing acknowledged, migrate to platform MCP module when one exists.
**Alternatives:**
- No MCP tools — capacity is infrastructure-only
- Platform-level MCP module for capacity tools — architecturally correct but no PlatformMcpTools exists yet
- Defer to follow-up — separate issue
**Rationale:** LLM agents need visibility into the capacity system to understand why work moves and who is overloaded. get_redistribution_history queries qhorus-scoped ledger data (HANDOFF entries from system:redistribution) and belongs in qhorus. get_actor_capacity and list_overloaded_actors read platform-scoped data — acceptable as temporary placement given no platform MCP surface exists.
**Trade-offs:** Three new @Tool methods. Two read platform-scoped data from qhorus surface — documented boundary crossing, migrate when platform MCP module is created.
**Sources:** QhorusMcpTools pattern (~53 @Tool methods, all qhorus-scoped), ActorCapacityView (platform-api), MessageLedgerEntryRepository queries
**Exploration:** quick → revised for boundary acknowledgment
**Status:** revised (R1-04: platform-scoped data boundary crossing acknowledged)

## D4: Separate per-channel routing and redistribution thresholds

**Choice:** Two nullable per-channel fields: Channel.routingCapacityThreshold (Layer 1 eidos OVERLOADED probe, falls back to casehub.eidos.routing.default-capacity-threshold default 0.8) and Channel.redistributionCapacityThreshold (Layer 3 executor filtering, falls back to casehub.capacity.redistribution.redistribute-threshold default 0.85)
**Alternatives:**
- Single routingCapacityThreshold for both (original) — forces compromise between prevention (near-zero cost) and cure (real cost: HANDOFF, re-routing, grace periods)
- No per-channel thresholds — global only, less granular
**Rationale:** Routing exclusion (prevention) and redistribution filtering (cure) have different risk profiles — the global config already separates them (0.8 vs 0.85). Per-channel thresholds should mirror this separation. A high-urgency channel may want routing exclusion at 0.7 (stop sending new work early) but redistribution at 0.85 (only move work when genuinely necessary).
**Trade-offs:** Two nullable columns instead of one, two migration columns. Worth the granularity for architectural consistency with the global-level separation.
**Sources:** Design spec Layer 1 (OVERLOADED probe), Layer 3 (executor filtering), global config keys (casehub.eidos.routing.default-capacity-threshold, casehub.capacity.redistribution.redistribute-threshold)
**Exploration:** quick → revised by reviewer challenge
**Depends on:** D1
**Status:** revised (R1-02: separated routing exclusion from redistribution filtering per-channel)

## D5: Executor-centric per-channel filtering (Approach B)

**Choice:** Policy stays actor-level pure function; executor filters obligations by channel's redistributionCapacityThreshold
**Alternatives:**
- Channel-aware policy (A) — policy injects stores, resolves channels internally. Violates SPI intent, mixes store queries with decision logic
- Threshold as pressure modifier (C) — non-linear mapping, hits CapacitySignal validation bounds, hard to debug
**Rationale:** RedistributionPolicy.evaluate() returns a single decision per actor — it can't express "redistribute some obligations but not others." Per-channel filtering belongs in the executor where per-obligation iteration already happens. Policy answers "what kind of action?", executor answers "which obligations qualify?"
**Trade-offs:** Decision logic split across policy (thresholds) and executor (channel filtering). Same split pattern already exists — executor handles grace periods and routing-failure escalation that policy doesn't know about. Executor must handle zero-qualifying edge case (post-filter set empty when pre-filter set is non-empty) — log and fall through to compress or escalate.
**Sources:** RedistributionPolicy SPI contract (single return type), RedistributionDelegate.redistribute() per-obligation loop, first-principles granularity analysis
**Exploration:** deep-analysis
**Depends on:** D1, D4
**Status:** captured

## D6: DefaultRedistributionPolicy becomes @DefaultBean upstream

**Choice:** DefaultRedistributionPolicy annotated with @DefaultBean @ApplicationScoped. Consumers provide their own @ApplicationScoped implementation to override — no @Alternative needed.
**Alternatives:**
- @Alternative @Priority(1) in qhorus (original) — fragile, collision-prone (@Priority(1) is easily overridden by any @Alternative with @Priority(2)+), poor discoverability
- Both — file platform issue, use @Alternative locally — unnecessary if upstream change lands with D1 enrichment
**Rationale:** @DefaultBean is the established platform pattern for SPI fallback implementations (engine @DefaultBean SPI spec, 2026-05-14). Makes the intent self-documenting: "I am a fallback." Consumer overrides require only @ApplicationScoped — no @Alternative, no @Priority, no quarkus.arc.selected-alternatives config. Combined with D1 enrichment in the same cross-repo PR.
**Trade-offs:** Requires upstream change to casehub-platform. Trivial alongside D1 enrichment.
**Sources:** 2026-05-14-defaultbean-spi-noops-design.md (engine precedent), Quarkus Arc @DefaultBean semantics
**Exploration:** quick → revised by reviewer challenge
**Depends on:** D1
**Status:** revised (R1-03 + R1-08: @DefaultBean replaces @Alternative, eliminating collision risk and incorrect FixedCurrentPrincipal precedent)

## D7: Watchdog containment supersedes redistribution — no active coordination

**Choice:** No coordination mechanism between CONTEXT_PRESSURE watchdog and CapacityPressureMonitor/redistribution executor. Watchdog containment actions take natural precedence via severity ordering.
**Alternatives:**
- Suppress watchdog containment when redistribution is active — adds coupling between independent systems, requires shared state
- Ordered execution: redistribute first, watchdog evaluates residual state — requires scheduler coordination between independent @Scheduled drivers
- Document double-action as expected and harmless — accurate but imprecise about the interaction model
**Rationale:** The two systems operate at different severity levels. CapacityPressureMonitor sweeps at compress-threshold (0.7) and fires CapacityPressureEvent for automated redistribution. The watchdog fires at configured condition thresholds (typically higher) for human alerting and containment (PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE). Natural ordering: redistribution fires first (lower threshold), reduces pressure, watchdog never fires. If pressure is extreme, watchdog containment fires — the executor's per-obligation try-catch in RedistributionDelegate handles HANDOFF failures from paused channels gracefully (LOG.warnf + continue). Guard #2 escalates if zero HANDOFFs succeed. Containment at extreme pressure is the correct response — redistribution is moot for a quarantined agent.
**Trade-offs:** Theoretical race where both fire simultaneously on the same sweep cycle. Harmless — executor degrades gracefully, and watchdog containment is the correct response at extreme pressure levels.
**Sources:** CapacityPressureMonitor.sweep() (platform, sweeps at compress-threshold), WatchdogConditionType.CONTEXT_PRESSURE (qhorus), RedistributionDelegate.redistribute() try-catch pattern
**Exploration:** implicit decision surfaced by reviewer
**Status:** captured (R1-06)

## D8: Redistribution scope — move all redistributable obligations

**Choice:** Executor moves ALL redistributable obligations (those with non-null capabilityTag). Full evacuation rather than graduated.
**Alternatives:**
- Graduated: move obligations until pressure drops below threshold — requires synchronous pressure re-check between HANDOFFs, which shows stale values because context window doesn't shrink until LLM processes summary
- Fixed fraction (e.g., 50% of redistributable, oldest first) — arbitrary cut-off, may under-redistribute
- Per-obligation pressure attribution: estimate each obligation's contribution to context pressure, move enough to cover the delta — requires per-channel context tracking not available in current architecture
**Rationale:** Pressure feedback is delayed — after a HANDOFF, ActorCapacityView.getCapacity() returns the same pressure because context hasn't been freed yet. A graduated "move until pressure drops" approach can't work synchronously without blocking the @ObservesAsync thread (rejected in platform D7). Sweep-based re-evaluation (60s) handles the outcome: if evacuation was excessive (agent now idle), it receives new work via normal routing. Under-redistribution is worse — the agent stays overloaded for another full sweep cycle.
**Trade-offs:** May disrupt obligations unnecessarily if only a few moves would have sufficed. Evacuated agent becomes idle while targets absorb load — CIRCULAR_DELEGATION watchdog catches cascade effects after the fact. Graduated approach should be revisited when per-obligation pressure attribution is available.
**Sources:** RedistributionDelegate.redistribute() (moves all with non-null capabilityTag), platform D5 ("executors can iterate until pressure drops"), platform D7 (sweep-based re-evaluation)
**Exploration:** implicit decision surfaced by reviewer
**Status:** captured (R1-07)

## D9: Aggregation strategy — max-pressure across signal types

**Choice:** Max-pressure aggregation across all CapacitySignalSource implementations. Already decided at platform level (platform D3): any single saturated dimension is a redistribution trigger.
**Alternatives:**
- Weighted average — misleading when one dimension is saturated (0.9 context + 0.1 commitments = 0.5 average, but agent can't process new work)
- Domain-priority ordering — adds configuration complexity without clear pre-release benefit
- Configurable strategy (max / weighted / domain-priority) — premature for the first version
**Rationale:** With CommitmentCountCapacitySource added (D2 revision), max-pressure means an agent at 0.9 commitment pressure but 0.3 context pressure is at 0.9 aggregate. This is correct for routing prevention — an agent near its obligation limit should not receive new work regardless of context window headroom. The urgency semantics differ between signal types (context saturation = physically unable to process vs commitment count = nearing capacity limit), but max-pressure is the safe default.
**Trade-offs:** May trigger routing exclusion from commitment count alone when the agent can still physically process work. Acceptable — prevention is cheaper than redistribution. Revisit if signal sources grow significantly or if commitment-only exclusion causes routing starvation in practice.
**Sources:** Platform D3 (max-pressure, hardcoded, not configurable), AggregatingActorCapacityView.getCapacity() implementation
**Exploration:** implicit decision surfaced by reviewer
**Status:** captured (R1-09)

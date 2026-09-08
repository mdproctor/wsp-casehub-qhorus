## D1: Qhorus provides its own RedistributionPolicy

**Choice:** QhorusRedistributionPolicy as @Alternative @Priority(1) overriding platform's DefaultRedistributionPolicy
**Alternatives:**
- Configure platform defaults — accepts 0.95+ escalation gap, executor compensates partially
- Enrich platform policy — upstream obligation/inactivity awareness, slower to land
**Rationale:** Platform's DefaultRedistributionPolicy maps >=0.95 to Escalate (not immediate Redistribute), ignores obligation count and inactivity. Qhorus needs the design spec's richer decision table. @Alternative @Priority(1) avoids cross-repo dependency.
**Trade-offs:** Platform's DefaultRedistributionPolicy is @ApplicationScoped not @DefaultBean — if platform later adds @DefaultBean, qhorus can migrate from @Alternative to plain @ApplicationScoped
**Sources:** platform DefaultRedistributionPolicy.java (threshold-only, no obligation/inactivity logic), design spec Layer 2 decision table
**Exploration:** quick
**Status:** captured

## D2: Context pressure signal only — no commitment-count source

**Choice:** Stick with ContextPressureCapacitySource; commitment count available via RedistributionContext.openObligationCount
**Alternatives:**
- CommitmentCountCapacitySource — maps obligations/max to pressure, needs per-agent max config
- Defer to follow-up — separate issue for additional signal sources
**Rationale:** Commitment count is already part of RedistributionContext, duplicating it as a separate CapacitySignalSource adds complexity for minimal benefit
**Trade-offs:** Less granular aggregate pressure (single signal type), but obligation count still influences policy decisions
**Sources:** CapacitySignalSource SPI, RedistributionContext.openObligationCount field
**Exploration:** quick
**Status:** captured

## D3: Add minimal MCP tool set for capacity observability

**Choice:** get_actor_capacity, list_overloaded_actors, get_redistribution_history (ledger query for sender system:redistribution)
**Alternatives:**
- No MCP tools — capacity is infrastructure-only
- Defer to follow-up — separate issue
**Rationale:** LLM agents need visibility into the capacity system to understand why work moves and who is overloaded
**Trade-offs:** Three new @Tool methods; redistribution history uses existing ledger data (HANDOFF entries from system:redistribution), no new storage
**Sources:** QhorusMcpTools pattern, MessageLedgerEntryRepository queries
**Exploration:** quick
**Status:** captured

## D4: Per-channel threshold controls both routing exclusion and redistribution filtering

**Choice:** Channel.routingCapacityThreshold used by eidos RAS (prevention) and executor (redistribution filtering)
**Alternatives:**
- Routing exclusion only — redistribution thresholds global, simpler but less granular
- Skip per-channel — global thresholds only
**Rationale:** Different channels have different urgency levels; a low-latency channel needs earlier redistribution than a batch channel
**Trade-offs:** Threshold evaluated in two places (eidos for routing, executor for redistribution); same threshold, different purposes
**Sources:** design spec Layer 1 (SelectionContext + OVERLOADED probe), design spec Layer 3 (executor)
**Exploration:** quick
**Status:** captured

## D5: Executor-centric per-channel filtering (Approach B)

**Choice:** Policy stays actor-level pure function; executor filters obligations by channel threshold
**Alternatives:**
- Channel-aware policy (A) — policy injects stores, resolves channels internally. Violates SPI intent, mixes store queries with decision logic
- Threshold as pressure modifier (C) — non-linear mapping, hits CapacitySignal validation bounds, hard to debug
**Rationale:** RedistributionPolicy.evaluate() returns a single decision per actor — it can't express "redistribute some obligations but not others." Per-channel filtering belongs in the executor where per-obligation iteration already happens. Policy answers "what kind of action?", executor answers "which obligations qualify?"
**Trade-offs:** Decision logic split across policy (thresholds) and executor (channel filtering). Same split pattern already exists — executor handles grace periods and routing-failure escalation that policy doesn't know about.
**Sources:** RedistributionPolicy SPI contract (single return type), RedistributionDelegate.redistribute() per-obligation loop, first-principles granularity analysis
**Exploration:** deep-analysis
**Depends on:** D1, D4
**Status:** captured

## D6: @Alternative @Priority(1) for CDI override pattern

**Choice:** QhorusRedistributionPolicy uses @Alternative @Priority(1) to override platform's DefaultRedistributionPolicy
**Alternatives:**
- Upstream @DefaultBean to platform — cleaner but requires cross-repo PR first
- Both — file platform issue, use @Alternative locally. Migrate later.
**Rationale:** Self-contained, no cross-repo dependency. Platform can adopt @DefaultBean pattern later.
**Trade-offs:** @Alternative @Priority is less discoverable than @DefaultBean replacement; requires quarkus.arc.selected-alternatives or @Priority to activate
**Sources:** Quarkus CDI priority ladder, existing qhorus @Alternative patterns (FixedCurrentPrincipal)
**Exploration:** quick
**Depends on:** D1
**Status:** captured

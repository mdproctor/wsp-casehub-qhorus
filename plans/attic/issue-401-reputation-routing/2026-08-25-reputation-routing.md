# Reputation-Aware Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #401 — E4: Reputation-aware routing
**Issue group:** #401

**Goal:** Bridge qhorus's `MessageService.dispatch()` to the platform's `AgentRoutingStrategy` SPI so `role:X` capability targets resolve to the best available agent using trust scores.

**Architecture:** A `RoutingBridge` bean detects `role:` prefixed targets at dispatch time, builds `AgentRoutingContext` + `List<AgentCandidate>` from qhorus types, delegates to the platform's `AgentRoutingStrategy.select()`, and maps the result to a resolved instanceId or rejection. A `@DefaultBean` fallback provides simple highest-trust routing when the engine isn't on the classpath.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-api routing SPIs, casehub-ledger trust services

## Global Constraints

- `casehub-api` routing SPIs (`AgentRoutingStrategy`, `AgentRoutingContext`, `AgentCandidate`, `RoutingResult`) are compile dependencies via `casehub-platform-api` → `casehub-api` transitive chain
- `TrustGateService` is in `casehub-ledger` runtime (already a qhorus dependency)
- Flyway domain migrations: V46 (channel), V2003 (ledger subclass)
- MCP tool names: `snake_case` per platform convention
- CDI-free unit tests must set `tracingConfig` to disabled (per CLAUDE.md)

---

## Batch 1: Foundation — RoutingBridge + DefaultBean + data model

### Task 1: RoutingRejectedException + Channel routing field + migrations

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/message/RoutingRejectedException.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — add `routingTrustThreshold` field
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java` — add builder setter
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` — add column
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` — add routing section
- Create: `runtime/src/main/resources/db/qhorus/migration/V46__channel_routing.sql`
- Create: `runtime/src/main/resources/db/qhorus/migration/V2003__ledger_routing_metadata.sql`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java` — four routing columns
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingBridgeTest.java` (partial — exception tests)

**Interfaces:**
- Produces: `RoutingRejectedException(String reason)` — unchecked exception
- Produces: `Channel.routingTrustThreshold()` — `Double` (nullable)
- Produces: `QhorusConfig.Routing.defaultTrustThreshold()` — `Optional<Double>`
- Produces: `MessageLedgerEntry.routingOriginalTarget()`, `.routingSelectedAgent()`, `.routingStrategy()`, `.routingCandidateCount()` — all nullable

- [ ] **Step 1: Create RoutingRejectedException**

```java
package io.casehub.qhorus.api.message;

public class RoutingRejectedException extends RuntimeException {
    public RoutingRejectedException(String reason) {
        super(reason);
    }
}
```

- [ ] **Step 2: Add routingTrustThreshold to Channel record**

Add field `Double routingTrustThreshold` after `enforcementExclusions`. Add to the canonical constructor, `toBuilder()`, and `Builder`. Add backward-compatible constructor that defaults it to `null`. Update `fromRequest()` to read from `ChannelCreateRequest`.

- [ ] **Step 3: Add routingTrustThreshold to ChannelCreateRequest**

Add `routingTrustThreshold(Double v)` setter to builder. Default: `null`.

- [ ] **Step 4: Add column to ChannelEntity**

Add `@Column(name = "routing_trust_threshold") Double routingTrustThreshold` to `ChannelEntity`. Map in `toDomain()` and `fromDomain()`.

- [ ] **Step 5: Add routing config to QhorusConfig**

```java
interface Routing {
    @WithDefault("0.0")
    double defaultTrustThreshold();
}
```

Add `Routing routing()` to `QhorusConfig`.

- [ ] **Step 6: Create V46 migration**

```sql
ALTER TABLE channel ADD COLUMN routing_trust_threshold DOUBLE PRECISION;
```

- [ ] **Step 7: Add four routing columns to MessageLedgerEntry**

Add nullable fields: `routingOriginalTarget` (String), `routingSelectedAgent` (String), `routingStrategy` (String), `routingCandidateCount` (Integer). Map with `@Column`.

- [ ] **Step 8: Create V2003 migration**

```sql
ALTER TABLE message_ledger_entry ADD COLUMN routing_original_target VARCHAR(255);
ALTER TABLE message_ledger_entry ADD COLUMN routing_selected_agent VARCHAR(255);
ALTER TABLE message_ledger_entry ADD COLUMN routing_strategy VARCHAR(100);
ALTER TABLE message_ledger_entry ADD COLUMN routing_candidate_count INTEGER;
```

- [ ] **Step 9: Compile and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add api/ runtime/src/main/
git commit -m "feat(#401): Channel routing field + ledger metadata + V46/V2003 migrations Refs #401"
```

---

### Task 2: RoutingBridge + SimpleHighestTrustRoutingStrategy

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/RoutingBridge.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/SimpleHighestTrustRoutingStrategy.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingBridgeTest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/SimpleHighestTrustRoutingStrategyTest.java`

**Interfaces:**
- Consumes: `AgentRoutingStrategy.select(AgentRoutingContext, List<AgentCandidate>)` — platform SPI
- Consumes: `InstanceService.findByCapability(String)` — returns `List<Instance>`
- Consumes: `TrustGateService.currentScore(String actorId)` — returns trust score
- Consumes: `Channel.routingTrustThreshold()`, `QhorusConfig.Routing.defaultTrustThreshold()`
- Consumes: `RoutingRejectedException` from Task 1
- Produces: `RoutingBridge.resolve(MessageDispatch, Channel)` → `RoutingBridge.RoutingOutcome(String resolvedTarget, String originalTarget, String strategyName, int candidateCount)`
- Produces: `SimpleHighestTrustRoutingStrategy.select()` → `RoutingResult`

- [ ] **Step 1: Write RoutingBridge unit tests**

CDI-free. Mock `AgentRoutingStrategy`, `InstanceService`, `TrustGateService`, `QhorusConfig`. Tests:
1. `resolve_nullTarget_returnsNull` — passthrough
2. `resolve_nonRoleTarget_returnsUnchanged` — passthrough for `"agent-007"`
3. `resolve_rolePrefix_delegatesToStrategy` — `"role:analyst"` extracts capability, builds context, calls strategy
4. `resolve_strategyReturnsSelected_returnsExecutorId` — maps Selected to instanceId
5. `resolve_strategyReturnsUnresolvable_throwsRoutingRejectedException`
6. `resolve_strategyReturnsEscalated_throwsRoutingRejectedException`
7. `resolve_buildsCorrectAgentRoutingContext` — verifies caseId=channelId, capabilityName, tenancyId, caseContext=content
8. `resolve_buildsCorrectAgentCandidates` — verifies workerId, capabilities, health mapping (online→READY, stale→UNAVAILABLE)
9. `resolve_filtersUnavailableCandidates` — stale/offline instances excluded from candidate list

- [ ] **Step 2: Run tests — verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=RoutingBridgeTest -pl runtime`
Expected: FAIL — `RoutingBridge` class does not exist

- [ ] **Step 3: Implement RoutingBridge**

```java
@ApplicationScoped
public class RoutingBridge {
    private final AgentRoutingStrategy routingStrategy;
    private final InstanceService instanceService;
    private final InstanceStore instanceStore;
    private final TrustGateService trustGateService;
    private final QhorusConfig config;
    private final ObjectMapper objectMapper;

    // Constructor injection

    public record RoutingOutcome(String resolvedTarget, String originalTarget,
                                  String strategyName, int candidateCount) {}

    public RoutingOutcome resolve(MessageDispatch dispatch, Channel channel, String tenancyId) {
        String target = dispatch.target();
        if (target == null || !target.startsWith("role:")) {
            return null;  // no routing needed
        }
        String capability = target.substring("role:".length());
        // Build AgentRoutingContext, build candidates, call strategy, map result
    }
}
```

- [ ] **Step 4: Run tests — verify GREEN**

- [ ] **Step 5: Write SimpleHighestTrustRoutingStrategy unit tests**

CDI-free. Tests:
1. `select_emptyCandidates_returnsUnresolvable`
2. `select_singleCandidate_returnsSelected`
3. `select_multipleCandidates_returnsHighestScore`
4. `select_allBelowThreshold_returnsUnresolvable` (when threshold passed via context)

- [ ] **Step 6: Run tests — verify RED**

- [ ] **Step 7: Implement SimpleHighestTrustRoutingStrategy**

```java
@DefaultBean
@ApplicationScoped
public class SimpleHighestTrustRoutingStrategy implements AgentRoutingStrategy {
    private final TrustGateService trustGateService;

    @Override
    public String id() { return "simple-highest-trust"; }

    @Override
    public RoutingResult select(AgentRoutingContext context, List<AgentCandidate> candidates) {
        // Filter by threshold, pick highest score, return Selected or Unresolvable
    }
}
```

- [ ] **Step 8: Run tests — verify GREEN**

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/ runtime/src/test/
git commit -m "feat(#401): RoutingBridge + SimpleHighestTrustRoutingStrategy with 13 unit tests Refs #401"
```

---

## Batch 2: Integration — wire into dispatch pipeline + ledger

### Task 3: Wire RoutingBridge into MessageService.dispatch() + ledger recording

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — inject RoutingBridge, call resolve() after rate limiter, replace target
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java` — record routing metadata
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageDispatch.java` (if needed — routing metadata carrier)
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingIntegrationTest.java`

**Interfaces:**
- Consumes: `RoutingBridge.resolve()` from Task 2
- Consumes: `MessageLedgerEntry` routing fields from Task 1

- [ ] **Step 1: Write integration test**

`@QuarkusTest`. Register two agents with capability `"analyst"` and different trust scores. Dispatch a COMMAND with `target: "role:analyst"`. Verify:
1. Message stored with resolved instanceId (not `role:analyst`)
2. Commitment opened against the resolved agent
3. Ledger entry has routing metadata (originalTarget, selectedAgent, strategyName, candidateCount)

- [ ] **Step 2: Run test — verify RED**

- [ ] **Step 3: Wire RoutingBridge into MessageService.dispatch()**

After the rate limiter check and before the ObligorTrustPolicy check, add:

```java
RoutingBridge.RoutingOutcome routingOutcome = null;
if (ch != null) {
    routingOutcome = routingBridge.resolve(dispatch, ch, effectiveTenancyId);
    if (routingOutcome != null) {
        dispatch = dispatch.withTarget(routingOutcome.resolvedTarget());
    }
}
```

Add `MessageDispatch.withTarget(String)` — returns a copy with the target replaced. Immutable pattern.

- [ ] **Step 4: Wire routing metadata into LedgerWriteService.record()**

Pass routing outcome through to `LedgerWriteService`. Set the four routing fields on `MessageLedgerEntry` when routing outcome is non-null.

- [ ] **Step 5: Run integration test — verify GREEN**

- [ ] **Step 6: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All existing tests pass (routing is a no-op for non-`role:` targets)

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/ runtime/src/test/
git commit -m "feat(#401): wire RoutingBridge into dispatch pipeline + ledger recording Refs #401"
```

---

## Batch 3: MCP tools + REST + CLAUDE.md

### Task 4: MCP tools + REST endpoint + CLAUDE.md

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — three new @Tool methods
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java` — PUT routing config
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java` — add routing field
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` — setRoutingConfig method
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/RoutingMcpToolsTest.java`
- Modify: `CLAUDE.md` — document routing bridge

**Interfaces:**
- Consumes: `RoutingBridge.resolve()` from Task 2, `ChannelService` from existing
- Produces: `set_routing_config(channel, trust_threshold)`, `get_routing_config(channel)`, `get_routing_candidates(capability, channel?)`

- [ ] **Step 1: Write MCP tool tests**

CDI-free unit tests extending `QhorusMcpToolsBase` pattern:
1. `setRoutingConfig_setsThreshold` — sets threshold, verifies channel updated
2. `getRoutingConfig_returnsConfig` — returns channel threshold + resolved global default
3. `getRoutingCandidates_listsAgents` — lists matching agents with scores and health
4. `getRoutingCandidates_withChannel_filtersbyThreshold` — channel threshold applied

- [ ] **Step 2: Run tests — verify RED**

- [ ] **Step 3: Add setRoutingConfig to ChannelService**

```java
@Transactional
public void setRoutingConfig(UUID channelId, Double trustThreshold) {
    // Update channel routing_trust_threshold
}
```

- [ ] **Step 4: Implement three MCP tools**

`set_routing_config`, `get_routing_config`, `get_routing_candidates` following existing `@Tool` patterns. Channel resolution via `resolveChannel()` (UUID or name).

- [ ] **Step 5: Add REST endpoint**

`PUT /api/channels/{id}/routing` on `ChannelResource`. Accepts `{"trustThreshold": 0.7}`. Delegates to `ChannelService.setRoutingConfig()`.

- [ ] **Step 6: Run tests — verify GREEN**

- [ ] **Step 7: Update CLAUDE.md**

Document `RoutingBridge`, `SimpleHighestTrustRoutingStrategy`, `role:` prefix convention, config property, MCP tools, V46/V2003 migrations.

- [ ] **Step 8: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: All modules green

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/ runtime/src/test/ CLAUDE.md
git commit -m "feat(#401): routing MCP tools + REST endpoint + CLAUDE.md Refs #401"
```

---

## References

- [2026-08-25-reputation-routing-design.md] — design spec
- [decisions.md] — 9 design decisions (revised after platform audit)
- `io.casehub.api.spi.routing.AgentRoutingStrategy` — platform routing SPI
- `io.casehub.engine.internal.routing.ComposableAgentRoutingStrategy` — engine default
- `io.casehub.ledger.routing.TrustCandidateClassifier` — trust maturity model
- `docs/platform/routing.md` — four-layer routing architecture
- Garden GE-20260616-17187e — TrustGateService delegation chain
- Garden GE-20260804-565c2c — ActorTrustScoreRepository cross-PU lookup
- Garden GE-20260625-aaf3d4 — JPA L1 cache staleness with requiringNew()
- GitHub #401 — E4 epic

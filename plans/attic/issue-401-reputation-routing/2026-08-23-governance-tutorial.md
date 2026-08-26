# Governance Tutorial Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Each task follows TDD and uses
> ide-tooling for structural editing. Steps use checkbox (`- [ ]`) syntax.

**Focal issue:** #407 — Tutorial example: governance in action
**Issue group:** #398, #399, #400, #407

**Goal:** Build a CausalGraphRenderer text utility + MCP tool, and a tutorial example module with 3 scenarios covering all Phase 1 governance features.

**Architecture:** CausalGraphRenderer is a static utility in runtime that converts CausalGraph (nodes+edges) to indented tree text. Exposed via `render_causal_graph` MCP tool. The tutorial module (`examples/governance-tutorial/`) is a Quarkus test module (no LLM, CI-safe) with three @QuarkusTest classes exercising causal graphs, containment, and enforcement.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, InMemory stores, H2

## Global Constraints

- Example module follows type-system pattern: pom.xml deps, application.properties, import-qhorus-test.sql
- No LLM, deterministic, CI-safe
- Text renderer is a pure function — no CDI, no state
- Must register in parent examples/pom.xml

---

## Batch 1: CausalGraphRenderer + MCP tool

### Task 1: CausalGraphRenderer + render_causal_graph MCP tool

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CausalGraphRenderer.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — add `render_causal_graph` tool
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/CausalGraphRendererTest.java`

**Interfaces:**
- Produces: `CausalGraphRenderer.render(CausalGraph)` → `String`
- Produces: `render_causal_graph(correlation_id)` MCP tool

- [ ] **Step 1: Write test for CausalGraphRenderer**

CDI-free test with constructed CausalGraph instances. Test cases:
- Single-node graph (root COMMAND only)
- Linear chain (COMMAND → STATUS → DONE, single channel)
- Cross-channel tree (COMMAND → HANDOFF → DONE across 2 channels)
- Truncated graph (truncated=true flag shown)
- Empty graph (no nodes)

Assertions on output structure: header line with correlation/outcome/duration, indented tree with channel tags, message types, actor IDs, content snippets, depth, elapsed time.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement CausalGraphRenderer**

Static `render(CausalGraph graph)` method. Algorithm:
1. Header: `Causal Graph — correlation {id} ({outcome}, {duration}s, {N} channels)`
2. Build parent→children map from edges
3. Walk tree from root (depth=0 node), render each node as:
   `[channelName] TYPE  actorId  "content"  depth=N  +elapsed`
4. Tree connectors: `├─` for non-last children, `└─` for last, `│   ` for continuation

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Add render_causal_graph MCP tool**

```java
@Tool(name = "render_causal_graph",
      description = "Render a correlation ID's causal graph as readable indented text. "
                  + "Shows cross-channel attribution tree with types, actors, content, timing.")
public String renderCausalGraph(
        @ToolArg(name = "correlation_id", description = "Correlation ID to trace") String correlationId,
        @ToolArg(name = "limit", description = "Max entries to include", required = false) Integer limit) {
    var graph = causalGraphService.buildGraph(correlationId,
            limit != null ? limit : 200, currentPrincipal.tenancyId());
    return CausalGraphRenderer.render(graph);
}
```

- [ ] **Step 6: Run full runtime tests**

- [ ] **Step 7: Commit**

---

## Batch 2: Tutorial example module + 3 scenarios

### Task 2: Module setup + Scenario 1 (causal tracing)

**Files:**
- Create: `examples/governance-tutorial/pom.xml`
- Modify: `examples/pom.xml` — add governance-tutorial module
- Create: `examples/governance-tutorial/src/test/resources/application.properties`
- Create: `examples/governance-tutorial/src/test/resources/import-qhorus-test.sql`
- Create: `examples/governance-tutorial/src/test/java/io/casehub/qhorus/examples/governance/CausalTracingScenarioTest.java`

**Interfaces:**
- Consumes: QhorusMcpTools (create_channel, register, send_message, get_causal_graph, render_causal_graph), CausalGraphRenderer

- [ ] **Step 1: Create module structure**

pom.xml matching type-system pattern. application.properties with InMemory stores. import-qhorus-test.sql for ledger table. Register in parent pom.

- [ ] **Step 2: Write Scenario 1 — cross-channel causal tracing**

@QuarkusTest test with @TestTransaction:
1. Create work + ops channels
2. Register agent-a and agent-b
3. Dispatch COMMAND on work channel (agent-a → "Analyze the dataset", correlationId=corrId)
4. Dispatch HANDOFF on work channel (agent-a → target="agent-b", same corrId)
5. Dispatch DONE on ops channel (agent-b → "Analysis complete", same corrId)
6. Call `get_causal_graph(corrId)` — assert nodes.size()=3, edges.size()=2, channelCount=2
7. Call `render_causal_graph(corrId)` — assert output contains channel names, all three message types, both actors
8. Print rendered output to stdout (the text IS the tutorial)

- [ ] **Step 3: Run test, verify it passes**

- [ ] **Step 4: Commit**

---

### Task 3: Scenario 2 (containment) + Scenario 3 (enforcement)

**Files:**
- Create: `examples/governance-tutorial/src/test/java/io/casehub/qhorus/examples/governance/ContainmentScenarioTest.java`
- Create: `examples/governance-tutorial/src/test/java/io/casehub/qhorus/examples/governance/EnforcementScenarioTest.java`

- [ ] **Step 1: Write Scenario 2 — cascade containment**

@QuarkusTest test:
1. Create channel, register agent
2. Register LOOP_DETECTED watchdog with QUARANTINE action, low similarity threshold
3. Send repeated similar messages from same sender to trigger loop detection
4. Call `watchdogService.evaluateAll()` directly
5. Assert: channel is paused, containment EVENT in ledger with structured telemetry
6. Render and print causal context

- [ ] **Step 2: Write Scenario 3 — enforcement modes**

@QuarkusTest test:
1. Create channel with protocol REQUEST_RESPONSE
2. Set enforcement mode to BLOCKING
3. Open multiple QUERYs to exceed max-open-queries threshold
4. Attempt another QUERY — assert `EnforcementBlockedException` thrown (wrapped in ToolCallException)
5. Verify enforcement EVENT in channel ledger
6. Set enforcement mode to QUARANTINE
7. Trigger another violation — assert exception + channel paused
8. Verify QUARANTINE containment actions completed

- [ ] **Step 3: Run all tutorial tests**

- [ ] **Step 4: Run full build (all modules)**

- [ ] **Step 5: Update CLAUDE.md — add governance-tutorial module**

- [ ] **Step 6: Commit**

---

## References

- `specs/issue-398-roadmap-phase1/2026-08-23-enforcement-modes-design.md` — enforcement spec
- `specs/issue-398-roadmap-phase1/2026-08-21-causal-graphs-design.md` — causal graphs spec
- `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CausalGraphService.java` — graph builder
- `examples/type-system/pom.xml` — example module pattern
- `examples/type-system/src/test/resources/application.properties` — test config pattern
- GitHub #407, #398, #399, #400

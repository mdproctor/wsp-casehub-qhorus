# Cross-Channel Causal Graphs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #398 — E1: Cross-channel causal graphs
**Issue group:** #398

**Goal:** Add cross-channel causal graph tracing — a query layer over existing ledger data that walks `causedByEntryId` links across channel boundaries and builds structured graphs from `correlationId`-grouped entries.

**Architecture:** New `CausalGraphService` owns graph construction logic, consumed by both MCP tools and a new REST resource. Repository gains a cross-channel ancestor walk method. Existing `get_causal_chain` MCP tool becomes cross-channel-capable via optional `channel` parameter. New `get_causal_graph` MCP tool returns nodes+edges graph structure.

**Tech Stack:** Java 21, Quarkus 3.32.2, H2 (test), JPA/Hibernate, quarkus-mcp-server

## Global Constraints

- No new database tables or Flyway migrations
- No changes to `LedgerWriteService` or message dispatch
- `get_obligation_activity` unchanged — flat chronological list serves a different semantic
- Channel ACLs are write-access controls; cross-channel reads within a tenancy are unrestricted
- HANDOFF is non-terminal — aligned with existing `get_obligation_chain` semantics
- Depth limit: 50 hops on all ancestor chain walks
- `channelStore.findByIds()` for batch channel lookup — never `channelService.listAll()`

---

## Batch 1: Foundation — repository + service layer

### Task 1: Cross-channel ancestor walk + depth limit

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/StubMessageLedgerEntryRepository.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryRepoTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntry` entity (existing), `EntityManager` (existing)
- Produces: `findAncestorChainCrossChannel(UUID entryId, String tenancyId)` → `List<MessageLedgerEntry>`

- [ ] **Step 1: Write failing test — cross-channel three-hop chain**

In `LedgerQueryRepoTest`, add test that creates entries across two channels linked by `causedByEntryId`, calls `findAncestorChainCrossChannel`, asserts full chain returned oldest-first.

```java
@Test
void findAncestorChainCrossChannel_threeHops_crossesChannelBoundary() {
    UUID ch1 = UUID.randomUUID();
    UUID ch2 = UUID.randomUUID();
    String tid = TenancyConstants.DEFAULT_TENANT_ID;

    // root on channel 1
    MessageLedgerEntry root = entry(ch1, "COMMAND", "agent-a", null, tid);
    repo.save(root);

    // hop 1 on channel 1 — caused by root
    MessageLedgerEntry hop1 = entry(ch1, "HANDOFF", "agent-a", root.id, tid);
    repo.save(hop1);

    // hop 2 on channel 2 — caused by hop1 (crosses channel boundary)
    MessageLedgerEntry hop2 = entry(ch2, "DONE", "agent-b", hop1.id, tid);
    repo.save(hop2);

    List<MessageLedgerEntry> chain = repo.findAncestorChainCrossChannel(hop2.id, tid);

    assertThat(chain).hasSize(3);
    assertThat(chain.get(0).id).isEqualTo(root.id);
    assertThat(chain.get(1).id).isEqualTo(hop1.id);
    assertThat(chain.get(2).id).isEqualTo(hop2.id);
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerQueryRepoTest#findAncestorChainCrossChannel_threeHops_crossesChannelBoundary -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: compile error — `findAncestorChainCrossChannel` does not exist.

- [ ] **Step 3: Implement `findAncestorChainCrossChannel` on `MessageLedgerEntryRepository`**

Use `ide_insert_member` to add after `findAncestorChain`:

```java
public List<MessageLedgerEntry> findAncestorChainCrossChannel(
        final UUID entryId, final String tenancyId) {
    final List<MessageLedgerEntry> chain = new ArrayList<>();
    UUID currentId = entryId;
    final Set<UUID> visited = new HashSet<>();
    final String tid = tenancyId(tenancyId);
    int depth = 0;
    while (currentId != null && !visited.contains(currentId) && depth < 50) {
        visited.add(currentId);
        final MessageLedgerEntry entry = em.find(MessageLedgerEntry.class, currentId);
        if (entry == null || !tid.equals(entry.tenancyId)) {
            break;
        }
        chain.add(entry);
        currentId = entry.causedByEntryId;
        depth++;
    }
    Collections.reverse(chain);
    return chain;
}
```

- [ ] **Step 4: Add stub to `StubMessageLedgerEntryRepository`**

Use `ide_insert_member` to add:

```java
public List<MessageLedgerEntry> findAncestorChainCrossChannel(
        final UUID entryId, final String tenancyId) {
    return List.of();
}
```

- [ ] **Step 5: Add stub to `LedgerQueryRepoTest.CapturingRepo`**

Use `ide_insert_member` to add:

```java
public List<MessageLedgerEntry> findAncestorChainCrossChannel(
        final UUID entryId, final String tenancyId) {
    return delegate.findAncestorChainCrossChannel(entryId, tenancyId);
}
```

- [ ] **Step 6: Run test — verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerQueryRepoTest#findAncestorChainCrossChannel_threeHops_crossesChannelBoundary -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 7: Write additional tests — cycle, depth limit, unknown entry, tenancy**

```java
@Test
void findAncestorChainCrossChannel_cycleDetection() {
    UUID ch = UUID.randomUUID();
    String tid = TenancyConstants.DEFAULT_TENANT_ID;
    // Create a cycle: A → B → A
    MessageLedgerEntry a = entry(ch, "COMMAND", "agent-a", null, tid);
    repo.save(a);
    MessageLedgerEntry b = entry(ch, "STATUS", "agent-b", a.id, tid);
    repo.save(b);
    // Manually set a.causedByEntryId = b.id to create cycle
    a.causedByEntryId = b.id;
    em.merge(a);
    em.flush();

    List<MessageLedgerEntry> chain = repo.findAncestorChainCrossChannel(b.id, tid);
    // Should not infinite loop — visited set stops it
    assertThat(chain).hasSizeLessThanOrEqualTo(2);
}

@Test
void findAncestorChainCrossChannel_unknownEntry_returnsEmpty() {
    List<MessageLedgerEntry> chain = repo.findAncestorChainCrossChannel(
            UUID.randomUUID(), TenancyConstants.DEFAULT_TENANT_ID);
    assertThat(chain).isEmpty();
}

@Test
void findAncestorChainCrossChannel_tenancyBoundary_stops() {
    UUID ch = UUID.randomUUID();
    String tid1 = "tenant-1";
    String tid2 = "tenant-2";

    MessageLedgerEntry root = entry(ch, "COMMAND", "agent-a", null, tid1);
    repo.save(root);
    MessageLedgerEntry child = entry(ch, "DONE", "agent-b", root.id, tid2);
    repo.save(child);

    // Walking from child in tenant-2 should stop — root is in tenant-1
    List<MessageLedgerEntry> chain = repo.findAncestorChainCrossChannel(child.id, tid2);
    assertThat(chain).hasSize(1);
    assertThat(chain.get(0).id).isEqualTo(child.id);
}
```

- [ ] **Step 8: Add 50-hop depth limit to existing `findAncestorChain`**

Use `ide_replace_member` to update `findAncestorChain` — add `int depth = 0;` counter and `&& depth < 50` guard to the while condition, increment `depth++` inside the loop. Same pattern as the new method.

- [ ] **Step 9: Write regression test for existing `findAncestorChain` depth limit**

```java
@Test
void findAncestorChain_depthLimitApplied() {
    // Existing single-channel method should also respect 50-hop limit
    // (verify no infinite loop with deep chains)
    UUID ch = UUID.randomUUID();
    String tid = TenancyConstants.DEFAULT_TENANT_ID;
    MessageLedgerEntry root = entry(ch, "COMMAND", "agent-a", null, tid);
    repo.save(root);

    List<MessageLedgerEntry> chain = repo.findAncestorChain(ch, root.id, tid);
    assertThat(chain).hasSize(1); // root entry only, no infinite loop
}
```

- [ ] **Step 10: Run all LedgerQueryRepoTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerQueryRepoTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: all PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntryRepository.java runtime/src/test/java/io/casehub/qhorus/runtime/ledger/StubMessageLedgerEntryRepository.java runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryRepoTest.java
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus commit -m "feat(#398): cross-channel ancestor walk + depth limit on findAncestorChain. Refs #398"
```

---

### Task 2: CausalGraphService — DTOs + graph building

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CausalGraphService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/CausalGraphServiceTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.findByCorrelationIdAcrossChannels(String, int, String)`, `ChannelStore.findByIds(Collection<UUID>)`, `CurrentPrincipal.tenancyId()`
- Produces: `CausalGraphService.buildGraph(String correlationId, int limit, String tenancyId)` → `CausalGraph`; inner records `CausalGraph`, `GraphNode`, `GraphEdge`

- [ ] **Step 1: Write failing test — single linear chain graph**

CDI-free unit test using Mockito for `MessageLedgerEntryRepository` and `ChannelStore`.

```java
@Test
void buildGraph_linearChain_producesCorrectNodesAndEdges() {
    UUID ch1 = UUID.randomUUID();
    String corrId = UUID.randomUUID().toString();
    String tid = TenancyConstants.DEFAULT_TENANT_ID;
    Instant t0 = Instant.parse("2026-08-21T10:00:00Z");

    MessageLedgerEntry command = testEntry(ch1, "COMMAND", "agent-a", corrId, null, t0);
    MessageLedgerEntry done = testEntry(ch1, "DONE", "agent-b", corrId, command.id, t0.plusSeconds(5));

    when(repo.findByCorrelationIdAcrossChannels(corrId, 100, tid))
            .thenReturn(List.of(command, done));
    when(channelStore.findByIds(Set.of(ch1)))
            .thenReturn(List.of(channel(ch1, "work")));

    var graph = service.buildGraph(corrId, 100, tid);

    assertThat(graph.correlationId()).isEqualTo(corrId);
    assertThat(graph.rootEntryId()).isEqualTo(command.id.toString());
    assertThat(graph.channelCount()).isEqualTo(1);
    assertThat(graph.channels()).containsExactly("work");
    assertThat(graph.outcome()).isEqualTo("FULFILLED");
    assertThat(graph.totalDurationMs()).isEqualTo(5000L);
    assertThat(graph.truncated()).isFalse();
    assertThat(graph.nodes()).hasSize(2);
    assertThat(graph.edges()).hasSize(1);
    assertThat(graph.edges().get(0).from()).isEqualTo(command.id.toString());
    assertThat(graph.edges().get(0).to()).isEqualTo(done.id.toString());
    assertThat(graph.edges().get(0).elapsedMs()).isEqualTo(5000L);
    assertThat(graph.nodes().get(0).depth()).isEqualTo(0); // root
    assertThat(graph.nodes().get(1).depth()).isEqualTo(1);
}
```

- [ ] **Step 2: Run test — verify compile error**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: FAIL — `CausalGraphService` does not exist.

- [ ] **Step 3: Implement `CausalGraphService` with DTOs and graph algorithm**

Create new file at `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CausalGraphService.java`:

```java
package io.casehub.qhorus.runtime.ledger;

import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.runtime.channel.Channel;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

@ApplicationScoped
public class CausalGraphService {

    @Inject MessageLedgerEntryRepository ledgerRepo;
    @Inject ChannelStore channelStore;

    public record CausalGraph(
            String correlationId,
            String rootEntryId,
            int channelCount,
            List<String> channels,
            Long totalDurationMs,
            String outcome,
            boolean truncated,
            List<GraphNode> nodes,
            List<GraphEdge> edges) {}

    public record GraphNode(
            String entryId,
            String channelId,
            String channelName,
            String messageType,
            String actorId,
            String occurredAt,
            String content,
            String causedByEntryId,
            int depth) {}

    public record GraphEdge(
            String from,
            String to,
            String type,
            Long elapsedMs) {}

    private static final Set<String> TERMINAL_TYPES = Set.of("DONE", "FAILURE", "DECLINE");

    @Transactional
    public CausalGraph buildGraph(String correlationId, int limit, String tenancyId) {
        List<MessageLedgerEntry> entries =
                ledgerRepo.findByCorrelationIdAcrossChannels(correlationId, limit, tenancyId);

        if (entries.isEmpty()) {
            return new CausalGraph(correlationId, null, 0, List.of(), null, "OPEN",
                    false, List.of(), List.of());
        }

        boolean truncated = entries.size() >= limit;

        // Index by ID
        Map<UUID, MessageLedgerEntry> byId = new LinkedHashMap<>();
        for (MessageLedgerEntry e : entries) {
            byId.put(e.id, e);
        }

        // Build edges
        List<GraphEdge> edges = new ArrayList<>();
        for (MessageLedgerEntry e : entries) {
            if (e.causedByEntryId != null && byId.containsKey(e.causedByEntryId)) {
                MessageLedgerEntry parent = byId.get(e.causedByEntryId);
                Long elapsed = (e.occurredAt != null && parent.occurredAt != null)
                        ? e.occurredAt.toEpochMilli() - parent.occurredAt.toEpochMilli()
                        : null;
                edges.add(new GraphEdge(
                        parent.id.toString(), e.id.toString(), "CAUSED_BY", elapsed));
            }
        }

        // Root selection — earliest entry with null causedByEntryId
        MessageLedgerEntry root = entries.stream()
                .filter(e -> e.causedByEntryId == null || !byId.containsKey(e.causedByEntryId))
                .min(Comparator.comparing(e -> e.occurredAt != null ? e.occurredAt : Instant.MAX))
                .orElse(null);

        // BFS for depth
        Map<UUID, Integer> depthMap = new HashMap<>();
        if (root != null) {
            depthMap.put(root.id, 0);
            // Build children map
            Map<UUID, List<UUID>> children = new HashMap<>();
            for (MessageLedgerEntry e : entries) {
                if (e.causedByEntryId != null && byId.containsKey(e.causedByEntryId)) {
                    children.computeIfAbsent(e.causedByEntryId, k -> new ArrayList<>()).add(e.id);
                }
            }
            Queue<UUID> queue = new ArrayDeque<>();
            queue.add(root.id);
            while (!queue.isEmpty()) {
                UUID current = queue.poll();
                int currentDepth = depthMap.get(current);
                for (UUID childId : children.getOrDefault(current, List.of())) {
                    if (!depthMap.containsKey(childId)) {
                        depthMap.put(childId, currentDepth + 1);
                        queue.add(childId);
                    }
                }
            }
        }

        // Channel name resolution
        Set<UUID> channelIds = entries.stream()
                .map(e -> e.channelId)
                .collect(Collectors.toSet());
        Map<UUID, String> channelNames = channelStore.findByIds(channelIds).stream()
                .collect(Collectors.toMap(
                        ch -> ((Channel) ch).id(),
                        ch -> ((Channel) ch).name(),
                        (a, b) -> a));

        // Build nodes
        List<GraphNode> nodes = entries.stream()
                .map(e -> new GraphNode(
                        e.id.toString(),
                        e.channelId.toString(),
                        channelNames.getOrDefault(e.channelId, "unknown"),
                        e.messageType,
                        e.actorId,
                        e.occurredAt != null ? e.occurredAt.toString() : null,
                        e.content,
                        e.causedByEntryId != null ? e.causedByEntryId.toString() : null,
                        depthMap.getOrDefault(e.id, -1)))
                .toList();

        // Outcome — precedence: FAILED > DECLINED > FULFILLED
        List<MessageLedgerEntry> terminals = entries.stream()
                .filter(e -> TERMINAL_TYPES.contains(e.messageType))
                .toList();
        String outcome;
        if (terminals.isEmpty()) {
            outcome = "OPEN";
        } else if (terminals.stream().anyMatch(e -> "FAILURE".equals(e.messageType))) {
            outcome = "FAILED";
        } else if (terminals.stream().anyMatch(e -> "DECLINE".equals(e.messageType))) {
            outcome = "DECLINED";
        } else {
            outcome = "FULFILLED";
        }

        // Duration — root to latest terminal
        Long totalDurationMs = null;
        if (root != null && root.occurredAt != null && !terminals.isEmpty()) {
            Instant latestTerminal = terminals.stream()
                    .map(e -> e.occurredAt)
                    .filter(Objects::nonNull)
                    .max(Instant::compareTo)
                    .orElse(null);
            if (latestTerminal != null) {
                totalDurationMs = latestTerminal.toEpochMilli() - root.occurredAt.toEpochMilli();
            }
        }

        List<String> channelNameList = channelIds.stream()
                .map(id -> channelNames.getOrDefault(id, "unknown"))
                .sorted()
                .toList();

        return new CausalGraph(
                correlationId,
                root != null ? root.id.toString() : null,
                channelIds.size(),
                channelNameList,
                totalDurationMs,
                outcome,
                truncated,
                nodes,
                edges);
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CausalGraphServiceTest#buildGraph_linearChain_producesCorrectNodesAndEdges -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 5: Write additional tests — branching, zero roots, truncation, outcome precedence**

```java
@Test
void buildGraph_branchingDelegation_failedPrecedence() {
    // COMMAND → HANDOFF to B → DONE (branch 1)
    //         → HANDOFF to C → FAILURE (branch 2)
    // Outcome should be FAILED (precedence over FULFILLED)
    // ...
}

@Test
void buildGraph_zeroRoots_allUnlinked() {
    // All entries have causedByEntryId pointing outside the result set
    // rootEntryId should be null, all depths -1
    // ...
}

@Test
void buildGraph_truncated_flagSet() {
    // Result size equals limit → truncated = true
    // ...
}

@Test
void buildGraph_emptyCorrelation_emptyGraph() {
    when(repo.findByCorrelationIdAcrossChannels(any(), anyInt(), any()))
            .thenReturn(List.of());
    var graph = service.buildGraph("unknown", 100, tid);
    assertThat(graph.nodes()).isEmpty();
    assertThat(graph.outcome()).isEqualTo("OPEN");
}

@Test
void buildGraph_unlinkedEntries_depthMinusOne() {
    // STATUS entry with no causedByEntryId alongside a COMMAND root
    // STATUS gets depth -1
    // ...
}
```

- [ ] **Step 6: Run all CausalGraphServiceTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CausalGraphServiceTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CausalGraphService.java runtime/src/test/java/io/casehub/qhorus/runtime/ledger/CausalGraphServiceTest.java
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus commit -m "feat(#398): CausalGraphService with graph building algorithm. Refs #398"
```

---

## Batch 2: Surface — MCP tools + REST

### Task 3: Update `get_causal_chain` + new `get_causal_graph` MCP tools

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — update `CausalChainEntry` record
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — update `getCausalChain`, add `getCausalGraph`
- Modify: `runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryToolsTest.java`

**Interfaces:**
- Consumes: `MessageLedgerEntryRepository.findAncestorChainCrossChannel(UUID, String)` (Task 1), `MessageLedgerEntryRepository.findAncestorChain(UUID, UUID, String)` (existing), `CausalGraphService.buildGraph(String, int, String)` (Task 2), `ChannelStore.findByIds(Collection<UUID>)` (existing)
- Produces: `getCausalChain(String entryId, String channel)` MCP tool (channel now optional), `getCausalGraph(String correlationId, Integer limit)` MCP tool

- [ ] **Step 1: Update `CausalChainEntry` record — add channelId, channelName, content**

Use `ide_replace_member` on `CausalChainEntry` in `QhorusMcpToolsBase.java` to add the three new fields:

```java
record CausalChainEntry(
        String entryId,
        String channelId,
        String channelName,
        String messageType,
        String actorId,
        String correlationId,
        String occurredAt,
        String content,
        String causedByEntryId) {}
```

- [ ] **Step 2: Update all existing `CausalChainEntry` construction sites**

Use `ide_find_references` on `CausalChainEntry` to find all construction sites. Update each to include the new fields. The existing `getCausalChain` creates entries at `QhorusMcpTools.java:1895-1901` — add `e.channelId.toString()`, channel name resolution, and `e.content`.

- [ ] **Step 3: Make `channel` optional on `getCausalChain`**

Use `ide_replace_member` on `getCausalChain` in `QhorusMcpTools.java`. Change `@ToolArg(name = "channel", ...)` to add `required = false`. Add conditional logic: when channel is null, call `findAncestorChainCrossChannel`; when provided, call existing `findAncestorChain`.

For the cross-channel path, batch-load channel names using `channelStore.findByIds()` for the unique `channelId` values in the chain.

For the single-channel path, use the resolved channel's name for all entries.

- [ ] **Step 4: Add `getCausalGraph` MCP tool**

Use `ide_insert_member` after `getCausalChain` in `QhorusMcpTools.java`:

```java
@Tool(name = "get_causal_graph", description = "Build a cross-channel causal graph for a correlation_id. "
        + "Returns nodes (ledger entries with channel, type, actor, depth) and edges (causedByEntryId links with elapsed time). "
        + "Edges are derived from causedByEntryId links — for complete cross-channel graphs, agents must pass "
        + "caused_by_entry_id when sending messages that continue delegations from other channels. "
        + "Cross-tenant delegation traces stop at the tenant boundary.")
@Transactional
public CausalGraphService.CausalGraph getCausalGraph(
        @ToolArg(name = "correlation_id", description = "Correlation ID to trace across all channels") String correlationId,
        @ToolArg(name = "limit", description = "Maximum entries to include (default 100, max 500)", required = false) Integer limit) {

    final int effectiveLimit = (limit != null && limit > 0) ? Math.min(limit, 500) : 100;
    return causalGraphService.buildGraph(correlationId, effectiveLimit, currentPrincipal.tenancyId());
}
```

Add `@Inject CausalGraphService causalGraphService;` field to `QhorusMcpTools`.

- [ ] **Step 5: Write tests in `LedgerQueryToolsTest`**

```java
// get_causal_chain — cross-channel (no channel param)
@Test
void getCausalChain_noChannel_crossesChannelBoundary() { ... }

// get_causal_chain — with channel preserves single-channel behavior
@Test
void getCausalChain_withChannel_singleChannelBehavior() { ... }

// get_causal_chain — CausalChainEntry includes new fields
@Test
void getCausalChain_entryIncludesChannelAndContent() { ... }

// get_causal_graph — returns CausalGraph
@Test
void getCausalGraph_returnsCausalGraph() { ... }

// get_causal_graph — unknown correlation returns empty
@Test
void getCausalGraph_unknownCorrelation_emptyGraph() { ... }
```

- [ ] **Step 6: Run all LedgerQueryToolsTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerQueryToolsTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ runtime/src/test/java/io/casehub/qhorus/ledger/LedgerQueryToolsTest.java
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus commit -m "feat(#398): cross-channel get_causal_chain + new get_causal_graph MCP tools. Refs #398"
```

---

### Task 4: CausalGraphResource — REST endpoints

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/CausalGraphResource.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/api/CausalGraphResourceTest.java`

**Interfaces:**
- Consumes: `CausalGraphService.buildGraph(String, int, String)` (Task 2), `MessageLedgerEntryRepository.findAncestorChainCrossChannel(UUID, String)` (Task 1), `ChannelStore.findByIds(Collection<UUID>)` (existing), `CurrentPrincipal.tenancyId()`
- Produces: `GET /api/causal-graph/{correlationId}` → `CausalGraph` JSON, `GET /api/causal-graph/attribution/{entryId}` → `List<AttributionEntry>` JSON

- [ ] **Step 1: Write failing integration test**

```java
@QuarkusTest
class CausalGraphResourceTest {

    @Test
    void getGraph_knownCorrelation_returnsGraph() {
        // Set up channel + dispatch COMMAND + DONE with same correlationId
        // GET /api/causal-graph/{correlationId}
        // Assert 200, nodes array, edges array, outcome
        given()
            .when().get("/api/causal-graph/" + correlationId)
            .then()
            .statusCode(200)
            .body("correlationId", is(correlationId))
            .body("nodes.size()", is(2))
            .body("outcome", is("FULFILLED"));
    }

    @Test
    void getGraph_unknownCorrelation_returnsEmptyGraph() {
        given()
            .when().get("/api/causal-graph/" + UUID.randomUUID())
            .then()
            .statusCode(200)
            .body("nodes.size()", is(0))
            .body("outcome", is("OPEN"));
    }

    @Test
    void getAttribution_knownEntry_returnsChain() {
        // Set up causal chain, GET /api/causal-graph/attribution/{entryId}
        given()
            .when().get("/api/causal-graph/attribution/" + entryId)
            .then()
            .statusCode(200)
            .body("size()", greaterThan(0));
    }

    @Test
    void getAttribution_invalidUuid_returns400() {
        given()
            .when().get("/api/causal-graph/attribution/not-a-uuid")
            .then()
            .statusCode(400);
    }
}
```

- [ ] **Step 2: Implement `CausalGraphResource`**

Create new file at `runtime/src/main/java/io/casehub/qhorus/runtime/api/CausalGraphResource.java`:

```java
package io.casehub.qhorus.runtime.api;

import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.identity.CurrentPrincipal;
import io.casehub.qhorus.runtime.ledger.CausalGraphService;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntry;
import io.casehub.qhorus.runtime.ledger.MessageLedgerEntryRepository;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.*;
import java.util.stream.Collectors;

@Path("/api/causal-graph")
@Produces(MediaType.APPLICATION_JSON)
public class CausalGraphResource {

    @Inject CausalGraphService causalGraphService;
    @Inject MessageLedgerEntryRepository ledgerRepo;
    @Inject ChannelStore channelStore;
    @Inject CurrentPrincipal currentPrincipal;

    @GET
    @Path("/{correlationId}")
    @Transactional
    public Response getGraph(@PathParam("correlationId") String correlationId,
                             @QueryParam("limit") @DefaultValue("100") int limit) {
        int effectiveLimit = Math.min(Math.max(limit, 1), 500);
        var graph = causalGraphService.buildGraph(
                correlationId, effectiveLimit, currentPrincipal.tenancyId());
        return Response.ok(graph).build();
    }

    @GET
    @Path("/attribution/{entryId}")
    @Transactional
    public Response getAttribution(@PathParam("entryId") String entryId) {
        UUID entryUuid;
        try {
            entryUuid = UUID.fromString(entryId);
        } catch (IllegalArgumentException e) {
            return Response.status(400)
                    .entity(Map.of("error", "Invalid entry ID: " + entryId))
                    .build();
        }

        List<MessageLedgerEntry> chain = ledgerRepo.findAncestorChainCrossChannel(
                entryUuid, currentPrincipal.tenancyId());

        // Resolve channel names
        Set<UUID> channelIds = chain.stream()
                .map(e -> e.channelId).collect(Collectors.toSet());
        Map<UUID, String> names = channelStore.findByIds(channelIds).stream()
                .collect(Collectors.toMap(
                        ch -> ((Channel) ch).id(),
                        ch -> ((Channel) ch).name(),
                        (a, b) -> a));

        var result = chain.stream().map(e -> Map.of(
                "entryId", e.id.toString(),
                "channelId", e.channelId.toString(),
                "channelName", names.getOrDefault(e.channelId, "unknown"),
                "messageType", e.messageType,
                "actorId", e.actorId != null ? e.actorId : "",
                "correlationId", e.correlationId != null ? e.correlationId : "",
                "occurredAt", e.occurredAt != null ? e.occurredAt.toString() : "",
                "content", e.content != null ? e.content : "",
                "causedByEntryId", e.causedByEntryId != null ? e.causedByEntryId.toString() : ""
        )).toList();

        return Response.ok(result).build();
    }
}
```

- [ ] **Step 3: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CausalGraphResourceTest -pl runtime -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: all PASS

- [ ] **Step 4: Run full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/slots/139/qhorus/pom.xml`
Expected: BUILD SUCCESS across all 17 modules

- [ ] **Step 5: Update CLAUDE.md with new tools and resource**

Add entries for `get_causal_graph`, the updated `get_causal_chain`, `CausalGraphService`, and `CausalGraphResource` to the project structure section.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/api/CausalGraphResource.java runtime/src/test/java/io/casehub/qhorus/runtime/api/CausalGraphResourceTest.java CLAUDE.md
git -C /Users/mdproctor/claude/casehub/slots/139/qhorus commit -m "feat(#398): CausalGraphResource REST API + CLAUDE.md update. Closes #398"
```

## References

- [2026-08-21-causal-graphs-design.md] — design spec this plan implements
- `QhorusMcpTools.java:1875-1903` — existing `get_causal_chain` implementation
- `QhorusMcpTools.java:2045-2079` — existing `get_obligation_activity` pattern
- `MessageLedgerEntryRepository.java:157-174` — existing `findAncestorChain`
- `MessageLedgerEntryRepository.java:358-369` — existing `findByCorrelationIdAcrossChannels`
- `QhorusMcpToolsBase.java:247-255` — existing `CausalChainEntry` record
- `ChannelResource.java` — REST resource pattern reference
- GitHub #398

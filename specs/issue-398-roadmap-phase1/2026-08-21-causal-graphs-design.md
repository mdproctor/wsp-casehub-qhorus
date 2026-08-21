# Cross-Channel Causal Graphs — Design Spec

**Issue:** #398 — E1: Cross-channel causal graphs
**Scale:** S | **Complexity:** Low
**Date:** 2026-08-21

---

## Problem

The attribution problem — "who delegated to whom in the chain that caused this failure?" — requires tracing across channels. The ledger has `causedByEntryId` and `correlationId` per entry. The data exists; the query layer doesn't.

Existing tools are channel-scoped or flat:
- `get_causal_chain(channel, entry_id)` walks `causedByEntryId` but stops at channel boundaries
- `get_obligation_chain(channel, correlation_id)` computes obligation enrichment for a single channel
- `get_obligation_activity(correlation_id)` is cross-channel but returns a flat chronological list with no graph structure

### Epic deliverable mapping

Issue #398 lists four deliverables. This spec addresses them as follows:

| Epic deliverable | Spec solution |
|---|---|
| `get_causal_graph(correlationId)` MCP tool | New tool — §2 |
| `get_attribution_chain(entryId)` MCP tool | Satisfied by making `get_causal_chain` cross-channel capable (§1). A separate tool with a new name is unnecessary — the existing tool already walks `causedByEntryId` backward; the only fix needed is removing the channel boundary. |
| Visual rendering as directed graph | Out of scope. The structured JSON (nodes + edges) is the rendering-ready data format. A future UI layer consumes this JSON to draw an interactive graph — building that UI is not part of this epic. |
| REST endpoint for external consumption | New `CausalGraphResource` — §3 |

## Solution

New `CausalGraphService` (§2a), two MCP tool changes (§1, §2), one new REST resource (§3), one new repository method (§4).

### 1. Fix `get_causal_chain` — make cross-channel capable

Make the `channel` parameter optional. When omitted, the `causedByEntryId` walk crosses channel boundaries by calling `findAncestorChainCrossChannel` (§4). When provided, the existing `findAncestorChain` is called, preserving current single-channel behavior.

**Signature change:**
```
get_causal_chain(entry_id, channel?)  // channel becomes optional
```

**Return type:** unchanged — `List<CausalChainEntry>`. Add `channelId`, `channelName`, and `content` fields to `CausalChainEntry` for cross-channel context and attribution analysis.

**Updated `CausalChainEntry` record:**
```java
record CausalChainEntry(
    String entryId,
    String channelId,      // NEW
    String channelName,    // NEW
    String messageType,
    String actorId,
    String correlationId,
    String occurredAt,
    String content,        // NEW — parity with GraphNode
    String causedByEntryId
)
```

**Channel name resolution:** For the cross-channel path, batch-load channel names using `channelStore.findByIds(channelIds)` for the unique channel IDs in the chain. For the single-channel path, use the already-resolved channel name.

### 2. New `get_causal_graph(correlation_id)` MCP tool

Fetches all entries sharing a `correlationId` across channels, builds a nodes+edges graph from `causedByEntryId` links, enriches with channel names and elapsed times.

**Signature:**
```
get_causal_graph(correlation_id, limit?)
```

**Return type:** `CausalGraph` record.

```java
record CausalGraph(
    String correlationId,
    String rootEntryId,       // earliest entry with null causedByEntryId (by occurredAt)
    int channelCount,
    List<String> channels,    // channel names (not IDs) — convenience summary
    Long totalDurationMs,     // root occurredAt to latest terminal entry occurredAt
    String outcome,           // FULFILLED, FAILED, DECLINED, OPEN (see outcome rules)
    boolean truncated,        // true when limit < total entries for this correlationId
    List<GraphNode> nodes,
    List<GraphEdge> edges
)

record GraphNode(
    String entryId,
    String channelId,
    String channelName,
    String messageType,
    String actorId,
    String occurredAt,
    String content,
    String causedByEntryId,
    int depth                 // 0 = root, -1 = unlinked
)

record GraphEdge(
    String from,              // parent entryId
    String to,                // child entryId
    String type,              // "CAUSED_BY"
    Long elapsedMs            // time between parent and child
)
```

### 2a. `CausalGraphService` — graph construction logic

A new `@ApplicationScoped` service that owns the graph building algorithm. Both `QhorusMcpTools.getCausalGraph()` and `CausalGraphResource` delegate to this service.

Injections: `MessageLedgerEntryRepository`, `ChannelStore` (for `findByIds`), `CurrentPrincipal`.

The DTO records (`CausalGraph`, `GraphNode`, `GraphEdge`) are defined as public static inner records on `CausalGraphService`, following the pattern where domain records live with the service that produces them. The existing `CausalChainEntry` stays in `QhorusMcpToolsBase` (moving it is out of scope — it would require updating all existing tests and callers).

**Graph building algorithm:**
1. Fetch all entries via `findByCorrelationIdAcrossChannels(correlationId, limit, tenancyId)`
2. Set `truncated = true` if the result size equals the limit (entries may have been dropped)
3. Index entries by `id` in a `Map<UUID, MessageLedgerEntry>`
4. For each entry with a non-null `causedByEntryId` that exists in the map, create a `CAUSED_BY` edge
5. **Root selection:** Find all entries with null `causedByEntryId`. Pick the earliest by `occurredAt` as the root. If zero entries have null `causedByEntryId` (root is outside the tenant or was deleted), set `rootEntryId = null` and all entries get depth -1 (all unlinked).
6. Compute `depth` via BFS from root. Entries with non-null `causedByEntryId` whose parent is not in the map get depth -1 (unlinked — parent was outside tenant boundary or truncated by limit).
7. Batch-load channel names using `channelStore.findByIds(uniqueChannelIds)` — NOT `channelService.listAll()`.
8. **Outcome computation (aligned with `get_obligation_chain`):** Terminal types are `DONE`, `FAILURE`, `DECLINE` — HANDOFF is explicitly non-terminal (it means delegation continues, not resolution). If multiple terminal entries exist on different branches, apply precedence: `FAILED > DECLINED > FULFILLED > OPEN`. If no terminal entry exists, outcome is `OPEN`.
9. **Duration computation:** `totalDurationMs` = root `occurredAt` to the latest terminal entry `occurredAt`. If no terminal or no root, `totalDurationMs = null`.

### Known limitation: cross-channel `causedByEntryId` links

`causedByEntryId` is set from either `dispatch.causedByEntryId()` or `dispatch.inReplyTo()` — both optional parameters. When a HANDOFF on channel A triggers a new COMMAND on channel B, the agent on channel B must explicitly pass `caused_by_entry_id` or `in_reply_to` to create the link. If the agent doesn't, no cross-channel edge exists.

In that case, `get_causal_graph` will show entries from both channels (they share a `correlationId`) but no edge connecting them. The graph has multiple disconnected sub-trees. This is a data-quality limitation, not a query limitation — the graph faithfully represents what the ledger recorded.

The tool description should note: "Edges are derived from `causedByEntryId` links. For complete cross-channel graphs, agents must pass `caused_by_entry_id` when sending messages that continue delegations from other channels."

### Known limitation: tenant boundary truncation

Cross-tenant delegation traces break at the tenant boundary (documented in `MessageLedgerEntryRepository` Javadoc, qhorus#265). Both `get_causal_graph` and `get_causal_chain` (cross-channel mode) silently stop at the boundary. The tool description should note this limitation so LLM callers know the graph may be incomplete for cross-tenant workflows.

### 3. New `CausalGraphResource` — REST API

```
GET /api/causal-graph/{correlationId}          → CausalGraph (from CausalGraphService)
GET /api/causal-graph/attribution/{entryId}    → List<CausalChainEntry>
```

The attribution endpoint takes only `entryId` — no `correlationId` in the path. The `causedByEntryId` walk is independent of correlation. The path `/api/causal-graph/attribution/{entryId}` is flat and semantically correct.

`CausalGraphResource` injects `CausalGraphService` and `MessageLedgerEntryRepository`. It does NOT import from `QhorusMcpToolsBase` — the graph records are on `CausalGraphService`, and the attribution chain reuses repository output directly with inline mapping to `CausalChainEntry` (or a REST-specific DTO).

Standard JAX-RS patterns following `ChannelResource` — dual-identity resolution not needed (correlationId is always a string, entryId is always a UUID).

### 4. Repository — cross-channel ancestor walk

New method on `MessageLedgerEntryRepository`:

```java
public List<MessageLedgerEntry> findAncestorChainCrossChannel(
        UUID entryId, String tenancyId) {
    // Same loop as findAncestorChain but without channelId guard
    // Tenancy boundary preserved (security boundary)
    // Depth limit: 50 hops (cycle guard via visited set)
}
```

The existing `findAncestorChain(channelId, entryId, tenancyId)` gains a matching 50-hop depth limit for consistency — previously it relied only on the visited set for cycle protection. The depth limit is a safety bound; real delegation chains are unlikely to exceed 10-20 hops.

**Access control note:** Channel ACLs (`allowedWriters`, `adminInstances`) control write access, not read access. Cross-channel reads within a tenancy are intentionally unrestricted — the tenancy boundary is the security boundary.

## What doesn't change

- `get_obligation_activity` — flat chronological list, different semantic
- `get_obligation_chain` — single-channel obligation enrichment
- `findByCorrelationIdAcrossChannels` — reused as-is for graph building
- No new database tables or migrations
- No changes to `LedgerWriteService` or message dispatch

## Testing

**Repository tests (`LedgerQueryRepoTest`):**
- `findAncestorChainCrossChannel` — three-hop chain across two channels, returns full chain
- `findAncestorChainCrossChannel` — cycle detection (visited set prevents infinite loop)
- `findAncestorChainCrossChannel` — depth limit at 50 hops
- `findAncestorChainCrossChannel` — unknown entry returns empty list
- `findAncestorChainCrossChannel` — tenancy boundary preserved (cross-tenant stops)
- `findAncestorChain` — verify 50-hop depth limit applies (regression for new limit)

**Service tests (`CausalGraphServiceTest`):**
- Graph with single root, linear chain — nodes, edges, depth, outcome correct
- Graph with branching (HANDOFF to two channels) — multiple terminals, precedence applied
- Graph with zero roots (root outside tenant) — all entries unlinked, rootEntryId null
- Graph with multiple null-causedByEntryId entries — earliest by occurredAt selected as root
- Graph with unlinked entries (STATUS/EVENT without causedByEntryId) — depth -1
- Graph with truncation (limit < total entries) — `truncated = true`
- Outcome precedence: FAILURE > DECLINE > DONE
- Duration computation: root to latest terminal entry
- Channel name resolution via `channelStore.findByIds` (not listAll)
- Empty graph for unknown correlationId

**MCP tool tests (`LedgerQueryToolsTest`):**
- `get_causal_chain` — without channel: crosses channel boundary, includes channelId/channelName/content
- `get_causal_chain` — with channel: preserves single-channel behavior (regression)
- `get_causal_graph` — delegates to CausalGraphService, returns CausalGraph

**Integration tests:**
- `CausalGraphResourceTest` — `GET /api/causal-graph/{correlationId}` returns CausalGraph JSON
- `CausalGraphResourceTest` — `GET /api/causal-graph/attribution/{entryId}` returns chain JSON
- End-to-end: dispatch COMMAND on channel A, HANDOFF to channel B, DONE on channel B → graph shows full cross-channel tree with edges

## References

- `QhorusMcpTools.java:1875` — existing `get_causal_chain` implementation
- `QhorusMcpTools.java:1841-1842` — terminal types definition (DONE, FAILURE, DECLINE — not HANDOFF)
- `QhorusMcpTools.java:2045` — existing `get_obligation_activity` (cross-channel flat list)
- `MessageLedgerEntryRepository.java:157-174` — existing `findAncestorChain` with channel boundary
- `MessageLedgerEntryRepository.java:358-369` — existing `findByCorrelationIdAcrossChannels`
- `QhorusMcpToolsBase.java:247-255` — existing `CausalChainEntry` record
- `ChannelResource.java:338-343` — existing `correlationChain` REST endpoint (single-channel)
- `ChannelStore.findByIds(Collection<UUID>)` — batch channel lookup
- `docs/roadmap-epics-2026.md` — Epic 1 definition
- Issue #398

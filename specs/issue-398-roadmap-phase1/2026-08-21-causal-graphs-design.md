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

## Solution

Two changes to the MCP tool surface, one new REST resource, one new repository method.

### 1. Fix `get_causal_chain` — make cross-channel capable

Make the `channel` parameter optional. When omitted, the `causedByEntryId` walk crosses channel boundaries. When provided, preserves current single-channel behavior.

**Signature change:**
```
get_causal_chain(entry_id, channel?)  // channel becomes optional
```

**Return type:** unchanged — `List<CausalChainEntry>`. Add `channelId` and `channelName` fields to `CausalChainEntry` for cross-channel context.

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
    String causedByEntryId
)
```

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
    String rootEntryId,
    int channelCount,
    List<String> channels,
    Long totalDurationMs,
    String outcome,           // FULFILLED, FAILED, DECLINED, DELEGATED, OPEN
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
    int depth                 // 0 = root
)

record GraphEdge(
    String from,              // parent entryId
    String to,                // child entryId
    String type,              // "CAUSED_BY"
    Long elapsedMs            // time between parent and child
)
```

**Graph building algorithm:**
1. Fetch all entries via `findByCorrelationIdAcrossChannels(correlationId, limit, tenancyId)`
2. Index entries by `id` in a `Map<UUID, MessageLedgerEntry>`
3. For each entry with a non-null `causedByEntryId` that exists in the map, create an edge
4. Compute `depth` via BFS from root (entry with null `causedByEntryId`)
5. Entries with no `causedByEntryId` and not the root are unlinked (depth = -1)
6. Batch-load channel names from the unique channel IDs (same pattern as `getObligationActivity`)
7. Compute `outcome` from the terminal entry type (DONE→FULFILLED, FAILURE→FAILED, DECLINE→DECLINED, HANDOFF→DELEGATED, no terminal→OPEN)
8. Compute `totalDurationMs` from root to terminal entry

**Limit:** default 100, max 500 (same as `get_obligation_activity`).

### 3. New `CausalGraphResource` — REST API

```
GET /api/causal-graph/{correlationId}
GET /api/causal-graph/{correlationId}/attribution/{entryId}
```

Both return the same JSON structures as the MCP tools. Standard JAX-RS patterns following `ChannelResource`.

`CausalGraphResource` injects `MessageLedgerEntryRepository`, `ChannelService`, and `CurrentPrincipal`. Request/response DTOs are the same records used by the MCP tools (placed in `QhorusMcpToolsBase` alongside existing DTOs).

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

The existing `findAncestorChain(channelId, entryId, tenancyId)` is unchanged — used by the single-channel path.

## What doesn't change

- `get_obligation_activity` — flat chronological list, different semantic
- `get_obligation_chain` — single-channel obligation enrichment
- `findByCorrelationIdAcrossChannels` — reused as-is for graph building
- `findAncestorChain` — single-channel version preserved
- No new database tables or migrations
- No changes to `LedgerWriteService` or message dispatch

## Testing

**Repository tests (`MessageLedgerEntryRepositoryTest` / `LedgerQueryRepoTest`):**
- `findAncestorChainCrossChannel` — three-hop chain across two channels, returns full chain
- `findAncestorChainCrossChannel` — cycle detection (visited set prevents infinite loop)
- `findAncestorChainCrossChannel` — depth limit at 50 hops
- `findAncestorChainCrossChannel` — unknown entry returns empty list
- `findAncestorChainCrossChannel` — tenancy boundary preserved (cross-tenant stops)

**MCP tool tests (`LedgerQueryToolsTest`):**
- `get_causal_chain` — without channel: crosses channel boundary
- `get_causal_chain` — with channel: preserves single-channel behavior (regression)
- `get_causal_chain` — CausalChainEntry includes channelId and channelName
- `get_causal_graph` — builds graph with nodes and edges from cross-channel entries
- `get_causal_graph` — enrichment: depth, elapsedMs, outcome, channelCount
- `get_causal_graph` — unlinked entries (no causedByEntryId) get depth -1
- `get_causal_graph` — unknown correlationId returns empty graph
- `get_causal_graph` — limit respected

**Integration tests:**
- `CausalGraphResourceTest` — REST endpoints return correct JSON for graph and attribution
- End-to-end: dispatch COMMAND on channel A, HANDOFF to channel B, DONE on channel B → graph shows full cross-channel tree

## References

- `QhorusMcpTools.java:1875` — existing `get_causal_chain` implementation
- `QhorusMcpTools.java:2045` — existing `get_obligation_activity` (cross-channel flat list)
- `MessageLedgerEntryRepository.java:157-174` — existing `findAncestorChain` with channel boundary
- `MessageLedgerEntryRepository.java:358-369` — existing `findByCorrelationIdAcrossChannels`
- `QhorusMcpToolsBase.java:247-255` — existing `CausalChainEntry` record
- `ChannelResource.java:338-343` — existing `correlationChain` REST endpoint (single-channel)
- `docs/roadmap-epics-2026.md` — Epic 1 definition
- Issue #398

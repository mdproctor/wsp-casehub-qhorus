# Decisions — #398 Cross-Channel Causal Graphs

## D1: Tool surface — fix existing + add new

**Choice:** Fix `get_causal_chain` to make `channel` optional (cross-channel when omitted) + add new `get_causal_graph(correlation_id)` tool. Keep `get_obligation_activity` as-is.
**Alternatives:**
- 4 new tools (get_causal_graph, get_attribution_chain, keep existing unchanged) — unnecessary duplication; get_attribution_chain IS get_causal_chain with channel optional
- Add `format=graph` param to get_obligation_activity — overloaded semantics confuse agents (timeline vs structure)
**Rationale:** Pre-release — fix the design flaw (channel-scoped causal walk). Two changes serve all four deliverables without API sprawl. get_obligation_activity remains useful as a flat chronological view.
**Trade-offs:** Callers of get_causal_chain that always passed channel see no change. Callers that didn't know about the channel boundary get new capability.
**Sources:** QhorusMcpTools.java:1875 (get_causal_chain), QhorusMcpTools.java:2045 (get_obligation_activity), MessageLedgerEntryRepository.java:157 (findAncestorChain channel boundary check)
**Exploration:** quick
**Status:** captured

## D2: Graph data model — nodes + edges with enrichment

**Choice:** JSON with top-level summary (correlationId, rootEntryId, channelCount, channels, totalDurationMs, outcome) + `nodes[]` array (entryId, channelId, channelName, messageType, actorId, occurredAt, content, causedByEntryId, depth) + `edges[]` array (from, to, type=CAUSED_BY, elapsedMs). Content included untruncated.
**Alternatives:**
- Tree-shaped JSON (children nested in parents) — harder to parse, doesn't handle fan-in, non-standard for graph interchange
- Flat list with parent refs only (like existing CausalChainEntry) — loses enrichment (elapsed, depth, channel names, outcome summary)
**Rationale:** Nodes+edges is the standard graph interchange format. Enrichment fields (depth, elapsed, channel names) make the output immediately useful for both agents and future UIs without additional queries. Content untruncated because attribution analysis needs full message text.
**Trade-offs:** Larger response payload for high-message-count correlations. Mitigated by existing `limit` pattern on `findByCorrelationIdAcrossChannels` (default 100, max 500).
**Sources:** Issue #398 deliverable: "channel-colored nodes, message-type edges"
**Exploration:** quick
**Status:** captured

## D3: REST endpoint placement — new CausalGraphResource

**Choice:** New `CausalGraphResource` at `/api/causal-graph` with two endpoints: `GET /{correlationId}` (graph) and `GET /{correlationId}/attribution/{entryId}` (linear chain).
**Alternatives:**
- Extend ChannelResource — wrong scope; causal graphs are cross-channel, ChannelResource is channel-scoped
- Add to A2AResource — wrong domain; causal graphs are internal governance, not A2A protocol
**Rationale:** Cross-channel data doesn't belong on channel-scoped resources. Follows AgentCardResource and A2AResource pattern of top-level resources for cross-cutting concerns.
**Trade-offs:** One more JAX-RS resource class. Minimal — the class is small.
**Sources:** ChannelResource.java (channel-scoped pattern), AgentCardResource.java (cross-cutting resource pattern)
**Exploration:** quick
**Status:** captured

## D4: Repository layer — new cross-channel walk method

**Choice:** Add `findAncestorChainCrossChannel(entryId, tenancyId)` to `MessageLedgerEntryRepository`. Same logic as `findAncestorChain` but without `channelId.equals()` guard. Depth limit 50 hops. Reuse existing `findByCorrelationIdAcrossChannels` for graph building.
**Alternatives:**
- Modify existing `findAncestorChain` to accept nullable channelId — muddies the API; single-channel walk is still valid and used by other tools
- Use recursive CTE in JPQL — H2 support is fragile for recursive CTEs; Java loop is simpler and already the pattern
**Rationale:** Separate method preserves existing single-channel behavior. Java-loop walk with visited set (existing pattern) handles cycles. 50-hop depth limit is generous for real delegation chains.
**Trade-offs:** Two similar methods in the repository. Acceptable — they have different channel-boundary semantics.
**Sources:** MessageLedgerEntryRepository.java:157-174 (findAncestorChain implementation)
**Exploration:** quick
**Status:** captured

# Decisions — #407 Tutorial Example: Governance in Action

## D1: CausalGraphRenderer exposure

**Choice:** Utility class in runtime + `render_causal_graph(correlation_id)` MCP tool
**Alternatives:**
- Utility only — defers MCP exposure; LLMs can't access rendered graphs without programmatic integration
- MCP tool only — no programmatic reuse from Java code
**Rationale:** Text-rendered causal graphs are specifically designed for LLM consumption — agents reading structured text output to understand attribution and causation chains. The MCP tool makes this immediately useful. The utility class enables programmatic reuse (tutorial tests, future tools).
**Trade-offs:** One extra @Tool method. Minimal — the utility does the work, the tool is a thin wrapper.
**Sources:** #398 design spec (causal graph readability for LLMs), platform dynamic MCP tools pattern
**Exploration:** quick
**Status:** captured

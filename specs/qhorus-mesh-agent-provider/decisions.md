# Decisions — Qhorus Mesh AgentProvider Bridge

## D1: Bidirectional integration

**Choice:** Both directions — server-side agents join mesh channels AND mesh tools can dispatch to AgentProvider backends
**Alternatives:**
- Server-side agents join mesh only — simpler, but MCP clients can't leverage AgentProvider
- MCP clients dispatch to AgentProvider only — misses the server-side agent participation scenario
**Rationale:** The mesh is the coordination layer for ALL agents, regardless of framework. Both inbound (AgentProvider → channel) and outbound (channel → AgentProvider) are needed.
**Trade-offs:** More complex bridge design; both directions must be tested
**Sources:** casehub-platform agent-api (AgentProvider, AgentBackend), qhorus RoutingBridge, claudony fleet manager #205
**Exploration:** quick
**Status:** captured

## D2: Agent lifecycle

**Choice:** Both persistent sessions and ephemeral invocations, designed equally
**Alternatives:**
- Persistent sessions first — ephemeral as degenerate case
- Ephemeral invoke first — simpler, no session management
**Rationale:** Fleet YAML (#246) declares persistent pools; ad-hoc MCP commands need ephemeral invocations. Both are first-class use cases.
**Trade-offs:** Must design both `AgentSession.query()` integration and `AgentProvider.invoke()` integration
**Sources:** claudony #246 (fleet deployment), #247 (fleet script runner), AgentSession/AgentProvider APIs
**Exploration:** quick
**Status:** captured

## D3: Module home

**Choice:** qhorus-mesh
**Alternatives:**
- Claudony — close to FleetScriptRunner but wrong dependency direction
- New module: qhorus-agent-bridge — additional module complexity without clear benefit
**Rationale:** The mesh relay already has ChannelService, MessageObserver, and all message routing infrastructure. Co-locating the bridge keeps message routing centralized.
**Trade-offs:** qhorus-mesh gains a dependency on casehub-platform agent-api
**Sources:** qhorus-mesh module (existing), A2AOutboundBackend pattern
**Exploration:** quick
**Status:** captured

## D4: Bridge mechanism

**Choice:** Both — ChannelBackend for inbound delivery to agents, MessageObserver for agent-initiated messages
**Alternatives:**
- ChannelBackend only — no agent-initiated messages
- MessageObserver only — no delivery guarantees, no cursor tracking
**Rationale:** ChannelBackend (AT_LEAST_ONCE) gives delivery guarantees via the existing delivery pump. MessageObserver enables agents to send proactive status updates, alerts, or initiate conversations.
**Trade-offs:** Two integration points to maintain; must prevent message loops (agent response → observer → agent again)
**Sources:** AgentChannelBackend SPI, A2AOutboundBackend (#396), MessageObserver dispatch pattern
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D5: Cache-aware prompt structuring

**Choice:** Bridge is cache-aware — separates stable context (systemPrompt) from per-message content (userPrompt)
**Alternatives:**
- Delegate to backend — simpler bridge, but backends must parse mesh prompts
- Protocol-level separation — MeshPromptContext record with structured fields
**Rationale:** AgentSessionInit already carries systemPrompt (set once at session open). The bridge puts channel description, agent briefing, and peer list in systemPrompt (cached by Claude). Per-message content goes to query(). This is the natural mapping to the existing API.
**Trade-offs:** Bridge knows about prompt structure, making it slightly coupled to the LLM interaction model
**Sources:** ClaudeAgentProvider (direct SDK, cache-aware), AgentSessionInit.systemPrompt(), Anthropic prompt caching docs
**Exploration:** quick
**Status:** captured

## D6: API surface — @McpDomain migration

**Choice:** Migrate MeshMcpTools from @Tool to @McpDomain pattern as part of this spec
**Alternatives:**
- Separate prerequisite issue — delays the AgentProvider work
- Keep @Tool — inconsistent with the rest of the platform
**Rationale:** MeshApi interface (with @McpDomain, @PlatformQuery, @PlatformMutation) IS the MeshOperations SPI we agreed to extract. The platform's code generator produces REST + GraphQL + MCP endpoints from this single interface. Using @Tool is the old pattern.
**Trade-offs:** Larger scope in this spec; mesh module needs casehub-platform generator dependency
**Sources:** ChannelsApi (@McpDomain pattern), ChannelsService (implementation pattern), casehub-platform generator
**Exploration:** quick
**Status:** captured

## D7: Relationship to fleet YAML

**Choice:** The bridge is the runtime execution layer for fleet-declared agent-to-channel wiring
**Alternatives:** None — this is a factual relationship, not a design choice
**Rationale:** Fleet YAML (#246) declares agents, pools, and channels with `dependsOn`. The `FleetNodeHandler` SPI handles provisioning. The mesh bridge connects provisioned AgentSessions to mesh channels at runtime. Fleet declares the wiring; the bridge executes it.
**Trade-offs:** Bridge design must align with FleetNodeHandler lifecycle (create, update, destroy)
**Sources:** claudony #246 (fleet YAML schema), #247 (FleetScriptRunner), FleetNodeHandler SPI
**Exploration:** quick
**Depends on:** D2, D3
**Status:** captured

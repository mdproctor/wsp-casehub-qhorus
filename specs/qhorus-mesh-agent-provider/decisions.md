# Decisions — Qhorus Mesh AgentProvider Bridge

## D1: Bridge scope

**Choice:** Channel participation — server-side agents join mesh channels as first-class participants, both receiving and sending messages
**Alternatives:**
- Both directions as separate integration modes — conflates channel-internal message flow with tool-level dispatch
- Server-side agents join mesh only (receive-only) — misses agent-initiated messages, status updates, and proactive alerts
- Outbound MCP tool dispatch to AgentProvider — redundant with direct CDI invocation; governed invocation already happens through channel COMMAND delivery
**Rationale:** The bridge integrates AgentProvider-managed agents into the channel abstraction. Messages flow bidirectionally WITHIN channels: the ChannelBackend delivers channel messages to the agent, and the agent posts messages back via MessageDispatcher. There is no separate "outbound dispatch" direction — governed invocation of agents happens through the channel (send a COMMAND, ChannelBackend delivers, agent responds). Direct AgentProvider invocation via CDI remains available for ungoverned use.
**Trade-offs:** The agent is a full channel participant with governance overhead (commitment tracking, speech-act enforcement, ledger audit). For lightweight invocations that don't need governance, direct CDI invocation is simpler.
**Sources:** casehub-platform agent-api (AgentProvider, AgentBackend), qhorus ChannelBackend SPI, A2AOutboundBackend pattern, claudony fleet manager #205
**Exploration:** quick
**Status:** revised — R1-02 correctly identified that "outbound dispatch" is either the channel delivery mechanism (part of channel participation) or redundant with direct CDI invocation. Collapsed to single integration concept: channel participation.

## D2: Agent lifecycle

**Choice:** Design for persistent sessions; ephemeral invocations as degenerate case
**Alternatives:**
- Both designed equally — ambiguous; masks the asymmetric complexity
- Ephemeral invoke first — simpler, but persistent sessions ARE the complex case that must be designed first
**Rationale:** Persistent sessions require session lifecycle management (open/close/interrupt), cursor tracking (which channel messages has the agent seen?), systemPrompt stability, and reconnection on failure. Ephemeral invocations are single-turn — map a message to an AgentSessionConfig, invoke, collect response. Designing the persistent case first ensures the hard problems (cursor tracking, session restart, cache management) are solved; ephemeral is the single-turn degenerate case.
**Trade-offs:** Must handle both AgentSession.query() and AgentProvider.invoke() but persistent is the primary design target
**Sources:** claudony #246 (fleet deployment), #247 (fleet script runner), AgentSession/AgentProvider APIs
**Exploration:** quick
**Status:** revised — R1-07 correctly identified that "designed equally" is ambiguous. Persistent sessions are the complex case; ephemeral is derived.

## D3: Module home

**Choice:** New module: `agent-bridge/` at the qhorus repo root
**Alternatives:**
- qhorus-mesh — mesh is a standalone Quarkus application (@QuarkusMain, quarkus-maven-plugin with build goal, quarkus-jdbc-h2 non-optional), not a library; placing the bridge here limits adoption to the mesh relay only
- Claudony — close to FleetScriptRunner but wrong dependency direction
**Rationale:** The bridge is a classpath-activated optional module, following the established pattern of a2a-outbound/, connector-backend/, slack-channel/, and notification-bridge/. Any Quarkus app that embeds casehub-qhorus can add agent-bridge/ to get AgentProvider-to-channel integration. The mesh application adds it as a dependency alongside its other capabilities. The routing infrastructure (ChannelService, MessageObserver, MessageDispatcher) lives in runtime-core/ and runtime/, not in mesh/ — the bridge depends on the runtime, not the mesh application.
**Trade-offs:** Additional module at the repo root (consistent with existing bridge pattern — one pom.xml and a package)
**Sources:** a2a-outbound/ (ChannelBackend bridge module pattern), connector-backend/, notification-bridge/, slack-channel/, qhorus-mesh pom.xml (confirmed: @QuarkusMain + quarkus-maven-plugin build goal + quarkus-jdbc-h2)
**Exploration:** quick
**Status:** revised — R1-01 correctly identified that qhorus-mesh is a standalone application with only MeshApp.java and MeshMcpTools.java. Follows established bridge module pattern.

## D4: Bridge mechanism

**Choice:** Both — ChannelBackend for channel-to-agent delivery, MessageObserver for agent-initiated messages. Two-layer loop guard: sender-based direct guard + per-bridge-instance depth counter for indirect loops.
**Alternatives:**
- ChannelBackend only — no agent-initiated messages
- MessageObserver only — no delivery guarantees, no cursor tracking
**Rationale:** ChannelBackend (AT_LEAST_ONCE) gives delivery guarantees via the existing delivery pump. MessageObserver enables agents to send proactive status updates, alerts, or initiate conversations. Loop prevention uses two complementary mechanisms:
**Direct loop guard:** The bridge maintains a set of managed agent instanceIds. In postTracked(), if message.sender() matches any managed agent, return immediately (matches A2AOutboundBackend.isExternalAgent() pattern). This prevents the cycle where an agent's RESPONSE is fanned out back to the ChannelBackend that triggered the invocation.
**Indirect loop guard:** A per-bridge-instance `AtomicInteger` tracks the number of currently in-flight agent invocations across all managed agents. When an invocation starts, the counter increments; when it completes, it decrements. If the counter exceeds a configurable max-depth threshold (default: 3), the bridge refuses to invoke and logs a warning. This catches cross-channel indirect loops (agent A → channel X → agent B → channel Y → agent A) regardless of correlationId, thread, or request scope. The `AtomicInteger` is JVM-wide and thread-safe, which is essential because AT_LEAST_ONCE delivery runs asynchronously on the DeliveryService pump thread (verified: ChannelGateway.fanOut() skips AT_LEAST_ONCE backends; delivery is signaled via DeliverySignalQueue after transaction commit). ThreadLocal and @RequestScoped counters do not propagate across this async boundary.
**Trade-offs:** Two integration points to maintain; per-bridge AtomicInteger adds a JVM-wide counter but imposes no measurable overhead
**Sources:** AgentChannelBackend SPI, A2AOutboundBackend (#396) — see resolver.isExternalAgent(message.sender()) guard in post(), ChannelGateway.fanOut() AT_LEAST_ONCE skip path, DeliverySignalQueue async delivery
**Exploration:** quick
**Depends on:** D1
**Status:** revised — R1-03 identified under-specification; R2-04 identified undefined scope for depth counter. Now specifies per-bridge-instance AtomicInteger with async delivery awareness.

## D5: Cache-aware prompt structuring

**Choice:** Bridge separates stable context (systemPrompt at session open) from volatile context (per-turn query prompt). Peer list snapshot at session open; session restart on significant peer topology changes.
**Alternatives:**
- Delegate to backend — simpler bridge, but backends must parse mesh prompts
- Protocol-level separation — MeshPromptContext record (rejected: volatile/stable separation is achievable within AgentSessionInit without a new abstraction)
**Rationale:** AgentSessionInit.systemPrompt() is set once at session open and cached by Anthropic's prefix-based caching. Stable context (channel description, agent briefing, initial peer snapshot) goes in systemPrompt for cache benefit. Volatile context (the actual message to process, recent channel activity) goes in the per-turn query() prompt. Peer list changes are infrequent relative to message frequency; a stale peer list is acceptable for most interactions. On significant topology changes (agent join/leave events observed via ChannelInitialisedEvent), the bridge closes and reopens the session. For ephemeral invocations (AgentSessionConfig), systemPrompt is constructed per-call with current peers — no staleness issue since there is no persistent session.
**Trade-offs:** Session restart on peer topology change loses the conversation history in that session; acceptable because peer changes are rare and the channel message history (the normative record) is independent of the agent session
**Sources:** ClaudeAgentProvider (direct SDK, cache-aware), AgentSessionInit.systemPrompt(), AgentSessionConfig (systemPrompt + userPrompt), Anthropic prompt caching docs
**Exploration:** quick
**Status:** revised — R1-04 correctly identified that peer list is volatile. Added explicit volatile/stable separation strategy and session restart policy.

## D6: API surface — @McpDomain migration

**Choice:** Migrate MeshMcpTools from @Tool to @McpDomain pattern as part of this spec
**Alternatives:**
- Separate prerequisite issue — possible but fragments the mesh API surface definition
- Keep @Tool — inconsistent with the rest of the platform (ChannelsApi already uses @McpDomain)
**Rationale:** The spec covers agent-mesh integration holistically — both the bridge mechanism (how agents participate in channels) and the tool surface (what operations agents can perform in the mesh). MeshApi interface (with @McpDomain, @PlatformQuery, @PlatformMutation) defines the mesh operations contract. Migrating the tool surface alongside the bridge ensures agents joining channels via the bridge use the same modern API pattern as ChannelsApi. The migration is a co-requisite of the spec's scope, not an independent task.
**Trade-offs:** Larger scope in this spec; mesh module needs casehub-platform generator dependency
**Sources:** ChannelsApi (@McpDomain pattern in api/spi/channels/), MeshMcpTools (current @Tool pattern in mesh/), casehub-platform generator
**Exploration:** quick
**Status:** captured

## D7: Relationship to fleet YAML

**Choice:** The bridge is the runtime execution layer for fleet-declared agent-to-channel wiring, designed with lifecycle extension points
**Alternatives:**
- Commit to FleetNodeHandler lifecycle — risks coupling to an SPI that doesn't exist (verified: ide_find_class returns 0 results for both FleetNodeHandler and FleetScriptRunner)
- Ignore fleet YAML — misses the primary integration consumer
**Rationale:** Fleet YAML (#246) declares agents, pools, and channels with `dependsOn`. FleetNodeHandler and FleetScriptRunner are referenced in #246/#247 but do not exist in any codebase. The bridge must not design against an assumed SPI shape. Instead, the bridge exposes lifecycle operations (create binding, update binding, destroy binding) as a stable SPI that FleetNodeHandler can call when it materializes. The bridge's lifecycle SPI is the contract; FleetNodeHandler is a future consumer of that contract.
**Trade-offs:** Bridge design is not shaped by FleetNodeHandler's assumed lifecycle; may need adaptation when FleetNodeHandler is actually built. Adapting a clean SPI is easier than redesigning a bridge built against wrong assumptions.
**Sources:** claudony #246 (fleet YAML schema), #247 (FleetScriptRunner) — both aspirational, no code exists
**Depends on:** D2, D3
**Exploration:** quick
**Status:** revised — R1-06 correctly identified that FleetNodeHandler doesn't exist. Inverted the dependency: bridge exposes lifecycle SPI, FleetNodeHandler consumes it.

## D8: Message type mapping, streaming model, and inbound filtering

**Choice:** Commitment-aware buffer-then-post model with typed speech-act mapping and explicit inbound message filter
**Alternatives:**
- Stream-through (post STATUS for each TextDelta) — high message volume, poor signal-to-noise in the normative ledger
- Full buffer (single RESPONSE at end, no intermediate STATUS) — no governance visibility into tool use
- Uniform RESPONSE+DONE for all commitment types — normatively redundant for COMMAND/QUERY (RESPONSE already fulfills; DONE is a silent no-op on a terminal commitment)
**Rationale:** The bridge maps AgentEvent types to qhorus speech acts with commitment-aware terminal message selection.

**Inbound message filter (ChannelBackend.postTracked):**
The ChannelBackend receives ALL messages fanned out on the channel. Only obligation-creating types trigger agent invocation:
- COMMAND, QUERY, PROPOSE → trigger AgentSession.query(content) or AgentProvider.invoke(config)
- STATUS, RESPONSE, EVENT, DONE, FAILURE, DECLINE, HANDOFF → skip invocation. For persistent sessions, these are optionally queued as conversation context for the next agent turn (appended to the query prompt to give the agent awareness of other participants' activity).

**Outbound speech-act mapping (AgentEvent → qhorus MessageType):**
- AgentEvent.TextDelta → buffered until InvocationComplete, then posted as a single RESPONSE message
- AgentEvent.ToolCallComplete → posted as STATUS (tool invocation visibility for governance/oversight)
- AgentEvent.ToolResult → posted as STATUS (tool result for audit trail)
- AgentEvent.ThinkingDelta → not posted to channels (agent-internal reasoning; not normatively accountable)
- Agent invocation failure (exception, timeout) → posted as FAILURE (resolves commitment immediately)

**Commitment-aware terminal message (replaces uniform RESPONSE+DONE):**
The terminal message depends on the inbound commitment type:
- COMMAND/QUERY commitment: RESPONSE only. RESPONSE carries the agent's text output AND fulfills the commitment (verified: MessageService.java RESPONSE case calls commitmentService.fulfill() for non-PROPOSE commitments). InvocationComplete metadata (inputTokens, outputTokens, thinkingTokens, durationMs, totalCostUsd) is carried as dispatch telemetry on the RESPONSE message. No separate DONE — it would find a terminal commitment and be a normative no-op (CommitmentService.fulfill() filters on c.state().isActive()).
- PROPOSE commitment: RESPONSE (informational — the != MessageType.PROPOSE guard prevents fulfillment) then DONE (fulfills the commitment = accepts the proposal). The dual-message pattern IS correct for PROPOSE because RESPONSE and DONE have distinct normative effects.
- No commitment (no correlationId): RESPONSE for content. No DONE needed — there is no commitment to fulfill.

**Trade-offs:** Buffering TextDelta means no real-time visibility of agent text output in the channel. Commitment-aware mapping adds a conditional branch in the terminal message logic but eliminates normative noise (redundant DONE entries) and attestation asymmetry from the ledger.
**Sources:** AgentEvent sealed interface, MessageService.java RESPONSE/DONE commitment handling (lines 481-486), CommitmentService.fulfill() isActive() filter, qhorus MessageType taxonomy (ADR-0005), StoredCommitmentAttestationPolicy
**Exploration:** surfaced by review (R1-09, R1-10, R2-01, R2-02, R2-03)
**Status:** revised — R2-01 identified RESPONSE+DONE normative redundancy for COMMAND/QUERY. R2-02 identified missing PROPOSE as inbound type. R2-03 identified missing inbound message filter. Now commitment-aware with explicit filter policy.

## D9: MCP server exposure to bridged agents

**Choice:** Configurable per-binding — bridge allows specifying which MCP servers to provide when opening an AgentSession
**Alternatives:**
- Always expose qhorus MCP server — makes every bridged agent fully mesh-aware but harder to contain
- Never expose — agents can only respond to delivered messages via the bridge, no autonomous mesh operations
**Rationale:** The binding configuration (per agent-channel pairing) specifies which MCP servers the agent receives via AgentSessionInit.mcpServers(). Governance-critical agents (oversight, coordinator roles) get the qhorus MCP server for autonomous mesh operations. Task-focused agents get no MCP servers — they receive messages via ChannelBackend delivery and respond through the bridge. This is a governance decision, not a technical one — the AgentSessionInit API already supports it.
**Trade-offs:** Per-binding configuration adds complexity to the binding model; must be documented to prevent accidental over-privileging of agents
**Sources:** AgentSessionInit.mcpServers(), AgentMcpServer, MeshMcpTools
**Exploration:** surfaced by review (R1-11)
**Status:** captured

## D10: Commitment lifecycle on agent failure

**Choice:** Bridge sends FAILURE automatically on agent invocation failure; no automatic retry
**Alternatives:**
- Automatic retry with backoff — masks transient failures, delays commitment resolution
- Delegate via HANDOFF — passes to another agent, but the bridge has no agent selection logic
- Do nothing (let commitment expire) — delayed failure detection, poor normative audit trail
**Rationale:** When an AgentProvider invocation fails (timeout, subprocess crash, API error) and the invocation was servicing a COMMAND that opened a Commitment, the bridge posts FAILURE to the channel with a diagnostic message. This resolves the commitment immediately via the standard attestation flow (FLAGGED/0.6 per StoredCommitmentAttestationPolicy). No retry — retry policy is a higher-level concern for the requester or a supervisor agent.
**Trade-offs:** Immediate FAILURE on first error is aggressive for transient failures. Acceptable because commitment expiry handles the timeout case, and the bridge translates agent lifecycle events to speech acts faithfully.
**Sources:** CommitmentService, StoredCommitmentAttestationPolicy, A2AOutboundBackend (throws RuntimeException on failure)
**Exploration:** surfaced by review (R1-12)
**Status:** captured

## D11: Tenancy context propagation

**Choice:** Bridge propagates the channel's tenancyId when opening sessions and dispatching messages
**Alternatives:**
- Use agent's own tenancy context — incorrect when a multi-tenant agent serves multiple channels
- Hardcode default tenant — breaks multi-tenant deployments
**Rationale:** When the bridge opens an AgentSession or invokes AgentProvider for a channel message, the tenancy context must match the channel's tenancy. InboundTenancyContext (@RequestScoped) provides the current tenant from the HTTP request. For background/scheduled operations (session keepalive, reconnection), the bridge uses the channel's stored tenancyId directly, following the QhorusSystemCurrentPrincipal (@QhorusSystem) pattern for background contexts. All messages dispatched by the bridge carry the originating channel's tenancyId.
**Trade-offs:** The bridge must propagate tenancy through async operations (agent sessions span multiple requests); requires explicit tenancy in the session binding, not reliance on request-scoped context alone
**Sources:** InboundTenancyContext (runtime/identity/), QhorusSystemCurrentPrincipal, Channel.tenancyId, TenancyContextFilter
**Exploration:** surfaced by review (R1-13)
**Status:** captured

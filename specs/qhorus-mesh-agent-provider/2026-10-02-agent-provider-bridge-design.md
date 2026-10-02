# Qhorus Mesh — AgentProvider Bridge + MeshApi Migration

**Date:** 2026-10-02
**Status:** Draft
**Branch:** issue-465-qhorus-mesh
**Issue:** casehubio/qhorus#465

## Problem

The qhorus-mesh relay currently supports only MCP clients (Claude Code sessions calling `mesh_register`, `mesh_send_message`). Server-side agents created via the platform's `AgentProvider` SPI — Claude (direct SDK), OpenAI, Gemini, Codex, langchain4j backends — cannot participate in mesh channels. Additionally, the mesh tools use the `@Tool` annotation pattern instead of the platform's `@McpDomain` pattern, which generates REST + MCP endpoints from a single interface.

## Solution

Two deliverables:

1. **agent-bridge module** — a new classpath-activated optional module (like `a2a-outbound/`, `connector-backend/`) that bridges `AgentProvider`-managed agents into qhorus channels as first-class participants
2. **MeshApi migration** — refactor `MeshMcpTools` from `@Tool` to the `@McpDomain` pattern (`MeshApi` interface + `MeshService` impl)

## Architecture

```
┌─────────────────────────────────────────────────────┐
│ Channel                                              │
│  ChannelBackend (AT_LEAST_ONCE)                     │
│  ┌──────────────────────────────┐                   │
│  │ AgentProviderBackend         │                   │
│  │  • receives COMMAND/QUERY    │                   │
│  │  • delivers to AgentSession  │◄── fanOut/delivery│
│  │  • posts RESPONSE back      │                   │
│  │  • sender-based loop guard  │                   │
│  └──────────────────────────────┘                   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ AgentBackend (casehub-platform-agent-api)             │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐           │
│  │ Claude   │ │ OpenAI   │ │langchain4j│           │
│  │ (direct) │ │ (direct) │ │(ChatModel)│           │
│  └──────────┘ └──────────┘ └───────────┘           │
└─────────────────────────────────────────────────────┘
```

### Module Placement

`agent-bridge/` at the qhorus repo root, following the established bridge module pattern (`a2a-outbound/`, `connector-backend/`, `notification-bridge/`). Any Quarkus app embedding `casehub-qhorus` adds `agent-bridge/` to get AgentProvider-to-channel integration. The mesh application adds it as a dependency.

The bridge depends on `casehub-qhorus-api` + `casehub-platform-agent-api` — it does not depend on `mesh/` (which is an application, not a library).

### Package

`io.casehub.qhorus.agent.bridge`

## Bridge Components

### AgentProviderBackend

`ChannelBackend` implementation with `AT_LEAST_ONCE` delivery guarantee. Registered per channel when an agent binding is created. Implements `ChannelBackend` (base interface — not `AgentChannelBackend`, which is the injection point for `QhorusChannelBackend` in `ChannelGateway`). Returns `ActorType.AGENT` from `actorType()`. Same pattern as `A2AOutboundBackend`.

**Inbound message filter (postTracked):**
- COMMAND, QUERY, PROPOSE → queue for agent invocation (see §Async delivery below)
- All other types (STATUS, RESPONSE, EVENT, DONE, FAILURE, DECLINE, HANDOFF) → skip invocation. For persistent sessions, optionally queued as conversation context for the next agent turn.

**Target-based routing:** Before invoking, check `message.target()`:
- Non-null, non-blank target matching `binding.agentInstanceId()` → invoke
- Non-null, non-blank target NOT matching → skip (message is for a different agent)
- Null/blank target (broadcast) → invoke all managed agents on the channel

**Async delivery:** `postTracked()` returns `PostResult.ALL_DELIVERED` immediately after accepting the message into the bridge's internal queue. Agent invocation runs asynchronously on a virtual thread (`Thread.ofVirtual().start()`). This prevents blocking the `DeliveryBatchExecutor`'s `@Transactional` `deliverBatch()` loop — LLM invocations take seconds to minutes, which would hold the JTA transaction open past the timeout (default 300s), exhaust executor threads, and starve other AT_LEAST_ONCE backends.

**Concurrency control:** Each binding has a `Semaphore(binding.maxConcurrency())` that gates agent invocation:
- **Persistent sessions:** `maxConcurrency = 1` (enforced, not configurable). `AgentSession` maintains conversation state — concurrent `query()` calls would interleave turns and corrupt the conversation history. Messages queue in FIFO order and are dispatched sequentially as the semaphore permit becomes available.
- **Ephemeral invocations:** `maxConcurrency` configurable per binding (default 3). Each `AgentBackend.invoke()` call is stateless, so bounded concurrency is safe. The default of 3 balances throughput with LLM provider rate limits.
- When the semaphore is full, messages remain in the bridge's internal queue. The virtual thread blocks on `semaphore.acquire()`, not on the delivery pump. This provides natural backpressure without affecting the `DeliveryBatchExecutor`.

The AT_LEAST_ONCE guarantee applies to delivery TO the bridge, not processing BY the agent. Once the bridge acknowledges, the delivery pump advances its cursor. If the agent invocation fails, the bridge posts FAILURE (per §Failure Handling). If the application crashes mid-invocation, the in-flight invocation is lost — this is a known limitation of ephemeral bindings (the cursor was advanced but the agent hadn't completed). On restart, bindings are recreated and the pump continues from the advanced cursor.

Agent dispatch: `AgentSession.query(content)` for persistent sessions, `AgentBackend.invoke(config)` for ephemeral (selected via `Instance<AgentBackend>` matching `binding.backendKey()` to `backend.key()`).

**Loop prevention:**
- **Direct guard:** Set of managed agent instanceIds. If `message.sender()` matches a managed agent, return immediately. Same pattern as `A2AOutboundBackend.isExternalAgent()`. This prevents the primary loop vector: an agent's own RESPONSE triggering re-invocation of itself.

**Indirect loop limitation:** Indirect loops (A → X → B → Y → A via MCP tool-dispatched COMMANDs) are not caught by the direct guard. A cross-bridge loop detection mechanism requires propagating invocation context through `MessageDispatcher.dispatch()` — a cross-cutting infrastructure change that affects the core message dispatch path. This is deferred to a separate design (see casehubio/qhorus#468). Indirect loops are only possible when agents have MCP server access, which is restricted to governance-critical agents per §MCP Server Exposure. For v1, the direct guard plus MCP access control provides sufficient protection.

### Outbound Speech-Act Mapping

AgentEvent → qhorus MessageType:

| AgentEvent | MessageType | Notes |
|---|---|---|
| TextDelta | (buffered) | Accumulated until InvocationComplete |
| InvocationComplete | RESPONSE | Carries accumulated text + invocation telemetry |
| ToolCallComplete | STATUS | Tool invocation visibility for governance |
| ToolResult | STATUS | Tool result for audit trail |
| ToolCallDelta | (not posted) | Streaming partial args — full call arrives in ToolCallComplete |
| ThinkingDelta | (not posted) | Agent-internal reasoning |
| Failure/timeout | FAILURE | Resolves commitment immediately |

**`InvocationComplete.isError` with buffered text:** When `isError=true` and text has been buffered from prior `TextDelta` events, the bridge discards the buffered text and posts FAILURE. The partial output is included as diagnostic context in the FAILURE message's `payload` field (not as a separate RESPONSE). Rationale: an errored invocation's partial output is unreliable; posting it as a RESPONSE would create a commitment-fulfilling message for content the agent didn't complete successfully.

### Commitment-Aware Terminal Messages

The terminal message depends on the inbound commitment type:

- **COMMAND/QUERY commitment:** RESPONSE only. RESPONSE fulfills the commitment (via `CommitmentService.fulfill()` for non-PROPOSE commitments). No separate DONE — it would find a terminal commitment and be a normative no-op.
- **PROPOSE commitment:** RESPONSE (informational, does not fulfill) then DONE (fulfills = accepts the proposal). Dual-message pattern is correct because RESPONSE and DONE have distinct normative effects for PROPOSE.
- **No commitment (no correlationId):** RESPONSE for content. No DONE needed.

Invocation telemetry (inputTokens, outputTokens, thinkingTokens, durationMs, totalCostUsd) carried as dispatch telemetry on the RESPONSE message.

### Agent Binding

A binding associates an agent identity with a channel and an `AgentBackend` key:

```java
public record AgentChannelBinding(
    UUID id,
    UUID channelId,
    String agentInstanceId,
    String backendKey,          // "claude", "openai", "langchain4j"
    String agentBriefing,       // agent-specific instructions — component of the assembled systemPrompt
    List<String> mcpServers,    // which MCP servers the agent gets
    boolean persistent,         // persistent session or ephemeral
    int maxConcurrency,         // max concurrent invocations (enforced=1 for persistent, default 3 ephemeral)
    int contextWindowSize,      // max recent messages for volatile context (default 20)
    Map<String, String> metadata,
    String tenancyId
) {}
```

**Lifecycle SPI** — the bridge exposes binding operations as a stable contract:
- `createBinding(AgentChannelBinding)` — registers the agent, opens session if persistent, registers ChannelBackend
- `updateBinding(UUID bindingId, ...)` — field changes trigger different levels of lifecycle action (see below)
- `destroyBinding(UUID bindingId)` — close session, deregister backend, deregister instance

**Update trigger categories:**

| Field(s) | Action | Rationale |
|-----------|--------|-----------|
| `agentBriefing`, `mcpServers` | Session restart (close + reopen with new `AgentSessionInit`) | These are set at session open time — `AgentSessionInit.systemPrompt()` and `AgentSessionInit.mcpServers()` |
| `backendKey`, `persistent`, `channelId` | Full rebind (destroy + recreate) | Switches `AgentBackend`, session model, or channel — incompatible with the existing session |
| `contextWindowSize`, `maxConcurrency`, `metadata` | Hot update (no restart) | Affects per-turn query assembly or bridge-level config, not the agent session |

Fleet YAML `FleetNodeHandler` (future, #247) will be a consumer of this SPI.

**Binding persistence:** Bindings are ephemeral — held in an in-memory `ConcurrentHashMap<UUID, AgentChannelBinding>` within the bridge. There is no JPA entity or Flyway migration. Consumers (e.g., `FleetNodeHandler`) are responsible for recreating bindings on application startup via the Lifecycle SPI. This is intentional: the bridge is a runtime component, and the source of truth for which agents should be bound to which channels is the consumer's configuration (fleet YAML, programmatic setup), not a database table. Persistent sessions are closed on shutdown and reopened when bindings are recreated at startup.

### Cache-Aware Prompt Structuring

The bridge separates stable and volatile context:

**Stable (systemPrompt, assembled at session open, cached by Claude):**
- Agent briefing (from `binding.agentBriefing()`)
- Channel description and governance rules
- Peer list snapshot (agents in the channel at session open time)

**Volatile (per-turn query prompt):**
- The message to process (COMMAND/QUERY content)
- Recent channel activity since last turn (bounded context window: last 20 messages or 4000 tokens, whichever is smaller; configurable per binding via `AgentChannelBinding.contextWindowSize`)

**Session restart policy:** The bridge maintains a `channelId → Set<peerInstanceIds>` mapping, populated from `MembershipManager.listMembers()` at session open time. On `InstanceRegisteredEvent` or `InstanceDeregisteredEvent`, the bridge checks whether the event's `instanceId` appears in any channel's peer set. If so, it queries `MembershipManager.listMembers()` for the affected channels, compares the new peer set to the cached one, and restarts only those sessions where the peer set actually changed (join/leave, not identical re-registration). Re-registrations with identical capabilities (see §Significant change filter) are skipped entirely. Peer changes are infrequent relative to message frequency.

**Significant change filter:** `InstanceRegisteredEvent` carries `previousCapabilities` and `currentCapabilities`. The bridge skips the event when `previousCapabilities.equals(currentCapabilities)` — this filters out heartbeat re-registrations. A new instance has `previousCapabilities = []` and always triggers evaluation. A deregistered instance always triggers evaluation.

### Failure Handling

When `AgentBackend.invoke()` or `AgentSession.query()` fails:
- Bridge posts FAILURE to the channel with diagnostic content (error type, message)
- Commitment (if any) resolves immediately via standard attestation flow (FLAGGED/0.6)
- No automatic retry — retry policy is a higher-level concern for the requester or supervisor agent

### Tenancy Context Propagation

The bridge propagates the channel's `tenancyId`:
- Session open: binding carries tenancyId from the channel
- Message dispatch: all messages dispatched by the bridge carry the originating channel's tenancyId
- Background operations (keepalive, reconnection): use channel's stored tenancyId directly, following the `QhorusSystemCurrentPrincipal` pattern

### MCP Server Exposure

Configurable per binding via `AgentChannelBinding.mcpServers`:
- Governance-critical agents (oversight, coordinator roles) get the qhorus MCP server for autonomous mesh operations
- Task-focused agents get no MCP servers — they receive via ChannelBackend and respond via the bridge
- This is a governance decision, not a technical one

## MeshApi Migration

### Before (current)

```java
@ApplicationScoped
public class MeshMcpTools {
    @Tool(description = "Register this session...")
    public String mesh_register(...) { ... }
}
```

### After

```java
// mesh/src/main/java/io/casehub/qhorus/mesh/MeshApi.java
@McpDomain(value = "qhorus/mesh", app = "qhorus-mesh",
           summary = "Mesh relay — register, discover peers, send messages")
public interface MeshApi {

    @PlatformMutation("Register a session with the mesh relay")
    MeshRegistration meshRegister(String instanceId, String description,
                                  Map<String, String> metadata);

    @PlatformMutation("Deregister a session from the mesh relay")
    void meshDeregister(String instanceId);

    @PlatformMutation("Send a message to a channel")
    MessageResult meshSendMessage(String channel, String sender,
                                   String type, String content);

    @PlatformQuery("Check messages in a channel")
    List<MessageSummary> meshCheckMessages(String channel, Long afterId);

    @PlatformMutation("Create a channel with metadata")
    ChannelDetail meshCreateChannel(String name, Map<String, String> metadata);

    @PlatformQuery("List channels filtered by metadata")
    List<ChannelDetail> meshListChannels(String metadataKey, String metadataValue);

    @PlatformQuery("Discover peers by metadata")
    List<PeerInfo> meshDiscoverPeers(String metadataKey, String metadataValue);
}
```

### Return Types

```java
public record MeshRegistration(UUID id, String instanceId, String description) {}

public record MessageResult(Long messageId, String channel, MessageType type) {}

public record ChannelDetail(UUID id, String name, Map<String, String> metadata) {}

public record MessageSummary(Long id, String sender, MessageType type, String content) {}

public record PeerInfo(String instanceId, String description, Map<String, String> metadata) {}
```

```java
// mesh/MeshService.java — implements MeshApi
@ApplicationScoped
public class MeshService implements MeshApi {
    @Inject InstanceService instanceService;
    @Inject ChannelService channelService;
    @Inject MessageDispatcher messageDispatcher;
    // ... implementation (moved from MeshMcpTools)
}
```

The `@McpDomain` generator produces REST and MCP endpoints from `MeshApi`. No `@Tool` annotations needed.

## Testing

### Bridge component tests (CDI-free)
- `AgentProviderBackendTest` — postTracked delivery, loop guard, message type filtering
- `SpeechActMappingTest` — AgentEvent → MessageType mapping, commitment-aware terminal messages
- `BindingLifecycleTest` — create/update/destroy binding, session restart on agentBriefing change

### Integration tests (@QuarkusTest)
- `AgentBridgeIntegrationTest` — end-to-end: create binding, send COMMAND to channel, verify agent receives via postTracked, verify RESPONSE dispatched back
- `LoopGuardIntegrationTest` — verify sender-based direct guard, target filtering, async queue acceptance
- `CacheStructureTest` — verify assembled systemPrompt / volatile query separation for persistent sessions

### MeshApi migration tests
- `MeshServiceTest` — unit test: verify each `MeshApi` method delegates correctly to `InstanceService`, `ChannelService`, `MessageDispatcher` (same behavior as `MeshMcpTools`, different return types)
- `MeshApiEndpointTest` (@QuarkusTest) — verify `@McpDomain` generates REST endpoints, verify MCP tool registration, verify old `@Tool` endpoints are removed

### SSE transport test
- One smoke test connecting an SSE client to the mesh relay, subscribing to a channel, verifying push events arrive when a bridged agent responds

## Dependencies

```
agent-bridge/
├── casehub-qhorus-api          (ChannelBackend, MembershipManager, MessageObserver, MessageDispatcher)
├── casehub-platform-agent-api  (AgentBackend, AgentSession, AgentSessionInit, AgentSessionConfig)
└── (test) casehub-qhorus-testing, casehub-qhorus-persistence-memory
```

The mesh application (`mesh/pom.xml`) adds `casehub-qhorus-agent-bridge` as a dependency.

## References

- `api/src/main/java/io/casehub/qhorus/api/gateway/ChannelBackend.java` — ChannelBackend SPI (base interface for bridge backends)
- `api/src/main/java/io/casehub/qhorus/api/gateway/AgentChannelBackend.java` — QhorusChannelBackend injection point (NOT for bridge backends)
- `api/src/main/java/io/casehub/qhorus/api/gateway/MessageObserver.java` — observer contract
- `a2a-outbound/src/main/java/.../A2AOutboundBackend.java` — AT_LEAST_ONCE bridge pattern, sender-based loop guard
- `runtime-core/src/main/java/.../message/RoutingBridge.java` — role:X → agent resolution
- `casehub-platform/agent-api/src/main/java/.../AgentBackend.java` — per-provider backend interface with `key()` routing
- `casehub-platform/agent-claude-core/src/main/java/.../ClaudeAgentProvider.java` — Claude direct SDK backend
- `casehub-platform/agent-langchain4j-core/src/main/java/.../ChatModelAgentProvider.java` — langchain4j catch-all
- `api/src/main/java/io/casehub/qhorus/api/spi/channels/ChannelsApi.java` — @McpDomain pattern reference
- `mesh/src/main/java/io/casehub/qhorus/mesh/MeshMcpTools.java` — current @Tool pattern to migrate
- claudony #246 — declarative fleet deployment (FleetNodeHandler, YAML schema)
- claudony #247 — standalone fleet script runner (FleetScriptRunner)

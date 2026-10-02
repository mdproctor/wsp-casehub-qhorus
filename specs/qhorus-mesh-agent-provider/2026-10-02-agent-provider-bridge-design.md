# Qhorus Mesh — AgentProvider Bridge + MeshApi Migration

**Date:** 2026-10-02
**Status:** Draft
**Branch:** issue-465-qhorus-mesh
**Issue:** casehubio/qhorus#465

## Problem

The qhorus-mesh relay currently supports only MCP clients (Claude Code sessions calling `mesh_register`, `mesh_send_message`). Server-side agents created via the platform's `AgentProvider` SPI — Claude (direct SDK), OpenAI, Gemini, Codex, langchain4j backends — cannot participate in mesh channels. Additionally, the mesh tools use the `@Tool` annotation pattern instead of the platform's `@McpDomain` pattern, which generates REST + GraphQL + MCP endpoints from a single interface.

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
│                                                      │
│  MessageObserver (LOCAL)                             │
│  ┌──────────────────────────────┐                   │
│  │ AgentInitiatedObserver       │                   │
│  │  • agent sends STATUS/EVENT  │                   │
│  │  • proactive alerts          │──► dispatch()     │
│  └──────────────────────────────┘                   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ AgentProvider (casehub-platform)                     │
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

`AgentChannelBackend` implementation with `AT_LEAST_ONCE` delivery guarantee. Registered per channel when an agent binding is created.

**Inbound message filter (postTracked):**
- COMMAND, QUERY, PROPOSE → trigger `AgentSession.query(content)` or `AgentProvider.invoke(config)`
- All other types (STATUS, RESPONSE, EVENT, DONE, FAILURE, DECLINE, HANDOFF) → skip invocation. For persistent sessions, optionally queued as conversation context for the next agent turn.

**Loop prevention — two layers:**
1. **Direct guard:** Set of managed agent instanceIds. If `message.sender()` matches a managed agent, return immediately. Same pattern as `A2AOutboundBackend.isExternalAgent()`.
2. **Indirect guard:** Per-bridge-instance `AtomicInteger` tracking in-flight invocations. If count exceeds `max-depth` (default 3), refuse invocation with WARN log. Catches cross-channel indirect loops. Uses `AtomicInteger` (not ThreadLocal) because AT_LEAST_ONCE delivery runs on the `DeliveryService` pump thread asynchronously.

### Outbound Speech-Act Mapping

AgentEvent → qhorus MessageType:

| AgentEvent | MessageType | Notes |
|---|---|---|
| TextDelta | (buffered) | Accumulated until InvocationComplete |
| InvocationComplete | RESPONSE | Carries accumulated text + invocation telemetry |
| ToolCallComplete | STATUS | Tool invocation visibility for governance |
| ToolResult | STATUS | Tool result for audit trail |
| ThinkingDelta | (not posted) | Agent-internal reasoning |
| Failure/timeout | FAILURE | Resolves commitment immediately |

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
    String systemPrompt,        // stable context for cache
    List<String> mcpServers,    // which MCP servers the agent gets
    boolean persistent,         // persistent session or ephemeral
    Map<String, String> metadata,
    String tenancyId
) {}
```

**Lifecycle SPI** — the bridge exposes binding operations as a stable contract:
- `createBinding(AgentChannelBinding)` — registers the agent, opens session if persistent, registers ChannelBackend
- `updateBinding(UUID bindingId, ...)` — update systemPrompt, mcpServers, etc. (session restart if systemPrompt changes)
- `destroyBinding(UUID bindingId)` — close session, deregister backend, deregister instance

Fleet YAML `FleetNodeHandler` (future, #247) will be a consumer of this SPI.

### Cache-Aware Prompt Structuring

The bridge separates stable and volatile context:

**Stable (systemPrompt, set at session open, cached by Claude):**
- Agent briefing (from binding)
- Channel description and governance rules
- Peer list snapshot (agents in the channel at session open time)

**Volatile (per-turn query prompt):**
- The message to process (COMMAND/QUERY content)
- Recent channel activity since last turn (optional context window)

**Session restart policy:** On significant peer topology changes (agent join/leave via `InstanceRegisteredEvent`/`InstanceDeregisteredEvent`), the bridge closes and reopens the session with updated systemPrompt. Peer changes are infrequent relative to message frequency.

### Failure Handling

When `AgentProvider.invoke()` or `AgentSession.query()` fails:
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
// api/spi/mesh/MeshApi.java
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

The `@McpDomain` generator produces REST, GraphQL, and MCP endpoints from `MeshApi`. No `@Tool` annotations needed.

## Testing

### Bridge component tests (CDI-free)
- `AgentProviderBackendTest` — postTracked delivery, loop guard, message type filtering
- `SpeechActMappingTest` — AgentEvent → MessageType mapping, commitment-aware terminal messages
- `BindingLifecycleTest` — create/update/destroy binding, session restart on systemPrompt change

### Integration tests (@QuarkusTest)
- `AgentBridgeIntegrationTest` — end-to-end: create binding, send COMMAND to channel, verify agent receives via postTracked, verify RESPONSE dispatched back
- `LoopGuardIntegrationTest` — verify sender-based and depth-based loop guards
- `CacheStructureTest` — verify systemPrompt/query separation for persistent sessions

### SSE transport test
- One smoke test connecting an SSE client to the mesh relay, subscribing to a channel, verifying push events arrive when a bridged agent responds

## Dependencies

```
agent-bridge/
├── casehub-qhorus-api          (ChannelBackend, MessageObserver, MessageDispatcher)
├── casehub-platform-agent-api  (AgentProvider, AgentBackend, AgentSession)
└── (test) casehub-qhorus-testing, casehub-qhorus-persistence-memory
```

The mesh application (`mesh/pom.xml`) adds `casehub-qhorus-agent-bridge` as a dependency.

## References

- `api/src/main/java/io/casehub/qhorus/api/gateway/AgentChannelBackend.java` — ChannelBackend SPI
- `api/src/main/java/io/casehub/qhorus/api/gateway/MessageObserver.java` — observer contract
- `a2a-outbound/src/main/java/.../A2AOutboundBackend.java` — AT_LEAST_ONCE bridge pattern, sender-based loop guard
- `runtime-core/src/main/java/.../message/RoutingBridge.java` — role:X → agent resolution
- `casehub-platform/agent-api/src/main/java/.../AgentProvider.java` — platform agent SPI
- `casehub-platform/agent-api/src/main/java/.../AgentBackend.java` — backend routing SPI
- `casehub-platform/agent-claude-core/src/main/java/.../ClaudeAgentProvider.java` — Claude direct SDK backend
- `casehub-platform/agent-langchain4j-core/src/main/java/.../ChatModelAgentProvider.java` — langchain4j catch-all
- `api/src/main/java/io/casehub/qhorus/api/spi/channels/ChannelsApi.java` — @McpDomain pattern reference
- `mesh/src/main/java/io/casehub/qhorus/mesh/MeshMcpTools.java` — current @Tool pattern to migrate
- claudony #246 — declarative fleet deployment (FleetNodeHandler, YAML schema)
- claudony #247 — standalone fleet script runner (FleetScriptRunner)

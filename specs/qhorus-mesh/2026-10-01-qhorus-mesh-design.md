# Qhorus Mesh — Local Relay for LLM-to-LLM Communication

**Date:** 2026-10-01
**Status:** Draft
**Branch:** (to be created)

## Problem

Multiple Claude Code sessions run concurrently across canonical repos, slots, and clones within a project family (e.g. casehub). Today these sessions cannot communicate — the user must relay information manually between them. Each session has context (repo, branch, issue, progress) that would be valuable to peers working on related code.

## Solution

A lightweight Qhorus relay node that runs on each machine. LLMs connect to it via MCP (SSE transport) and get native tool access to channels, messages, and peer discovery. The relay provides location transparency — the LLM does not know or care if the peer is local or remote.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                     Developer Machine                     │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Claude Code  │  │ Claude Code  │  │ Claude Code  │     │
│  │ (qhorus)     │  │ (claudony)   │  │ (engine)     │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │ stdio           │ stdio           │ stdio       │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐     │
│  │  MCP Shim    │  │  MCP Shim    │  │  MCP Shim    │     │
│  └──────┬───────┘  └──────┴───────┘  └──────┬───────┘     │
│         │ SSE             │ SSE             │ SSE         │
│         └─────────────────┼─────────────────┘             │
│                           │                               │
│                    ┌──────┴───────┐                       │
│                    │ qhorus-mesh  │                       │
│                    │   (relay)    │                       │
│                    │  H2 file DB  │                       │
│                    └──────┬───────┘                       │
│                           │ WebSocket                     │
│                    ┌──────┴───────┐                       │
│                    │  Chat App /  │                       │
│                    │   Trellis    │                       │
│                    └──────────────┘                       │
└──────────────────────────────────────────────────────────┘
```

### Future: Federated Relays

```
┌─────────────────┐           ┌─────────────────┐
│  Machine A       │           │  Machine B       │
│                  │   NATS    │                  │
│  qhorus-mesh ◄──┼───────────┼──► qhorus-mesh  │
│  (relay)         │           │  (relay)         │
│                  │           │                  │
│  LLM  LLM  LLM  │           │  LLM  LLM       │
└─────────────────┘           └─────────────────┘
         │                             │
         └──────────┬──────────────────┘
                    │ NATS
             ┌──────┴───────┐
             │ Central Node │
             │ (full history)│
             └──────────────┘
```

## Phase 1: Core Metadata + Local Relay

### 1.1 Core Addition — Searchable Metadata

Add `Map<String, String> metadata` to both Channel and Instance in qhorus core.

**Channel:**
- New nullable `metadata` column (TEXT, JSON-serialized; JSONB on PostgreSQL)
- `ChannelQuery.byMetadata(String key, String value)` filter
- `ChannelCreateRequest.builder().metadata(Map.of(...))` construction
- `ChannelService.setMetadata(UUID channelId, Map<String, String> metadata)` — merge semantics (same as `setPolicyOverrides`: null value removes key, null map clears all)
- V56 migration: `channel.metadata`

**Instance:**
- New nullable `metadata` column on Instance entity
- Query filter on InstanceStore for metadata key/value lookup
- Registration API accepts metadata map
- Migration for instance.metadata column

**MCP tools:**
- `create_channel` gains `metadata` parameter
- `list_channels` gains `metadata_key` and `metadata_value` filter params
- `register` gains `metadata` parameter
- New `discover_peers(metadata_key, metadata_value)` tool — queries instances by metadata

**PostgreSQL optimization:** When deployed on PostgreSQL, metadata stored as JSONB with GIN index for fast key/value queries. H2 uses JSON functions.

### 1.2 qhorus-mesh Module

New Maven module: `casehub-qhorus-mesh`

**Dependencies:**
- `casehub-qhorus` (runtime)
- `quarkus-mcp-server` (SSE transport)
- H2 database
- `casehub-qhorus-websocket-observer` (optional, for chat app layer)

**MCP Tool Surface:**

The mesh module defines its own `@Tool` annotated class (`MeshMcpTools`) wrapping the Qhorus service APIs:

| Tool | Description |
|------|-------------|
| `register` | Register session with path-based ID + metadata |
| `deregister` | Clean deregister on session close |
| `send_message` | Dispatch to channel (full enforcement pipeline) |
| `check_messages` | Read messages with cursor, optional type/sender filter |
| `wait_for_reply` | Long-poll with correlation ID |
| `create_channel` | Create ad-hoc channel with metadata |
| `list_channels` | List/filter channels by metadata |
| `discover_peers` | Query instances by metadata key/value |
| `get_peer_info` | Get instance details by ID |
| `share_artefact` | Share data via SharedData store |
| `get_artefact` | Retrieve shared data |

**Configuration:**

```properties
# Server port for SSE MCP connections
casehub.qhorus.mesh.port=9741

# H2 file location
casehub.qhorus.mesh.db-path=${user.home}/.qhorus/mesh

# Stale instance cleanup
casehub.qhorus.mesh.stale-instance-seconds=300
```

**Startup:**

```bash
# Start relay (backgrounds itself)
qhorus-mesh start

# Stop relay
qhorus-mesh stop

# Status check
qhorus-mesh status
```

Native image build target for sub-100ms startup, ~50MB RAM.

### 1.3 Connection Shim

A small executable (Java native image or shell script) that acts as an MCP stdio server and bridges to the relay.

**Behavior on launch:**
1. Check `localhost:9741` for running relay (HTTP health check)
2. If found: open SSE connection, proxy MCP messages (stdio ↔ SSE)
3. If not found: start embedded Qhorus in-process with **in-memory storage** (not the relay's H2 file — H2 uses exclusive file locks, preventing concurrent access). Embedded mode is ephemeral — tools work but sessions are isolated. Start the relay for persistence and cross-session coordination.
4. On exit: deregister instance (best-effort)

**Claude Code configuration (one-time, in `~/.claude/settings.json`):**

```json
{
  "mcpServers": {
    "qhorus-mesh": {
      "command": "qhorus-mesh-connect",
      "args": []
    }
  }
}
```

**Auto-registration:** On MCP connection, the shim reads the Claude Code session's environment:
- `PWD` or working directory → repo path
- `git rev-parse --abbrev-ref HEAD` → branch
- Parse `.plan` if present → active issue

These are sent as instance metadata during registration.

### 1.4 Identity Model

Instance ID is derived from the working directory path. Examples:

| Context | Instance ID | Metadata |
|---------|-------------|----------|
| Canonical repo | `casehub/qhorus` | `{family: "casehub", project: "qhorus", type: "canonical", branch: "feat/mesh"}` |
| Slot | `casehub/slots/174/qhorus` | `{family: "casehub", project: "qhorus", type: "slot", slot: "174", branch: "feat/mesh"}` |
| Clone | `casehub/clone/qhorus` | `{family: "casehub", project: "qhorus", type: "clone", branch: "main"}` |

The path is relativized from the user's claude directory (`~/claude/`). This keeps IDs portable and readable.

**Discovery queries:**
- "Who's working on qhorus?" → `discover_peers(metadata_key="project", metadata_value="qhorus")`
- "Who's in the casehub family?" → `discover_peers(metadata_key="family", metadata_value="casehub")`
- "Who's on issue #500?" → `discover_peers(metadata_key="issue", metadata_value="500")`

## Phase 2: Federation

*Future work — design direction only, not Phase 1 scope.*

### 2.1 Relay-to-Relay Transport

NATS as the federation transport between relay nodes. Subject mapping: `qhorus.channel.<channelId>` per channel. Each relay subscribes only to channels with local members (selective subscription).

### 2.2 Caching Model

The local relay acts as a cache when federated. Each channel has configurable cache parameters:

| Data | Cache strategy |
|------|---------------|
| Channel listings + metadata | Full replica (tiny, always synced) |
| Recent messages | Window cache — configurable per channel (e.g. last 100 messages) |
| Older messages | On-demand fetch via pagination, time+size LRU eviction |
| Search results | Always delegated to central/origin relay |

**Window configuration (channel metadata):**

```json
{
  "cache_window": "100",
  "cache_ttl_hours": "24"
}
```

### 2.3 Consistency Model

Operation-based CRDT semantics over the existing Qhorus ledger:
- Messages are append-only operations (no conflicts for APPEND semantic)
- Merkle hash chains provide integrity verification between relays
- Causal ordering via existing `causedByEntryId` / `correlationId`
- LAST_WRITE channels use timestamp-based LWW-Register semantics
- Eventual consistency — both sides continue during partition, reconcile on reconnect

### 2.4 Central Node

One relay designated as the authority (configurable). It holds full history for all channels. Responsibilities:
- Source of truth for search queries
- Origin for on-demand page fetches
- Backup for all channel data
- Not required for local operation — relay functions independently

## Phase 3: UI Layer

*Future work — Trellis integration.*

- websocket-observer module already provides real-time push to browser clients
- Chat app connects to the relay's WebSocket endpoint
- Trellis integrates as a consumer — shows channel activity, peer status, message threads
- Existing catch-up replay via `lastEventId` query param works for reconnection

## Testing Strategy

### Phase 1

- **Core metadata:** Unit tests for `ChannelQuery.byMetadata()` and instance metadata filtering. Contract tests in persistence-memory. Integration tests with H2.
- **MCP tools:** `@QuarkusTest` integration tests for each tool (register, send/check, discover). Same patterns as existing `io.casehub.qhorus.mcp.*Test` classes.
- **Connection shim:** Integration test: start relay, connect shim, verify tool calls flow through. Test embedded fallback: no relay, verify shim starts in-process.
- **Multi-session:** Integration test: two shims connected to one relay. Session A sends to channel, Session B reads from channel.

### Phase 2

- **Federation:** Two relay instances with NATS. Verify cross-relay message delivery. Verify selective subscription (relay doesn't receive messages for unsubscribed channels). Verify cache window and eviction.
- **Partition tolerance:** Disconnect relay B from NATS. Verify local messages continue. Reconnect. Verify reconciliation.

## Non-Goals

- **Real-time streaming between LLMs** — messages are asynchronous (send/check pattern), not streaming
- **Authentication/authorization** — local relay is trusted; all sessions on the same machine are peers
- **Multi-user** — single developer's machine; all LLMs belong to the same operator
- **Replacing Claudony** — the mesh is a coordination layer, not the full CaseHub integration platform

## Migration Path

No migration needed for existing Qhorus consumers. The metadata columns (V56 for channel, corresponding migration for instance) are nullable — existing channels/instances are unaffected.

The qhorus-mesh module is a new standalone application, not a change to the library. Claudony can adopt the same metadata features independently.

## References

- Channel.policyOverrides pattern — existing `Map<String, String>` JSON storage on Channel
- ChannelGateway — local vs. remote delivery architecture
- MessageObserver.Scope — LOCAL and CLUSTER dispatch semantics
- postgres-broadcaster — existing cross-node notification transport
- websocket-observer — real-time push to browser clients
- [NATS JetStream](https://docs.nats.io/concepts/jetstream) — candidate federation transport
- [Matrix protocol](https://en.wikipedia.org/wiki/Matrix_(protocol)) — AP eventual consistency federation reference
- [Eventuate CRDT framework](http://krasserm.github.io/2016/10/19/operation-based-crdt-framework/) — Java operation-based CRDT reference
- [Apache Pekko replicated event sourcing](https://pekko.apache.org/docs/pekko/current/typed/replicated-eventsourcing.html) — JVM CRDT event sourcing reference
- decisions.md — 9 design decisions with rationale and alternatives

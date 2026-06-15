# Channel Backend Families — Three-Family Architecture

**Date:** 2026-06-15
**Status:** Approved — pending implementation plan
**Replaces:** casehubio/qhorus#261 (scope reduced; see below)

---

## Context

Issue #261 proposed a `casehub-qhorus-slack-channel` module with Slack-specific entities
(BotBinding, ThreadCache, normaliser, REST binding resource). The insight: the binding and
threading pattern is entirely generic across Slack, Zulip, Gitter, Teams — only the wire
format and client are platform-specific.

At the same time, exploring the design surfaced a third integration family (model endpoints)
and a need to name the existing `connector-backend` family clearly. This spec covers all
three families.

---

## Three-Family Architecture

External integrations fall into three genuinely distinct categories. The distinction is not
arbitrary — each family has a different capability profile, credential model, and threading
semantics.

| Family | Examples | Threading | Credentials | Actor type | Direction |
|--------|----------|-----------|-------------|------------|-----------|
| **Push** | Email, SMS, webhook | None | In connector | System | Outbound-primary |
| **Chat** | Slack, Zulip, Gitter, Teams | Platform thread handle | Per channel in Qhorus | Human | Bidirectional |
| **Model** | Claude, OpenAI, Ollama | Conversation history | Per channel in Qhorus | Agent | Bidirectional |

**Decision rule for new integrations:**
- Does it need conversational threading and per-channel credentials? → Chat family
- Does it answer queries or run agentic sessions? → Model family
- Does it fire content at a destination without needing a reply thread? → Push family

---

## Module Structure (Approach A)

```
casehub-qhorus/
├── connector-backend/     ← push family (existing; label clarified, no code change)
├── chat-channel/          ← new generic: BotBinding, ThreadCache, BotChannelBackend
├── model-channel/         ← new: ChatModel backend + AgentProvider backend
├── slack-channel/         ← thin adapter on chat-channel (replaces original #261 scope)
└── (zulip-channel, ...)   ← future thin adapters, same pattern
```

**Dependency graph:**

```
slack-channel    ──▶  chat-channel
zulip-channel    ──▶  chat-channel
model-channel    ──▶  casehub-platform-agent-api  (AgentProvider SPI)
                 ──▶  langchain4j-core             (ChatModel interface only)
connector-backend ──▶  casehub-connectors-core     (unchanged)
```

---

## Push Family — connector-backend

No code changes. The existing `casehub-qhorus-connector-backend` artifact serves this family.
Documentation and CLAUDE.md updated to name its role explicitly: notification-style delivery
(email, SMS, webhook, generic HTTP) via `ConnectorService`. Not the right home for
chat-platform integrations.

---

## Chat Family — chat-channel Generic Module

### BotBinding — JOINED inheritance

Base table `bot_binding`:

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| channel_id | UUID FK | |
| adapter_type | VARCHAR | discriminator: "slack", "zulip", etc. |
| enabled | BOOLEAN | |
| created_at | TIMESTAMP | |

Each adapter adds a joined sub-table with platform-specific columns:
- `slack_bot_binding(bot_token, slack_channel_id, workspace_id)`
- `zulip_bot_binding(api_key, org_url, stream_name)`

Follows the same JOINED inheritance pattern as `MessageLedgerEntry`. Base migration V23
(next available per CLAUDE.md); each adapter owns its sub-table migration in its own module.

### ThreadCache

Generic — no platform knowledge.

| Column | Type | Notes |
|--------|------|-------|
| channel_id | UUID | PK part |
| correlation_id | UUID | PK part |
| platform_thread_handle | VARCHAR | Slack: thread_ts · Zulip: topic name · etc. |
| created_at | TIMESTAMP | |

PK: `(channel_id, correlation_id)`. One handle per conversation per channel. Migration V23 (same file as `bot_binding`).

### BotChannelBackend — abstract base

```java
abstract class BotChannelBackend<B extends BotBinding, P>
        implements HumanParticipatingChannelBackend {

    // Concrete — lifecycle
    void onChannelInitialised(ChannelInitialisedEvent)
        // register self if BotBinding exists for channel
    void open(ChannelRef, Map<String,String> metadata)
    void close(ChannelRef)

    // Concrete — outbound (called by ChannelGateway fanout)
    void post(ChannelRef, OutboundMessage)
        // lookup B binding → lookup thread handle from ThreadCache
        // → deliver(message, binding, threadHandle)

    // Protected — adapter webhook resource calls normalise() then this
    void route(InboundHumanMessage msg, ChannelRef channel, String threadHandle)
        // → update ThreadCache → gateway.receiveHumanMessage()

    // Abstract — adapter implements
    InboundHumanMessage normalise(P payload)   // P is adapter-specific payload type
    void deliver(OutboundMessage message, B binding, String threadHandle)
    Optional<String> resolveInboundThreadHandle(P payload)
}
```

Inbound traffic arrives via the adapter's own webhook endpoint (e.g. `SlackWebhookResource`),
which calls `onInbound()`. The generic handles Qhorus routing; the adapter handles
normalisation and delivery only.

### BotBindingResource — base REST

`GET /{channel}/binding` and `DELETE /{channel}/binding` live in the base class.
`POST`/`PUT` are adapter-specific — request body shape differs per platform. Adapters extend
and add a typed create/update endpoint.

---

## Model Family — model-channel Module

Two backend implementations in one module.

### ModelBinding entity

Single table — no JOINED inheritance needed (both backends share the same config shape):

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| channel_id | UUID FK | |
| backend_type | VARCHAR | "chat" or "agent" |
| system_prompt | TEXT | |
| max_history_depth | INT | messages to include as context |
| enabled | BOOLEAN | |

Migration V24 (after chat-channel's V23).

### ChatModelChannelBackend

Injects `@Any Instance<ChatModel>` (optional — warns and no-ops if absent). On QUERY:

1. Fetch last `max_history_depth` messages from `MessageStore` by correlationId
2. Build `ChatRequest` (system prompt + history + new message)
3. `chatModel.chat(request)` → post RESPONSE to channel

Stateless per call. When platform#100 ships (`ChatModel` adapter backed by `AgentSession`),
prompt caching arrives automatically via classpath presence — no changes to this backend.

### AgentChannelBackend

Injects `AgentProvider`. Manages `ConcurrentHashMap<UUID, AgentSession>` keyed by
correlation ID.

- **First QUERY for a correlation**: open session with system prompt → query
- **Subsequent QUERYs**: send only the new message to existing session (history held in subprocess)
- **DONE / FAILURE / DECLINE**: close and remove session from map
- **Crash recovery** (`AgentProcessException`): rebuild history from `MessageStore` →
  open fresh session → replay. History is durable in Qhorus; crash recovery does not require
  platform#100 to solve it internally.

Session lifetime is scoped to a Qhorus correlation ID — survives across HTTP requests.
(Noted in platform#100 comment as a requirement for the multi-turn `AgentSession` API.)

### Sender identity

Both backends dispatch as `"system:model-channel"` with `ActorType.AGENT`. Consistent with
`"system:connector:{id}"` pattern in `ConnectorQhorusMeshBridge`. Not a registered Qhorus
instance.

### ModelBindingResource

`POST/PUT /{channel}/binding` — typed create/update (backend type, system prompt, max
history depth). `GET` and `DELETE` follow the same base pattern as `BotBindingResource`.

---

## Slack Adapter — slack-channel (revised #261 scope)

Thin layer on `chat-channel`. Implements the three abstract methods:

- `normalise(payload)` — maps Slack webhook event to `InboundHumanMessage`
- `deliver(message, SlackBotBinding, threadHandle)` — calls `SlackBotClient.postMessage()`
  with `thread_ts` for reply threading
- `resolveInboundThreadHandle(payload)` — extracts Slack `thread_ts`

Adds:
- `SlackBotBinding extends BotBinding` — `(bot_token, slack_channel_id, workspace_id)` columns
- `SlackWebhookResource` — receives Slack event POST, calls `onInbound()`
- `SlackBindingResource extends BotBindingResource` — typed create/update
- Flyway migration for `slack_bot_binding` table

Scale: S · Low (most work is in the generic `chat-channel`).

---

## Issue Changes

| Issue | Change |
|-------|--------|
| #261 | Scope reduced to thin adapter; blocked by new chat-channel issue |
| New (qhorus) | `chat-channel` generic module — M · Med · prerequisite for #261 |
| New (qhorus) | `model-channel` module — M · Med |
| parent#241 | Expand scope: add three-family architecture and new module structure |
| New (parent) | Document three-family channel backend pattern in PLATFORM.md |
| casehubio/connectors | Heads-up issue: two-family model; Zulip/Gitter = bot clients, not ConnectorService connectors |
| claudony | Heads-up issue: model-channel is coming; use it rather than ad-hoc LLM wiring |

---

## Dependencies and Sequencing

```
chat-channel (new issue)
    └── slack-channel (#261, revised)
    └── zulip-channel (future)

platform#58  (multi-turn AgentSession)
    └── platform#100 (ChatModel adapter + caching)
            └── model-channel benefits automatically (no code change)

model-channel (new issue) — can start now; caching arrives later via platform#100
```

`model-channel` can be built immediately against `ChatModel` and the current single-shot
`AgentProvider`. The persistent-session caching improvement arrives when platform#100 ships.

---

## References

- platform#55 — `casehub-platform-agent` (AgentProvider SPI) — CLOSED, shipped
- platform#58 — multi-turn `AgentSession` API — OPEN
- platform#100 — `ChatModel` adapter backed by `AgentSession` — OPEN
- casehubio/qhorus#261 — original slack-channel issue (scope revised by this spec)
- `docs/specs/2026-04-13-qhorus-design.md` — primary design spec

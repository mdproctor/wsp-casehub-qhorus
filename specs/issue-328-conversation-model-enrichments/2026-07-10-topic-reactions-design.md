# Topic and Reactions Design — Session 1 of #328

**Date:** 2026-07-10
**Epic:** #328 (conversation model enrichments for qhorus-native chat UI)
**Issues:** #329 (Topic), #330 (Reactions)
**Branch:** `issue-328-conversation-model-enrichments`

---

## Context

Qhorus provides the peer-to-peer communication mesh for multi-agent AI systems. The conversation model currently supports channels with typed messages (9-type speech-act taxonomy), correlation-based threading, and an immutable audit ledger. The chat UI being built in casehubio/connectors (connectors#61) needs two missing layers:

1. **Topics** — named, persistent sub-conversations within channels (Zulip model)
2. **Reactions** — lightweight emoji reactions on messages

These are the first two deliverables of the conversation model enrichments epic, chosen because they're additive (no breaking changes), independent of each other, and unblock connectors#64 (reaction palette) and connectors#66 (topic navigator).

### Research Basis

Design informed by exhaustive research across 11 chat platforms and 6 agent communication frameworks (full research: `casehubio/connectors workspace specs/2026-07-07-conversation-model-research.md`). Key finding: Zulip's mandatory topic model is the strongest structural pattern for agent communication — named, persistent sub-conversations solve the "5 agents, 5 tasks, 1 channel" interleaved-stream problem.

---

## Design Decisions

### Topic: Hybrid Entity + Denormalized String

Three approaches were evaluated:

| Approach | Pros | Cons |
|----------|------|------|
| **1. String on Message only** (Zulip's current impl) | Simple reads, no JOIN | O(n) rename, no place for metadata, expensive `list_topics` (GROUP BY) |
| **2. Topic entity with FK** | Rename = 1 row update | JOIN on every message poll (hot path), heavier dispatch |
| **3. Hybrid: Topic entity + denormalized string** | Fast reads AND efficient listing | Denormalization sync on rename (infrequent) |

**Selected: Option 3.** This avoids [Zulip's known architectural regret](https://github.com/zulip/zulip/issues/1191) (open since 2014 — string-on-message with no Topic entity makes listing and metadata expensive) while avoiding the JOIN cost of a pure FK approach on the hottest query path (message polling).

### Ledger Relationship: Organizational, Not Normative

The immutable audit ledger records speech acts. Topics are organizational metadata — they scope conversations but don't create obligations, attestations, or commitments. The design follows the [Matrix event DAG](https://matrix-org.github.io/synapse/latest/development/room-dag-concepts.html) and [CQRS/Event Sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) pattern:

- **Ledger (write model):** records the original topic at dispatch time — immutable
- **Message table (read model):** stores the current topic — mutable via rename
- **Rename EVENT:** bridges the two — a system EVENT in the ledger documents the administrative action

An auditor reconstructing history sees: "Messages 1–50 were dispatched in topic 'auth-review'. At time T2, topic 'auth-review' was renamed to 'auth-investigation' [EVENT entry]. Messages 51+ were dispatched in topic 'auth-investigation'."

### Reactions: Outside the Normative Layer

Reactions are deliberately non-normative. A thumbs-up on a STATUS message is a lightweight acknowledgment without the formal weight of a RESPONSE (which triggers commitment resolution). This boundary is correct: humans observing agent conversations want to signal "seen" without triggering obligation machinery.

### Deferred Operations

`moveTopic` (#335) and `mergeTopics` (#336) are deferred. Move changes `message.channelId`, breaking `Commitment.channelId` references — a normative integrity violation. Merge mixes correlation chains across conversations. Both need separate design work. Rename is safe because it doesn't touch any normative field.

Also deferred: topic-aware channel digest (#337), topic-aware projections (#338), `get_reactions_batch` MCP tool (#339).

---

## Normative System Complementarity

Topics are organizational, not normative — but they complement the normative layer:

1. **Obligation context.** The ledger records the original topic at dispatch time. Auditors can answer: "what was the obligation context when this commitment was created?" The topic doesn't create the obligation — the speech act does — but it provides the human-readable scope.

2. **Topic resolution vs obligation state.** Resolving a topic ("this conversation is done") is independent of obligation state. A topic can be resolved while obligations are still OPEN (human decided to move on), or all obligations can be FULFILLED while the topic stays unresolved. This deliberate decoupling is correct: obligations are formal (normative), topic resolution is informal (organizational).

3. **Topics within the normative channel layout.** The existing layout uses `case-*/work`, `case-*/observe`, `case-*/oversight` for role-based structure. Topics add task-based structure within each role — replacing the need to create separate channels per sub-task. Role separation (normative) and topic scoping (organizational) are orthogonal and composable.

4. **Rename EVENT in the ledger.** Topic rename emits a system EVENT — making organizational changes normatively visible. The normative record stays intact while the organizational label evolves.

5. **Reactions as non-normative acknowledgment.** A reaction is a lightweight signal outside the speech-act taxonomy. It complements the formal RESPONSE/DONE/DECLINE cycle without competing with it.

---

## Chat System Integration

The epic exists to support the connectors chat UI (connectors#61). The conversation hierarchy is:

```
Space → Channel → Topic → Thread (correlation chain)
```

Topics are the missing middle layer. Integration points:

- `list_topics` feeds the topic sidebar (like Zulip's left panel)
- `check_messages` returns messages with `topic` field — UI groups by topic
- `resolve_topic` gives the UI a "mark done" affordance per sub-conversation
- `ReactionChangedEvent` CDI event → connectors' chat backend pushes via WebSocket for live reaction updates
- The `topic` field on `MessageReceivedEvent` lets external backends (Slack, connectors) route messages to the correct UI panel
- Connectors chat-demo already has `<qhorus-reaction-bar>` — maps `ReactionGroup` to the UI's `Reaction[]` type

---

## Data Model

### Topic

```java
// api/message/Topic.java
public record Topic(
        Long id,
        UUID channelId,
        String name,
        boolean resolved,
        Instant resolvedAt,
        String resolvedBy,
        Instant createdAt,
        String tenancyId) {}

// runtime/message/TopicEntity.java — JPA entity
// UNIQUE(channel_id, name, tenancy_id)
```

### Message Changes

```java
// api/message/Message.java — add field:
String topic    // nullable, null treated as "general"

// api/message/MessageDispatch.java — add field:
String topic    // flows through dispatch pipeline; null → "general" at build()

// api/store/query/MessageQuery.java — add field:
String topic    // filter messages by topic

// api/gateway/MessageReceivedEvent.java — add field:
String topic    // so observers know the topic context
```

### MessageLedgerEntry

```java
// runtime/ledger/MessageLedgerEntry.java — add column:
@Column(name = "topic")
public String topic;    // original topic at dispatch time, immutable
```

### Reaction

```java
// api/message/Reaction.java
public record Reaction(
        Long id,
        Long messageId,
        String emoji,
        String actorId,
        Instant createdAt,
        String tenancyId) {}

// runtime/message/ReactionEntity.java — JPA entity
// UNIQUE(message_id, emoji, actor_id)
// INDEX on message_id
```

### Topic Naming Rules

- Free-form text (not slug-restricted like channels — topics are human-readable labels)
- Trimmed whitespace, reject blank/empty
- Max 200 characters
- Case-insensitive matching, case-preserving storage
- `"general"` is reserved — cannot be resolved, renamed, or deleted
- First occurrence sets the stored case

### What's NOT in the Model

- No `topicId` FK on Message — denormalized string avoids JOIN on hot polling path
- No `channelId` on Reaction — derived from message; cleanup uses subquery on delete
- Reactions do NOT create ledger entries — UI metadata, not speech acts
- No many-to-many: a message belongs to exactly one topic

---

## Store Layer

### TopicStore

```java
// api/store/TopicStore.java
public interface TopicStore {
    Topic put(Topic topic);
    Optional<Topic> find(UUID channelId, String name);    // case-insensitive
    Optional<Topic> findById(Long id);
    List<Topic> findByChannel(UUID channelId);
    void resolve(UUID channelId, String name, String actorId);
    void unresolve(UUID channelId, String name);
    int rename(UUID channelId, String oldName, String newName);
    void delete(UUID channelId, String name);
    void deleteAll(UUID channelId);
}
```

### MessageStore Addition

```java
// api/store/MessageStore.java — add method:
int updateTopicName(UUID channelId, String oldTopic, String newTopic);
```

### ReactionStore

```java
// api/store/ReactionStore.java
public interface ReactionStore {
    Reaction react(Long messageId, String emoji, String actorId, String tenancyId);
    boolean unreact(Long messageId, String emoji, String actorId);
    List<Reaction> findByMessage(Long messageId);
    Map<Long, List<Reaction>> findByMessages(Collection<Long> messageIds);
    void deleteByMessage(Long messageId);
    void deleteByChannel(UUID channelId);    // subquery via message.channel_id
}
```

### Reactive Parity

`ReactiveTopicStore` and `ReactiveReactionStore` with `Uni<T>` returns. Gated by `@IfBuildProperty(casehub.qhorus.reactive.enabled)`.

### InMemory Implementations

`InMemoryTopicStore`, `InMemoryReactionStore` + reactive variants in `persistence-memory/`. All `@Alternative @Priority(1)`.

---

## Service Layer

### TopicService

```java
@ApplicationScoped
public class TopicService {

    // Called by MessageService.dispatch() on every message
    Topic ensureExists(UUID channelId, String topicName, String tenancyId);

    // Query
    List<TopicSummary> listTopics(UUID channelId);
    // record TopicSummary(String name, long messageCount, Instant lastActivityAt,
    //                     boolean resolved, Instant resolvedAt)

    // State
    void resolve(UUID channelId, String topicName, String actorId);
    void unresolve(UUID channelId, String topicName);

    // Rename
    RenameResult rename(UUID channelId, String oldName, String newName, String actorId);
    // 1. topicStore.rename()
    // 2. messageStore.updateTopicName() — bulk update
    // 3. emit system EVENT: {action: "topic-renamed", channelId, oldName, newName, count}
    // record RenameResult(String oldName, String newName, int messagesUpdated)
}
```

**Dispatch integration:** `MessageDispatch.build()` defaults null topic to `"general"`. `MessageService.dispatch()` sets `entity.topic` from `dispatch.topic()` before persist, then calls `topicService.ensureExists()` to upsert the Topic record. Flow: `MessageDispatch.topic` → `MessageEntity.topic` (persisted) → `topicService.ensureExists()` (Topic record upserted) → `Message.topic` (returned).

### ReactionService

```java
@ApplicationScoped
public class ReactionService {

    Reaction react(Long messageId, String emoji, String actorId);
    boolean unreact(Long messageId, String emoji, String actorId);
    List<ReactionGroup> getReactions(Long messageId);
    Map<Long, List<ReactionGroup>> getReactionsBatch(Collection<Long> messageIds);
    // record ReactionGroup(String emoji, int count, List<String> actorIds)
}
```

**ReactionService does NOT flow through MessageService.dispatch().** Reactions bypass the dispatch pipeline entirely — no enforcement gate, no ledger entry, no MessageObserver notification. Reactions have their own CDI event for live notification:

```java
// api/message/ReactionChangedEvent.java
public record ReactionChangedEvent(
        Long messageId,
        String emoji,
        String actorId,
        boolean added) {}
```

### Reactive Parity

`ReactiveTopicService`, `ReactiveReactionService` — gated by `@IfBuildProperty`, mirroring blocking services with `Uni<T>`.

### Cleanup Integration

`delete_channel(force=true)` flow gains two steps before `messageStore.deleteAll()`:
1. `reactionStore.deleteByChannel(channelId)`
2. `topicStore.deleteAll(channelId)`

---

## MCP Tool Surface

### Updates to Existing Tools

**`send_message`** gains optional `topic` parameter:
- Null → `"general"`
- Passed through `MessageDispatch.topic`

**`MessageSummary`** gains `topic` field — included in `check_messages` and `search_messages` responses.

**`search_messages`** gains optional `topic` parameter for server-side topic filtering.

### New Topic Tools

```
list_topics(channel) → List<TopicSummary>
resolve_topic(channel, topic_name) → TopicDetail
unresolve_topic(channel, topic_name) → TopicDetail
rename_topic(channel, old_name, new_name) → RenameResult
```

### New Reaction Tools

```
react(message_id, emoji) → ReactionResult
unreact(message_id, emoji) → ReactionResult
get_reactions(message_id) → List<ReactionGroup>
```

Actor ID derived from caller's registered instance or CurrentPrincipal.

### Tool Discipline

All new tools are `@Tool`-annotated public methods. Non-`@Tool` convenience overloads are package-private per `ToolOverloadDiscoverabilityTest`. All mirrored in `ReactiveQhorusMcpTools` with `Uni<T>` returns.

---

## Flyway Migrations

Next V is 28 per CLAUDE.md. Pre-release — no deployed instances, no data migration concerns.

```sql
-- V28: topic column on message table
ALTER TABLE message ADD COLUMN topic VARCHAR(200);

-- V29: topic table
CREATE TABLE topic (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    channel_id UUID NOT NULL,
    name VARCHAR(200) NOT NULL,
    resolved BOOLEAN NOT NULL DEFAULT FALSE,
    resolved_at TIMESTAMP,
    resolved_by VARCHAR(255),
    tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT uq_topic_channel_name_tenancy UNIQUE (channel_id, name, tenancy_id),
    CONSTRAINT fk_topic_channel FOREIGN KEY (channel_id) REFERENCES channel(id)
);

-- V30: reaction table
CREATE TABLE reaction (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    message_id BIGINT NOT NULL,
    emoji VARCHAR(100) NOT NULL,
    actor_id VARCHAR(255) NOT NULL,
    tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT uq_reaction_message_emoji_actor UNIQUE (message_id, emoji, actor_id),
    CONSTRAINT fk_reaction_message FOREIGN KEY (message_id) REFERENCES message(id)
);
CREATE INDEX idx_reaction_message_id ON reaction(message_id);

-- V31: topic column on message_ledger_entry
ALTER TABLE message_ledger_entry ADD COLUMN topic VARCHAR(200);
```

---

## Testing Strategy

| Area | Test class | Module | Approach |
|------|-----------|--------|----------|
| TopicStore contract | `TopicStoreContractTest` (abstract) | persistence-memory | Blocking + reactive runners |
| ReactionStore contract | `ReactionStoreContractTest` (abstract) | persistence-memory | Blocking + reactive runners |
| TopicService | `TopicServiceTest` | runtime | CDI-free unit test, InMemory stores |
| ReactionService | `ReactionServiceTest` | runtime | CDI-free unit test, InMemory stores |
| Topic dispatch integration | `TopicDispatchTest` | runtime | `@QuarkusTest`, verify topic flows dispatch → message → ledger |
| Topic MCP tools | `TopicToolTest` | runtime | `@QuarkusTest`, `@Tool` methods |
| Reaction MCP tools | `ReactionToolTest` | runtime | `@QuarkusTest`, `@Tool` methods |
| Rename with EVENT audit | `TopicRenameTest` | runtime | `@QuarkusTest`, verify ledger entries preserve original topic |
| Channel delete cascade | existing `ChannelToolTest` | runtime | Add cases for reaction + topic cleanup |
| Flyway schema | `FlywayMigrationSchemaTest` | runtime | Verify V28–V31 produce correct schema |

Testing conventions applied per CLAUDE.md:
- CDI-free unit tests set `service.tracingConfig` to disabled-tracing impl
- Integration tests asserting observer dispatch use `QuarkusTransaction.requiringNew()`
- `LedgerWriteService.record()` uses `REQUIRES_NEW` — setup inside `@Test` body
- Unique channel names per test to avoid `RateLimiter` cross-test interference
- `import-qhorus-test.sql` for `ledger_subject_sequence` table

---

## File Impact Summary

### New Files

| Path | What |
|------|------|
| `api/.../message/Topic.java` | Topic record |
| `api/.../message/Reaction.java` | Reaction record |
| `api/.../message/ReactionChangedEvent.java` | CDI event for live notification |
| `api/.../store/TopicStore.java` | Topic store interface |
| `api/.../store/ReactiveTopicStore.java` | Reactive topic store interface |
| `api/.../store/ReactionStore.java` | Reaction store interface |
| `api/.../store/ReactiveReactionStore.java` | Reactive reaction store interface |
| `runtime/.../message/TopicEntity.java` | JPA entity |
| `runtime/.../message/TopicService.java` | Service bean |
| `runtime/.../message/ReactiveTopicService.java` | Reactive service |
| `runtime/.../message/ReactionEntity.java` | JPA entity |
| `runtime/.../message/ReactionService.java` | Service bean |
| `runtime/.../message/ReactiveReactionService.java` | Reactive service |
| `runtime/.../store/jpa/JpaTopicStore.java` | JPA store impl |
| `runtime/.../store/jpa/ReactiveJpaTopicStore.java` | Reactive JPA store |
| `runtime/.../store/jpa/JpaReactionStore.java` | JPA store impl |
| `runtime/.../store/jpa/ReactiveJpaReactionStore.java` | Reactive JPA store |
| `persistence-memory/.../InMemoryTopicStore.java` | InMemory store |
| `persistence-memory/.../InMemoryReactiveTopicStore.java` | InMemory reactive store |
| `persistence-memory/.../InMemoryReactionStore.java` | InMemory store |
| `persistence-memory/.../InMemoryReactiveReactionStore.java` | InMemory reactive store |
| `db/qhorus/migration/V28__*.sql` through `V31__*.sql` | Flyway migrations |

### Modified Files

| Path | Change |
|------|--------|
| `api/.../message/Message.java` | Add `topic` field + builder method |
| `api/.../message/MessageDispatch.java` | Add `topic` field + builder method + null→"general" default |
| `api/.../store/MessageStore.java` | Add `updateTopicName()` method |
| `api/.../store/query/MessageQuery.java` | Add `topic` field + builder method + `matches()` |
| `api/.../gateway/MessageReceivedEvent.java` | Add `topic` field |
| `runtime/.../message/MessageEntity.java` | Add `topic` column + fromDomain/toDomain |
| `runtime/.../message/MessageService.java` | Call `topicService.ensureExists()` after persist |
| `runtime/.../message/ReactiveMessageService.java` | Same for reactive path |
| `runtime/.../ledger/MessageLedgerEntry.java` | Add `topic` column |
| `runtime/.../ledger/LedgerWriteService.java` | Set `entry.topic` from dispatch |
| `runtime/.../ledger/ReactiveLedgerWriteService.java` | Same for reactive path |
| `runtime/.../mcp/QhorusMcpToolsBase.java` | Add `topic` to `MessageSummary`, `toMessageSummary()` |
| `runtime/.../mcp/QhorusMcpTools.java` | Add `topic` param to `sendMessage`, new topic + reaction tools |
| `runtime/.../mcp/ReactiveQhorusMcpTools.java` | Same for reactive tools |
| `runtime/.../store/jpa/JpaMessageStore.java` | Add `updateTopicName()` impl |
| `persistence-memory/.../InMemoryMessageStore.java` | Add `updateTopicName()` impl |
| `persistence-memory/.../InMemoryReactiveMessageStore.java` | Same for reactive |

---

## Cross-Repo Integration Test Distribution

Each repo tests its own contribution to the integrated result. Qhorus tests that topics work. Engine tests that choreography flows through topics. Connectors tests that the UI reads topics. Blocks tests that all three compose correctly — the end-to-end story invisible from any single repo.

| Repo | Tests | What they prove |
|------|-------|-----------------|
| **qhorus** | Topic/Reaction unit + integration | Primitives work: dispatch flows topic to message and ledger, rename preserves original in ledger, reactions are idempotent, cleanup cascades |
| **engine** | Choreography-over-channels | Engine workflow dispatches COMMANDs with topic context, tracks which choreography step maps to which topic, workflow metadata flows through MessageDispatch |
| **connectors** | Chat UI reads topics and reactions | Topic sidebar populated from `list_topics`, messages grouped by topic, reaction pills rendered from `get_reactions`, WebSocket push for `ReactionChangedEvent` |
| **blocks** | Full stack end-to-end | Engine choreography → qhorus dispatch with topic → `ConversationProjection` scoped by topic → `ConversationRenderer` shows topic grouping with choreography progress overlay. Wires the full path and asserts the rendered conversation matches the engine's workflow definition |

Cross-repo follow-up issues: engine#701, connectors#80, blocks#49.

---

## Future Directions

Beyond the current epic (#328), several conversation model aids would help LLMs communicate and coordinate more efficiently:

1. **Structured content schemas per topic.** A `topic.contentSchema` (optional JSON Schema) would let senders and receivers agree on message format without token-expensive negotiation. An orchestrator declares "messages in this topic conform to `{findings: [{severity, location, description}]}`" — both sides know the contract.

2. **Progress signals as first-class metadata.** Structured progress on STATUS messages (`{step: 3, totalSteps: 7, percentComplete: 42, currentActivity: "..."}`) would let orchestrators make intelligent timeout decisions and the UI render progress bars — without parsing natural language.

3. **Pinned messages per topic.** A pointer from Topic to a specific message ID marking "this is the authoritative requirement" or "this is the agreed approach." Agents joining a topic late read the pin first, not the whole scroll.

4. **Engine choreography visibility in conversations.** The engine (casehub-engine) owns workflow definition and enforcement — "code review protocol = COMMAND(review) → STATUS(findings) → RESPONSE(feedback) → DONE(revised)." Qhorus carries the conversation trace. The chat UI renders both together: the message stream with a workflow progress overlay showing "step 3 of 7: waiting for reviewer response" and "this conversation deviated from the protocol at step 4." The dovetail: engine dispatches messages through qhorus with workflow context metadata (workflow ID, step ID), qhorus persists and delivers them, the normative layer validates obligation fulfillment, and the chat UI renders the conversation with the engine's choreography as navigational context — not enforcement (that's the engine's job) but comprehension (which is the UI's job).

5. **Attention/mention signals.** Richer than the current `target` field — explicit `@mentions`, read receipts, "needs your input" markers that reduce polling waste when multiple agents share a channel.

The thread connecting all of these: reducing ambiguity without adding ceremony. LLMs are good at natural language but waste capacity resolving ambiguity. Each aid provides structure that LLMs can lean on — schema tells them the format, progress tells them the state, pins tell them the context, choreographies tell them the sequence.

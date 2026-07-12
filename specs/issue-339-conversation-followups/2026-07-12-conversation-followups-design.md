# Conversation Model Followups — Design Spec

**Date:** 2026-07-12
**Branch:** issue-339-conversation-followups
**Covers:** #339, #336, #335, #340, #333

Five follow-up issues from the #328 conversation model enrichments epic. Ordered by implementation dependency: #339 (independent), #336 (depends on topics), #335 (depends on topics + commitments), #340 (independent), #333 (depends on #332 membership).

---

## 1. get_reactions_batch MCP tool (#339)

### Problem

`ReactionService.getReactionsBatch(Collection<Long>)` exists and returns `Map<Long, List<ReactionGroup>>`, but no MCP tool exposes it. External MCP clients fetching reactions for a message list must call `get_reactions` N times.

### Design

Add a `get_reactions_batch` MCP tool to `QhorusMcpTools` that delegates to `ReactionService.getReactionsBatch()`.

**MCP tool signature:**
```
get_reactions_batch(message_ids: List<Long>) → Map<Long, List<ReactionGroup>>
```

No new store methods, no migration, no service changes. Blocking tool only — reactive parity follows the existing `@IfBuildProperty` pattern in `ReactiveQhorusMcpTools`.

### Validation

- `message_ids` must be non-null, non-empty, and limited to 200 entries maximum
- Individual IDs are not validated for existence — missing messages return empty reaction lists (consistent with `getReactions` for non-existent messages)

---

## 2. mergeTopics (#336)

### Problem

Two topics in a channel contain related messages that should be combined. No merge operation exists — `rename_topic` blocks when the target name already exists.

### Design

Add `TopicService.merge()` — mechanically identical to rename where the target already exists.

**Service method:**
```java
TopicService.merge(UUID channelId, String sourceTopic, String targetTopic, String actorId) → MergeResult
```

```java
record MergeResult(String sourceTopic, String targetTopic, int messagesUpdated) {}
```

**Mechanics:**

*Service (`TopicService.merge`):*
1. Validate: source topic exists, target topic exists, source != target, source is not "general"
2. Update all messages with `topic=source` to `topic=target` via `MessageStore.updateTopicName(channelId, source, target)` (existing method)
3. Delete source `Topic` record via `TopicStore.delete(channelId, source)`

*MCP tool (`merge_topics` — `@Transactional`):*
1. Resolve channel via `resolveChannel()`
2. Call `topicService.merge(channelId, sourceTopic, targetTopic, actorId)`
3. Emit audit EVENT with merge details via `messageService.dispatch()` (follows `rename_topic` pattern)

**MCP tool:**
```
merge_topics(channel, source_topic, target_topic, caller_instance_id?)
```

### Key decisions

- **Open commitments: allowed.** Topics are organizational, not normative. Commitments reference `channelId` and `correlationId`, not topic. Merging doesn't change normative context.
- **No merge markers.** The ledger preserves original topics (immutable). Query the ledger for pre-merge history.
- **Source Topic record deleted.** Source's resolved state is lost — target's resolved state governs. Merging into a resolved target does NOT auto-unresolve; the admin can explicitly unresolve if the merged content warrants it. Resolved state is always a deliberate admin decision, not a side effect of message reorganisation.
- **"general" cannot be the source.** You can't empty the default topic. But merging INTO "general" is allowed — collapsing a topic back into the default is a valid operation.

### Migration

None — uses existing `MessageStore.updateTopicName()` and `TopicStore.delete()`.

---

## 3. moveTopic (#335)

### Problem

A topic in channel A belongs in channel B. No operation exists to re-parent messages across channels.

### Design

Add `TopicService.move()` — updates `message.channelId` for all messages in a topic from source to target channel.

**Service method:**
```java
TopicService.move(UUID sourceChannelId, String topicName, UUID targetChannelId, String actorId) → MoveResult
```

```java
record MoveResult(String topicName, UUID sourceChannelId, UUID targetChannelId, int messagesUpdated) {}
```

**New store methods:**
```java
// MessageStore (blocking)
int updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId);
// ReactiveMessageStore
Uni<Integer> updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId);

// CommitmentStore (blocking)
List<Commitment> findByIds(Collection<UUID> ids);
// ReactiveCommitmentStore
Uni<List<Commitment>> findByIds(Collection<UUID> ids);
```

**Mechanics:**

*MCP tool (`move_topic` — `@Transactional`):*
1. Resolve source and target channels via `resolveChannel()`
2. Validate: source != target, `sourceChannel.tenancyId == targetChannel.tenancyId`, target channel semantic is APPEND or COLLECT (reject EPHEMERAL, BARRIER, LAST_WRITE)
3. Call `topicService.move(sourceChannelId, topicName, targetChannelId, actorId)`
4. Emit audit EVENTs in both channels via `messageService.dispatch()`: `topic-moved-out` in source, `topic-moved-in` in target (follows `rename_topic` pattern)

*Service (`TopicService.move`):*
1. Validate: topic exists in source, topic is not "general"
2. **Commitment gate:** query messages in source channel/topic with non-null `commitmentId`. Collect distinct commitment IDs, batch-check via `CommitmentStore.findByIds(Collection<UUID>)` (new method). If any commitment is non-terminal (OPEN, ACKNOWLEDGED) → throw `IllegalStateException` listing the blocking commitments by correlationId.
3. Move messages: `messageStore.updateChannelId(sourceChannelId, topicName, targetChannelId)`
4. Move topic record: if target channel has a topic with the same name → messages merge into it (target's resolved state is not modified — same rule as mergeTopics). If not → create new Topic record in target channel. Delete source Topic record.

**MCP tool:**
```
move_topic(source_channel, topic_name, target_channel, caller_instance_id?)
```

### Key decisions

- **Block on open commitments.** Commitments are social contracts opened under channel A's enforcement context (allowed writers, type policies, rate limits). Moving the underlying messages while the obligation is live creates normative ambiguity. Requiring terminal state forces deliberate resolution before reorganization.
- **Ledger entries NOT mutated.** The ledger preserves the original `channelId` (immutable). Divergence between ledger and message table is by design — same principle as topic rename.
- **Delivery cursor limitation (known, accepted).** Moved messages retain their original IDs (allocated in the source channel's sequence). These IDs may be lower than the target channel's AT_LEAST_ONCE delivery cursor, so they won't be re-delivered to the target channel's backends. moveTopic is administrative reorganization of historical messages, not real-time dispatch.
- **Reactions follow messages.** Keyed by `messageId`, not `channelId`. No action needed.
- **"general" cannot be moved.** The default topic is structural.
- **Same-tenancy enforced.** Source and target must share `tenancyId`. Implicitly enforced at the MCP layer (tenancy-scoped store queries prevent cross-tenant channel resolution), explicitly validated for defense-in-depth.
- **Target semantic: APPEND or COLLECT only.** EPHEMERAL, BARRIER, and LAST_WRITE channels have structural semantics incompatible with historical message insertion. LAST_WRITE expects single-value state. EPHEMERAL clears on read. BARRIER gates on declared contributors.
- **Channel policies not re-validated.** Moved messages were validated against the source channel's policies at dispatch time. The enforcement gate is dispatch-time, not retroactive. Moved messages may include senders not in target's `allowedWriters` or message types not in target's `allowedTypes`/`deniedTypes` — this is an intentional consequence of administrative reorganisation. `distinctSendersByChannel()` will reflect the actual senders present.

### Migration

None — `updateChannelId` is a new UPDATE query on existing columns. No DDL changes.

---

## 4. ArtefactType.DEBATE + Gap 1 closure (#340)

### Problem

The chat-demo UI defines `ArtefactType.DEBATE` which doesn't exist in Qhorus's enum. ChannelMembership lacks `displayName`.

### Design

**Gap 2 (DEBATE):** Add `DEBATE` to the `ArtefactType` enum.

```java
public enum ArtefactType {
    DOCUMENT, CODE, CASE, WORK_ITEM, CHANNEL, MESSAGE, EXTERNAL, DEBATE
}
```

No migration, no downstream impact. Pure enum addition.

**Gap 1 (displayName): closed as design-intentional.** Display name is a property of identity, not membership. Qhorus is communication infrastructure, not an identity service. Consumers resolve display names via their own identity layer: connectors use `ChatPlatform.Member.displayName()`, MCP agents use `Instance.description`. Adding `displayName` to `ChannelMembership` would create a denormalized second source of truth with no refresh mechanism.

---

## 5. Presence with heartbeat degradation (#333)

### Problem

No concept of member availability in Qhorus. The chat UI needs online/away/offline indicators. Agent orchestration needs to know which agents are available for work.

### Design

#### Data model

```java
// api/channel/PresenceStatus.java
public enum PresenceStatus {
    ONLINE,      // actively connected, heartbeat recent
    AVAILABLE,   // connected, ready for work (idle agent)
    BUSY,        // connected, working on something
    AWAY,        // heartbeat stale — connected but not active (computed)
    OFFLINE      // heartbeat expired or never seen (computed)
}
```

```java
// api/channel/Presence.java
public record Presence(
    String memberId,
    PresenceStatus status,          // effective status (may be degraded from reported)
    PresenceStatus reportedStatus,  // what the client last reported
    Instant lastSeenAt,
    String statusMessage            // nullable — "In a meeting", "Processing case-456"
) {}
```

Two status fields: `reportedStatus` is what the client sent in the last heartbeat. `status` is the effective status after timeout degradation. Consumers display `status`; diagnostics can check `reportedStatus` to see what the client intended.

#### Storage: Caffeine cache

Presence is ephemeral — no database, no JPA entity, no Flyway migration.

```java
Cache<String, PresenceEntry> cache = Caffeine.newBuilder()
    .expireAfterWrite(offlineTimeout)  // default 10 min
    .build();
```

Internal `PresenceEntry` holds `reportedStatus`, `lastSeenAt`, `statusMessage`. The `Presence` record returned to callers computes `status` from the entry's staleness.

#### Status computation: lazy, on read

No `@Scheduled` evaluator. Status degradation is computed at read time:

```
now - lastSeenAt < awayTimeout  → reportedStatus (ONLINE/AVAILABLE/BUSY)
now - lastSeenAt ≥ awayTimeout  → AWAY
cache miss (expired/never seen) → OFFLINE
```

#### Service

```java
// runtime/channel/PresenceService.java
@ApplicationScoped
public class PresenceService {

    void heartbeat(String memberId, PresenceStatus status, String statusMessage);
    // status restricted to ONLINE, AVAILABLE, BUSY — throws IllegalArgumentException for AWAY/OFFLINE

    Presence getPresence(String memberId);

    List<Presence> getChannelPresence(UUID channelId);
    // Queries membershipService.listMembers(channelId), then looks up each member's presence

    void setOffline(String memberId);
    // Immediate cache removal — e.g., on disconnect
}
```

#### Configuration

```java
// runtime/config/PresenceConfig.java
@ConfigMapping(prefix = "casehub.qhorus.presence")
public interface PresenceConfig {
    @WithDefault("PT2M")
    Duration awayTimeout();

    @WithDefault("PT10M")
    Duration offlineTimeout();

    @WithDefault("PT30S")
    Duration heartbeatInterval();  // advisory — returned to clients so they know how often to heartbeat
}
```

**Invariant:** `awayTimeout < offlineTimeout` — otherwise the Caffeine cache evicts entries before the AWAY threshold is reached, making AWAY unreachable (members jump directly from reported status to OFFLINE). Validate at `PresenceService` startup; throw `IllegalStateException` if violated.

#### MCP tools

```
set_presence(status, status_message?, member_id?)   → Presence
get_presence(member_id)                             → Presence
get_channel_presence(channel)                       → List<Presence>
```

`set_presence` is the heartbeat entry point. Calling it refreshes `lastSeenAt` and updates the reported status. `member_id` defaults to `currentPrincipal.actorId()` — matching the `react`/`unreact` pattern. Optional override for system/admin use. AWAY and OFFLINE are rejected — they are computed-only statuses derived from heartbeat absence.

#### Testing

`PresenceService` takes a `java.time.Clock` injected via CDI (producer provides `Clock.systemUTC()` in production). Tests provide a controllable clock to verify degradation:
- Heartbeat with ONLINE → verify status
- Advance time past awayTimeout → verify AWAY
- Advance time past offlineTimeout (or call setOffline) → verify OFFLINE
- getChannelPresence → verify batch lookup against membership

No `InMemoryPresenceStore` needed — the Caffeine cache IS in-memory.

### Key decisions

- **No CDI event for presence changes.** Heartbeats are frequent (30s). Events every heartbeat are noisy. The UI polls `get_channel_presence`. Server-side reactive events can be added when a concrete use case emerges.
- **No reactive parity.** Caffeine has no blocking I/O. The blocking service is sub-microsecond. Reactive MCP tools call it directly.
- **No Flyway migration.** Cache-only.
- **No PresenceStore SPI.** Extract one when multi-node support (Redis, distributed cache) is needed. Pre-release single-node doesn't need the abstraction.
- **Single-node only.** Caffeine is JVM-local. Consistent with ARC42 §12's "Cross-node fanOut gap" constraint — presence queries on node B won't see heartbeats sent to node A. Acceptable for pre-release single-node deployment.

---

## Implementation Order

1. **#339** — get_reactions_batch (independent, smallest)
2. **#336** — mergeTopics (uses existing store methods)
3. **#335** — moveTopic (new store method, commitment gate)
4. **#340** — ArtefactType.DEBATE (single enum addition)
5. **#333** — Presence (largest, depends on membership from #332)

---

## Cross-cutting

- **No Flyway migration** for any of these issues. All changes are either in-memory (presence), enum additions (DEBATE), new queries on existing columns (moveTopic), or pure service/MCP additions.
- **Reactive parity** for MCP tools follows the existing `@IfBuildProperty` pattern. Presence is an exception — no reactive service, blocking service called from both stacks.
- **Store method additions** across all implementations (blocking, reactive, InMemory, JPA) for `updateChannelId()` and `findByIds()`. `CrossTenant*Store` interfaces are separate (not extending the base stores) and do not need the new methods.

# Rich ArtefactRef + Channel Membership Design — Session 2 of #328

**Date:** 2026-07-11
**Epic:** #328 (conversation model enrichments for qhorus-native chat UI)
**Issues:** #331 (Rich ArtefactRef), #332 (Channel Membership)
**Branch:** `issue-328-conversation-model-enrichments`

---

## Context

Session 1 delivered Topics (#329) and Reactions (#330). Session 2 tackles two independent enrichments:

1. **Rich ArtefactRef (#331)** — replace the opaque `List<UUID>` artefact reference model with structured `ArtefactRef` records carrying type, label, and optional selection scope. Unblocks connectors#65 (rich artefact references with selection scope).

2. **Channel Membership (#332)** — add a user-facing membership model distinct from the existing ACL layer (`allowedWriters`, `adminInstances`). Tracks who IS in a channel (not just who CAN write), with roles and unread tracking. Unblocks connectors#68 (channel membership and presence model). Foundation for #333 (Presence).

### Design Principle: Pipeline Unification

The current `artefactRefs` field uses three different representations across the pipeline — `List<UUID>` (API domain), `String` (dispatch/entity/view CSV), and `List<String>` (MCP summary). This representational split predates rich refs and is already inconsistent. Pre-release, we unify: `List<ArtefactRef>` as the canonical type everywhere in the API/domain layer, with JSON serialization only at the entity storage boundary.

---

## #331: Rich ArtefactRef

### Data Model

Three new records in `api/message/`:

#### ArtefactRef

```java
// api/message/ArtefactRef.java
public record ArtefactRef(
    String uri,              // UUID string, URL, or typed prefix (case:123, channel:<uuid>)
    ArtefactType type,       // structural category for UI rendering
    String label,            // human-readable display text; nullable
    SelectionScope scope     // nullable — for anchored references
) {
    public ArtefactRef {
        if (uri == null || uri.isBlank()) throw new IllegalArgumentException("uri is required");
        if (type == null) throw new IllegalArgumentException("type is required");
    }
}
```

**URI scheme:** Flexible — UUIDs for SharedData-backed artifacts, URLs for external resources, typed prefixes for cross-system links (`case:123`, `channel:<uuid>`, `message:<id>`). The `type` enum disambiguates interpretation; the URI scheme provides system-level routing.

#### ArtefactType

```java
// api/message/ArtefactType.java
public enum ArtefactType {
    DOCUMENT,    // specs, design docs, markdown
    CODE,        // source code files
    CASE,        // case management case
    WORK_ITEM,   // work item / task
    CHANNEL,     // another qhorus channel
    MESSAGE,     // another qhorus message (cross-reference)
    EXTERNAL     // external URL or opaque reference
}
```

Seven values. `DEBATE` from the original issue spec was dropped — it is drafthouse-specific and `EXTERNAL` with a `debate:` URI scheme covers it. Adding enum values later is trivial (pre-release).

#### SelectionScope

```java
// api/message/SelectionScope.java
public record SelectionScope(
    Integer startLine,       // for code (nullable)
    Integer endLine,         // for code (nullable)
    Integer startOffset,     // for text character offsets (nullable)
    Integer endOffset,       // for text character offsets (nullable)
    String selectedText      // display when source unavailable (nullable)
) {}
```

All fields nullable. Use line-based for code, offset-based for text, `selectedText` for display when the source artifact is unavailable. This is a **generalisation** of drafthouse's `io.casehub.drafthouse.SelectionScope`, not a drop-in replacement. Key differences:

- Drafthouse's `SelectionScope` has a required `DocumentSide` enum (LEFT/RIGHT for diff views), required non-blank `selectedText`, and validated `int` line ranges. These are drafthouse-specific constraints for A/B document comparison.
- Qhorus's `SelectionScope` is a platform primitive: all fields nullable, adds character offsets for text selections, no domain-specific `DocumentSide`.

The migration path for drafthouse is **composition**: drafthouse retains its `DocumentSide` alongside qhorus's `SelectionScope` (e.g., `DebateContext(DocumentSide side, SelectionScope scope)`), not a simple type swap. The `DocumentSide` concept is inherent to drafthouse's diff-view domain and does not belong in a platform primitive.

### Pipeline Unification

| Type | Before | After |
|------|--------|-------|
| `Message.artefactRefs` | `List<UUID>` | `List<ArtefactRef>` |
| `MessageDispatch.artefactRefs` | `String` (CSV UUIDs) | `List<ArtefactRef>` |
| `MessageEntity.artefactRefs` | `String` (CSV UUIDs) | `String` (JSON) |
| `NormalisedMessage.artefactRefs` | `String` (CSV UUIDs) | `List<ArtefactRef>` (nullable) |
| `MessageView.artefactRefs` | `String` (CSV UUIDs) | `List<ArtefactRef>` |
| `DispatchResult.artefactRefs` | `List<UUID>` | `List<ArtefactRef>` |
| `MessageSummary.artefactRefs` | `List<String>` | `List<ArtefactRef>` |
| `OutboundMessage.artefactRefs` | (not present) | `List<ArtefactRef>` (nullable) |

**OutboundMessage** gains `List<ArtefactRef> artefactRefs` so backends receiving fan-out messages can render artefact references in real-time (essential for connectors#65 chat UI). Backends that don't use artefactRefs simply ignore the field.

**Normaliser handling:** `DefaultInboundNormaliser` continues to pass `null` for artefactRefs — the type change from `String` to `List<ArtefactRef>` is mechanical. Human chat UIs don't produce structured `ArtefactRef` objects in the raw `InboundHumanMessage`. Backend-specific normalisers (e.g., `ConnectorNormaliser`) that receive rich artefact references from their input format construct `List<ArtefactRef>` via `InboundHumanMessage.metadata()` or a future field extension — that's a connector-backend concern.

The entity layer is the sole serialization boundary. `MessageEntity` uses a JPA `@Converter` (`ArtefactRefListConverter`) to handle `List<ArtefactRef>` ↔ JSON `String` — entities aren't CDI beans so ObjectMapper can't be injected; the converter uses a static ObjectMapper instance. `fromDomain()` and `toDomain()` work with `List<ArtefactRef>` directly; the converter handles persistence. The `artefact_refs` column stays `TEXT`.

### Auto-Claim Changes

**Current:** Every UUID in artefactRefs must exist in SharedData; all are auto-claimed for the sender.

**New:** Selective auto-claim based on URI resolvability.
1. For each `ArtefactRef`, attempt `UUID.fromString(ref.uri())`
2. If it parses as UUID, look up in `dataStore.findByIds()` — if found, auto-claim for sender; if NOT found, reject with `IllegalArgumentException` (preserves existing contract — dangling UUID refs are data integrity violations)
3. Non-UUID URIs (`case:123`, `https://...`) bypass validation and claim/release entirely
4. Auto-release on commitment resolution unchanged — scans refs for UUID-backed ones

This preserves the existing lifecycle for SharedData-backed artifacts while allowing opaque cross-system references that have no lifecycle within qhorus.

### No Schema Migration

The `artefact_refs` column is already `TEXT` (V1__initial_schema.sql). Content format changes from CSV UUIDs to JSON array. Pre-release — no deployed data to migrate, no Flyway migration needed.

### MCP Tool Surface

**`send_message`** — `artefact_refs` parameter changes from `List<String>` (UUID strings) to `String` (JSON-encoded array of ArtefactRef objects). Parsed at the MCP boundary into `List<ArtefactRef>`. Backward-compatible shorthand: a plain UUID string is accepted and auto-wrapped as `ArtefactRef(uri=uuid, type=DOCUMENT, label=null, scope=null)` — `DOCUMENT` because the only valid plain-UUID usage pattern is SharedData document references.

**`check_messages` / `search_messages`** — `MessageSummary.artefactRefs` carries full `List<ArtefactRef>` structure in responses.

**New tool:** `get_artefact_refs(message_id)` → `List<ArtefactRef>` — convenience for fetching refs from a specific message.

### Testing Strategy

| Area | Test class | Module | Approach |
|------|-----------|--------|----------|
| ArtefactRef record validation | `ArtefactRefTest` | api | Unit test — null uri/type rejected |
| JSON round-trip | `EntityConversionTest` (extend) | runtime | Verify JSON serialization/deserialization preserves all fields including scope |
| Auto-claim selective | `ArtefactAutoClaimTest` (extend) | runtime | UUID refs auto-claimed; non-UUID refs bypass claim |
| Rich refs in check_messages | `ArtefactRefsTest` (extend) | runtime | `@QuarkusTest` — verify full structure in MessageSummary |
| Mixed ref types | new test | runtime | Message with UUID + URL + typed prefix refs — all persist correctly |
| Null scope / null label | new test | runtime | Verify nullable fields round-trip without NPE |

### File Impact

**New files:**
- `api/message/ArtefactRef.java`
- `api/message/ArtefactType.java`
- `api/message/SelectionScope.java`

**Modified files:**
- `api/message/Message.java` — `List<UUID>` → `List<ArtefactRef>`
- `api/message/MessageDispatch.java` — `String` → `List<ArtefactRef>`; Builder validation updated
- `api/message/DispatchResult.java` — `List<UUID>` → `List<ArtefactRef>`
- `api/message/MessageView.java` — `String` → `List<ArtefactRef>`
- `api/gateway/NormalisedMessage.java` — `String` → `List<ArtefactRef>`
- `runtime/message/MessageEntity.java` — `List<ArtefactRef>` field with `@Convert` via `ArtefactRefListConverter`; remove `joinUuids()`/`parseUuids()` helpers
- `runtime/message/ArtefactRefListConverter.java` — new JPA `@Converter(autoApply = false)` for `List<ArtefactRef>` ↔ JSON String
- `runtime/QhorusEntityMapper.java` — `toMessageView()` passes `artefactRefs` through directly (no more `joinUuids()` call)
- `runtime/mcp/QhorusMcpToolsBase.java` — `MessageSummary.artefactRefs` type change to `List<ArtefactRef>`; `toMessageSummary()` updated
- `runtime/mcp/QhorusMcpTools.java` — `sendMessage()` artefact_refs parameter and auto-claim logic
- `runtime/mcp/ReactiveQhorusMcpTools.java` — same for reactive
- `api/gateway/OutboundMessage.java` — add `List<ArtefactRef> artefactRefs` field
- `runtime/gateway/ChannelGateway.java` — pass artefactRefs when constructing OutboundMessage in `fanOut()` and `deliverRemote()`
- `runtime/gateway/DefaultInboundNormaliser.java` — artefactRefs type change (null → null, mechanical)
- `runtime/mcp/QhorusMcpToolsBase.java` — `ChannelDigest.artefactRefCount` computation updated: iterate `ref.uri()` instead of `ref.toString()` for dedup; rename `artefactUuids` variable
- `persistence-memory/InMemoryMessageStore.java` — artefactRefs type in stored domain objects
- `persistence-memory/InMemoryReactiveMessageStore.java` — same
- All existing artefactRefs tests — updated for new types

**No changes needed (confirmed):**
- `connector-backend/ConnectorChannelBackend.java` — receives `OutboundMessage` via `post()` but only accesses `message.content()`; no artefactRefs handling

---

## #332: Channel Membership

### Data Model

New records in `api/channel/` (membership is a channel concept):

#### ChannelMembership

```java
// api/channel/ChannelMembership.java
public record ChannelMembership(
    Long id,
    UUID channelId,
    String memberId,         // agent ID or human user ID
    MemberRole role,
    String tenancyId,
    Instant joinedAt,
    Long lastReadMessageId   // nullable — for unread tracking
) {}
```

**`lastReadMessageId` (Long) instead of `lastReadAt` (Instant)** from the original issue spec. Message IDs are sequence-generated and monotonically increasing:
- No timestamp collision edge cases (sub-millisecond messages)
- Efficient unread count: `SELECT COUNT(*) FROM message WHERE channel_id = ? AND id > ? AND sender != ? AND message_type != 'EVENT'`
- Precise cursor semantics — no ambiguity about which messages at the same instant

#### MemberRole

```java
// api/channel/MemberRole.java
public enum MemberRole {
    PARTICIPANT,    // can send and receive messages
    OBSERVER,       // can receive only (read-only view)
    MODERATOR       // can manage topics, membership, channel settings
}
```

Maps to existing backend taxonomy: `HumanParticipatingChannelBackend` → PARTICIPANT, `HumanObserverChannelBackend` → OBSERVER. MODERATOR complements `adminInstances` ACL for topic and membership management.

#### UnreadCount

```java
// api/channel/UnreadCount.java
public record UnreadCount(
    UUID channelId,
    String channelName,
    long count,
    Long latestMessageId
) {}
```

### Relationship to Existing Concepts

| Concept | Layer | Question Answered |
|---------|-------|-------------------|
| `allowedWriters` | Authorization (ACL) | Who CAN write |
| `adminInstances` | Authorization (ACL) | Who CAN manage |
| `ChannelBackend` | Runtime SPI | Which backends receive messages |
| **ChannelMembership** | User-facing | Who IS in the channel |

**Membership does NOT replace ACL.** A write attempt still checks `allowedWriters`. A MODERATOR member can manage topics and membership but channel config changes (rate limits, type constraints) still require `adminInstances`.

### Store

```java
// api/store/ChannelMembershipStore.java
public interface ChannelMembershipStore {
    ChannelMembership put(ChannelMembership membership);
    Optional<ChannelMembership> find(UUID channelId, String memberId);
    List<ChannelMembership> findByChannel(UUID channelId);
    List<ChannelMembership> findByMember(String memberId, String tenancyId);
    void updateRole(UUID channelId, String memberId, MemberRole role);
    void updateLastReadMessageId(UUID channelId, String memberId, Long messageId);
    boolean delete(UUID channelId, String memberId);
    void deleteAll(UUID channelId);
}
```

Plus `ReactiveChannelMembershipStore` with `Uni<T>` returns. Standard pattern: `JpaChannelMembershipStore`, `InMemoryChannelMembershipStore`, `InMemoryReactiveChannelMembershipStore`.

### Service

```java
// runtime/channel/ChannelMembershipService.java
@ApplicationScoped
public class ChannelMembershipService {
    ChannelMembership join(UUID channelId, String memberId, MemberRole role, String tenancyId);
    void leave(UUID channelId, String memberId);
    List<ChannelMembership> listMembers(UUID channelId);
    void markRead(UUID channelId, String memberId, Long messageId);
    Map<UUID, UnreadCount> getUnreadCounts(String memberId, String tenancyId);
}
```

- `join()` is idempotent — if already a member, updates the role and preserves `joinedAt`. Accepts `tenancyId` explicitly; callers derive it from `CurrentPrincipal` or channel context. **On first join, initializes `lastReadMessageId` to the current max message ID for the channel** — "clean slate" semantic (Slack/Discord behavior: history is visible but not marked unread). If the channel has no messages, initializes to 0.
- `leave()` removes membership; no-op if not a member
- `markRead()` only advances `lastReadMessageId` forward (never backwards). When `messageId` is null, the MCP tool layer resolves "latest" by querying the max message ID for the channel before calling the service — the service always receives a concrete ID.
- `getUnreadCounts()` joins membership against message table, scoped by `tenancyId`: for each channel, count messages with `id > lastReadMessageId AND sender != memberId AND message_type != 'EVENT'` (own messages excluded — they aren't "unseen"; EVENT messages excluded — they are system telemetry, not user-facing communication, consistent with `check_messages` default behavior)

Plus `ReactiveChannelMembershipService` with `Uni<T>` returns, gated by `@IfBuildProperty`.

### Auto-Membership

Auto-membership is created lazily on first human interaction, NOT at backend registration time. A backend is infrastructure (e.g., `ConnectorChannelBackend` with backendId `"connector-human"`) — it serves potentially many human users, so creating a single membership for the backend is semantically wrong. Instead:

- **`ChannelGateway.receiveHumanMessage()`** — after normalisation, if no membership exists for the sender (e.g., `"human:alice"`), auto-create with `MemberRole.PARTICIPANT`. The sender ID from `NormalisedMessage.senderInstanceId()` IS the `memberId`.
- **`ChannelGateway.receiveObserverSignal()`** — auto-create with `MemberRole.OBSERVER` for `"human:" + signal.externalSenderId()`.

`AgentChannelBackend` and custom agent backends do NOT auto-create memberships. Agents join explicitly via `join_channel` MCP tool when they want to be visible as members.

Implementation: `receiveHumanMessage()` calls `membershipService.join()` after normalisation — outside any synchronized block, using the normalised sender ID. `join()` is idempotent, so repeated messages from the same human are no-ops. **Tenancy derivation:** the gateway resolves `tenancyId` from the channel entity via `crossTenantChannelStore.findById(channelId)` — the channel is always available in the gateway context and is the authoritative tenancy source for non-MCP entry points (webhooks, connector callbacks) where `CurrentPrincipal` may not be set.

### Channel Delete Cascade

`delete_channel(force=true)` flow gains `membershipStore.deleteAll(channelId)` before existing cleanup. FK has `ON DELETE CASCADE` as safety net.

### Flyway Migration

V32 — `channel_membership` table:

```sql
-- V32__channel_membership.sql
CREATE TABLE channel_membership (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    channel_id UUID NOT NULL,
    member_id VARCHAR(255) NOT NULL,
    member_role VARCHAR(50) NOT NULL,
    tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
    joined_at TIMESTAMP NOT NULL,
    last_read_message_id BIGINT,
    CONSTRAINT uq_membership_channel_member UNIQUE (channel_id, member_id),
    CONSTRAINT fk_membership_channel FOREIGN KEY (channel_id) REFERENCES channel(id) ON DELETE CASCADE
);
CREATE INDEX idx_membership_member_id ON channel_membership(member_id);
```

### MCP Tool Surface

```
join_channel(channel, role?)           — memberId from caller's registered instance; tenancyId from CurrentPrincipal
leave_channel(channel)
list_members(channel)                  — returns ChannelMembership[]
mark_channel_read(channel, message_id?) — updates lastReadMessageId; null = latest (MCP tool resolves to max message ID before calling service)
get_unread_counts()                    — returns unread counts across all channels for caller, scoped by tenancyId from CurrentPrincipal; excludes caller's own messages
```

### Testing Strategy

| Area | Test class | Module | Approach |
|------|-----------|--------|----------|
| ChannelMembershipStore contract | `ChannelMembershipStoreContractTest` (abstract) | persistence-memory | Blocking + reactive runners |
| ChannelMembershipService | `ChannelMembershipServiceTest` | runtime | CDI-free unit test, InMemory stores |
| Join idempotency | in service test | runtime | Join twice — role updated, joinedAt preserved |
| markRead forward-only | in service test | runtime | Advance works, regress is no-op |
| Unread counts | in service test | runtime | Correct counts after markRead |
| Auto-membership | `ChannelGatewayMembershipTest` | runtime | `@QuarkusTest` — first human message auto-creates membership; observer signal auto-creates OBSERVER |
| Channel delete cascade | extend `ChannelToolTest` | runtime | Verify memberships cleaned up |
| MCP tools | `MembershipToolTest` | runtime | `@QuarkusTest` — all MCP tool methods |
| Flyway schema | extend `FlywayMigrationSchemaTest` | runtime | Verify V32 produces correct schema |

### File Impact

**New files:**
- `api/channel/ChannelMembership.java`
- `api/channel/MemberRole.java`
- `api/channel/UnreadCount.java`
- `api/store/ChannelMembershipStore.java`
- `api/store/ReactiveChannelMembershipStore.java`
- `runtime/channel/ChannelMembershipEntity.java`
- `runtime/channel/ChannelMembershipService.java`
- `runtime/channel/ReactiveChannelMembershipService.java`
- `runtime/store/jpa/JpaChannelMembershipStore.java`
- `runtime/store/jpa/ReactiveJpaChannelMembershipStore.java`
- `persistence-memory/InMemoryChannelMembershipStore.java`
- `persistence-memory/InMemoryReactiveChannelMembershipStore.java`
- `db/qhorus/migration/V32__channel_membership.sql`

**Modified files:**
- `runtime/gateway/ChannelGateway.java` — auto-membership on first human message via `receiveHumanMessage()` and `receiveObserverSignal()`
- `runtime/mcp/QhorusMcpTools.java` — new membership tools; delete_channel gains membership cleanup
- `runtime/mcp/ReactiveQhorusMcpTools.java` — same for reactive
- `runtime/mcp/QhorusMcpToolsBase.java` — MembershipSummary record
- `runtime/src/test/resources/import-qhorus-test.sql` — membership table DDL for tests
- `examples/type-system/src/test/resources/application.properties` — add InMemory membership stores to selected-alternatives

---

## Implementation Order

1. **#331 first** — touches more existing code; better to stabilize the pipeline change before adding new entities
2. **#332 second** — greenfield entity/service/tools; builds on stabilized pipeline

---

## Cross-Repo Follow-ups

| Repo | Issue | What |
|------|-------|------|
| connectors | #65 | Rich artefact references with selection scope — depends on #331 |
| connectors | #68 | Channel membership and presence model — depends on #332 |
| drafthouse | (new) | Migrate `SelectionScope` to compose with qhorus API's `SelectionScope` — replace `SelectionScope(DocumentSide, int, int, String)` with a wrapper holding `DocumentSide` alongside qhorus's `SelectionScope` |
| qhorus | (new) | Update issue #332 and connectors#68 to reflect `lastReadMessageId` (Long) instead of `lastReadAt` (Instant) |

# Rich ArtefactRef + Channel Membership Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #331 — Rich ArtefactRef, #332 — Channel Membership
**Issue group:** #328, #329, #330, #331, #332, #333, #334

**Goal:** Replace opaque `List<UUID>` artefact references with structured `ArtefactRef` records carrying type, label, and selection scope; add user-facing channel membership with roles and unread tracking.

**Architecture:** #331 unifies `artefactRefs` to `List<ArtefactRef>` across all 8 pipeline types, with JSON serialization via a JPA `@Converter` at the entity boundary. Auto-claim narrows to UUID-backed refs only. #332 adds `ChannelMembership` entity/store/service following the existing TopicStore pattern, with lazy auto-membership on first human interaction in `ChannelGateway`, and ID-based unread tracking.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM (H2 tests), Jackson ObjectMapper, Flyway, quarkus-mcp-server 1.11.1

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
- Use `mvn` not `./mvnw`
- After API visibility changes, run `mvn install` from project root (not scoped `mvn test`)
- Named datasource `qhorus` — all JPA entities, stores, and queries use the qhorus PU
- `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")` gates reactive beans
- `@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)` gates blocking beans that conflict with reactive counterparts
- InMemory stores: `@Alternative @Priority(1)` in `persistence-memory/`
- CDI-free unit tests: set `service.tracingConfig` to disabled-tracing impl
- `ToolOverloadDiscoverabilityTest` — non-`@Tool` public overloads of `@Tool` method names silently drop the tool. Use package-private visibility.
- `import-qhorus-test.sql` for non-JPA tables in test H2
- Pre-release — breaking changes cost nothing

---

### Task 1: ArtefactRef API Types

New records in `api/message/`. Compiles independently — no existing code changes.

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/message/ArtefactRef.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/ArtefactType.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/SelectionScope.java`
- Test: `api/src/test/java/io/casehub/qhorus/api/message/ArtefactRefTest.java`

**Produces:**
- `ArtefactRef(String uri, ArtefactType type, String label, SelectionScope scope)` — compact constructor validates uri non-null/non-blank, type non-null
- `ArtefactType` enum: `DOCUMENT, CODE, CASE, WORK_ITEM, CHANNEL, MESSAGE, EXTERNAL`
- `SelectionScope(Integer startLine, Integer endLine, Integer startOffset, Integer endOffset, String selectedText)` — all nullable

- [ ] **Step 1: Write ArtefactRef validation tests**

```java
package io.casehub.qhorus.api.message;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ArtefactRefTest {

    @Test
    void validRef_allFields() {
        var scope = new SelectionScope(1, 10, null, null, "selected text");
        var ref = new ArtefactRef("doc:spec.md", ArtefactType.DOCUMENT, "Design Spec", scope);
        assertThat(ref.uri()).isEqualTo("doc:spec.md");
        assertThat(ref.type()).isEqualTo(ArtefactType.DOCUMENT);
        assertThat(ref.label()).isEqualTo("Design Spec");
        assertThat(ref.scope()).isNotNull();
        assertThat(ref.scope().startLine()).isEqualTo(1);
        assertThat(ref.scope().selectedText()).isEqualTo("selected text");
    }

    @Test
    void validRef_nullLabelAndScope() {
        var ref = new ArtefactRef("https://example.com", ArtefactType.EXTERNAL, null, null);
        assertThat(ref.label()).isNull();
        assertThat(ref.scope()).isNull();
    }

    @Test
    void nullUri_rejected() {
        assertThatThrownBy(() -> new ArtefactRef(null, ArtefactType.DOCUMENT, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("uri");
    }

    @Test
    void blankUri_rejected() {
        assertThatThrownBy(() -> new ArtefactRef("  ", ArtefactType.DOCUMENT, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("uri");
    }

    @Test
    void nullType_rejected() {
        assertThatThrownBy(() -> new ArtefactRef("doc:spec.md", null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("type");
    }

    @Test
    void allArtefactTypes_serializable() {
        for (ArtefactType type : ArtefactType.values()) {
            var ref = new ArtefactRef("test:" + type.name(), type, null, null);
            assertThat(ref.type()).isEqualTo(type);
        }
        assertThat(ArtefactType.values()).hasSize(7);
    }

    @Test
    void selectionScope_allNullable() {
        var scope = new SelectionScope(null, null, null, null, null);
        assertThat(scope.startLine()).isNull();
        assertThat(scope.endLine()).isNull();
        assertThat(scope.startOffset()).isNull();
        assertThat(scope.endOffset()).isNull();
        assertThat(scope.selectedText()).isNull();
    }

    @Test
    void selectionScope_lineBased() {
        var scope = new SelectionScope(10, 20, null, null, null);
        assertThat(scope.startLine()).isEqualTo(10);
        assertThat(scope.endLine()).isEqualTo(20);
    }

    @Test
    void selectionScope_offsetBased() {
        var scope = new SelectionScope(null, null, 100, 200, "selected");
        assertThat(scope.startOffset()).isEqualTo(100);
        assertThat(scope.endOffset()).isEqualTo(200);
    }
}
```

- [ ] **Step 2: Run tests — verify they fail (types don't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ArtefactRefTest
```

Expected: compilation failure — `ArtefactRef`, `ArtefactType`, `SelectionScope` not found.

- [ ] **Step 3: Implement the three records**

Create `SelectionScope.java`:
```java
package io.casehub.qhorus.api.message;

public record SelectionScope(
    Integer startLine,
    Integer endLine,
    Integer startOffset,
    Integer endOffset,
    String selectedText) {}
```

Create `ArtefactType.java`:
```java
package io.casehub.qhorus.api.message;

public enum ArtefactType {
    DOCUMENT, CODE, CASE, WORK_ITEM, CHANNEL, MESSAGE, EXTERNAL
}
```

Create `ArtefactRef.java`:
```java
package io.casehub.qhorus.api.message;

public record ArtefactRef(
    String uri,
    ArtefactType type,
    String label,
    SelectionScope scope) {

    public ArtefactRef {
        if (uri == null || uri.isBlank()) throw new IllegalArgumentException("uri is required");
        if (type == null) throw new IllegalArgumentException("type is required");
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ArtefactRefTest
```

Expected: all 8 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/message/ArtefactRef.java \
        api/src/main/java/io/casehub/qhorus/api/message/ArtefactType.java \
        api/src/main/java/io/casehub/qhorus/api/message/SelectionScope.java \
        api/src/test/java/io/casehub/qhorus/api/message/ArtefactRefTest.java
git commit -m "feat(#331): ArtefactRef, ArtefactType, SelectionScope API records

Refs #331"
```

---

### Task 2: Pipeline Type Unification + Entity Converter

Change `artefactRefs` from UUID-based to `List<ArtefactRef>` across all API types and the entity layer. This is a single atomic change — all types must change together for compilation.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/Message.java` — `List<UUID>` → `List<ArtefactRef>`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java` — `String` → `List<ArtefactRef>`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/DispatchResult.java` — `List<UUID>` → `List<ArtefactRef>`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageView.java` — `String` → `List<ArtefactRef>`
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/NormalisedMessage.java` — `String` → `List<ArtefactRef>`
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java` — add `List<ArtefactRef>` field
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ArtefactRefListConverter.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageEntity.java` — use converter, remove `joinUuids`/`parseUuids`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java` — pass through directly
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — OutboundMessage construction gains artefactRefs
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — MessageSummary type change, toMessageSummary, ChannelDigest
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DefaultInboundNormaliser.java` — null `List<ArtefactRef>`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` — receiveHumanMessage passes `List<ArtefactRef>`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/conversion/EntityConversionTest.java` — extend with JSON round-trip

**Interfaces:**
- Consumes: `ArtefactRef`, `ArtefactType`, `SelectionScope` from Task 1
- Produces: All pipeline types use `List<ArtefactRef>`; `ArtefactRefListConverter` handles JSON ↔ `List<ArtefactRef>`

- [ ] **Step 1: Write JSON round-trip test for the converter**

Add to `EntityConversionTest.java`:

```java
@Test
void message_richArtefactRefs_jsonRoundTrip() {
    var scope = new SelectionScope(1, 10, null, null, "selected");
    var refs = List.of(
        new ArtefactRef(UUID.randomUUID().toString(), ArtefactType.DOCUMENT, "Design spec", scope),
        new ArtefactRef("https://example.com", ArtefactType.EXTERNAL, null, null),
        new ArtefactRef("case:123", ArtefactType.CASE, "Bug report", null)
    );
    Message msg = Message.builder()
            .channelId(UUID.randomUUID()).sender("agent-1")
            .messageType(MessageType.STATUS).actorType(ActorType.AGENT)
            .artefactRefs(refs).build();

    MessageEntity entity = MessageEntity.fromDomain(msg);
    assertThat(entity.artefactRefs).isNotNull();
    assertThat(entity.artefactRefs).startsWith("[");

    Message restored = entity.toDomain();
    assertThat(restored.artefactRefs()).hasSize(3);
    assertThat(restored.artefactRefs().get(0).type()).isEqualTo(ArtefactType.DOCUMENT);
    assertThat(restored.artefactRefs().get(0).label()).isEqualTo("Design spec");
    assertThat(restored.artefactRefs().get(0).scope().startLine()).isEqualTo(1);
    assertThat(restored.artefactRefs().get(1).type()).isEqualTo(ArtefactType.EXTERNAL);
    assertThat(restored.artefactRefs().get(1).label()).isNull();
    assertThat(restored.artefactRefs().get(2).uri()).isEqualTo("case:123");
}

@Test
void message_nullArtefactRefs_jsonRoundTrip() {
    Message msg = Message.builder()
            .channelId(UUID.randomUUID()).sender("agent-1")
            .messageType(MessageType.STATUS).actorType(ActorType.AGENT)
            .artefactRefs(null).build();

    MessageEntity entity = MessageEntity.fromDomain(msg);
    assertThat(entity.artefactRefs).isNull();

    Message restored = entity.toDomain();
    assertThat(restored.artefactRefs()).isNull();
}
```

- [ ] **Step 2: Run test — verify compilation fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime
```

Expected: compilation failure — types don't match yet.

- [ ] **Step 3: Create ArtefactRefListConverter**

```java
package io.casehub.qhorus.runtime.message;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.qhorus.api.message.ArtefactRef;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;

import java.util.List;

@Converter
public class ArtefactRefListConverter implements AttributeConverter<List<ArtefactRef>, String> {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final TypeReference<List<ArtefactRef>> TYPE_REF = new TypeReference<>() {};

    @Override
    public String convertToDatabaseColumn(List<ArtefactRef> refs) {
        if (refs == null || refs.isEmpty()) return null;
        try {
            return MAPPER.writeValueAsString(refs);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize ArtefactRef list", e);
        }
    }

    @Override
    public List<ArtefactRef> convertToEntityAttribute(String json) {
        if (json == null || json.isBlank()) return null;
        try {
            return MAPPER.readValue(json, TYPE_REF);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to deserialize ArtefactRef list", e);
        }
    }
}
```

- [ ] **Step 4: Update all API types**

Use `ide_edit_member` for each record/class change. The changes (all in `api/`):

1. **Message.java**: Change `List<UUID> artefactRefs` → `List<ArtefactRef> artefactRefs`. Update compact constructor defensive copy. Update Builder field and method types.
2. **MessageDispatch.java**: Change `String artefactRefs` → `List<ArtefactRef> artefactRefs`. Update Builder field/method. Remove CSV-related logic.
3. **DispatchResult.java**: Change `List<UUID> artefactRefs` → `List<ArtefactRef> artefactRefs`. Update compact constructor.
4. **MessageView.java**: Change `String artefactRefs` → `List<ArtefactRef> artefactRefs`.
5. **NormalisedMessage.java**: Change `String artefactRefs` → `List<ArtefactRef> artefactRefs`.
6. **OutboundMessage.java**: Add `List<ArtefactRef> artefactRefs` as a new record component (8th field, after `senderActorType`).

- [ ] **Step 5: Update MessageEntity**

Replace `joinUuids()`/`parseUuids()` with converter-based approach:
- Change field: `public String artefactRefs` → add `@Convert(converter = ArtefactRefListConverter.class)` annotation, keep as `public String artefactRefs` (the converter handles the mapping)
- Actually — the `@Convert` approach requires the field to be `List<ArtefactRef>` on the entity. But the column is `TEXT`. Use: field type `List<ArtefactRef>` with `@Convert` on the field, and `@Column(name = "artefact_refs", columnDefinition = "TEXT")`.
- Update `fromDomain()`: `e.artefactRefs = msg.artefactRefs()` (direct assignment)
- Update `toDomain()`: pass `artefactRefs` directly (no parsing needed)
- Delete `joinUuids()` and `parseUuids()` private methods

- [ ] **Step 6: Update QhorusEntityMapper**

In `toMessageView()`: change `joinUuids(msg.artefactRefs())` → `msg.artefactRefs()` (direct passthrough).

- [ ] **Step 7: Update MessageSummary and toMessageSummary()**

In `QhorusMcpToolsBase`:
- Change `MessageSummary.artefactRefs` from `List<String>` to `List<ArtefactRef>`
- Update `toMessageSummary()`: change `m.artefactRefs().stream().map(UUID::toString).toList()` → `m.artefactRefs() != null ? m.artefactRefs() : List.of()`
- Update `ChannelDigest` computation: `artefactRefCount` iterates `List<ArtefactRef>` instead of `List<UUID>`

- [ ] **Step 8: Update MessageService OutboundMessage construction**

Both `fanOut()` call sites in `MessageService.dispatch()` (lines ~237 and ~352): add `dispatch.artefactRefs()` as 8th argument to `new OutboundMessage(...)`.

Same for `ReactiveMessageService` if it has equivalent fanOut calls.

- [ ] **Step 9: Update DefaultInboundNormaliser**

Change `null` (6th arg) in `new NormalisedMessage(...)` from `String null` to `List<ArtefactRef> null` — mechanical, same value.

- [ ] **Step 10: Update ChannelGateway.receiveHumanMessage()**

The `new MessageDispatch(...)` canonical constructor call: change `n.artefactRefs()` — already passes through; type changed automatically from the NormalisedMessage change.

- [ ] **Step 11: Update ChannelGateway.deliverRemote()**

Check if `deliverRemote()` constructs `OutboundMessage` — if so, add `artefactRefs` from the `Message` domain object. Pass `msg.artefactRefs()`.

- [ ] **Step 12: Fix remaining compilation errors**

Run `ide_diagnostics` across the project. Fix any remaining callers:
- `InMemoryMessageStore` / `InMemoryReactiveMessageStore` — if they reference artefactRefs type
- Test files that construct `Message` or `MessageDispatch` with old artefactRefs types
- `ArtefactRefsTest`, `ArtefactAutoClaimTest`, `LastWriteArtefactRefsTest`, `MessageTest`

- [ ] **Step 13: Run the round-trip test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EntityConversionTest
```

Expected: PASS — JSON round-trip works.

- [ ] **Step 14: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: all modules compile and tests pass.

- [ ] **Step 15: Commit**

```bash
git add -A
git commit -m "feat(#331): unify artefactRefs pipeline to List<ArtefactRef>

Replace List<UUID>/String/List<String> with List<ArtefactRef> across
Message, MessageDispatch, DispatchResult, MessageView, NormalisedMessage,
OutboundMessage, MessageSummary. JPA ArtefactRefListConverter handles
JSON serialization at the entity boundary.

Refs #331"
```

---

### Task 3: MCP Tool Auto-Claim Update

Update `sendMessage()` in `QhorusMcpTools` and `ReactiveQhorusMcpTools` for selective auto-claim and backward-compatible UUID shorthand.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/mcp/ArtefactAutoClaimTest.java` — extend
- Test: `runtime/src/test/java/io/casehub/qhorus/mcp/ArtefactRefsTest.java` — extend

**Interfaces:**
- Consumes: `ArtefactRef`, `ArtefactType` from Task 1; unified pipeline from Task 2
- Produces: `sendMessage()` accepts `String artefactRefsJson` at MCP boundary, parses to `List<ArtefactRef>`

- [ ] **Step 1: Write test for UUID shorthand auto-wrap**

Add to `ArtefactRefsTest`:

```java
@Test
void sendMessageWithPlainUuidRef_autoWrapsAsDocument() {
    // Plain UUID string should auto-wrap as ArtefactRef(uri=uuid, type=DOCUMENT)
    // ... setup channel, artefact, instance ...
    // send_message with artefact_refs as JSON: ["<uuid-string>"]
    // verify the stored message has ArtefactRef with type=DOCUMENT
}
```

- [ ] **Step 2: Write test for mixed ref types**

```java
@Test
void sendMessageWithMixedRefTypes_uuidClaimedUrlBypassed() {
    // Create a SharedData artefact (UUID-backed)
    // send_message with artefact_refs: [
    //   {uri: "<uuid>", type: "DOCUMENT", label: "Spec"},
    //   {uri: "https://example.com", type: "EXTERNAL", label: "Link"}
    // ]
    // Verify: UUID ref auto-claimed, URL ref persisted but no claim
}
```

- [ ] **Step 3: Write test for dangling UUID rejection**

```java
@Test
void sendMessageWithDanglingUuidRef_rejected() {
    // send_message with artefact_refs containing a valid UUID format
    // that does NOT exist in SharedData → IllegalArgumentException
}
```

- [ ] **Step 4: Update sendMessage() in QhorusMcpTools**

Change `@ToolArg` for `artefact_refs`:
- Type: `String artefactRefsJson` (JSON-encoded)
- Parse logic: if input is a JSON array of strings → auto-wrap each as `ArtefactRef(uri, DOCUMENT, null, null)`; if JSON array of objects → deserialize as `List<ArtefactRef>`
- Selective auto-claim: extract UUID-backed refs, batch-validate against dataStore, claim for sender
- Non-UUID refs: skip validation and claim
- Build `List<ArtefactRef>` and pass to `MessageDispatch.builder().artefactRefs(refs)`

- [ ] **Step 5: Update sendMessage() in ReactiveQhorusMcpTools**

Mirror the blocking changes for the reactive variant.

- [ ] **Step 6: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ArtefactRefsTest,ArtefactAutoClaimTest"
```

- [ ] **Step 7: Add get_artefact_refs tool**

New `@Tool` method in `QhorusMcpTools`:
```java
@Tool(name = "get_artefact_refs", description = "Get artefact references for a message.")
public List<ArtefactRef> getArtefactRefs(
    @ToolArg(name = "message_id", description = "Message ID") Long messageId) {
    Message msg = messageStore.find(messageId)
            .orElseThrow(() -> new IllegalArgumentException("Message not found: " + messageId));
    return msg.artefactRefs() != null ? msg.artefactRefs() : List.of();
}
```

Mirror in `ReactiveQhorusMcpTools`.

- [ ] **Step 8: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#331): selective auto-claim, UUID shorthand, get_artefact_refs tool

UUID-backed refs validated against SharedData and auto-claimed. Non-UUID
refs bypass validation. Plain UUID strings auto-wrap as DOCUMENT type.
New get_artefact_refs MCP tool.

Refs #331"
```

---

### Task 4: Channel Membership API + Store + Flyway

New API types, store interface, and Flyway migration for channel membership.

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelMembership.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/MemberRole.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/UnreadCount.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/ChannelMembershipStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveChannelMembershipStore.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V32__channel_membership.sql`

**Produces:**
- `ChannelMembership(Long id, UUID channelId, String memberId, MemberRole role, String tenancyId, Instant joinedAt, Long lastReadMessageId)`
- `MemberRole { PARTICIPANT, OBSERVER, MODERATOR }`
- `UnreadCount(UUID channelId, String channelName, long count, Long latestMessageId)`
- `ChannelMembershipStore` interface — `put`, `find`, `findByChannel`, `findByMember`, `updateRole`, `updateLastReadMessageId`, `delete`, `deleteAll`
- `ReactiveChannelMembershipStore` — `Uni<T>` variants of above

- [ ] **Step 1: Create API types**

`MemberRole.java`:
```java
package io.casehub.qhorus.api.channel;

public enum MemberRole {
    PARTICIPANT, OBSERVER, MODERATOR
}
```

`ChannelMembership.java`:
```java
package io.casehub.qhorus.api.channel;

import java.time.Instant;
import java.util.UUID;

public record ChannelMembership(
    Long id,
    UUID channelId,
    String memberId,
    MemberRole role,
    String tenancyId,
    Instant joinedAt,
    Long lastReadMessageId) {}
```

`UnreadCount.java`:
```java
package io.casehub.qhorus.api.channel;

import java.util.UUID;

public record UnreadCount(
    UUID channelId,
    String channelName,
    long count,
    Long latestMessageId) {}
```

- [ ] **Step 2: Create store interfaces**

`ChannelMembershipStore.java`:
```java
package io.casehub.qhorus.api.store;

import io.casehub.qhorus.api.channel.ChannelMembership;
import io.casehub.qhorus.api.channel.MemberRole;
import java.util.*;

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

`ReactiveChannelMembershipStore.java` — same with `Uni<T>` returns.

- [ ] **Step 3: Create Flyway migration V32**

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

- [ ] **Step 4: Compile API module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#332): ChannelMembership API types, store interfaces, V32 migration

ChannelMembership record, MemberRole enum, UnreadCount record,
ChannelMembershipStore + ReactiveChannelMembershipStore interfaces.
V32 Flyway migration creates channel_membership table.

Refs #332"
```

---

### Task 5: Entity + JPA Store + InMemory Stores

JPA entity, JPA store implementation, and InMemory stores (blocking + reactive) with contract tests.

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelMembershipEntity.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelMembershipStore.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaChannelMembershipStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelMembershipStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveChannelMembershipStore.java`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/ChannelMembershipStoreContractTest.java`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryChannelMembershipStoreTest.java`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactiveChannelMembershipStoreTest.java`
- Modify: `runtime/src/test/resources/import-qhorus-test.sql` — add channel_membership DDL

**Interfaces:**
- Consumes: `ChannelMembership`, `MemberRole`, `ChannelMembershipStore`, `ReactiveChannelMembershipStore` from Task 4
- Produces: `ChannelMembershipEntity` JPA entity; `JpaChannelMembershipStore` @ApplicationScoped; `InMemoryChannelMembershipStore` @Alternative @Priority(1)

- [ ] **Step 1: Write contract test (abstract base)**

```java
package io.casehub.qhorus.persistence.memory.contract;

import io.casehub.qhorus.api.channel.ChannelMembership;
import io.casehub.qhorus.api.channel.MemberRole;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

abstract class ChannelMembershipStoreContractTest {

    protected abstract ChannelMembership put(ChannelMembership m);
    protected abstract java.util.Optional<ChannelMembership> find(UUID channelId, String memberId);
    protected abstract java.util.List<ChannelMembership> findByChannel(UUID channelId);
    protected abstract void updateRole(UUID channelId, String memberId, MemberRole role);
    protected abstract void updateLastReadMessageId(UUID channelId, String memberId, Long messageId);
    protected abstract boolean delete(UUID channelId, String memberId);
    protected abstract void deleteAll(UUID channelId);

    private UUID channelId;

    @BeforeEach
    void setUp() { channelId = UUID.randomUUID(); }

    @Test
    void putAndFind() {
        var m = new ChannelMembership(null, channelId, "agent-1", MemberRole.PARTICIPANT, "default", Instant.now(), null);
        var saved = put(m);
        assertThat(saved.id()).isNotNull();
        var found = find(channelId, "agent-1");
        assertThat(found).isPresent();
        assertThat(found.get().role()).isEqualTo(MemberRole.PARTICIPANT);
    }

    @Test
    void findByChannel_returnsAll() {
        put(new ChannelMembership(null, channelId, "agent-1", MemberRole.PARTICIPANT, "default", Instant.now(), null));
        put(new ChannelMembership(null, channelId, "agent-2", MemberRole.OBSERVER, "default", Instant.now(), null));
        assertThat(findByChannel(channelId)).hasSize(2);
    }

    @Test
    void updateRole() {
        put(new ChannelMembership(null, channelId, "agent-1", MemberRole.PARTICIPANT, "default", Instant.now(), null));
        updateRole(channelId, "agent-1", MemberRole.MODERATOR);
        assertThat(find(channelId, "agent-1").get().role()).isEqualTo(MemberRole.MODERATOR);
    }

    @Test
    void updateLastReadMessageId() {
        put(new ChannelMembership(null, channelId, "agent-1", MemberRole.PARTICIPANT, "default", Instant.now(), 0L));
        updateLastReadMessageId(channelId, "agent-1", 42L);
        assertThat(find(channelId, "agent-1").get().lastReadMessageId()).isEqualTo(42L);
    }

    @Test
    void delete() {
        put(new ChannelMembership(null, channelId, "agent-1", MemberRole.PARTICIPANT, "default", Instant.now(), null));
        assertThat(delete(channelId, "agent-1")).isTrue();
        assertThat(find(channelId, "agent-1")).isEmpty();
    }

    @Test
    void deleteAll() {
        put(new ChannelMembership(null, channelId, "agent-1", MemberRole.PARTICIPANT, "default", Instant.now(), null));
        put(new ChannelMembership(null, channelId, "agent-2", MemberRole.OBSERVER, "default", Instant.now(), null));
        deleteAll(channelId);
        assertThat(findByChannel(channelId)).isEmpty();
    }

    @Test
    void delete_nonexistent_returnsFalse() {
        assertThat(delete(channelId, "nobody")).isFalse();
    }
}
```

- [ ] **Step 2: Implement InMemoryChannelMembershipStore**

Follow existing `InMemoryTopicStore` pattern — `ConcurrentHashMap`, `@Alternative @Priority(1)`.

- [ ] **Step 3: Write blocking contract test runner**

```java
class InMemoryChannelMembershipStoreTest extends ChannelMembershipStoreContractTest {
    private final InMemoryChannelMembershipStore store = new InMemoryChannelMembershipStore();
    // delegate all abstract methods to store
}
```

- [ ] **Step 4: Run contract tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryChannelMembershipStoreTest
```

- [ ] **Step 5: Implement InMemoryReactiveChannelMembershipStore + reactive runner**

Wraps blocking store with `Uni.createFrom().item(...)`. Reactive runner calls `.await().indefinitely()`.

- [ ] **Step 6: Implement ChannelMembershipEntity**

```java
@Entity(name = "ChannelMembership")
@Table(name = "channel_membership")
public class ChannelMembershipEntity extends PanacheEntityBase {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long id;
    @Column(name = "channel_id", nullable = false)
    public UUID channelId;
    @Column(name = "member_id", nullable = false)
    public String memberId;
    @Enumerated(EnumType.STRING)
    @Column(name = "member_role", nullable = false)
    public MemberRole memberRole;
    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId;
    @Column(name = "joined_at", nullable = false)
    public Instant joinedAt;
    @Column(name = "last_read_message_id")
    public Long lastReadMessageId;

    public ChannelMembership toDomain() { ... }
    public static ChannelMembershipEntity fromDomain(ChannelMembership m) { ... }
}
```

- [ ] **Step 7: Implement JpaChannelMembershipStore**

Follow `JpaTopicStore` pattern — `@ApplicationScoped`, inject `@PersistenceUnit("qhorus") EntityManager`.

- [ ] **Step 8: Implement ReactiveJpaChannelMembershipStore**

`@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")` — follow `ReactiveJpaTopicStore` pattern.

- [ ] **Step 9: Update import-qhorus-test.sql**

Add `CREATE TABLE IF NOT EXISTS channel_membership (...)` DDL.

- [ ] **Step 10: Update examples/type-system application.properties**

Add `InMemoryChannelMembershipStore` and `InMemoryReactiveChannelMembershipStore` to `quarkus.arc.selected-alternatives` if the reactive stack is enabled.

- [ ] **Step 11: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat(#332): ChannelMembershipEntity, JPA + InMemory stores, contract tests

JPA entity with qhorus PU, blocking + reactive JPA stores,
InMemory + InMemoryReactive stores with contract test suite.
V32 migration for channel_membership table.

Refs #332"
```

---

### Task 6: ChannelMembershipService

Service layer with CDI-free unit tests.

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelMembershipService.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelMembershipService.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMembershipServiceTest.java`

**Interfaces:**
- Consumes: `ChannelMembershipStore`, `MessageStore` (for unread counts)
- Produces: `join()`, `leave()`, `listMembers()`, `markRead()`, `getUnreadCounts()`

- [ ] **Step 1: Write service unit tests**

CDI-free tests using `InMemoryChannelMembershipStore` and `InMemoryMessageStore`:

```java
@Test
void join_createsNewMembership() { ... }

@Test
void join_existingMember_updatesRolePreservesJoinedAt() { ... }

@Test
void join_initializesLastReadMessageIdToMaxMessageId() { ... }

@Test
void join_emptyChannel_initializesLastReadMessageIdToZero() { ... }

@Test
void leave_removesMembership() { ... }

@Test
void leave_nonMember_noOp() { ... }

@Test
void markRead_advancesForward() { ... }

@Test
void markRead_regressIsNoOp() { ... }

@Test
void getUnreadCounts_excludesOwnMessages() { ... }

@Test
void getUnreadCounts_excludesEventMessages() { ... }

@Test
void getUnreadCounts_multipleChannels() { ... }
```

- [ ] **Step 2: Implement ChannelMembershipService**

```java
@ApplicationScoped
public class ChannelMembershipService {
    @Inject ChannelMembershipStore membershipStore;
    @Inject MessageStore messageStore;

    public ChannelMembership join(UUID channelId, String memberId, MemberRole role, String tenancyId) {
        var existing = membershipStore.find(channelId, memberId);
        if (existing.isPresent()) {
            membershipStore.updateRole(channelId, memberId, role);
            return membershipStore.find(channelId, memberId).orElseThrow();
        }
        Long maxId = messageStore.maxId(channelId);
        return membershipStore.put(new ChannelMembership(
                null, channelId, memberId, role, tenancyId,
                Instant.now(), maxId != null ? maxId : 0L));
    }

    public void leave(UUID channelId, String memberId) {
        membershipStore.delete(channelId, memberId);
    }

    public List<ChannelMembership> listMembers(UUID channelId) {
        return membershipStore.findByChannel(channelId);
    }

    public void markRead(UUID channelId, String memberId, Long messageId) {
        var m = membershipStore.find(channelId, memberId);
        if (m.isPresent() && (m.get().lastReadMessageId() == null || messageId > m.get().lastReadMessageId())) {
            membershipStore.updateLastReadMessageId(channelId, memberId, messageId);
        }
    }

    public Map<UUID, UnreadCount> getUnreadCounts(String memberId, String tenancyId) {
        // For each channel the member belongs to, count messages
        // with id > lastReadMessageId AND sender != memberId AND type != EVENT
        ...
    }
}
```

Note: `messageStore.maxId(channelId)` may not exist yet. If not, add it to `MessageStore` interface and implement in both JPA and InMemory stores. Also add `messageStore.countUnread(channelId, Long afterId, String excludeSender)` for unread counts.

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelMembershipServiceTest
```

- [ ] **Step 4: Implement ReactiveChannelMembershipService**

`@IfBuildProperty` gated. Delegates to reactive stores with `Uni<T>`.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#332): ChannelMembershipService — join, leave, markRead, unreadCounts

Idempotent join with clean-slate unread. Forward-only markRead. Unread
counts exclude own messages and EVENTs. CDI-free unit tests.

Refs #332"
```

---

### Task 7: Auto-Membership in ChannelGateway

Lazy auto-membership on first human interaction.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/ChannelGatewayMembershipTest.java`

**Interfaces:**
- Consumes: `ChannelMembershipService.join()` from Task 6
- Produces: Auto-membership on `receiveHumanMessage()` and `receiveObserverSignal()`

- [ ] **Step 1: Write integration test**

`@QuarkusTest` — send a human message, verify membership auto-created:

```java
@Test
void receiveHumanMessage_autoCreatesMembership() {
    // Create channel
    // Send inbound human message via gateway.receiveHumanMessage()
    // Verify: membershipStore.find(channelId, "human:alice") returns PARTICIPANT
}

@Test
void receiveHumanMessage_existingMember_noChange() {
    // Create channel + existing membership
    // Send another human message
    // Verify: role unchanged, joinedAt unchanged
}

@Test
void receiveObserverSignal_autoCreatesObserverMembership() {
    // Create channel
    // Send observer signal via gateway.receiveObserverSignal()
    // Verify: membershipStore.find(channelId, "human:bob") returns OBSERVER
}
```

- [ ] **Step 2: Inject ChannelMembershipService into ChannelGateway**

Add `ChannelMembershipService membershipService` field.

- [ ] **Step 3: Add auto-membership in receiveHumanMessage()**

After normalisation, before dispatch:
```java
String tenancyId = crossTenantChannelStore.findById(channel.id())
        .map(Channel::tenancyId).orElse(TenancyConstants.DEFAULT_TENANT_ID);
membershipService.join(channel.id(), n.senderInstanceId(), MemberRole.PARTICIPANT, tenancyId);
```

- [ ] **Step 4: Add auto-membership in receiveObserverSignal()**

```java
String senderId = "human:" + signal.externalSenderId();
String tenancyId = crossTenantChannelStore.findById(channel.id())
        .map(Channel::tenancyId).orElse(TenancyConstants.DEFAULT_TENANT_ID);
membershipService.join(channel.id(), senderId, MemberRole.OBSERVER, tenancyId);
```

- [ ] **Step 5: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelGatewayMembershipTest
```

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#332): lazy auto-membership on first human interaction

receiveHumanMessage() auto-creates PARTICIPANT membership.
receiveObserverSignal() auto-creates OBSERVER membership.
Tenancy derived from channel entity.

Refs #332"
```

---

### Task 8: Membership MCP Tools + delete_channel Integration

MCP tools for membership management and channel delete cascade.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — `MembershipSummary` record
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — 5 new tools + delete_channel update
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — reactive mirrors
- Test: `runtime/src/test/java/io/casehub/qhorus/mcp/MembershipToolTest.java`

**Interfaces:**
- Consumes: `ChannelMembershipService` from Task 6
- Produces: `join_channel`, `leave_channel`, `list_members`, `mark_channel_read`, `get_unread_counts` MCP tools

- [ ] **Step 1: Write MCP tool tests**

```java
@QuarkusTest
class MembershipToolTest {
    @Test void joinChannel_createsMembership() { ... }
    @Test void joinChannel_withRole_setsRole() { ... }
    @Test void leaveChannel_removesMembership() { ... }
    @Test void listMembers_returnsAll() { ... }
    @Test void markChannelRead_updatesLastRead() { ... }
    @Test void markChannelRead_nullMessageId_usesLatest() { ... }
    @Test void getUnreadCounts_returnsCorrectCounts() { ... }
    @Test void deleteChannel_cleansUpMemberships() { ... }
}
```

- [ ] **Step 2: Add MembershipSummary record to QhorusMcpToolsBase**

```java
public record MembershipSummary(
    String channelId, String channelName, String memberId,
    String role, String joinedAt, Long lastReadMessageId) {}
```

- [ ] **Step 3: Implement 5 MCP tools in QhorusMcpTools**

`join_channel`, `leave_channel`, `list_members`, `mark_channel_read`, `get_unread_counts` — all `@Tool`-annotated. Channel resolution via `resolveChannel()`. MemberId from instance context.

- [ ] **Step 4: Update delete_channel**

Add `membershipStore.deleteAll(ch.id())` before existing cleanup (before `reactionStore.deleteByChannel`).

- [ ] **Step 5: Mirror in ReactiveQhorusMcpTools**

- [ ] **Step 6: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MembershipToolTest
```

- [ ] **Step 7: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#332): membership MCP tools + delete_channel cascade

join_channel, leave_channel, list_members, mark_channel_read,
get_unread_counts tools. delete_channel cleans up memberships.

Refs #332"
```

---

### Task 9: CLAUDE.md + FlywayMigrationSchemaTest + Final Verification

Update project documentation and verify Flyway migration produces correct schema.

**Files:**
- Modify: `CLAUDE.md` — add #331/#332 conventions
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/migration/FlywayMigrationSchemaTest.java` — verify V32

- [ ] **Step 1: Extend FlywayMigrationSchemaTest**

Add assertions for `channel_membership` table columns, constraints, indexes.

- [ ] **Step 2: Run migration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FlywayMigrationSchemaTest
```

- [ ] **Step 3: Update CLAUDE.md**

Add conventions for:
- `ArtefactRef`, `ArtefactType`, `SelectionScope` — API types in `api/message/`
- `ArtefactRefListConverter` — JPA converter in runtime
- `ChannelMembership`, `MemberRole`, `UnreadCount` — API types in `api/channel/`
- `ChannelMembershipStore` — store pattern, `ChannelMembershipService` — service
- V32 migration, next domain migration is V33
- Testing conventions: InMemory membership stores, auto-membership in gateway tests
- Lazy auto-membership on first human interaction, NOT backend registration

- [ ] **Step 4: Run full build one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: all modules compile and all tests pass.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "docs(#331,#332): update CLAUDE.md — ArtefactRef pipeline, ChannelMembership, V32 boundary

Refs #331, #332"
```

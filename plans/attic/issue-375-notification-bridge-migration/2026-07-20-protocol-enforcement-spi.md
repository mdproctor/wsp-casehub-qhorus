# Channel Protocol Enforcement SPI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #357 — channel protocol enforcement SPI
**Issue group:** #357

**Goal:** Add a pluggable ChannelProtocol SPI with 4 built-in protocols
that surface advisory violations in DispatchResult at dispatch time.

**Architecture:** SPI interface + ProtocolContext in `api/spi/`. ProtocolRegistry
(CDI discovery, ProjectionRegistry pattern) and 4 built-in protocol beans in
`runtime/message/protocol/`. Two new store methods. Two new Channel fields
(protocols, protocolParticipants). Protocol evaluation runs in
MessageService.dispatch() after CorrelationIntegrityChecker, before LAST_WRITE.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, SmallRye Config, H2 (tests)

## Global Constraints

- Flyway domain migrations use V39 (V38 taken by channel_summary)
- Advisory prefix convention: `[PROTOCOL_NAME] description`
- Channel list fields normalise null → `List.of()` (not null-preserving)
- All enforcement is advisory — protocols never throw
- `recentMessages` ordering: oldest-first (ascending by ID)
- IntelliJ MCP required for all code operations
- Pre-release platform: no backward compat concerns

---

### Task 1: SPI Contract — ChannelProtocol + ProtocolContext

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/ProtocolContext.java`

**Interfaces:**
- Consumes: `MessageView` from `api/message/`, `Commitment` from `api/message/`
- Produces: `ChannelProtocol.protocolName(): String`, `ChannelProtocol.evaluate(ProtocolContext): List<String>`, `ProtocolContext` record

- [ ] **Step 1: Create ChannelProtocol interface**

```java
package io.casehub.qhorus.api.spi;

import java.util.List;

public interface ChannelProtocol {
    String protocolName();
    List<String> evaluate(ProtocolContext context);
}
```

Use `ide_create_file` for `api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java`.

- [ ] **Step 2: Create ProtocolContext record**

```java
package io.casehub.qhorus.api.spi;

import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.MessageView;

import java.util.List;
import java.util.UUID;

public record ProtocolContext(
    UUID channelId,
    String channelName,
    MessageType incomingType,
    String sender,
    String correlationId,
    List<String> protocolParticipants,
    List<MessageView> recentMessages,
    List<Commitment> activeCommitments
) {
    public ProtocolContext {
        protocolParticipants = protocolParticipants != null ? List.copyOf(protocolParticipants) : List.of();
        recentMessages = recentMessages != null ? List.copyOf(recentMessages) : List.of();
        activeCommitments = activeCommitments != null ? List.copyOf(activeCommitments) : List.of();
    }
}
```

Use `ide_create_file` for `api/src/main/java/io/casehub/qhorus/api/spi/ProtocolContext.java`.

- [ ] **Step 3: Build api module to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/main/java/io/casehub/qhorus/api/spi/ChannelProtocol.java api/src/main/java/io/casehub/qhorus/api/spi/ProtocolContext.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): ChannelProtocol SPI + ProtocolContext record in api/spi/"
```

---

### Task 2: Channel Record — Add protocols and protocolParticipants Fields

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V39__channel_protocols.sql`

**Interfaces:**
- Consumes: nothing new
- Produces: `Channel.protocols(): List<String>`, `Channel.protocolParticipants(): List<String>`, `ChannelCreateRequest.Builder.protocols(List<String>)`, `ChannelCreateRequest.Builder.protocolParticipants(List<String>)`

- [ ] **Step 1: Add fields to Channel record**

Add `List<String> protocols` and `List<String> protocolParticipants` to the `Channel` record — after `reviewerInstances`, before `tenancyId`. Update the compact constructor to normalise both: `protocols = protocols != null ? List.copyOf(protocols) : List.of();` (same for `protocolParticipants`). Update the backward-compat constructor to pass `null, null` for the two new fields (delegates to canonical). Update `fromRequest()` to pass `req.protocols()` and `req.protocolParticipants()`. Update `toBuilder()` and `Builder` class with the two new fields and builder methods. Update `Builder.build()`.

Use `ide_edit_member` on `Channel` for the record declaration, compact constructor, backward-compat constructor, `fromRequest`, `toBuilder`, and `Builder` class.

- [ ] **Step 2: Add fields to ChannelCreateRequest**

Add `List<String> protocols` and `List<String> protocolParticipants` to the `ChannelCreateRequest` record — after `reviewerInstances`, before connector fields. Update compact constructor: normalise both to `List.of()` like `allowedWriters`. Update backward-compat constructors to pass `null, null`. Add builder fields and methods `.protocols(List<String>)`, `.protocolParticipants(List<String>)`. Update `Builder.build()`.

Use `ide_edit_member` on `ChannelCreateRequest` for each changed member.

- [ ] **Step 3: Add columns to ChannelEntity**

Add `protocols` and `protocolParticipants` fields (TEXT nullable, CSV pattern matching `allowedWriters`). Update `fromDomain()` to map: `e.protocols = joinCsv(channel.protocols());` and `e.protocolParticipants = joinCsv(channel.protocolParticipants());`. Update `toDomain()` to pass `splitCsv(protocols)` and `splitCsv(protocolParticipants)` in the constructor call.

Use `ide_insert_member` for the two new fields (after `reviewerInstances`). Use `ide_edit_member` for `fromDomain` and `toDomain`.

- [ ] **Step 4: Create V39 migration**

```sql
ALTER TABLE channel ADD COLUMN protocols TEXT;
ALTER TABLE channel ADD COLUMN protocol_participants TEXT;
```

Use `ide_create_file` for `runtime/src/main/resources/db/qhorus/migration/V39__channel_protocols.sql`.

- [ ] **Step 5: Verify compilation across all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -q`

This will surface any callers of Channel/ChannelCreateRequest constructors that need updating (e.g., in `persistence-memory/`, `connector-backend/`, `testing/`, `examples/`). Fix all compilation errors before proceeding.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): Channel + ChannelCreateRequest + ChannelEntity gain protocols and protocolParticipants fields; V39 migration"
```

---

### Task 3: Store Methods — findRecent + findOpenByChannelId

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/MessageStore.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CommitmentStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryMessageStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCommitmentStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveMessageStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveCommitmentStore.java`
- Modify: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/MessageStoreContractTest.java`
- Modify: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/CommitmentStoreContractTest.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveMessageStore.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveCommitmentStore.java`
- Modify: JPA store implementations in `runtime/`

**Interfaces:**
- Consumes: `MessageView`, `Commitment`, `CommitmentState.isActive()`, `MessageType.EVENT`
- Produces: `MessageStore.findRecent(UUID, int): List<MessageView>`, `CommitmentStore.findOpenByChannelId(UUID): List<Commitment>`

- [ ] **Step 1: Write contract tests for findRecent**

Add to `MessageStoreContractTest`:

```java
@Test
void findRecent_returnsLastNMessages_excludingEvents_oldestFirst() {
    // Setup: put 5 messages (mix of types including EVENT)
    // Assert: findRecent(channelId, 3) returns 3 non-EVENT messages, oldest-first
}

@Test
void findRecent_emptyChannel_returnsEmptyList() {
    // Assert: findRecent(randomUUID, 10) returns empty list
}

@Test
void findRecent_limitLargerThanAvailable_returnsAll() {
    // Setup: put 2 non-EVENT messages
    // Assert: findRecent(channelId, 50) returns 2 messages oldest-first
}
```

- [ ] **Step 2: Add findRecent to MessageStore interface**

```java
List<MessageView> findRecent(UUID channelId, int limit);
```

Use `ide_insert_member` on `MessageStore` after `updateChannelId`.

- [ ] **Step 3: Implement InMemoryMessageStore.findRecent**

```java
@Override
public List<MessageView> findRecent(UUID channelId, int limit) {
    List<Message> filtered = store.values().stream()
            .filter(m -> m.channelId().equals(channelId))
            .filter(m -> m.messageType() != MessageType.EVENT)
            .sorted(Comparator.comparingLong(Message::id).reversed())
            .limit(limit)
            .toList();
    List<MessageView> views = new ArrayList<>(filtered.size());
    for (int i = filtered.size() - 1; i >= 0; i--) {
        views.add(QhorusEntityMapper.toMessageView(filtered.get(i)));
    }
    return views;
}
```

Note: `QhorusEntityMapper.toMessageView(Message)` is in `runtime/`. InMemoryMessageStore is in `persistence-memory/` which does not depend on `runtime/`. Instead, construct `MessageView` directly from `Message` fields inline (same field mapping as `QhorusEntityMapper.toMessageView`).

Use `ide_insert_member` on `InMemoryMessageStore`.

- [ ] **Step 4: Run contract tests to verify findRecent passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest="InMemoryMessageStoreTest" -q`

- [ ] **Step 5: Write contract tests for findOpenByChannelId**

Add to `CommitmentStoreContractTest`:

```java
@Test
void findOpenByChannelId_returnsOpenAndAcknowledged() {
    // Setup: create commitments in various states (OPEN, ACKNOWLEDGED, FULFILLED, DECLINED)
    // Assert: only OPEN and ACKNOWLEDGED returned (CommitmentState.isActive())
}

@Test
void findOpenByChannelId_emptyChannel_returnsEmpty() {
    // Assert: returns empty list for channel with no commitments
}

@Test
void findOpenByChannelId_doesNotReturnOtherChannels() {
    // Setup: OPEN commitment on channel A, OPEN on channel B
    // Assert: findOpenByChannelId(channelA) returns only channel A's commitment
}
```

- [ ] **Step 6: Add findOpenByChannelId to CommitmentStore interface**

```java
List<Commitment> findOpenByChannelId(UUID channelId);
```

Use `ide_insert_member` on `CommitmentStore` after `findByChannel`.

- [ ] **Step 7: Implement InMemoryCommitmentStore.findOpenByChannelId**

```java
@Override
public List<Commitment> findOpenByChannelId(UUID channelId) {
    return store.values().stream()
            .filter(c -> c.channelId().equals(channelId))
            .filter(c -> c.state().isActive())
            .toList();
}
```

Use `ide_insert_member` on `InMemoryCommitmentStore`.

- [ ] **Step 8: Run contract tests to verify findOpenByChannelId passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest="InMemoryCommitmentStoreTest" -q`

- [ ] **Step 9: Add reactive counterparts**

Add `findRecentAsync(UUID, int): Uni<List<MessageView>>` to `ReactiveMessageStore`. Implement in `InMemoryReactiveMessageStore` (delegate to blocking, wrap in `Uni.createFrom().item()`).

Add `findOpenByChannelIdAsync(UUID): Uni<List<Commitment>>` to `ReactiveCommitmentStore`. Implement in `InMemoryReactiveCommitmentStore`.

Run reactive contract tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest="InMemoryReactiveMessageStoreTest,InMemoryReactiveCommitmentStoreTest" -q`

- [ ] **Step 10: Add JPA implementations**

Add `findRecent` to `JpaMessageStore` (runtime/):
```sql
SELECT m FROM Message m WHERE m.channelId = ?1 AND m.messageType != io.casehub.qhorus.runtime.message.MessageType.EVENT ORDER BY m.id DESC
```
Limit via `setMaxResults(limit)`. Reverse the result list. Map to `MessageView` via `QhorusEntityMapper.toMessageView()`.

Add `findOpenByChannelId` to `JpaCommitmentStore`:
```sql
SELECT c FROM Commitment c WHERE c.channelId = ?1 AND (c.state = 'OPEN' OR c.state = 'ACKNOWLEDGED')
```

Add reactive JPA counterparts if reactive JPA store classes exist.

- [ ] **Step 11: Add stubs to runtime test stubs**

Check if `StubReactiveCommitmentStore` and `StubReactiveMessageStore` in `runtime/src/test/` need the new methods added (they throw `UnsupportedOperationException` — just add the new method with the same pattern).

- [ ] **Step 12: Compile all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -q`

Fix any compilation errors from missing implementations.

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): MessageStore.findRecent + CommitmentStore.findOpenByChannelId — store methods for protocol evaluation"
```

---

### Task 4: ProtocolRegistry + Config

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/protocol/ProtocolRegistry.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/ProtocolRegistryTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java`

**Interfaces:**
- Consumes: `ChannelProtocol` from `api/spi/`
- Produces: `ProtocolRegistry.forProtocols(List<String>): List<ChannelProtocol>`, `ProtocolRegistry.allNames(): Set<String>`, `QhorusConfig.Protocol` sub-interface

- [ ] **Step 1: Write ProtocolRegistryTest (CDI-free)**

Follow the `ProjectionRegistryTest` pattern exactly. Tests:

```java
package io.casehub.qhorus.runtime.message.protocol;

// Tests:
// - forProtocols_returnsMatchedProtocols_inOrder
// - forProtocols_skipsUnknownNames (returns partial, no throw)
// - forProtocols_emptyList_returnsEmpty
// - allNames_returnsSortedSet
// - duplicateName_throwsIllegalStateException_atConstruction
// - nullProtocolName_throwsIllegalStateException_atConstruction
// - blankProtocolName_throwsIllegalStateException_atConstruction
```

Stub `ChannelProtocol` implementation for tests — returns protocolName from constructor arg, evaluate returns empty list.

Use `ide_create_file`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ProtocolRegistryTest" -q`
Expected: FAIL — class does not exist yet

- [ ] **Step 3: Create ProtocolRegistry**

```java
package io.casehub.qhorus.runtime.message.protocol;

@ApplicationScoped
public class ProtocolRegistry {
    private final Map<String, ChannelProtocol> registry;

    @Inject
    ProtocolRegistry(@Any Instance<ChannelProtocol> protocols) {
        this(buildMap(protocols));
    }

    ProtocolRegistry(List<? extends ChannelProtocol> protocols) {
        this(buildMap(protocols));
    }

    private ProtocolRegistry(Map<String, ChannelProtocol> registry) {
        this.registry = registry;
    }

    // buildMap — same validation pattern as ProjectionRegistry
    // forProtocols — iterate names, skip unknowns with LOG.warn
    // allNames — TreeSet from registry.keySet()
}
```

Use `ide_create_file`.

- [ ] **Step 4: Add Protocol config to QhorusConfig**

```java
Protocol protocol();

interface Protocol {
    @WithDefault("50")
    int lookbackSize();

    RequestResponse requestResponse();
    TaskCompletion taskCompletion();
    ContributionRequired contributionRequired();

    interface RequestResponse {
        @WithDefault("3")
        int maxOpenQueries();
    }
    interface TaskCompletion {
        @WithDefault("3")
        int maxOpenCommands();
    }
    interface ContributionRequired {
        @WithDefault("2")
        int maxConsecutive();
    }
}
```

Use `ide_insert_member` on `QhorusConfig` for the `protocol()` method, then `ide_insert_member` for the `Protocol` interface.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ProtocolRegistryTest" -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): ProtocolRegistry + QhorusConfig.Protocol — CDI registry and configuration"
```

---

### Task 5: Built-in Protocols — REQUEST_RESPONSE + TASK_COMPLETION

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/protocol/RequestResponseProtocol.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/protocol/TaskCompletionProtocol.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/RequestResponseProtocolTest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/TaskCompletionProtocolTest.java`

**Interfaces:**
- Consumes: `ProtocolContext.activeCommitments()`, `QhorusConfig.Protocol`
- Produces: `RequestResponseProtocol` bean (name "REQUEST_RESPONSE"), `TaskCompletionProtocol` bean (name "TASK_COMPLETION")

- [ ] **Step 1: Write RequestResponseProtocolTest (CDI-free)**

Construct `ProtocolContext` directly with test data. No mocks needed — pure function of context.

```java
// Tests:
// - noActiveCommitments_noAdvisory
// - openQueriesUnderThreshold_noAdvisory
// - openQueriesAtThreshold_advisoryOnNewQuery (prefix: [REQUEST_RESPONSE])
// - nonResponseWithOpenQueries_advisory
// - responseWithOpenQueries_noAdvisory
// - commandCommitmentsIgnored (only QUERY filtered)
```

- [ ] **Step 2: Write TaskCompletionProtocolTest (CDI-free)**

```java
// Tests:
// - noActiveCommitments_noAdvisory
// - openCommandsAtThreshold_advisoryOnNewCommand
// - senderIsObligorWithOpenObligation_advisory
// - senderIsNotObligor_noObligorAdvisory
// - queryCommitmentsIgnored (only COMMAND filtered)
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="RequestResponseProtocolTest,TaskCompletionProtocolTest" -q`

- [ ] **Step 4: Implement RequestResponseProtocol**

```java
@ApplicationScoped
public class RequestResponseProtocol implements ChannelProtocol {
    @Inject QhorusConfig config;

    @Override public String protocolName() { return "REQUEST_RESPONSE"; }

    @Override public List<String> evaluate(ProtocolContext ctx) {
        List<Commitment> openQueries = ctx.activeCommitments().stream()
                .filter(c -> c.messageType() == MessageType.QUERY)
                .toList();
        List<String> advisories = new ArrayList<>();
        int threshold = config.protocol().requestResponse().maxOpenQueries();
        if (ctx.incomingType() == MessageType.QUERY && openQueries.size() >= threshold) {
            advisories.add("[REQUEST_RESPONSE] " + openQueries.size()
                + " unanswered QUERYs in channel '" + ctx.channelName()
                + "' — consider waiting for responses");
        }
        if (ctx.incomingType() != MessageType.RESPONSE && !openQueries.isEmpty()) {
            advisories.add("[REQUEST_RESPONSE] channel '" + ctx.channelName()
                + "' has open QUERYs awaiting RESPONSE");
        }
        return advisories;
    }
}
```

- [ ] **Step 5: Implement TaskCompletionProtocol**

Similar pattern — filter `activeCommitments` to COMMAND type, threshold check, obligor check.

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="RequestResponseProtocolTest,TaskCompletionProtocolTest" -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): REQUEST_RESPONSE + TASK_COMPLETION built-in protocols"
```

---

### Task 6: Built-in Protocols — ROUND_ROBIN + CONTRIBUTION_REQUIRED

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/protocol/RoundRobinProtocol.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/protocol/ContributionRequiredProtocol.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/RoundRobinProtocolTest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/ContributionRequiredProtocolTest.java`

**Interfaces:**
- Consumes: `ProtocolContext.recentMessages()`, `ProtocolContext.protocolParticipants()`, `QhorusConfig.Protocol`
- Produces: `RoundRobinProtocol` bean (name "ROUND_ROBIN"), `ContributionRequiredProtocol` bean (name "CONTRIBUTION_REQUIRED")

- [ ] **Step 1: Write RoundRobinProtocolTest (CDI-free)**

```java
// Tests:
// - noParticipants_noAdvisory (skips when empty)
// - singleParticipant_noAdvisory (skips when ≤1)
// - correctTurn_noAdvisory
// - wrongTurn_advisory (with [ROUND_ROBIN] prefix)
// - noMessageHistory_noAdvisory (first message)
// - nonParticipantSender_noAdvisory_doesNotAdvanceTurn
// - wrapAround_afterLastParticipant_firstSpeaksAgain
```

Build `ProtocolContext` with explicit `protocolParticipants` and `recentMessages` (oldest-first `MessageView` list). Use `MessageView` constructor directly.

- [ ] **Step 2: Write ContributionRequiredProtocolTest (CDI-free)**

```java
// Tests:
// - noRecentMessages_noAdvisory
// - senderBelowConsecutiveThreshold_noAdvisory
// - senderAtConsecutiveThreshold_advisoryWithMissingContributors
// - allParticipantsContributing_noAdvisory
// - eventMessagesExcludedFromCount
// - fallbackParticipants_derivedFromDistinctSenders
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="RoundRobinProtocolTest,ContributionRequiredProtocolTest" -q`

- [ ] **Step 4: Implement RoundRobinProtocol**

```java
@ApplicationScoped
public class RoundRobinProtocol implements ChannelProtocol {
    @Override public String protocolName() { return "ROUND_ROBIN"; }

    @Override public List<String> evaluate(ProtocolContext ctx) {
        List<String> participants = ctx.protocolParticipants();
        if (participants.size() <= 1) return List.of();
        if (!participants.contains(ctx.sender())) return List.of();

        // Find last participant message (scan oldest-first list backward)
        String lastParticipantSender = null;
        for (int i = ctx.recentMessages().size() - 1; i >= 0; i--) {
            MessageView mv = ctx.recentMessages().get(i);
            if (mv.type() != MessageType.EVENT && participants.contains(mv.sender())) {
                lastParticipantSender = mv.sender();
                break;
            }
        }
        if (lastParticipantSender == null) return List.of();

        int lastIdx = participants.indexOf(lastParticipantSender);
        String expected = participants.get((lastIdx + 1) % participants.size());
        if (ctx.sender().equals(expected)) return List.of();

        return List.of("[ROUND_ROBIN] expected '" + expected
            + "' to speak next in channel '" + ctx.channelName()
            + "', got '" + ctx.sender() + "'");
    }
}
```

- [ ] **Step 5: Implement ContributionRequiredProtocol**

Scan `recentMessages` from end backward, count consecutive non-EVENT messages from `ctx.sender()`. If count >= threshold, collect participants who haven't contributed between the first of those consecutive messages and now.

```java
@ApplicationScoped
public class ContributionRequiredProtocol implements ChannelProtocol {
    @Inject QhorusConfig config;

    @Override public String protocolName() { return "CONTRIBUTION_REQUIRED"; }

    @Override public List<String> evaluate(ProtocolContext ctx) {
        int threshold = config.protocol().contributionRequired().maxConsecutive();
        List<String> participants = ctx.protocolParticipants().isEmpty()
                ? deriveParticipants(ctx.recentMessages())
                : ctx.protocolParticipants();
        if (participants.size() <= 1) return List.of();

        // Count consecutive messages from sender at tail (oldest-first, scan backward)
        int consecutive = 0;
        for (int i = ctx.recentMessages().size() - 1; i >= 0; i--) {
            MessageView mv = ctx.recentMessages().get(i);
            if (mv.type() == MessageType.EVENT) continue;
            if (mv.sender().equals(ctx.sender())) consecutive++;
            else break;
        }
        // The incoming message would make it consecutive + 1
        if (consecutive + 1 < threshold) return List.of();

        Set<String> contributed = new HashSet<>();
        // Scan the consecutive run to find who else contributed
        // (nobody — they're all from ctx.sender())
        List<String> missing = participants.stream()
                .filter(p -> !p.equals(ctx.sender()))
                .toList();
        if (missing.isEmpty()) return List.of();

        return List.of("[CONTRIBUTION_REQUIRED] '" + ctx.sender()
            + "' has sent " + (consecutive + 1)
            + " consecutive messages in channel '" + ctx.channelName()
            + "' without contributions from: " + String.join(", ", missing));
    }

    private List<String> deriveParticipants(List<MessageView> messages) {
        return messages.stream()
                .filter(m -> m.type() != MessageType.EVENT)
                .map(MessageView::sender)
                .distinct()
                .toList();
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="RoundRobinProtocolTest,ContributionRequiredProtocolTest" -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): ROUND_ROBIN + CONTRIBUTION_REQUIRED built-in protocols"
```

---

### Task 7: Dispatch Pipeline Integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` (if exists)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` (new setProtocols/setProtocolParticipants methods)
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java` (new interface methods)

**Interfaces:**
- Consumes: `ProtocolRegistry.forProtocols()`, `MessageStore.findRecent()`, `CommitmentStore.findOpenByChannelId()`, `QhorusConfig.Protocol.lookbackSize()`
- Produces: advisories in `DispatchResult.advisories()`

- [ ] **Step 1: Add setProtocols and setProtocolParticipants to ChannelManager interface**

```java
Channel setProtocols(UUID channelId, List<String> protocols);
Channel setProtocolParticipants(UUID channelId, List<String> protocolParticipants);
```

Use `ide_insert_member` on `ChannelManager` after `setReviewerInstances`.

- [ ] **Step 2: Implement in ChannelService**

Follow `setAllowedWriters` pattern exactly:

```java
@Override
@Transactional
public Channel setProtocols(UUID channelId, List<String> protocols) {
    Channel ch = channelStore.find(channelId)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelId));
    return channelStore.put(ch.toBuilder().protocols(protocols).build());
}

@Override
@Transactional
public Channel setProtocolParticipants(UUID channelId, List<String> protocolParticipants) {
    Channel ch = channelStore.find(channelId)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelId));
    return channelStore.put(ch.toBuilder().protocolParticipants(protocolParticipants).build());
}
```

Use `ide_insert_member` on `ChannelService` after `setReviewerInstances`.

- [ ] **Step 3: Add ProtocolRegistry injection to MessageService**

Add field: `@Inject ProtocolRegistry protocolRegistry;`

Use `ide_insert_member` on `MessageService` after `correlationIntegrityChecker` field.

- [ ] **Step 4: Add protocol evaluation block to dispatch()**

Insert the protocol evaluation block after the `correlationIntegrityChecker.check()` block (around line 230 in the current source), before the LAST_WRITE check. The block:

```java
if (ch != null && !ch.protocols().isEmpty()) {
    List<ChannelProtocol> activeProtocols = protocolRegistry.forProtocols(ch.protocols());
    if (!activeProtocols.isEmpty()) {
        List<MessageView> recent = messageStore.findRecent(ch.id(), config.protocol().lookbackSize());
        List<Commitment> activeCommitments = commitmentStore.findOpenByChannelId(ch.id());
        ProtocolContext protocolCtx = new ProtocolContext(
                ch.id(), ch.name(), dispatch.type(), dispatch.sender(),
                dispatch.correlationId(), ch.protocolParticipants(), recent, activeCommitments);
        for (ChannelProtocol protocol : activeProtocols) {
            List<String> violations = protocol.evaluate(protocolCtx);
            for (String v : violations) { LOG.warn(v); }
            if (!violations.isEmpty()) {
                advisories = new ArrayList<>(advisories);
                advisories.addAll(violations);
            }
        }
    }
    if (span != null) {
        span.addEvent("qhorus.enforcement.protocol");
    }
}
```

Use `ide_read_file` to find the exact insertion point (after `correlationIntegrityChecker` block), then use `ide_replace_member` on `dispatch` method with the full updated body.

Note: `advisories` is already mutable by this point (it may have been converted to `ArrayList` by the correlation checker block).

- [ ] **Step 5: Add reactive parity to ReactiveMessageService**

Mirror the same block with async queries: `messageStore.findRecentAsync()` and `commitmentStore.findOpenByChannelIdAsync()` composed via `Uni.combine().all().unis(...).asTuple()`.

- [ ] **Step 6: Update ReactiveChannelManager and ReactiveChannelService**

Add `setProtocols` and `setProtocolParticipants` to the reactive ChannelManager/Service interfaces if they exist, following the same pattern as blocking.

- [ ] **Step 7: Compile and run existing tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: all existing tests pass (protocol evaluation is a no-op when `ch.protocols()` is empty — which is the default).

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): protocol evaluation in MessageService.dispatch() — advisory pipeline integration"
```

---

### Task 8: MCP Tools

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` (if reactive tools exist)
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/ToolOverloadDiscoverabilityTest.java`

**Interfaces:**
- Consumes: `ProtocolRegistry.allNames()`, `ProtocolRegistry.forProtocols()`, `ChannelService.setProtocols()`, `ChannelService.setProtocolParticipants()`
- Produces: 4 new `@Tool` methods + `create_channel` modified

- [ ] **Step 1: Add ProtocolRegistry injection to QhorusMcpTools**

Add field: `@Inject ProtocolRegistry protocolRegistry;`

- [ ] **Step 2: Add list_protocols tool**

```java
@Tool(description = "List all registered protocol names")
public List<String> listProtocols() {
    return new ArrayList<>(protocolRegistry.allNames());
}
```

- [ ] **Step 3: Add set_channel_protocols tool**

```java
@Tool(description = "Set the protocols for a channel (full replacement)")
public ChannelDetail setChannelProtocols(String channel, String protocols) {
    Channel ch = resolveChannel(channel);
    List<String> protocolList = splitCsv(protocols);
    // Validate: ROUND_ROBIN requires protocolParticipants
    if (protocolList.contains("ROUND_ROBIN") && ch.protocolParticipants().isEmpty()) {
        throw new IllegalArgumentException(
            "ROUND_ROBIN requires protocolParticipants — set them first with set_protocol_participants");
    }
    Channel updated = channelService.setProtocols(ch.id(), protocolList);
    return toChannelDetail(updated, messageStore.countByChannel(ch.id()));
}
```

- [ ] **Step 4: Add set_protocol_participants tool**

```java
@Tool(description = "Set the protocol participants for a channel (full replacement)")
public ChannelDetail setProtocolParticipants(String channel, String participants) {
    Channel ch = resolveChannel(channel);
    Channel updated = channelService.setProtocolParticipants(ch.id(), splitCsv(participants));
    return toChannelDetail(updated, messageStore.countByChannel(ch.id()));
}
```

- [ ] **Step 5: Add get_channel_protocols tool**

```java
@Tool(description = "Get the protocols and protocol participants for a channel")
public Map<String, Object> getChannelProtocols(String channel) {
    Channel ch = resolveChannel(channel);
    return Map.of(
        "protocols", ch.protocols(),
        "protocol_participants", ch.protocolParticipants());
}
```

- [ ] **Step 6: Add protocols and protocol_participants to create_channel**

Add parameters to the existing `createChannel` `@Tool` method. Pass through to `ChannelCreateRequest.builder()`:
```java
.protocols(splitCsv(protocols))
.protocolParticipants(splitCsv(protocolParticipants))
```

- [ ] **Step 7: Update ToolOverloadDiscoverabilityTest**

The test uses reflection to ensure no public non-`@Tool` methods share names with `@Tool` methods. Verify the new methods don't create overload conflicts.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ToolOverloadDiscoverabilityTest" -q`

- [ ] **Step 8: Add ChannelDetail protocol fields**

Check `QhorusEntityMapper.toChannelDetail()` — it builds `ChannelDetail` from `Channel`. Add `protocols` and `protocolParticipants` to `ChannelDetail` if they're not already surfaced. Use `ide_find_class` to locate `ChannelDetail` and check its fields.

- [ ] **Step 9: Compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -q`

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#357): MCP tools — list_protocols, set_channel_protocols, set_protocol_participants, get_channel_protocols + create_channel protocols param"
```

---

### Task 9: Integration Test + Cross-Module Compilation

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/ProtocolDispatchIntegrationTest.java`
- Modify: various files for compilation fixes

**Interfaces:**
- Consumes: all prior tasks
- Produces: end-to-end proof that protocol advisories appear in DispatchResult

- [ ] **Step 1: Write integration test**

```java
@QuarkusTest
@TestTransaction
class ProtocolDispatchIntegrationTest {

    @Inject MessageService messageService;
    @Inject ChannelService channelService;
    @Inject InstanceService instanceService;

    @Test
    void dispatch_withRoundRobinProtocol_producesAdvisory_whenOutOfTurn() {
        // Register two instances
        // Create channel with protocols=["ROUND_ROBIN"], protocolParticipants=["agent-a","agent-b"]
        // Send message from agent-a (first turn — no advisory)
        // Send message from agent-a again (out of turn — advisory expected)
        // Assert: result.advisories() contains "[ROUND_ROBIN] expected 'agent-b'"
    }

    @Test
    void dispatch_withContributionRequired_producesAdvisory_whenConsecutive() {
        // Create channel with protocols=["CONTRIBUTION_REQUIRED"], participants=["a","b"]
        // Send 2 messages from "a" (threshold default=2)
        // Assert: result.advisories() contains "[CONTRIBUTION_REQUIRED]"
    }

    @Test
    void dispatch_withNoProtocols_producesNoProtocolAdvisory() {
        // Create channel without protocols
        // Send messages
        // Assert: no protocol-prefixed advisories in result
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ProtocolDispatchIntegrationTest" -q`
Expected: PASS

- [ ] **Step 3: Full build across all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q`

Fix any compilation errors in `examples/`, `connector-backend/`, `testing/`, `slack-channel/`, etc. These modules may reference Channel or ChannelCreateRequest constructors that need the new fields.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "test(#357): protocol dispatch integration test + cross-module compilation fixes"
```

---

### Task 10: CLAUDE.md Update

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: all prior tasks
- Produces: updated project documentation

- [ ] **Step 1: Add protocol documentation to CLAUDE.md**

Add a section documenting:
- `ChannelProtocol` SPI location and pattern
- Built-in protocol names and what they do
- `Channel.protocols` and `Channel.protocolParticipants` field semantics
- `ProtocolContext` fields and ordering convention
- Config keys under `casehub.qhorus.protocol.*`
- V39 migration
- Testing patterns (CDI-free with constructed ProtocolContext)

Place after the existing channel summary documentation. Follow the existing style — concise, focused on what a developer needs to know.

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs(#357): CLAUDE.md — protocol enforcement SPI conventions and testing patterns"
```

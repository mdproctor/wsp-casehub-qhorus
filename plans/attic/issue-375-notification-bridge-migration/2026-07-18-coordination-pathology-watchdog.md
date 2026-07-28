# Coordination Pathology Watchdog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #354 — coordination pathology watchdog conditions
**Issue group:** #354

**Goal:** Add 4 new watchdog conditions (LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER) that detect coordination pathologies observable from the message stream.

**Architecture:** Inline expansion of the existing WatchdogEvaluationService pattern — 4 new evaluate* methods, 4 AlertContext records, one shared Jaccard similarity utility. The Watchdog record gains `similarityPct` (Integer, nullable) and `conditionType` migrates from String to WatchdogConditionType enum.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, H2 (test), Flyway

## Global Constraints

- `WatchdogConditionType` enum is the single source of truth for condition type identity — no parallel String representations
- `WatchdogEntity.toDomain()` must handle unrecognized DB values gracefully via `WatchdogConditionType.fromString()` returning Optional (rollback safety)
- Content similarity uses whitespace-tokenised Jaccard — no embedding models, no external dependencies
- All evaluation methods use `CrossTenant*Store` interfaces and explicit `tenancyId` (per PP-20260609-67996e)
- Tests use `@QuarkusTest` + `@TestProfile(WatchdogEnabledProfile)` + `@TestTransaction` with unique channel names per test
- Next Flyway domain migration: V36

---

### Task 1: Data Model — Watchdog record, entity, migration, String→enum

The foundational change. Everything else depends on this.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogConditionType.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/Watchdog.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/query/WatchdogQuery.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEntity.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryWatchdogStore.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V36__watchdog_similarity_pct.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java` (existing tests must still pass)

**Interfaces:**
- Produces: `WatchdogConditionType.fromString(String) → Optional<WatchdogConditionType>`
- Produces: `Watchdog` record with `WatchdogConditionType conditionType` and `Integer similarityPct`
- Produces: `Watchdog.builder(WatchdogConditionType, String)` and `Watchdog.toBuilder()`
- Produces: `WatchdogQuery.byConditionType(WatchdogConditionType)`

- [ ] **Step 1: Write Flyway V36 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V36__watchdog_similarity_pct.sql`:

```sql
ALTER TABLE watchdog ADD COLUMN similarity_pct INTEGER;
```

- [ ] **Step 2: Add 4 new enum values and fromString() to WatchdogConditionType**

Use `ide_edit_member` on `WatchdogConditionType` (`file: api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogConditionType.java`, `member: WatchdogConditionType`):

```java
public enum WatchdogConditionType {
    BARRIER_STUCK, APPROVAL_PENDING, AGENT_STALE, CHANNEL_IDLE, QUEUE_DEPTH,
    CONTEXT_PRESSURE,
    LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER;

    public static Optional<WatchdogConditionType> fromString(String value) {
        try {
            return Optional.of(valueOf(value));
        } catch (IllegalArgumentException e) {
            return Optional.empty();
        }
    }
}
```

Add `import java.util.Optional;` to the file.

- [ ] **Step 3: Add similarityPct to Watchdog record, change conditionType to enum**

Use `ide_edit_member` on `Watchdog` (`file: api/src/main/java/io/casehub/qhorus/api/watchdog/Watchdog.java`, `member: Watchdog`).

The record gains:
- `conditionType` changes from `String` to `WatchdogConditionType`
- `Integer similarityPct` added as 6th field (after `thresholdCount`, before `notificationChannel`)
- `builder()` takes `WatchdogConditionType` instead of `String`
- `Builder.conditionType` field and constructor param become `WatchdogConditionType`
- `Builder` gains `similarityPct(Integer)` method
- `toBuilder()` carries `similarityPct`

```java
public record Watchdog(
        UUID id,
        WatchdogConditionType conditionType,
        String targetName,
        Integer thresholdSeconds,
        Integer thresholdCount,
        Integer similarityPct,
        String notificationChannel,
        String createdBy,
        String tenancyId,
        Instant createdAt,
        Instant lastFiredAt) {

    public Builder toBuilder() {
        return new Builder(conditionType, targetName).id(id)
                .thresholdSeconds(thresholdSeconds).thresholdCount(thresholdCount)
                .similarityPct(similarityPct)
                .notificationChannel(notificationChannel).createdBy(createdBy)
                .tenancyId(tenancyId).createdAt(createdAt).lastFiredAt(lastFiredAt);
    }

    public static Builder builder(WatchdogConditionType conditionType, String targetName) {
        return new Builder(conditionType, targetName);
    }

    public static final class Builder {
        private final WatchdogConditionType conditionType;
        private final String targetName;
        private UUID id;
        private Integer thresholdSeconds;
        private Integer thresholdCount;
        private Integer similarityPct;
        private String notificationChannel;
        private String createdBy;
        private String tenancyId;
        private Instant createdAt;
        private Instant lastFiredAt;

        private Builder(WatchdogConditionType conditionType, String targetName) {
            this.conditionType = conditionType;
            this.targetName = targetName;
        }

        public Builder id(UUID v) { this.id = v; return this; }
        public Builder thresholdSeconds(Integer v) { this.thresholdSeconds = v; return this; }
        public Builder thresholdCount(Integer v) { this.thresholdCount = v; return this; }
        public Builder similarityPct(Integer v) { this.similarityPct = v; return this; }
        public Builder notificationChannel(String v) { this.notificationChannel = v; return this; }
        public Builder createdBy(String v) { this.createdBy = v; return this; }
        public Builder tenancyId(String v) { this.tenancyId = v; return this; }
        public Builder createdAt(Instant v) { this.createdAt = v; return this; }
        public Builder lastFiredAt(Instant v) { this.lastFiredAt = v; return this; }

        public Watchdog build() {
            return new Watchdog(id, conditionType, targetName, thresholdSeconds,
                    thresholdCount, similarityPct, notificationChannel, createdBy,
                    tenancyId, createdAt, lastFiredAt);
        }
    }
}
```

- [ ] **Step 4: Update WatchdogQuery for enum conditionType**

Use `ide_edit_member` on `WatchdogQuery` (`file: api/src/main/java/io/casehub/qhorus/api/store/query/WatchdogQuery.java`, `member: WatchdogQuery`).

Change `conditionType` field from `String` to `WatchdogConditionType`. Update `byConditionType()`, `matches()`, `Builder.conditionType()`, and `toBuilder()`.

```java
public final class WatchdogQuery {

    private final WatchdogConditionType conditionType;
    private final String tenancyId;

    private WatchdogQuery(Builder b) {
        this.conditionType = b.conditionType;
        this.tenancyId = b.tenancyId;
    }

    public static WatchdogQuery all() {
        return new Builder().build();
    }

    public static WatchdogQuery byConditionType(WatchdogConditionType conditionType) {
        return new Builder().conditionType(conditionType).build();
    }

    public static WatchdogQuery byTenancy(String tenancyId) {
        return new Builder().tenancyId(tenancyId).build();
    }

    public static Builder builder() {
        return new Builder();
    }

    public WatchdogConditionType conditionType() {
        return conditionType;
    }

    public String tenancyId() {
        return tenancyId;
    }

    public boolean matches(Watchdog w) {
        if (conditionType != null && conditionType != w.conditionType()) {
            return false;
        }
        if (tenancyId != null && !tenancyId.equals(w.tenancyId())) {
            return false;
        }
        return true;
    }

    public Builder toBuilder() {
        return new Builder().conditionType(conditionType).tenancyId(tenancyId);
    }

    public static final class Builder {
        private WatchdogConditionType conditionType;
        private String tenancyId;

        public Builder conditionType(WatchdogConditionType v) {
            this.conditionType = v;
            return this;
        }

        public Builder tenancyId(String v) {
            this.tenancyId = v;
            return this;
        }

        public WatchdogQuery build() {
            return new WatchdogQuery(this);
        }
    }
}
```

- [ ] **Step 5: Update WatchdogEntity for similarityPct and String↔enum conversion**

Use `ide_edit_member` on `WatchdogEntity` (`file: runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEntity.java`).

Add field:
```java
@Column(name = "similarity_pct")
public Integer similarityPct;
```

Update `fromDomain()` to store enum as String:
```java
e.conditionType = w.conditionType().name();
e.similarityPct = w.similarityPct();
```

Update `toDomain()` to parse with graceful fallback:
```java
public io.casehub.qhorus.api.watchdog.Watchdog toDomain() {
    WatchdogConditionType type = WatchdogConditionType.fromString(conditionType).orElse(null);
    if (type == null) {
        return null;
    }
    return new io.casehub.qhorus.api.watchdog.Watchdog(
            id, type, targetName, thresholdSeconds, thresholdCount,
            similarityPct, notificationChannel, createdBy, tenancyId,
            createdAt, lastFiredAt);
}
```

- [ ] **Step 6: Update InMemoryWatchdogStore to use toBuilder()**

Use `ide_replace_member` on `InMemoryWatchdogStore.put()` (`file: persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryWatchdogStore.java`, `member: put`):

```java
@Override
public Watchdog put(Watchdog watchdog) {
    UUID id = watchdog.id() != null ? watchdog.id() : UUID.randomUUID();
    Instant createdAt = watchdog.createdAt() != null ? watchdog.createdAt() : Instant.now();
    if (watchdog.id() == null || watchdog.createdAt() == null) {
        watchdog = watchdog.toBuilder().id(id).createdAt(createdAt).build();
    }
    store.put(watchdog.id(), watchdog);
    return watchdog;
}
```

- [ ] **Step 7: Update evaluateAll() — enum switch + toBuilder() for fired block**

In `WatchdogEvaluationService.evaluateAll()`, two changes:

1. Switch from string literals to enum:
```java
boolean fired = switch (w.conditionType()) {
    case BARRIER_STUCK -> evaluateBarrierStuck(w, now);
    case APPROVAL_PENDING -> evaluateApprovalPending(w, now);
    case AGENT_STALE -> evaluateAgentStale(w, now);
    case CHANNEL_IDLE -> evaluateChannelIdle(w, now);
    case QUEUE_DEPTH -> evaluateQueueDepth(w, now);
    case CONTEXT_PRESSURE -> evaluateContextPressure(w, now);
    case LOOP_DETECTED -> false;       // placeholder until Task 4
    case OBLIGATION_FAN_OUT -> false;   // placeholder until Task 5
    case CONVERSATION_STALL -> false;   // placeholder until Task 6
    case ECHO_CHAMBER -> false;         // placeholder until Task 7
};
```

2. Replace positional constructor in fired block with toBuilder():
```java
if (fired) {
    Watchdog updated = w.toBuilder().lastFiredAt(now).build();
    watchdogStore.put(updated);
}
```

3. Filter null entries from `crossTenantWatchdogStore.listAll()` (graceful toDomain() returns null for unknown types):
```java
List<Watchdog> watchdogs = crossTenantWatchdogStore.listAll().stream()
        .filter(Objects::nonNull)
        .toList();
```

- [ ] **Step 8: Run existing tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest`

All existing tests must pass. The enum migration and `similarityPct` addition are backward-compatible (builder defaults to null).

Also run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryWatchdogStoreTest`

- [ ] **Step 9: Fix compilation in all modules and test**

Run `ide_diagnostics` on modified files. Fix any compilation errors from the String→enum change across modules. The ripple includes:
- `JpaWatchdogStore` — `conditionType` filtering may use String comparison
- `ReactiveJpaWatchdogStore` — same
- `ReactiveWatchdogService.register()` — takes String, needs enum parsing
- `WatchdogEnabledTest` — constructs Watchdog with String conditionType
- `WatchdogQueryTest` — uses String conditionType
- `WatchdogServiceContractTest` — uses String conditionType
- `ConnectorAlertBridgeTest` — not affected (constructs AlertContext directly)

Use `ide_find_references` on `Watchdog.builder` to find all call sites.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests` to verify full compilation.

Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,persistence-memory,api`

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): Watchdog data model — similarityPct, conditionType String→enum

- Add Integer similarityPct to Watchdog record, entity, and builder
- Migrate conditionType from String to WatchdogConditionType enum
- Add WatchdogConditionType.fromString() for rollback safety
- WatchdogEntity.toDomain() returns null for unrecognized types
- evaluateAll() uses enum switch + toBuilder() for fired block
- InMemoryWatchdogStore.put() uses toBuilder() instead of positional ctor
- Flyway V36: watchdog.similarity_pct column
- Add 4 new enum values: LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER

Refs #354"
```

---

### Task 2: AlertContext Records

4 new sealed-interface permits in `api/watchdog/`.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/LoopDetectedContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/ObligationFanOutContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/ConversationStallContext.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/EchoChamberContext.java`

**Interfaces:**
- Consumes: `WatchdogConditionType` enum (from Task 1)
- Produces: 4 AlertContext records used by evaluators (Tasks 4-7) and ConnectorAlertBridge (Task 8)

- [ ] **Step 1: Update AlertContext sealed permits**

Use `ide_edit_member` on `AlertContext` (`file: api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java`, `member: AlertContext`):

```java
public sealed interface AlertContext
        permits BarrierStuckContext, ApprovalPendingContext,
                AgentStaleContext, ChannelIdleContext, QueueDepthContext,
                ContextPressureContext,
                LoopDetectedContext, ObligationFanOutContext,
                ConversationStallContext, EchoChamberContext {

    WatchdogConditionType conditionType();
}
```

- [ ] **Step 2: Create LoopDetectedContext**

Use `ide_create_file` (`file: api/src/main/java/io/casehub/qhorus/api/watchdog/LoopDetectedContext.java`):

```java
package io.casehub.qhorus.api.watchdog;

import java.util.UUID;

public record LoopDetectedContext(
        UUID channelId, String channelName,
        String sender, int messageCount, double maxSimilarity
) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.LOOP_DETECTED; }
}
```

- [ ] **Step 3: Create ObligationFanOutContext**

Use `ide_create_file` (`file: api/src/main/java/io/casehub/qhorus/api/watchdog/ObligationFanOutContext.java`):

```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;
import java.util.UUID;

public record ObligationFanOutContext(
        UUID channelId, String channelName,
        int staleCount, List<String> correlationIds
) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.OBLIGATION_FAN_OUT; }
}
```

- [ ] **Step 4: Create ConversationStallContext**

Use `ide_create_file` (`file: api/src/main/java/io/casehub/qhorus/api/watchdog/ConversationStallContext.java`):

```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;
import java.util.UUID;

public record ConversationStallContext(
        UUID channelId, String channelName,
        int stalledCount, List<String> correlationIds, long stalledSeconds
) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.CONVERSATION_STALL; }
}
```

- [ ] **Step 5: Create EchoChamberContext**

Use `ide_create_file` (`file: api/src/main/java/io/casehub/qhorus/api/watchdog/EchoChamberContext.java`):

```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;
import java.util.UUID;

public record EchoChamberContext(
        UUID channelId, String channelName,
        List<String> participants, double maxSimilarity
) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.ECHO_CHAMBER; }
}
```

- [ ] **Step 6: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`

Note: `ConnectorAlertBridge.buildBody()` will now fail compilation because the sealed switch is no longer exhaustive. This is expected — Task 8 fixes it. For now, verify the api module compiles.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): AlertContext records for 4 pathology conditions

- LoopDetectedContext: channelId, channelName, sender, messageCount, maxSimilarity
- ObligationFanOutContext: channelId, channelName, staleCount, correlationIds
- ConversationStallContext: channelId, channelName, stalledCount, correlationIds, stalledSeconds
- EchoChamberContext: channelId, channelName, participants, maxSimilarity

Refs #354"
```

---

### Task 3: JaccardSimilarity Utility + Tests

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/JaccardSimilarity.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/JaccardSimilarityTest.java`

**Interfaces:**
- Produces: `JaccardSimilarity.similarity(String, String) → double` (package-private, used by Tasks 4 and 7)

- [ ] **Step 1: Write the failing tests**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/JaccardSimilarityTest.java`:

```java
package io.casehub.qhorus.runtime.watchdog;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class JaccardSimilarityTest {

    @Test
    void identicalStrings_returnsOne() {
        assertThat(JaccardSimilarity.similarity("hello world", "hello world"))
                .isCloseTo(1.0, within(0.001));
    }

    @Test
    void disjointStrings_returnsZero() {
        assertThat(JaccardSimilarity.similarity("alpha beta", "gamma delta"))
                .isCloseTo(0.0, within(0.001));
    }

    @Test
    void partialOverlap_returnsCorrectRatio() {
        // tokens: {hello, world} ∩ {hello, there} = {hello}, union = {hello, world, there}
        assertThat(JaccardSimilarity.similarity("hello world", "hello there"))
                .isCloseTo(1.0 / 3.0, within(0.001));
    }

    @Test
    void caseInsensitive() {
        assertThat(JaccardSimilarity.similarity("Hello World", "hello world"))
                .isCloseTo(1.0, within(0.001));
    }

    @Test
    void nullInput_returnsZero() {
        assertThat(JaccardSimilarity.similarity(null, "hello")).isCloseTo(0.0, within(0.001));
        assertThat(JaccardSimilarity.similarity("hello", null)).isCloseTo(0.0, within(0.001));
    }

    @Test
    void bothEmpty_returnsOne() {
        assertThat(JaccardSimilarity.similarity("", "")).isCloseTo(1.0, within(0.001));
    }

    @Test
    void oneEmpty_returnsZero() {
        assertThat(JaccardSimilarity.similarity("", "hello")).isCloseTo(0.0, within(0.001));
    }

    @Test
    void punctuationStripping_reducesStructuredContentFalsePositives() {
        // JSON-like content: shared structural tokens ({, }, :, ",) should be stripped
        // leaving only meaningful tokens to compare
        String json1 = "{\"action\": \"deploy\", \"target\": \"prod\"}";
        String json2 = "{\"action\": \"rollback\", \"target\": \"staging\"}";
        assertThat(JaccardSimilarity.similarity(json1, json2))
                .isLessThan(0.7);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JaccardSimilarityTest`

Expected: compilation error — `JaccardSimilarity` does not exist.

- [ ] **Step 3: Implement JaccardSimilarity**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/JaccardSimilarity.java`:

```java
package io.casehub.qhorus.runtime.watchdog;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;
import java.util.stream.Collectors;

final class JaccardSimilarity {

    private static final String PUNCTUATION = "{}[]():\",\'=;";

    static double similarity(String a, String b) {
        if (a == null || b == null) {
            return 0.0;
        }
        Set<String> tokensA = tokenize(a);
        Set<String> tokensB = tokenize(b);
        if (tokensA.isEmpty() && tokensB.isEmpty()) {
            return 1.0;
        }
        if (tokensA.isEmpty() || tokensB.isEmpty()) {
            return 0.0;
        }
        long intersection = tokensA.stream().filter(tokensB::contains).count();
        Set<String> union = new HashSet<>(tokensA);
        union.addAll(tokensB);
        return (double) intersection / union.size();
    }

    private static Set<String> tokenize(String text) {
        return Arrays.stream(text.split("\\s+"))
                .map(t -> stripPunctuation(t).toLowerCase())
                .filter(t -> !t.isEmpty())
                .collect(Collectors.toSet());
    }

    private static String stripPunctuation(String token) {
        StringBuilder sb = new StringBuilder(token.length());
        for (int i = 0; i < token.length(); i++) {
            char c = token.charAt(i);
            if (PUNCTUATION.indexOf(c) < 0) {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    private JaccardSimilarity() {}
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JaccardSimilarityTest`

Expected: all 8 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): JaccardSimilarity — whitespace-tokenised content comparison

Package-private utility for LOOP_DETECTED and ECHO_CHAMBER.
Punctuation stripping reduces false positives on structured content.

Refs #354"
```

---

### Task 4: evaluateLoopDetected + Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java`

**Interfaces:**
- Consumes: `JaccardSimilarity.similarity(String, String)` (from Task 3)
- Consumes: `CrossTenantMessageStore.scan(MessageQuery)` (existing)
- Consumes: `LoopDetectedContext` (from Task 2)

- [ ] **Step 1: Write failing tests**

Add to `WatchdogEvaluationServiceTest`:

```java
@Test
@TestTransaction
void evaluateLoopDetected_firesAlert_whenSenderRepeatsSimilarContent() {
    String chName = "loop-fire-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-loop-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.LOOP_DETECTED, ch.name())
            .thresholdSeconds(600).thresholdCount(3).similarityPct(70)
            .notificationChannel(notifCh.name()).createdBy("test").build());

    for (int i = 0; i < 3; i++) {
        messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
                .messageType(MessageType.STATUS).content("Processing task step " + i + " for the workflow")
                .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    }

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertFalse(alerts.isEmpty(), "LOOP_DETECTED should fire when sender repeats similar content");
    assertTrue(alerts.get(0).content().contains("LOOP_DETECTED"));
}

@Test
@TestTransaction
void evaluateLoopDetected_noAlert_whenContentIsDissimilar() {
    String chName = "loop-nofire-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-loop-nf-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.LOOP_DETECTED, ch.name())
            .thresholdSeconds(600).thresholdCount(3).similarityPct(70)
            .notificationChannel(notifCh.name()).createdBy("test").build());

    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Starting database migration")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Deploying frontend assets to CDN")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Running integration test suite")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when messages are dissimilar");
}

@Test
@TestTransaction
void evaluateLoopDetected_noAlert_whenSimilarMessagesInterleavedWithDissimilar() {
    String chName = "loop-interleave-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-loop-il-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.LOOP_DETECTED, ch.name())
            .thresholdSeconds(600).thresholdCount(3).similarityPct(70)
            .notificationChannel(notifCh.name()).createdBy("test").build());

    // Similar messages interleaved with a different one — consecutive algorithm should NOT fire
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Processing task step for the workflow")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Completely different unrelated message about deployment")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Processing task step for the workflow")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when similar messages are interleaved (consecutive algorithm)");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest#evaluateLoopDetected*`

Expected: FAIL (evaluateLoopDetected returns false placeholder).

- [ ] **Step 3: Implement evaluateLoopDetected**

Add private method to `WatchdogEvaluationService` using `ide_insert_member` (after `evaluateContextPressure`):

```java
private boolean evaluateLoopDetected(Watchdog w, Instant now) {
    int repetitionCount = w.thresholdCount() != null ? w.thresholdCount() : 5;
    int windowSeconds = w.thresholdSeconds() != null ? w.thresholdSeconds() : 300;
    double similarityThreshold = w.similarityPct() != null ? w.similarityPct() / 100.0 : 0.70;
    Instant cutoff = now.minusSeconds(windowSeconds);

    List<Channel> channels = crossTenantChannelStore.listAll().stream()
            .filter(ch -> "*".equals(w.targetName()) || ch.name().equals(w.targetName()))
            .toList();

    boolean fired = false;
    for (Channel ch : channels) {
        List<Message> recent = crossTenantMessageStore.scan(
                MessageQuery.builder().channelId(ch.id())
                        .excludeTypes(List.of(MessageType.EVENT))
                        .limit(repetitionCount * 3).descending(true).build());

        Map<String, List<Message>> bySender = recent.stream()
                .filter(m -> m.createdAt() != null && m.createdAt().isAfter(cutoff))
                .collect(Collectors.groupingBy(Message::sender));

        for (var entry : bySender.entrySet()) {
            List<Message> msgs = entry.getValue();
            if (msgs.size() < repetitionCount) {
                continue;
            }
            msgs.sort(Comparator.comparing(Message::createdAt));

            int longestRun = 0;
            int currentRun = 0;
            double maxSim = 0.0;
            for (int i = 1; i < msgs.size(); i++) {
                double sim = JaccardSimilarity.similarity(msgs.get(i - 1).content(), msgs.get(i).content());
                if (sim >= similarityThreshold) {
                    currentRun++;
                    maxSim = Math.max(maxSim, sim);
                } else {
                    longestRun = Math.max(longestRun, currentRun);
                    currentRun = 0;
                }
            }
            longestRun = Math.max(longestRun, currentRun);

            if (longestRun >= repetitionCount - 1) {
                String summary = "LOOP_DETECTED: sender='" + entry.getKey()
                                 + "' repeated " + (longestRun + 1) + " similar messages on '" + ch.name() + "'";
                fireAlert(w, summary,
                        new LoopDetectedContext(ch.id(), ch.name(), entry.getKey(),
                                longestRun + 1, maxSim), now);
                fired = true;
            }
        }
    }
    return fired;
}
```

Update the switch case in `evaluateAll()` from `case LOOP_DETECTED -> false;` to `case LOOP_DETECTED -> evaluateLoopDetected(w, now);`.

Add required imports: `LoopDetectedContext`, `Comparator`, `Collectors`, `Map`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest`

Expected: all tests PASS (including the 3 new ones).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): LOOP_DETECTED — consecutive-pair content similarity detection

Detects agents repeating similar content. Uses Jaccard similarity on
consecutive message pairs (O(N)), not all-pairs (O(N²)).
Interleaved dissimilar messages break the run.

Refs #354"
```

---

### Task 5: evaluateObligationFanOut + Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java`

**Interfaces:**
- Consumes: `CrossTenantCommitmentStore.findOpenByChannel(UUID)` (existing)
- Consumes: `CrossTenantMessageStore.count(MessageQuery)` (existing)
- Consumes: `ObligationFanOutContext` (from Task 2)

- [ ] **Step 1: Write failing tests**

Add to `WatchdogEvaluationServiceTest`:

```java
@Test
@TestTransaction
void evaluateObligationFanOut_firesAlert_whenCommandHasNoResponse() {
    String chName = "fanout-fire-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-fanout-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.OBLIGATION_FAN_OUT, ch.name())
            .thresholdSeconds(0).notificationChannel(notifCh.name()).createdBy("test").build());

    // OPEN COMMAND commitment, unacknowledged, old
    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(UUID.randomUUID().toString())
            .obligor("agent-b").requester("agent-a")
            .createdAt(Instant.now().minusSeconds(600))
            .build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertFalse(alerts.isEmpty(), "OBLIGATION_FAN_OUT should fire when COMMAND has no response");
    assertTrue(alerts.get(0).content().contains("OBLIGATION_FAN_OUT"));
}

@Test
@TestTransaction
void evaluateObligationFanOut_noAlert_whenCommitmentIsAcknowledged() {
    String chName = "fanout-ack-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-fanout-ack-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.OBLIGATION_FAN_OUT, ch.name())
            .thresholdSeconds(0).notificationChannel(notifCh.name()).createdBy("test").build());

    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(UUID.randomUUID().toString())
            .obligor("agent-b").requester("agent-a")
            .acknowledgedAt(Instant.now().minusSeconds(300))
            .createdAt(Instant.now().minusSeconds(600))
            .build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when commitment is acknowledged");
}

@Test
@TestTransaction
void evaluateObligationFanOut_noAlert_whenStatusMessageExists() {
    String chName = "fanout-status-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-fanout-st-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    String corrId = UUID.randomUUID().toString();
    watchdogStore.put(Watchdog.builder(WatchdogConditionType.OBLIGATION_FAN_OUT, ch.name())
            .thresholdSeconds(0).notificationChannel(notifCh.name()).createdBy("test").build());

    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(corrId)
            .obligor("agent-b").requester("agent-a")
            .createdAt(Instant.now().minusSeconds(600))
            .build());

    // STATUS message exists on the correlationId — agent is engaged
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.STATUS).content("Working on it")
            .correlationId(corrId).actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when STATUS message exists on correlationId");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest#evaluateObligationFanOut*`

- [ ] **Step 3: Implement evaluateObligationFanOut**

Add private method to `WatchdogEvaluationService` using `ide_insert_member`:

```java
private boolean evaluateObligationFanOut(Watchdog w, Instant now) {
    int deadlineSeconds = w.thresholdSeconds() != null ? w.thresholdSeconds() : 300;
    Instant cutoff = now.minusSeconds(deadlineSeconds);

    List<Channel> channels = crossTenantChannelStore.listAll().stream()
            .filter(ch -> "*".equals(w.targetName()) || ch.name().equals(w.targetName()))
            .toList();

    boolean fired = false;
    for (Channel ch : channels) {
        List<Commitment> stale = crossTenantCommitmentStore.findOpenByChannel(ch.id()).stream()
                .filter(c -> c.messageType() == MessageType.COMMAND)
                .filter(c -> c.acknowledgedAt() == null)
                .filter(c -> c.createdAt() != null && c.createdAt().isBefore(cutoff))
                .filter(c -> {
                    long responseCount = crossTenantMessageStore.count(
                            MessageQuery.builder().channelId(ch.id())
                                    .correlationId(c.correlationId())
                                    .excludeTypes(List.of(MessageType.COMMAND, MessageType.EVENT))
                                    .build());
                    return responseCount == 0;
                })
                .toList();

        if (!stale.isEmpty()) {
            List<String> corrIds = stale.stream()
                    .map(Commitment::correlationId).limit(5).toList();
            String summary = "OBLIGATION_FAN_OUT: " + stale.size()
                             + " unresponded obligation(s) on '" + ch.name() + "'";
            fireAlert(w, summary,
                    new ObligationFanOutContext(ch.id(), ch.name(), stale.size(), corrIds), now);
            fired = true;
        }
    }
    return fired;
}
```

Update the switch case from `case OBLIGATION_FAN_OUT -> false;` to `case OBLIGATION_FAN_OUT -> evaluateObligationFanOut(w, now);`.

Add import: `ObligationFanOutContext`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): OBLIGATION_FAN_OUT — detect COMMAND obligations with zero engagement

Fires when OPEN COMMAND commitments have no ACKNOWLEDGE and no
STATUS/response messages within the deadline threshold.

Refs #354"
```

---

### Task 6: evaluateConversationStall + Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java`

**Interfaces:**
- Consumes: `CrossTenantCommitmentStore.findOpenByChannel(UUID)` (existing)
- Consumes: `CrossTenantMessageStore.scan(MessageQuery)` (existing)
- Consumes: `ConversationStallContext` (from Task 2)

- [ ] **Step 1: Write failing tests**

Add to `WatchdogEvaluationServiceTest`:

```java
@Test
@TestTransaction
void evaluateConversationStall_firesAlert_whenNoTerminalResolution() {
    String chName = "stall-fire-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-stall-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CONVERSATION_STALL, ch.name())
            .thresholdSeconds(0).notificationChannel(notifCh.name()).createdBy("test").build());

    // OPEN commitment, old enough to be stalled
    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(UUID.randomUUID().toString())
            .obligor("agent-b").requester("agent-a")
            .createdAt(Instant.now().minusSeconds(600))
            .build());

    // Only STATUS messages — no terminal resolution
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.STATUS).content("Still working")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertFalse(alerts.isEmpty(), "CONVERSATION_STALL should fire when no terminal resolution");
    assertTrue(alerts.get(0).content().contains("CONVERSATION_STALL"));
}

@Test
@TestTransaction
void evaluateConversationStall_noAlert_whenRecentTerminalExists() {
    String chName = "stall-term-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-stall-t-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    String corrId = UUID.randomUUID().toString();
    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CONVERSATION_STALL, ch.name())
            .thresholdSeconds(600).notificationChannel(notifCh.name()).createdBy("test").build());

    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(corrId)
            .obligor("agent-b").requester("agent-a")
            .createdAt(Instant.now().minusSeconds(1200))
            .build());

    // Recent DONE message on a different correlation — channel is resolving things
    String otherCorrId = UUID.randomUUID().toString();
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.DONE).content("Completed task")
            .correlationId(otherCorrId).actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    // But this commitment has a recent DONE on its own correlationId
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.DONE).content("Completed")
            .correlationId(corrId).actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when correlation has recent terminal resolution");
}

@Test
@TestTransaction
void evaluateConversationStall_noAlert_whenNoActiveCommitments() {
    String chName = "stall-empty-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-stall-e-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CONVERSATION_STALL, ch.name())
            .thresholdSeconds(0).notificationChannel(notifCh.name()).createdBy("test").build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when channel has no active commitments");
}

@Test
@TestTransaction
void evaluateConversationStall_noAlert_whenCommitmentsAreYoung() {
    String chName = "stall-young-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-stall-y-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CONVERSATION_STALL, ch.name())
            .thresholdSeconds(600).notificationChannel(notifCh.name()).createdBy("test").build());

    // Commitment created 10 seconds ago — younger than 600s threshold
    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(UUID.randomUUID().toString())
            .obligor("agent-b").requester("agent-a")
            .createdAt(Instant.now().minusSeconds(10))
            .build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when all commitments are younger than threshold");
}

@Test
@TestTransaction
void evaluateConversationStall_firesAlert_whenOneStalledAndOneResolved() {
    String chName = "stall-partial-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-stall-p-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CONVERSATION_STALL, ch.name())
            .thresholdSeconds(0).notificationChannel(notifCh.name()).createdBy("test").build());

    // Commitment A: stalled (no resolution message)
    String corrA = UUID.randomUUID().toString();
    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(corrA)
            .obligor("agent-b").requester("agent-a")
            .createdAt(Instant.now().minusSeconds(600))
            .build());

    // Commitment B: recently resolved (DONE message on its correlationId)
    String corrB = UUID.randomUUID().toString();
    commitmentStore.save(Commitment.builder()
            .state(CommitmentState.OPEN).messageType(MessageType.COMMAND)
            .channelId(ch.id()).correlationId(corrB)
            .obligor("agent-c").requester("agent-a")
            .createdAt(Instant.now().minusSeconds(600))
            .build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-c")
            .messageType(MessageType.DONE).content("Done")
            .correlationId(corrB).actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertFalse(alerts.isEmpty(), "Should fire when one correlation is stalled even if another resolved");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest#evaluateConversationStall*`

- [ ] **Step 3: Implement evaluateConversationStall**

Add private method using `ide_insert_member`:

```java
private static final List<MessageType> RESOLUTION_TYPES = List.of(
        MessageType.DONE, MessageType.FAILURE, MessageType.DECLINE, MessageType.HANDOFF);

private boolean evaluateConversationStall(Watchdog w, Instant now) {
    int stallSeconds = w.thresholdSeconds() != null ? w.thresholdSeconds() : 600;
    Instant cutoff = now.minusSeconds(stallSeconds);

    List<Channel> channels = crossTenantChannelStore.listAll().stream()
            .filter(ch -> "*".equals(w.targetName()) || ch.name().equals(w.targetName()))
            .toList();

    boolean fired = false;
    for (Channel ch : channels) {
        List<Commitment> active = crossTenantCommitmentStore.findOpenByChannel(ch.id());
        if (active.isEmpty()) {
            continue;
        }

        List<Commitment> aged = active.stream()
                .filter(c -> c.createdAt() != null && c.createdAt().isBefore(cutoff))
                .toList();
        if (aged.isEmpty()) {
            continue;
        }

        List<Commitment> stalled = aged.stream()
                .filter(c -> {
                    List<Message> resolutions = crossTenantMessageStore.scan(
                            MessageQuery.builder().channelId(ch.id())
                                    .correlationId(c.correlationId())
                                    .limit(100).descending(true).build()).stream()
                            .filter(m -> RESOLUTION_TYPES.contains(m.messageType()))
                            .toList();
                    if (resolutions.isEmpty()) {
                        return true;
                    }
                    Message latest = resolutions.get(0);
                    return latest.createdAt() != null && latest.createdAt().isBefore(cutoff);
                })
                .toList();

        if (!stalled.isEmpty()) {
            long maxStallSeconds = stalled.stream()
                    .mapToLong(c -> now.getEpochSecond() - c.createdAt().getEpochSecond())
                    .max().orElse(0L);
            List<String> corrIds = stalled.stream()
                    .map(Commitment::correlationId).limit(5).toList();
            String summary = "CONVERSATION_STALL: " + stalled.size()
                             + " stalled correlation(s) on '" + ch.name() + "'";
            fireAlert(w, summary,
                    new ConversationStallContext(ch.id(), ch.name(), stalled.size(),
                            corrIds, maxStallSeconds), now);
            fired = true;
        }
    }
    return fired;
}
```

Update the switch case from `case CONVERSATION_STALL -> false;` to `case CONVERSATION_STALL -> evaluateConversationStall(w, now);`.

Add import: `ConversationStallContext`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): CONVERSATION_STALL — per-correlation terminal resolution detection

Detects channels with active work where individual correlations are not
completing. Per-correlation checking prevents recently-resolved siblings
from masking stalled ones. Age guard prevents false positives on new
channels.

Refs #354"
```

---

### Task 7: evaluateEchoChamber + Tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java`

**Interfaces:**
- Consumes: `JaccardSimilarity.similarity(String, String)` (from Task 3)
- Consumes: `CrossTenantMessageStore.scan(MessageQuery)` (existing)
- Consumes: `EchoChamberContext` (from Task 2)

- [ ] **Step 1: Write failing tests**

Add to `WatchdogEvaluationServiceTest`:

```java
@Test
@TestTransaction
void evaluateEchoChamber_firesAlert_whenMultipleSendersRelayIdenticalContent() {
    String chName = "echo-fire-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-echo-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.ECHO_CHAMBER, ch.name())
            .thresholdSeconds(600).thresholdCount(2).similarityPct(70)
            .notificationChannel(notifCh.name()).createdBy("test").build());

    // Agent A and B echoing similar content back and forth (2+ cross-sender pairs)
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("The deployment pipeline is running for the production environment")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.STATUS).content("The deployment pipeline is running for the production environment now")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Confirmed the deployment pipeline is running for the production environment")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.STATUS).content("Yes the deployment pipeline is running for the production environment")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertFalse(alerts.isEmpty(), "ECHO_CHAMBER should fire when agents echo similar content");
    assertTrue(alerts.get(0).content().contains("ECHO_CHAMBER"));
}

@Test
@TestTransaction
void evaluateEchoChamber_noAlert_whenContentIsTransformed() {
    String chName = "echo-transform-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-echo-t-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.ECHO_CHAMBER, ch.name())
            .thresholdSeconds(600).thresholdCount(2).similarityPct(70)
            .notificationChannel(notifCh.name()).createdBy("test").build());

    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Analyze the quarterly revenue data")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.STATUS).content("Revenue grew 15% year over year driven by subscription renewals")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire when content is meaningfully transformed");
}

@Test
@TestTransaction
void evaluateEchoChamber_noAlert_whenOnlySingleSender() {
    String chName = "echo-single-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-echo-s-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.ECHO_CHAMBER, ch.name())
            .thresholdSeconds(600).thresholdCount(2).similarityPct(70)
            .notificationChannel(notifCh.name()).createdBy("test").build());

    // Only one sender — echo requires 2+ participants
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Same content repeated")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Same content repeated again")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire with only one sender (below min_agents)");
}

@Test
@TestTransaction
void evaluateEchoChamber_noAlert_whenOnlyOneSimilarPair() {
    String chName = "echo-onepair-" + UUID.randomUUID();
    Channel ch = channelStore.put(Channel.builder(chName).semantic(ChannelSemantic.APPEND).build());
    Channel notifCh = channelStore.put(Channel.builder("notif-echo-op-" + UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build());

    watchdogStore.put(Watchdog.builder(WatchdogConditionType.ECHO_CHAMBER, ch.name())
            .thresholdSeconds(600).thresholdCount(2).similarityPct(70)
            .notificationChannel(notifCh.name()).createdBy("test").build());

    // One forwarded message is legitimate — need 2+ pairs for echo chamber
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-a")
            .messageType(MessageType.STATUS).content("Deploy the application to production cluster immediately")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());
    messageStore.put(Message.builder().channelId(ch.id()).sender("agent-b")
            .messageType(MessageType.STATUS).content("Deploy the application to production cluster immediately")
            .actorType(ActorType.AGENT).createdAt(Instant.now()).build());

    watchdogService.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.forChannel(notifCh.id()));
    assertTrue(alerts.isEmpty(), "Should not fire with only one similar cross-sender pair");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest#evaluateEchoChamber*`

- [ ] **Step 3: Implement evaluateEchoChamber**

Add private method using `ide_insert_member`:

```java
private boolean evaluateEchoChamber(Watchdog w, Instant now) {
    int windowSeconds = w.thresholdSeconds() != null ? w.thresholdSeconds() : 300;
    int minAgents = w.thresholdCount() != null ? w.thresholdCount() : 2;
    double similarityThreshold = w.similarityPct() != null ? w.similarityPct() / 100.0 : 0.70;
    Instant cutoff = now.minusSeconds(windowSeconds);

    List<Channel> channels = crossTenantChannelStore.listAll().stream()
            .filter(ch -> "*".equals(w.targetName()) || ch.name().equals(w.targetName()))
            .toList();

    boolean fired = false;
    for (Channel ch : channels) {
        List<Message> recent = crossTenantMessageStore.scan(
                MessageQuery.builder().channelId(ch.id())
                        .excludeTypes(List.of(MessageType.EVENT))
                        .limit(50).descending(true).build()).stream()
                .filter(m -> m.createdAt() != null && m.createdAt().isAfter(cutoff))
                .toList();

        Map<String, List<Message>> bySender = recent.stream()
                .collect(Collectors.groupingBy(Message::sender));

        if (bySender.size() < minAgents) {
            continue;
        }

        int similarPairs = 0;
        double maxSim = 0.0;
        Set<String> participants = new HashSet<>();
        List<String> senders = new ArrayList<>(bySender.keySet());
        for (int i = 0; i < senders.size(); i++) {
            for (int j = i + 1; j < senders.size(); j++) {
                for (Message ma : bySender.get(senders.get(i))) {
                    for (Message mb : bySender.get(senders.get(j))) {
                        double sim = JaccardSimilarity.similarity(ma.content(), mb.content());
                        if (sim >= similarityThreshold) {
                            similarPairs++;
                            maxSim = Math.max(maxSim, sim);
                            participants.add(senders.get(i));
                            participants.add(senders.get(j));
                        }
                    }
                }
            }
        }

        if (similarPairs >= 2) {
            String summary = "ECHO_CHAMBER: " + similarPairs
                             + " echoed message pair(s) on '" + ch.name() + "'";
            fireAlert(w, summary,
                    new EchoChamberContext(ch.id(), ch.name(),
                            List.copyOf(participants), maxSim), now);
            fired = true;
        }
    }
    return fired;
}
```

Update the switch case from `case ECHO_CHAMBER -> false;` to `case ECHO_CHAMBER -> evaluateEchoChamber(w, now);`.

Add imports: `EchoChamberContext`, `HashSet`, `Set`, `ArrayList`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WatchdogEvaluationServiceTest`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): ECHO_CHAMBER — detect content relayed without transformation

Detects sustained inter-agent echo (≥2 similar cross-sender pairs),
not isolated forwarding. Jaccard similarity on whitespace-tokenised
content with punctuation stripping.

Refs #354"
```

---

### Task 8: MCP Tools + ConnectorAlertBridge

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ReactiveWatchdogService.java`
- Modify: `connectors/src/main/java/io/casehub/qhorus/connectors/ConnectorAlertBridge.java`
- Modify: `connectors/src/test/java/io/casehub/qhorus/connectors/ConnectorAlertBridgeTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/WatchdogEnabledTest.java`

**Interfaces:**
- Consumes: `WatchdogConditionType` enum, `Watchdog.similarityPct()`, all 4 new AlertContext records

- [ ] **Step 1: Update WatchdogSummary record**

Use `ide_edit_member` on `WatchdogSummary` in `QhorusMcpToolsBase` to add `similarityPct`:

```java
public record WatchdogSummary(
        String id,
        String conditionType,
        String targetName,
        Integer thresholdSeconds,
        Integer thresholdCount,
        Integer similarityPct,
        String notificationChannel,
        String createdBy,
        String createdAt,
        String lastFiredAt) {
}
```

- [ ] **Step 2: Update toWatchdogSummary()**

Use `ide_replace_member` on `toWatchdogSummary` in `QhorusMcpToolsBase`:

```java
return new WatchdogSummary(
        w.id().toString(),
        w.conditionType().name(),
        w.targetName(),
        w.thresholdSeconds(),
        w.thresholdCount(),
        w.similarityPct(),
        w.notificationChannel(),
        w.createdBy(),
        w.createdAt() != null ? w.createdAt().toString() : null,
        w.lastFiredAt() != null ? w.lastFiredAt().toString() : null);
```

- [ ] **Step 3: Update QhorusMcpTools.registerWatchdog()**

Use `ide_edit_member` on `registerWatchdog` in `QhorusMcpTools` to:
1. Update `condition_type` description to include all 10 types
2. Add `similarity_pct` parameter
3. Parse `conditionType` String to enum

```java
@Tool(name = "register_watchdog", description = "Register a watchdog ...")
@Transactional
public WatchdogSummary registerWatchdog(
        @ToolArg(name = "condition_type", description = "BARRIER_STUCK | APPROVAL_PENDING | AGENT_STALE | CHANNEL_IDLE | QUEUE_DEPTH | CONTEXT_PRESSURE | LOOP_DETECTED | OBLIGATION_FAN_OUT | CONVERSATION_STALL | ECHO_CHAMBER") String conditionType,
        @ToolArg(name = "target_name", description = "Channel name, instance_id, or '*' for all") String targetName,
        @ToolArg(name = "threshold_seconds", description = "Time threshold in seconds (for time-based conditions)", required = false) Integer thresholdSeconds,
        @ToolArg(name = "threshold_count", description = "Count threshold (for QUEUE_DEPTH, LOOP_DETECTED repetitions, ECHO_CHAMBER min agents)", required = false) Integer thresholdCount,
        @ToolArg(name = "similarity_pct", description = "Content similarity percentage threshold 0-100 (for LOOP_DETECTED, ECHO_CHAMBER)", required = false) Integer similarityPct,
        @ToolArg(name = "notification_channel", description = "Channel to post alert events to") String notificationChannel,
        @ToolArg(name = "created_by", description = "Who is registering this watchdog") String createdBy) {
    requireWatchdogEnabled();
    WatchdogConditionType type = WatchdogConditionType.fromString(conditionType)
            .orElseThrow(() -> new IllegalArgumentException(
                    "Unknown condition_type '" + conditionType + "'. Valid: " + Arrays.toString(WatchdogConditionType.values())));
    Watchdog w = watchdogStore.put(Watchdog.builder(type, targetName)
            .thresholdSeconds(thresholdSeconds).thresholdCount(thresholdCount)
            .similarityPct(similarityPct)
            .notificationChannel(notificationChannel).createdBy(createdBy)
            .tenancyId(currentPrincipal.tenancyId()).build());
    return toWatchdogSummary(w);
}
```

- [ ] **Step 4: Update ReactiveWatchdogService.register()**

Use `ide_replace_member` on `register` in `ReactiveWatchdogService` to add `similarityPct` parameter:

```java
public Uni<Watchdog> register(String conditionType, String targetName, Integer thresholdSeconds,
                                    Integer thresholdCount, Integer similarityPct,
                                    String notificationChannel, String createdBy, String tenancyId) {
    return Panache.withTransaction("qhorus", () -> {
        WatchdogConditionType type = WatchdogConditionType.fromString(conditionType)
                .orElseThrow(() -> new IllegalArgumentException("Unknown condition_type: " + conditionType));
        Watchdog w = Watchdog.builder(type, targetName)
                .thresholdSeconds(thresholdSeconds)
                .thresholdCount(thresholdCount)
                .similarityPct(similarityPct)
                .notificationChannel(notificationChannel)
                .createdBy(createdBy)
                .tenancyId(tenancyId)
                .build();
        return watchdogStore.put(w);
    });
}
```

- [ ] **Step 5: Update ReactiveQhorusMcpTools.registerWatchdog()**

Use `ide_edit_member` on `registerWatchdog` in `ReactiveQhorusMcpTools` — add `similarity_pct` parameter and pass to `watchdogService.register()`:

```java
@Tool(name = "register_watchdog", description = "Register a watchdog ...")
public Uni<WatchdogSummary> registerWatchdog(
        @ToolArg(name = "condition_type", description = "BARRIER_STUCK | APPROVAL_PENDING | AGENT_STALE | CHANNEL_IDLE | QUEUE_DEPTH | CONTEXT_PRESSURE | LOOP_DETECTED | OBLIGATION_FAN_OUT | CONVERSATION_STALL | ECHO_CHAMBER") String conditionType,
        @ToolArg(name = "target_name", description = "Channel name, instance_id, or '*' for all") String targetName,
        @ToolArg(name = "threshold_seconds", description = "Time threshold in seconds", required = false) Integer thresholdSeconds,
        @ToolArg(name = "threshold_count", description = "Count threshold", required = false) Integer thresholdCount,
        @ToolArg(name = "similarity_pct", description = "Content similarity percentage threshold 0-100 (for LOOP_DETECTED, ECHO_CHAMBER)", required = false) Integer similarityPct,
        @ToolArg(name = "notification_channel", description = "Channel to post alert events to") String notificationChannel,
        @ToolArg(name = "created_by", description = "Who is registering this watchdog") String createdBy) {
    requireWatchdogEnabled();
    String tenancyId = currentPrincipal.tenancyId();
    return watchdogService.register(conditionType, targetName, thresholdSeconds, thresholdCount,
            similarityPct, notificationChannel, createdBy, tenancyId)
            .map(this::toWatchdogSummary);
}
```

- [ ] **Step 6: Update ConnectorAlertBridge.buildBody() — add 4 cases**

Use `ide_replace_member` on `buildBody` in `ConnectorAlertBridge` to add the 4 new sealed cases:

```java
case LoopDetectedContext c -> event.summary()
    + "\nChannel: " + c.channelName()
    + "\nSender: " + c.sender()
    + "\nRepeated messages: " + c.messageCount()
    + "\nMax similarity: " + Math.round(c.maxSimilarity() * 100) + "%";

case ObligationFanOutContext c -> event.summary()
    + "\nChannel: " + c.channelName()
    + "\nStale obligations: " + c.staleCount()
    + "\nCorrelation IDs: " + String.join(", ", c.correlationIds());

case ConversationStallContext c -> event.summary()
    + "\nChannel: " + c.channelName()
    + "\nStalled correlations: " + c.stalledCount()
    + "\nCorrelation IDs: " + String.join(", ", c.correlationIds())
    + "\nLongest stall: " + c.stalledSeconds() + "s";

case EchoChamberContext c -> event.summary()
    + "\nChannel: " + c.channelName()
    + "\nParticipants: " + String.join(", ", c.participants())
    + "\nMax similarity: " + Math.round(c.maxSimilarity() * 100) + "%";
```

Add imports for the 4 new context types.

- [ ] **Step 7: Fix WatchdogEnabledTest compilation**

Update all `Watchdog.builder(String, ...)` calls to `Watchdog.builder(WatchdogConditionType.X, ...)`. Use `ide_find_references` to find all call sites in test files.

- [ ] **Step 8: Build and test all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

All modules must compile and all tests must pass.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#354): MCP tools + ConnectorAlertBridge for pathology conditions

- register_watchdog gains similarity_pct param and enum validation
- WatchdogSummary includes similarityPct
- ConnectorAlertBridge exhaustive switch covers all 10 conditions
- ReactiveWatchdogService.register() gains similarityPct param

Refs #354"
```

---

### Task 9: Full Build Verification + CLAUDE.md Update

- [ ] **Step 1: Run the full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

All 1759+ tests must pass. If any fail, fix before proceeding.

- [ ] **Step 2: Update CLAUDE.md**

Add documentation for the new conditions and the conditionType enum migration:
- `WatchdogConditionType` is now the source of truth (not String) for condition type dispatch
- `WatchdogConditionType.fromString(String) → Optional` for DB/MCP boundary parsing
- `WatchdogEntity.toDomain()` returns null for unrecognized condition types (rollback safety)
- V36 migration: `watchdog.similarity_pct`
- New conditions: LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER with parameter mapping

- [ ] **Step 3: Commit CLAUDE.md**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "docs: CLAUDE.md — coordination pathology watchdog conventions

Refs #354"
```

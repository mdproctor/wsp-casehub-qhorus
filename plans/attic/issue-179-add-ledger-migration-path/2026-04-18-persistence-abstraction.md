# Persistence Abstraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a `Store` + `scan(Query)` persistence abstraction layer across all five Qhorus domains so the JPA/Panache implementation can be swapped without touching services or MCP tools.

**Architecture:** Per-domain store interfaces (`ChannelStore`, `MessageStore`, `InstanceStore`, `DataStore`, `WatchdogStore`) in `runtime/store/`; JPA implementations in `runtime/store/jpa/`; in-memory implementations in a new `testing/` module activated via CDI `@Alternative @Priority(1)`. Services inject stores instead of calling Panache entity statics; business logic stays in services.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM + Panache, CDI, JUnit 5, AssertJ, RestAssured, H2 (test), quarkus-mcp-server 1.11.1.

**Test strategy:**
- **Unit tests** — Query `matches()` predicates: pure Java, no framework, run anywhere.
- **Integration tests** — JPA store tests: `@QuarkusTest` + H2, inject `@Named` store directly.
- **End-to-end / happy path** — existing 561 `@QuarkusTest` MCP tool tests; must stay green after every service migration.
- **In-memory contract tests** — same scenarios as integration tests, run against `InMemory*Store` directly.

**Commits:** all commits reference an issue with `Refs #N` or `Closes #N`. Create the epic and child issues in Task 1 before writing any code.

---

## File Map

**New — runtime module:**
```
runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/
    ChannelStore.java
    MessageStore.java
    InstanceStore.java
    DataStore.java
    WatchdogStore.java
    query/
        ChannelQuery.java
        MessageQuery.java
        InstanceQuery.java
        DataQuery.java
        WatchdogQuery.java
    jpa/
        JpaChannelStore.java
        JpaMessageStore.java
        JpaInstanceStore.java
        JpaDataStore.java
        JpaWatchdogStore.java

runtime/src/test/java/io/quarkiverse/qhorus/store/
    JpaChannelStoreTest.java
    JpaMessageStoreTest.java
    JpaInstanceStoreTest.java
    JpaDataStoreTest.java
    JpaWatchdogStoreTest.java
    query/
        ChannelQueryTest.java
        MessageQueryTest.java
        InstanceQueryTest.java
        DataQueryTest.java
        WatchdogQueryTest.java
```

**New — testing module:**
```
testing/pom.xml
testing/src/main/java/io/quarkiverse/qhorus/testing/
    InMemoryChannelStore.java
    InMemoryMessageStore.java
    InMemoryInstanceStore.java
    InMemoryDataStore.java
    InMemoryWatchdogStore.java
testing/src/test/java/io/quarkiverse/qhorus/testing/
    InMemoryChannelStoreTest.java
    InMemoryMessageStoreTest.java
    InMemoryInstanceStoreTest.java
    InMemoryDataStoreTest.java
    InMemoryWatchdogStoreTest.java
```

**New — examples module:**
```
examples/pom.xml
examples/src/main/java/io/quarkiverse/qhorus/examples/
    GettingStartedExample.java
examples/src/test/java/io/quarkiverse/qhorus/examples/
    GettingStartedExampleTest.java
```

**Modified:**
```
pom.xml                                            — add testing/, examples/ modules
runtime/src/main/java/.../channel/ChannelService.java
runtime/src/main/java/.../message/MessageService.java
runtime/src/main/java/.../instance/InstanceService.java
runtime/src/main/java/.../data/DataService.java
runtime/src/main/java/.../watchdog/WatchdogEvaluationService.java
docs/DESIGN.md
adr/0002-persistence-abstraction-store-pattern.md  — new
adr/INDEX.md
docs/superpowers/specs/2026-04-18-persistence-abstraction-design.md — already written
```

---

## Task 1: GitHub Epic and Child Issues

**Files:** none (GitHub only)

- [ ] **Step 1: Create the epic issue**

```bash
gh issue create --repo mdproctor/quarkus-qhorus \
  --label "enhancement,epic" \
  --title "Persistence abstraction — Store + scan(Query) pattern" \
  --body "$(cat <<'EOF'
Introduce a proper DB abstraction layer so the JPA/Panache implementation
can be swapped without touching services or MCP tools. Aligns with the
quarkus-workitems Store + scan(Query) pattern.

Design spec: docs/superpowers/specs/2026-04-18-persistence-abstraction-design.md
Cross-project rationale: docs/specs/2026-04-17-persistence-abstraction-strategy.md

Child issues:
- Module scaffolding (testing/ + examples/)
- Store interfaces + Query objects
- Channel domain (JpaChannelStore + ChannelService migration)
- Message domain (JpaMessageStore + MessageService migration)
- Instance domain (JpaInstanceStore + InstanceService migration)
- Data domain (JpaDataStore + DataService migration)
- Watchdog domain (JpaWatchdogStore + WatchdogEvaluationService migration)
- In-memory stores (testing/ module)
- ADR-0002 + DESIGN.md + examples
EOF
)"
```

Note the epic issue number — call it `#EPIC` in subsequent steps.

- [ ] **Step 2: Create child issues**

```bash
gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — Maven module scaffolding (testing/, examples/)" \
  --body "Add testing/ and examples/ Maven modules to the parent pom. Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — Store interfaces + Query value objects (all 5 domains)" \
  --body "Define ChannelStore, MessageStore, InstanceStore, DataStore, WatchdogStore interfaces and their Query value objects with matches() predicate. Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — Channel domain (JpaChannelStore + ChannelService migration)" \
  --body "Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — Message domain (JpaMessageStore + MessageService migration)" \
  --body "Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — Instance domain (JpaInstanceStore + InstanceService migration)" \
  --body "Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — Data domain (JpaDataStore + DataService migration)" \
  --body "Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — Watchdog domain (JpaWatchdogStore + WatchdogEvaluationService migration)" \
  --body "Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — In-memory stores (testing/ module)" \
  --body "Refs #EPIC"

gh issue create --repo mdproctor/quarkus-qhorus --label enhancement \
  --title "Persistence abstraction — ADR-0002 + DESIGN.md + examples + documentation" \
  --body "Refs #EPIC"
```

Note all issue numbers. You will use them throughout this plan.

---

## Task 2: Maven Module Scaffolding

**Files:**
- Modify: `pom.xml`
- Create: `testing/pom.xml`
- Create: `examples/pom.xml`

- [ ] **Step 1: Add modules to parent pom**

Edit `/Users/mdproctor/claude/quarkus-qhorus/pom.xml` — replace `<modules>` block:

```xml
<modules>
  <module>runtime</module>
  <module>deployment</module>
  <module>testing</module>
  <module>examples</module>
</modules>
```

- [ ] **Step 2: Create testing/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.quarkiverse.qhorus</groupId>
    <artifactId>quarkus-qhorus-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>quarkus-qhorus-testing</artifactId>
  <name>Quarkus Qhorus - Testing</name>
  <description>In-memory store implementations for fast unit testing without a database</description>

  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.qhorus</groupId>
      <artifactId>quarkus-qhorus</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>
    <!-- Test scope only -->
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 3: Create examples/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.quarkiverse.qhorus</groupId>
    <artifactId>quarkus-qhorus-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>quarkus-qhorus-examples</artifactId>
  <name>Quarkus Qhorus - Examples</name>
  <description>Usage examples showing Store injection, in-memory testing, and MCP tool patterns</description>

  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.qhorus</groupId>
      <artifactId>quarkus-qhorus</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkiverse.qhorus</groupId>
      <artifactId>quarkus-qhorus-testing</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 4: Verify build compiles with new modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -q
```

Expected: BUILD SUCCESS (empty modules are fine).

- [ ] **Step 5: Commit**

```bash
git add pom.xml testing/pom.xml examples/pom.xml
git commit -m "feat: add testing/ and examples/ Maven modules

Closes #SCAFFOLDING_ISSUE"
```

---

## Task 3: Store Interfaces

**Files:** Create all 5 store interfaces in `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/`

No tests for pure interfaces — they are tested via their implementations.

- [ ] **Step 1: Create ChannelStore.java**

```java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;

public interface ChannelStore {
    Channel put(Channel channel);
    Optional<Channel> find(UUID id);
    Optional<Channel> findByName(String name);
    List<Channel> scan(ChannelQuery query);
    void delete(UUID id);
}
```

- [ ] **Step 2: Create MessageStore.java**

```java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;

public interface MessageStore {
    Message put(Message message);
    Optional<Message> find(Long id);
    List<Message> scan(MessageQuery query);
    void deleteAll(UUID channelId);
    void delete(Long id);
    int countByChannel(UUID channelId);
}
```

- [ ] **Step 3: Create InstanceStore.java**

```java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;

public interface InstanceStore {
    Instance put(Instance instance);
    Optional<Instance> find(UUID id);
    Optional<Instance> findByInstanceId(String instanceId);
    List<Instance> scan(InstanceQuery query);
    void putCapabilities(UUID instanceId, List<String> tags);
    void deleteCapabilities(UUID instanceId);
    List<String> findCapabilities(UUID instanceId);
    void delete(UUID id);
}
```

- [ ] **Step 4: Create DataStore.java**

```java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.query.DataQuery;

public interface DataStore {
    SharedData put(SharedData data);
    Optional<SharedData> find(UUID id);
    Optional<SharedData> findByKey(String key);
    List<SharedData> scan(DataQuery query);
    ArtefactClaim putClaim(ArtefactClaim claim);
    void deleteClaim(UUID artefactId, UUID instanceId);
    int countClaims(UUID artefactId);
    void delete(UUID id);
}
```

- [ ] **Step 5: Create WatchdogStore.java**

```java
package io.quarkiverse.qhorus.runtime.store;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;

public interface WatchdogStore {
    Watchdog put(Watchdog watchdog);
    Optional<Watchdog> find(UUID id);
    List<Watchdog> scan(WatchdogQuery query);
    void delete(UUID id);
}
```

- [ ] **Step 6: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/
git commit -m "feat(store): add Store interfaces for all 5 domains

ChannelStore, MessageStore, InstanceStore, DataStore, WatchdogStore.
KV semantics: put/find/scan + domain-specific methods.

Refs #INTERFACES_ISSUE"
```

---

## Task 4: Query Value Objects + Unit Tests

**Files:**
- Create: `runtime/src/main/java/.../store/query/ChannelQuery.java` (and 4 others)
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/query/ChannelQueryTest.java` (and 4 others)

Each query has: immutable fields, static factories, builder, `boolean matches(T entity)` predicate for in-memory implementations.

- [ ] **Step 1: Write failing ChannelQuery unit test**

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/query/ChannelQueryTest.java`:

```java
package io.quarkiverse.qhorus.store.query;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;

class ChannelQueryTest {

    @Test
    void all_matchesAnyChannel() {
        Channel ch = new Channel();
        ch.paused = false;
        assertThat(ChannelQuery.all().matches(ch)).isTrue();
    }

    @Test
    void paused_matchesPausedChannel() {
        Channel ch = new Channel();
        ch.paused = true;
        assertThat(ChannelQuery.paused().matches(ch)).isTrue();
    }

    @Test
    void paused_doesNotMatchActiveChannel() {
        Channel ch = new Channel();
        ch.paused = false;
        assertThat(ChannelQuery.paused().matches(ch)).isFalse();
    }

    @Test
    void bySemantic_matchesCorrectSemantic() {
        Channel ch = new Channel();
        ch.semantic = ChannelSemantic.APPEND;
        assertThat(ChannelQuery.bySemantic(ChannelSemantic.APPEND).matches(ch)).isTrue();
        assertThat(ChannelQuery.bySemantic(ChannelSemantic.COLLECT).matches(ch)).isFalse();
    }

    @Test
    void builder_combinesPredicates() {
        Channel ch = new Channel();
        ch.paused = true;
        ch.semantic = ChannelSemantic.BARRIER;

        ChannelQuery q = ChannelQuery.builder().paused(true).semantic(ChannelSemantic.BARRIER).build();
        assertThat(q.matches(ch)).isTrue();

        ChannelQuery nonMatch = ChannelQuery.builder().paused(true).semantic(ChannelSemantic.APPEND).build();
        assertThat(nonMatch.matches(ch)).isFalse();
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelQueryTest -q 2>&1 | tail -5
```

Expected: FAIL — `ChannelQuery` does not exist yet.

- [ ] **Step 3: Create ChannelQuery.java**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/query/ChannelQuery.java`:

```java
package io.quarkiverse.qhorus.runtime.store.query;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;

public final class ChannelQuery {

    private final String namePattern;
    private final ChannelSemantic semantic;
    private final Boolean paused;

    private ChannelQuery(Builder b) {
        this.namePattern = b.namePattern;
        this.semantic = b.semantic;
        this.paused = b.paused;
    }

    public static ChannelQuery all() { return new Builder().build(); }
    public static ChannelQuery paused() { return new Builder().paused(true).build(); }
    public static ChannelQuery byName(String pattern) { return new Builder().namePattern(pattern).build(); }
    public static ChannelQuery bySemantic(ChannelSemantic s) { return new Builder().semantic(s).build(); }
    public static Builder builder() { return new Builder(); }

    public String namePattern() { return namePattern; }
    public ChannelSemantic semantic() { return semantic; }
    public Boolean paused() { return paused; }

    public boolean matches(Channel ch) {
        if (paused != null && !paused.equals(ch.paused)) return false;
        if (semantic != null && !semantic.equals(ch.semantic)) return false;
        if (namePattern != null && (ch.name == null || !ch.name.matches(namePattern.replace("*", ".*")))) return false;
        return true;
    }

    public Builder toBuilder() {
        return new Builder().namePattern(namePattern).semantic(semantic).paused(paused);
    }

    public static final class Builder {
        private String namePattern;
        private ChannelSemantic semantic;
        private Boolean paused;

        public Builder namePattern(String v) { this.namePattern = v; return this; }
        public Builder semantic(ChannelSemantic v) { this.semantic = v; return this; }
        public Builder paused(Boolean v) { this.paused = v; return this; }
        public ChannelQuery build() { return new ChannelQuery(this); }
    }
}
```

- [ ] **Step 4: Run to verify ChannelQuery tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelQueryTest -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, 5 tests passing.

- [ ] **Step 5: Create MessageQuery.java**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/query/MessageQuery.java`:

```java
package io.quarkiverse.qhorus.runtime.store.query;

import java.util.List;
import java.util.UUID;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;

public final class MessageQuery {

    private final UUID channelId;
    private final Long afterId;
    private final Integer limit;
    private final List<MessageType> excludeTypes;
    private final String sender;
    private final String target;
    private final String contentPattern;
    private final Long inReplyTo;

    private MessageQuery(Builder b) {
        this.channelId = b.channelId;
        this.afterId = b.afterId;
        this.limit = b.limit;
        this.excludeTypes = b.excludeTypes;
        this.sender = b.sender;
        this.target = b.target;
        this.contentPattern = b.contentPattern;
        this.inReplyTo = b.inReplyTo;
    }

    public static MessageQuery forChannel(UUID channelId) {
        return new Builder().channelId(channelId).build();
    }
    public static MessageQuery poll(UUID channelId, Long afterId, int limit) {
        return new Builder().channelId(channelId).afterId(afterId).limit(limit).build();
    }
    public static MessageQuery replies(UUID channelId, Long inReplyTo) {
        return new Builder().channelId(channelId).inReplyTo(inReplyTo).build();
    }
    public static Builder builder() { return new Builder(); }

    public UUID channelId() { return channelId; }
    public Long afterId() { return afterId; }
    public Integer limit() { return limit; }
    public List<MessageType> excludeTypes() { return excludeTypes; }
    public String sender() { return sender; }
    public String target() { return target; }
    public String contentPattern() { return contentPattern; }
    public Long inReplyTo() { return inReplyTo; }

    public boolean matches(Message m) {
        if (channelId != null && !channelId.equals(m.channelId)) return false;
        if (afterId != null && m.id != null && m.id <= afterId) return false;
        if (sender != null && !sender.equals(m.sender)) return false;
        if (target != null && !target.equals(m.target)) return false;
        if (inReplyTo != null && !inReplyTo.equals(m.inReplyTo)) return false;
        if (excludeTypes != null && excludeTypes.contains(m.messageType)) return false;
        if (contentPattern != null && (m.content == null ||
            !m.content.toLowerCase().contains(contentPattern.toLowerCase()))) return false;
        return true;
    }

    public Builder toBuilder() {
        return new Builder().channelId(channelId).afterId(afterId).limit(limit)
            .excludeTypes(excludeTypes).sender(sender).target(target)
            .contentPattern(contentPattern).inReplyTo(inReplyTo);
    }

    public static final class Builder {
        private UUID channelId;
        private Long afterId;
        private Integer limit;
        private List<MessageType> excludeTypes;
        private String sender;
        private String target;
        private String contentPattern;
        private Long inReplyTo;

        public Builder channelId(UUID v) { this.channelId = v; return this; }
        public Builder afterId(Long v) { this.afterId = v; return this; }
        public Builder limit(Integer v) { this.limit = v; return this; }
        public Builder excludeTypes(List<MessageType> v) { this.excludeTypes = v; return this; }
        public Builder sender(String v) { this.sender = v; return this; }
        public Builder target(String v) { this.target = v; return this; }
        public Builder contentPattern(String v) { this.contentPattern = v; return this; }
        public Builder inReplyTo(Long v) { this.inReplyTo = v; return this; }
        public MessageQuery build() { return new MessageQuery(this); }
    }
}
```

- [ ] **Step 6: Write and run MessageQueryTest**

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/query/MessageQueryTest.java`:

```java
package io.quarkiverse.qhorus.store.query;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;

class MessageQueryTest {

    @Test
    void forChannel_matchesCorrectChannel() {
        UUID channelId = UUID.randomUUID();
        Message m = new Message();
        m.channelId = channelId;
        m.messageType = MessageType.REQUEST;

        assertThat(MessageQuery.forChannel(channelId).matches(m)).isTrue();
        assertThat(MessageQuery.forChannel(UUID.randomUUID()).matches(m)).isFalse();
    }

    @Test
    void excludeTypes_filtersOutEventMessages() {
        Message m = new Message();
        m.messageType = MessageType.EVENT;
        m.channelId = UUID.randomUUID();

        MessageQuery q = MessageQuery.builder()
            .channelId(m.channelId)
            .excludeTypes(List.of(MessageType.EVENT))
            .build();

        assertThat(q.matches(m)).isFalse();
    }

    @Test
    void afterId_filtersOlderMessages() {
        Message m = new Message();
        m.channelId = UUID.randomUUID();
        m.id = 5L;
        m.messageType = MessageType.REQUEST;

        assertThat(MessageQuery.poll(m.channelId, 5L, 10).matches(m)).isFalse();
        assertThat(MessageQuery.poll(m.channelId, 4L, 10).matches(m)).isTrue();
    }

    @Test
    void contentPattern_matchesCaseInsensitive() {
        Message m = new Message();
        m.channelId = UUID.randomUUID();
        m.messageType = MessageType.REQUEST;
        m.content = "Hello World";

        MessageQuery q = MessageQuery.builder().channelId(m.channelId).contentPattern("hello").build();
        assertThat(q.matches(m)).isTrue();

        MessageQuery noMatch = MessageQuery.builder().channelId(m.channelId).contentPattern("goodbye").build();
        assertThat(noMatch.matches(m)).isFalse();
    }
}
```

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="MessageQueryTest" -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, 4 tests passing.

- [ ] **Step 7: Create InstanceQuery.java**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/query/InstanceQuery.java`:

```java
package io.quarkiverse.qhorus.runtime.store.query;

import java.time.Instant;

import io.quarkiverse.qhorus.runtime.instance.Instance;

public final class InstanceQuery {

    private final String capability;
    private final String status;
    private final Instant staleOlderThan;

    private InstanceQuery(Builder b) {
        this.capability = b.capability;
        this.status = b.status;
        this.staleOlderThan = b.staleOlderThan;
    }

    public static InstanceQuery all() { return new Builder().build(); }
    public static InstanceQuery online() { return new Builder().status("online").build(); }
    public static InstanceQuery byCapability(String tag) { return new Builder().capability(tag).build(); }
    public static InstanceQuery staleOlderThan(Instant threshold) {
        return new Builder().staleOlderThan(threshold).build();
    }
    public static Builder builder() { return new Builder(); }

    public String capability() { return capability; }
    public String status() { return status; }
    public Instant staleOlderThan() { return staleOlderThan; }

    // Note: capability matching on the Instance entity requires joining to the Capability table.
    // In-memory stores must load capabilities separately and check inclusion.
    public boolean matches(Instance inst) {
        if (status != null && !status.equals(inst.status)) return false;
        if (staleOlderThan != null && (inst.lastSeen == null || !inst.lastSeen.isBefore(staleOlderThan))) return false;
        // capability filter is applied by the store, not here, as it requires a join
        return true;
    }

    public Builder toBuilder() {
        return new Builder().capability(capability).status(status).staleOlderThan(staleOlderThan);
    }

    public static final class Builder {
        private String capability;
        private String status;
        private Instant staleOlderThan;

        public Builder capability(String v) { this.capability = v; return this; }
        public Builder status(String v) { this.status = v; return this; }
        public Builder staleOlderThan(Instant v) { this.staleOlderThan = v; return this; }
        public InstanceQuery build() { return new InstanceQuery(this); }
    }
}
```

- [ ] **Step 8: Create DataQuery.java**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/query/DataQuery.java`:

```java
package io.quarkiverse.qhorus.runtime.store.query;

import io.quarkiverse.qhorus.runtime.data.SharedData;

public final class DataQuery {

    private final String createdBy;
    private final Boolean complete;

    private DataQuery(Builder b) {
        this.createdBy = b.createdBy;
        this.complete = b.complete;
    }

    public static DataQuery all() { return new Builder().build(); }
    public static DataQuery complete() { return new Builder().complete(true).build(); }
    public static DataQuery byCreator(String instanceId) { return new Builder().createdBy(instanceId).build(); }
    public static Builder builder() { return new Builder(); }

    public String createdBy() { return createdBy; }
    public Boolean complete() { return complete; }

    public boolean matches(SharedData d) {
        if (createdBy != null && !createdBy.equals(d.createdBy)) return false;
        if (complete != null && !complete.equals(d.complete)) return false;
        return true;
    }

    public Builder toBuilder() { return new Builder().createdBy(createdBy).complete(complete); }

    public static final class Builder {
        private String createdBy;
        private Boolean complete;

        public Builder createdBy(String v) { this.createdBy = v; return this; }
        public Builder complete(Boolean v) { this.complete = v; return this; }
        public DataQuery build() { return new DataQuery(this); }
    }
}
```

- [ ] **Step 9: Create WatchdogQuery.java**

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/query/WatchdogQuery.java`:

```java
package io.quarkiverse.qhorus.runtime.store.query;

import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;

public final class WatchdogQuery {

    private final Boolean enabled;
    private final String conditionType;

    private WatchdogQuery(Builder b) {
        this.enabled = b.enabled;
        this.conditionType = b.conditionType;
    }

    public static WatchdogQuery all() { return new Builder().build(); }
    public static WatchdogQuery enabled() { return new Builder().enabled(true).build(); }
    public static Builder builder() { return new Builder(); }

    public Boolean enabled() { return enabled; }
    public String conditionType() { return conditionType; }

    public boolean matches(Watchdog w) {
        if (enabled != null && !enabled.equals(w.enabled)) return false;
        if (conditionType != null && !conditionType.equals(w.conditionType)) return false;
        return true;
    }

    public Builder toBuilder() { return new Builder().enabled(enabled).conditionType(conditionType); }

    public static final class Builder {
        private Boolean enabled;
        private String conditionType;

        public Builder enabled(Boolean v) { this.enabled = v; return this; }
        public Builder conditionType(String v) { this.conditionType = v; return this; }
        public WatchdogQuery build() { return new WatchdogQuery(this); }
    }
}
```

- [ ] **Step 10: Write and run InstanceQueryTest, DataQueryTest, WatchdogQueryTest**

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/query/InstanceQueryTest.java`:

```java
package io.quarkiverse.qhorus.store.query;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;

class InstanceQueryTest {

    @Test
    void online_matchesOnlineInstance() {
        Instance inst = new Instance();
        inst.status = "online";
        assertThat(InstanceQuery.online().matches(inst)).isTrue();
    }

    @Test
    void online_doesNotMatchOfflineInstance() {
        Instance inst = new Instance();
        inst.status = "offline";
        assertThat(InstanceQuery.online().matches(inst)).isFalse();
    }

    @Test
    void staleOlderThan_matchesInstanceNotSeenSince() {
        Instance inst = new Instance();
        inst.lastSeen = Instant.now().minusSeconds(300);
        assertThat(InstanceQuery.staleOlderThan(Instant.now().minusSeconds(60)).matches(inst)).isTrue();
    }

    @Test
    void staleOlderThan_doesNotMatchRecentInstance() {
        Instance inst = new Instance();
        inst.lastSeen = Instant.now();
        assertThat(InstanceQuery.staleOlderThan(Instant.now().minusSeconds(60)).matches(inst)).isFalse();
    }
}
```

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/query/DataQueryTest.java`:

```java
package io.quarkiverse.qhorus.store.query;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.query.DataQuery;

class DataQueryTest {

    @Test
    void complete_matchesCompleteArtefact() {
        SharedData d = new SharedData();
        d.complete = true;
        assertThat(DataQuery.complete().matches(d)).isTrue();
    }

    @Test
    void complete_doesNotMatchIncompleteArtefact() {
        SharedData d = new SharedData();
        d.complete = false;
        assertThat(DataQuery.complete().matches(d)).isFalse();
    }

    @Test
    void byCreator_matchesCorrectCreator() {
        SharedData d = new SharedData();
        d.createdBy = "agent-1";
        assertThat(DataQuery.byCreator("agent-1").matches(d)).isTrue();
        assertThat(DataQuery.byCreator("agent-2").matches(d)).isFalse();
    }
}
```

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/query/WatchdogQueryTest.java`:

```java
package io.quarkiverse.qhorus.store.query;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;

class WatchdogQueryTest {

    @Test
    void enabled_matchesEnabledWatchdog() {
        Watchdog w = new Watchdog();
        w.enabled = true;
        assertThat(WatchdogQuery.enabled().matches(w)).isTrue();
    }

    @Test
    void enabled_doesNotMatchDisabledWatchdog() {
        Watchdog w = new Watchdog();
        w.enabled = false;
        assertThat(WatchdogQuery.enabled().matches(w)).isFalse();
    }

    @Test
    void all_matchesAnyWatchdog() {
        Watchdog w = new Watchdog();
        w.enabled = false;
        assertThat(WatchdogQuery.all().matches(w)).isTrue();
    }
}
```

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="InstanceQueryTest,DataQueryTest,WatchdogQueryTest" -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, all tests passing.

- [ ] **Step 11: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/query/ \
        runtime/src/test/java/io/quarkiverse/qhorus/store/query/
git commit -m "feat(store): add Query value objects with matches() predicates for all 5 domains

Unit tests verify predicate logic for ChannelQuery, MessageQuery, InstanceQuery,
DataQuery, WatchdogQuery. matches() is used by in-memory store implementations.

Refs #INTERFACES_ISSUE"
```

---

## Task 5: Channel Domain — JpaChannelStore + ChannelService Migration

**Files:**
- Create: `runtime/src/main/java/.../store/jpa/JpaChannelStore.java`
- Modify: `runtime/src/main/java/.../channel/ChannelService.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaChannelStoreTest.java`

- [ ] **Step 1: Write failing integration test**

Read `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/Channel.java` first to understand fields (name, semantic, paused, rateLimitPerChannel, etc.).

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaChannelStoreTest.java`:

```java
package io.quarkiverse.qhorus.store;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.store.ChannelStore;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;

@QuarkusTest
class JpaChannelStoreTest {

    @Inject
    ChannelStore channelStore;

    @Test
    @TestTransaction
    void put_persistsChannelAndAssignsId() {
        Channel ch = new Channel();
        ch.name = "put-test-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;

        Channel saved = channelStore.put(ch);

        assertThat(saved.id).isNotNull();
        assertThat(saved.name).isEqualTo(ch.name);
    }

    @Test
    @TestTransaction
    void find_returnsChannel_whenExists() {
        Channel ch = new Channel();
        ch.name = "find-test-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;
        channelStore.put(ch);

        Optional<Channel> found = channelStore.find(ch.id);

        assertThat(found).isPresent();
        assertThat(found.get().name).isEqualTo(ch.name);
    }

    @Test
    @TestTransaction
    void find_returnsEmpty_whenNotFound() {
        assertThat(channelStore.find(UUID.randomUUID())).isEmpty();
    }

    @Test
    @TestTransaction
    void findByName_returnsChannel_whenExists() {
        Channel ch = new Channel();
        ch.name = "named-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.COLLECT;
        channelStore.put(ch);

        Optional<Channel> found = channelStore.findByName(ch.name);

        assertThat(found).isPresent();
        assertThat(found.get().semantic).isEqualTo(ChannelSemantic.COLLECT);
    }

    @Test
    @TestTransaction
    void scan_pausedQuery_returnsOnlyPausedChannels() {
        String suffix = UUID.randomUUID().toString();

        Channel active = new Channel();
        active.name = "active-" + suffix;
        active.semantic = ChannelSemantic.APPEND;
        active.paused = false;
        channelStore.put(active);

        Channel paused = new Channel();
        paused.name = "paused-" + suffix;
        paused.semantic = ChannelSemantic.APPEND;
        paused.paused = true;
        channelStore.put(paused);

        List<Channel> results = channelStore.scan(ChannelQuery.paused());

        assertThat(results).anyMatch(c -> c.name.equals(paused.name));
        assertThat(results).noneMatch(c -> c.name.equals(active.name));
    }

    @Test
    @TestTransaction
    void delete_removesChannel() {
        Channel ch = new Channel();
        ch.name = "delete-test-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;
        channelStore.put(ch);

        channelStore.delete(ch.id);

        assertThat(channelStore.find(ch.id)).isEmpty();
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaChannelStoreTest -q 2>&1 | tail -10
```

Expected: FAIL — `ChannelStore` has no CDI implementation yet.

- [ ] **Step 3: Create JpaChannelStore.java**

Read `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java` first to find the exact Panache queries to copy.

Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaChannelStore.java`:

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.store.ChannelStore;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;

@ApplicationScoped
public class JpaChannelStore implements ChannelStore {

    @Override
    @Transactional
    public Channel put(Channel channel) {
        channel.persistAndFlush();
        return channel;
    }

    @Override
    public Optional<Channel> find(UUID id) {
        return Optional.ofNullable(Channel.findById(id));
    }

    @Override
    public Optional<Channel> findByName(String name) {
        return Channel.find("name", name).firstResultOptional();
    }

    @Override
    public List<Channel> scan(ChannelQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Channel WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int paramIdx = 1;

        if (q.paused() != null) {
            jpql.append(" AND paused = ?").append(paramIdx++);
            params.add(q.paused());
        }
        if (q.semantic() != null) {
            jpql.append(" AND semantic = ?").append(paramIdx++);
            params.add(q.semantic());
        }
        if (q.namePattern() != null) {
            jpql.append(" AND name LIKE ?").append(paramIdx++);
            params.add(q.namePattern().replace("*", "%"));
        }

        return Channel.list(jpql.toString(), params.toArray());
    }

    @Override
    @Transactional
    public void delete(UUID id) {
        Channel.deleteById(id);
    }
}
```

- [ ] **Step 4: Run store tests — should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaChannelStoreTest -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, 5 tests passing.

- [ ] **Step 5: Migrate ChannelService to inject ChannelStore**

Read `runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java` in full.

Add `@Inject ChannelStore channelStore;` field. Replace each Panache static call with the corresponding store method. For example:

```java
// Before (in ChannelService):
return Channel.find("name", name).firstResultOptional();

// After:
return channelStore.findByName(name);
```

```java
// Before:
channel.persistAndFlush();

// After:
channelStore.put(channel);
```

Follow this pattern for every Panache call in `ChannelService`. Do not change any business logic — only swap the data access calls.

- [ ] **Step 6: Run full test suite — verify 561 tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: `Tests run: 561+, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaChannelStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/channel/ChannelService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/store/JpaChannelStoreTest.java
git commit -m "feat(store): JpaChannelStore + ChannelService migration

TDD: JpaChannelStoreTest (5 integration tests) written first.
ChannelService now injects ChannelStore; all 561 existing tests pass.

Refs #CHANNEL_ISSUE"
```

---

## Task 6: Message Domain — JpaMessageStore + MessageService Migration

**Files:**
- Create: `runtime/src/main/java/.../store/jpa/JpaMessageStore.java`
- Modify: `runtime/src/main/java/.../message/MessageService.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaMessageStoreTest.java`

- [ ] **Step 1: Write failing integration test**

Read `MessageService.java` and `Message.java` to understand fields (channelId, sender, messageType, content, correlationId, inReplyTo, target, etc.).

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaMessageStoreTest.java`:

```java
package io.quarkiverse.qhorus.store;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.store.ChannelStore;
import io.quarkiverse.qhorus.runtime.store.MessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class JpaMessageStoreTest {

    @Inject MessageStore messageStore;
    @Inject ChannelStore channelStore;

    private Channel makeChannel() {
        Channel ch = new Channel();
        ch.name = "msg-test-" + UUID.randomUUID();
        ch.semantic = ChannelSemantic.APPEND;
        return channelStore.put(ch);
    }

    @Test
    @TestTransaction
    void put_persistsMessage() {
        Channel ch = makeChannel();
        Message m = new Message();
        m.channelId = ch.id;
        m.sender = "agent-1";
        m.messageType = MessageType.REQUEST;
        m.content = "hello";

        Message saved = messageStore.put(m);

        assertThat(saved.id).isNotNull();
    }

    @Test
    @TestTransaction
    void scan_forChannel_returnsMessagesForChannel() {
        Channel ch = makeChannel();
        Message m = new Message();
        m.channelId = ch.id;
        m.sender = "agent-1";
        m.messageType = MessageType.REQUEST;
        m.content = "hello";
        messageStore.put(m);

        List<Message> results = messageStore.scan(MessageQuery.forChannel(ch.id));

        assertThat(results).hasSize(1);
        assertThat(results.get(0).content).isEqualTo("hello");
    }

    @Test
    @TestTransaction
    void scan_excludesEventMessages() {
        Channel ch = makeChannel();

        Message req = new Message();
        req.channelId = ch.id;
        req.sender = "a";
        req.messageType = MessageType.REQUEST;
        req.content = "req";
        messageStore.put(req);

        Message evt = new Message();
        evt.channelId = ch.id;
        evt.sender = "a";
        evt.messageType = MessageType.EVENT;
        evt.content = "evt";
        messageStore.put(evt);

        List<Message> results = messageStore.scan(
            MessageQuery.builder().channelId(ch.id)
                .excludeTypes(List.of(MessageType.EVENT)).build());

        assertThat(results).hasSize(1);
        assertThat(results.get(0).messageType).isEqualTo(MessageType.REQUEST);
    }

    @Test
    @TestTransaction
    void countByChannel_returnsCorrectCount() {
        Channel ch = makeChannel();
        for (int i = 0; i < 3; i++) {
            Message m = new Message();
            m.channelId = ch.id;
            m.sender = "a";
            m.messageType = MessageType.REQUEST;
            m.content = "msg-" + i;
            messageStore.put(m);
        }

        assertThat(messageStore.countByChannel(ch.id)).isEqualTo(3);
    }

    @Test
    @TestTransaction
    void deleteAll_removesAllMessagesInChannel() {
        Channel ch = makeChannel();
        Message m = new Message();
        m.channelId = ch.id;
        m.sender = "a";
        m.messageType = MessageType.REQUEST;
        m.content = "bye";
        messageStore.put(m);

        messageStore.deleteAll(ch.id);

        assertThat(messageStore.scan(MessageQuery.forChannel(ch.id))).isEmpty();
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaMessageStoreTest -q 2>&1 | tail -5
```

Expected: FAIL.

- [ ] **Step 3: Create JpaMessageStore.java**

Read `MessageService.java` to extract the exact Panache queries used there. Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaMessageStore.java`:

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.store.MessageStore;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;

@ApplicationScoped
public class JpaMessageStore implements MessageStore {

    @Override
    @Transactional
    public Message put(Message message) {
        message.persistAndFlush();
        return message;
    }

    @Override
    public Optional<Message> find(Long id) {
        return Optional.ofNullable(Message.findById(id));
    }

    @Override
    public List<Message> scan(MessageQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Message WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.channelId() != null) {
            jpql.append(" AND channelId = ?").append(idx++);
            params.add(q.channelId());
        }
        if (q.afterId() != null) {
            jpql.append(" AND id > ?").append(idx++);
            params.add(q.afterId());
        }
        if (q.sender() != null) {
            jpql.append(" AND sender = ?").append(idx++);
            params.add(q.sender());
        }
        if (q.target() != null) {
            jpql.append(" AND target = ?").append(idx++);
            params.add(q.target());
        }
        if (q.inReplyTo() != null) {
            jpql.append(" AND inReplyTo = ?").append(idx++);
            params.add(q.inReplyTo());
        }
        if (q.excludeTypes() != null && !q.excludeTypes().isEmpty()) {
            jpql.append(" AND messageType NOT IN ?").append(idx++);
            params.add(q.excludeTypes());
        }
        if (q.contentPattern() != null) {
            jpql.append(" AND LOWER(content) LIKE LOWER(?").append(idx++).append(")");
            params.add("%" + q.contentPattern() + "%");
        }
        jpql.append(" ORDER BY id ASC");

        List<Message> results = Message.list(jpql.toString(), params.toArray());
        if (q.limit() != null) {
            return results.stream().limit(q.limit()).toList();
        }
        return results;
    }

    @Override
    @Transactional
    public void deleteAll(UUID channelId) {
        Message.delete("channelId", channelId);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        Message.deleteById(id);
    }

    @Override
    public int countByChannel(UUID channelId) {
        return (int) Message.count("channelId", channelId);
    }
}
```

- [ ] **Step 4: Run store tests — should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaMessageStoreTest -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Migrate MessageService to inject MessageStore**

Read `MessageService.java` in full. Add `@Inject MessageStore messageStore;`. Replace all Panache static calls with the corresponding store methods. Do not change business logic.

- [ ] **Step 6: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: all tests passing, BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaMessageStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/message/MessageService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/store/JpaMessageStoreTest.java
git commit -m "feat(store): JpaMessageStore + MessageService migration

TDD: JpaMessageStoreTest (5 integration tests). scan(MessageQuery) handles
cursor pagination, excludeTypes, contentPattern, sender, inReplyTo filtering.

Refs #MESSAGE_ISSUE"
```

---

## Task 7: Instance Domain — JpaInstanceStore + InstanceService Migration

**Files:**
- Create: `runtime/src/main/java/.../store/jpa/JpaInstanceStore.java`
- Modify: `runtime/src/main/java/.../instance/InstanceService.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaInstanceStoreTest.java`

- [ ] **Step 1: Write failing integration test**

Read `InstanceService.java`, `Instance.java`, `Capability.java` to understand fields.

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaInstanceStoreTest.java`:

```java
package io.quarkiverse.qhorus.store;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.InstanceStore;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class JpaInstanceStoreTest {

    @Inject InstanceStore instanceStore;

    private Instance makeInstance(String id) {
        Instance inst = new Instance();
        inst.instanceId = id + "-" + UUID.randomUUID();
        inst.description = "test instance";
        inst.status = "online";
        inst.lastSeen = Instant.now();
        return instanceStore.put(inst);
    }

    @Test
    @TestTransaction
    void put_persistsInstance() {
        Instance saved = makeInstance("inst");
        assertThat(saved.id).isNotNull();
    }

    @Test
    @TestTransaction
    void findByInstanceId_returnsInstance() {
        Instance inst = makeInstance("findable");
        assertThat(instanceStore.findByInstanceId(inst.instanceId))
            .isPresent()
            .hasValueSatisfying(i -> assertThat(i.instanceId).isEqualTo(inst.instanceId));
    }

    @Test
    @TestTransaction
    void scan_online_returnsOnlyOnlineInstances() {
        Instance online = makeInstance("online");
        online.status = "online";
        instanceStore.put(online);

        Instance offline = makeInstance("offline");
        offline.status = "offline";
        instanceStore.put(offline);

        List<Instance> results = instanceStore.scan(InstanceQuery.online());
        assertThat(results).anyMatch(i -> i.instanceId.equals(online.instanceId));
        assertThat(results).noneMatch(i -> i.instanceId.equals(offline.instanceId));
    }

    @Test
    @TestTransaction
    void putCapabilities_andFindCapabilities_roundTrip() {
        Instance inst = makeInstance("capable");
        instanceStore.putCapabilities(inst.id, List.of("llm", "vision"));

        List<String> caps = instanceStore.findCapabilities(inst.id);
        assertThat(caps).containsExactlyInAnyOrder("llm", "vision");
    }

    @Test
    @TestTransaction
    void putCapabilities_replacesExistingCapabilities() {
        Instance inst = makeInstance("replaceable");
        instanceStore.putCapabilities(inst.id, List.of("old-cap"));
        instanceStore.putCapabilities(inst.id, List.of("new-cap"));

        assertThat(instanceStore.findCapabilities(inst.id)).containsExactly("new-cap");
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaInstanceStoreTest -q 2>&1 | tail -5
```

Expected: FAIL.

- [ ] **Step 3: Create JpaInstanceStore.java**

Read `InstanceService.java` to extract Panache queries. Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaInstanceStore.java`:

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.quarkiverse.qhorus.runtime.instance.Capability;
import io.quarkiverse.qhorus.runtime.instance.Instance;
import io.quarkiverse.qhorus.runtime.store.InstanceStore;
import io.quarkiverse.qhorus.runtime.store.query.InstanceQuery;

@ApplicationScoped
public class JpaInstanceStore implements InstanceStore {

    @Override
    @Transactional
    public Instance put(Instance instance) {
        instance.persistAndFlush();
        return instance;
    }

    @Override
    public Optional<Instance> find(UUID id) {
        return Optional.ofNullable(Instance.findById(id));
    }

    @Override
    public Optional<Instance> findByInstanceId(String instanceId) {
        return Instance.find("instanceId", instanceId).firstResultOptional();
    }

    @Override
    public List<Instance> scan(InstanceQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Instance WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.status() != null) {
            jpql.append(" AND status = ?").append(idx++);
            params.add(q.status());
        }
        if (q.staleOlderThan() != null) {
            jpql.append(" AND lastSeen < ?").append(idx++);
            params.add(q.staleOlderThan());
        }
        if (q.capability() != null) {
            // join via EXISTS subquery on capability table
            jpql.append(" AND id IN (SELECT c.instanceId FROM Capability c WHERE c.tag = ?").append(idx++).append(")");
            params.add(q.capability());
        }

        return Instance.list(jpql.toString(), params.toArray());
    }

    @Override
    @Transactional
    public void putCapabilities(UUID instanceId, List<String> tags) {
        Capability.delete("instanceId", instanceId);
        for (String tag : tags) {
            Capability cap = new Capability();
            cap.instanceId = instanceId;
            cap.tag = tag;
            cap.persist();
        }
    }

    @Override
    @Transactional
    public void deleteCapabilities(UUID instanceId) {
        Capability.delete("instanceId", instanceId);
    }

    @Override
    public List<String> findCapabilities(UUID instanceId) {
        return Capability.<Capability>list("instanceId", instanceId)
            .stream().map(c -> c.tag).toList();
    }

    @Override
    @Transactional
    public void delete(UUID id) {
        Capability.delete("instanceId", id);
        Instance.deleteById(id);
    }
}
```

- [ ] **Step 4: Run store tests — should pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaInstanceStoreTest -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Migrate InstanceService**

Read `InstanceService.java`. Add `@Inject InstanceStore instanceStore;`. Replace all Panache calls with store methods. Keep business logic unchanged.

- [ ] **Step 6: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: all tests passing.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaInstanceStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/instance/InstanceService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/store/JpaInstanceStoreTest.java
git commit -m "feat(store): JpaInstanceStore + InstanceService migration

putCapabilities is replace-all (delete-then-insert). Capability subquery
handles capability-based instance filtering in scan().

Refs #INSTANCE_ISSUE"
```

---

## Task 8: Data Domain — JpaDataStore + DataService Migration

**Files:**
- Create: `runtime/src/main/java/.../store/jpa/JpaDataStore.java`
- Modify: `runtime/src/main/java/.../data/DataService.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaDataStoreTest.java`

- [ ] **Step 1: Write failing integration test**

Read `DataService.java`, `SharedData.java`, `ArtefactClaim.java`.

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaDataStoreTest.java`:

```java
package io.quarkiverse.qhorus.store;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.DataStore;
import io.quarkiverse.qhorus.runtime.store.query.DataQuery;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class JpaDataStoreTest {

    @Inject DataStore dataStore;

    private SharedData makeData(String key) {
        SharedData d = new SharedData();
        d.dataKey = key + "-" + UUID.randomUUID();
        d.content = "test content";
        d.createdBy = "agent-1";
        d.complete = true;
        d.sizeBytes = 12L;
        return dataStore.put(d);
    }

    @Test
    @TestTransaction
    void put_persistsSharedData() {
        SharedData saved = makeData("put-test");
        assertThat(saved.id).isNotNull();
    }

    @Test
    @TestTransaction
    void findByKey_returnsData() {
        SharedData d = makeData("findable");
        assertThat(dataStore.findByKey(d.dataKey))
            .isPresent()
            .hasValueSatisfying(found -> assertThat(found.content).isEqualTo("test content"));
    }

    @Test
    @TestTransaction
    void scan_complete_returnsOnlyCompleteArtefacts() {
        SharedData complete = makeData("complete");
        complete.complete = true;
        dataStore.put(complete);

        SharedData incomplete = makeData("incomplete");
        incomplete.complete = false;
        dataStore.put(incomplete);

        assertThat(dataStore.scan(DataQuery.complete()))
            .anyMatch(d -> d.dataKey.equals(complete.dataKey))
            .noneMatch(d -> d.dataKey.equals(incomplete.dataKey));
    }

    @Test
    @TestTransaction
    void putClaim_andCountClaims_roundTrip() {
        SharedData d = makeData("claimed");

        ArtefactClaim claim = new ArtefactClaim();
        claim.artefactId = d.id;
        claim.instanceId = UUID.randomUUID();
        dataStore.putClaim(claim);

        assertThat(dataStore.countClaims(d.id)).isEqualTo(1);
    }

    @Test
    @TestTransaction
    void deleteClaim_reducesCount() {
        SharedData d = makeData("release-test");
        UUID instanceId = UUID.randomUUID();

        ArtefactClaim claim = new ArtefactClaim();
        claim.artefactId = d.id;
        claim.instanceId = instanceId;
        dataStore.putClaim(claim);

        dataStore.deleteClaim(d.id, instanceId);

        assertThat(dataStore.countClaims(d.id)).isEqualTo(0);
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaDataStoreTest -q 2>&1 | tail -5
```

- [ ] **Step 3: Create JpaDataStore.java**

Read `DataService.java` to extract Panache queries. Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaDataStore.java`:

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.quarkiverse.qhorus.runtime.data.ArtefactClaim;
import io.quarkiverse.qhorus.runtime.data.SharedData;
import io.quarkiverse.qhorus.runtime.store.DataStore;
import io.quarkiverse.qhorus.runtime.store.query.DataQuery;

@ApplicationScoped
public class JpaDataStore implements DataStore {

    @Override
    @Transactional
    public SharedData put(SharedData data) {
        data.persistAndFlush();
        return data;
    }

    @Override
    public Optional<SharedData> find(UUID id) {
        return Optional.ofNullable(SharedData.findById(id));
    }

    @Override
    public Optional<SharedData> findByKey(String key) {
        return SharedData.find("dataKey", key).firstResultOptional();
    }

    @Override
    public List<SharedData> scan(DataQuery q) {
        StringBuilder jpql = new StringBuilder("FROM SharedData WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.createdBy() != null) {
            jpql.append(" AND createdBy = ?").append(idx++);
            params.add(q.createdBy());
        }
        if (q.complete() != null) {
            jpql.append(" AND complete = ?").append(idx++);
            params.add(q.complete());
        }

        return SharedData.list(jpql.toString(), params.toArray());
    }

    @Override
    @Transactional
    public ArtefactClaim putClaim(ArtefactClaim claim) {
        claim.persistAndFlush();
        return claim;
    }

    @Override
    @Transactional
    public void deleteClaim(UUID artefactId, UUID instanceId) {
        ArtefactClaim.delete("artefactId = ?1 AND instanceId = ?2", artefactId, instanceId);
    }

    @Override
    public int countClaims(UUID artefactId) {
        return (int) ArtefactClaim.count("artefactId", artefactId);
    }

    @Override
    @Transactional
    public void delete(UUID id) {
        ArtefactClaim.delete("artefactId", id);
        SharedData.deleteById(id);
    }
}
```

- [ ] **Step 4: Run store tests, migrate DataService, run full suite, commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaDataStoreTest -q 2>&1 | tail -5
```

Read `DataService.java`. Add `@Inject DataStore dataStore;`. Replace Panache calls with store methods.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | grep -E "Tests run:|BUILD"
```

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaDataStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/data/DataService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/store/JpaDataStoreTest.java
git commit -m "feat(store): JpaDataStore + DataService migration

Refs #DATA_ISSUE"
```

---

## Task 9: Watchdog Domain — JpaWatchdogStore + WatchdogEvaluationService Migration

**Files:**
- Create: `runtime/src/main/java/.../store/jpa/JpaWatchdogStore.java`
- Modify: `runtime/src/main/java/.../watchdog/WatchdogEvaluationService.java`
- Create: `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaWatchdogStoreTest.java`

- [ ] **Step 1: Write failing integration test**

Read `WatchdogEvaluationService.java` and `Watchdog.java` to understand fields (enabled, conditionType, etc.).

Create `runtime/src/test/java/io/quarkiverse/qhorus/store/JpaWatchdogStoreTest.java`:

```java
package io.quarkiverse.qhorus.store;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.store.WatchdogStore;
import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class JpaWatchdogStoreTest {

    @Inject WatchdogStore watchdogStore;

    private Watchdog makeWatchdog(boolean enabled) {
        Watchdog w = new Watchdog();
        // read Watchdog.java and populate all required fields
        w.enabled = enabled;
        return watchdogStore.put(w);
    }

    @Test
    @TestTransaction
    void put_persistsWatchdog() {
        Watchdog saved = makeWatchdog(true);
        assertThat(saved.id).isNotNull();
    }

    @Test
    @TestTransaction
    void scan_enabled_returnsOnlyEnabledWatchdogs() {
        Watchdog enabled = makeWatchdog(true);
        Watchdog disabled = makeWatchdog(false);

        List<Watchdog> results = watchdogStore.scan(WatchdogQuery.enabled());
        assertThat(results).anyMatch(w -> w.id.equals(enabled.id));
        assertThat(results).noneMatch(w -> w.id.equals(disabled.id));
    }

    @Test
    @TestTransaction
    void delete_removesWatchdog() {
        Watchdog w = makeWatchdog(true);
        watchdogStore.delete(w.id);
        assertThat(watchdogStore.find(w.id)).isEmpty();
    }
}
```

**Important:** After reading `Watchdog.java`, populate `makeWatchdog()` with all required (non-null) fields. The placeholder above must be completed before running.

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaWatchdogStoreTest -q 2>&1 | tail -5
```

- [ ] **Step 3: Create JpaWatchdogStore.java**

Read `WatchdogEvaluationService.java` to extract Panache queries. Create `runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaWatchdogStore.java`:

```java
package io.quarkiverse.qhorus.runtime.store.jpa;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.quarkiverse.qhorus.runtime.store.WatchdogStore;
import io.quarkiverse.qhorus.runtime.store.query.WatchdogQuery;
import io.quarkiverse.qhorus.runtime.watchdog.Watchdog;

@ApplicationScoped
public class JpaWatchdogStore implements WatchdogStore {

    @Override
    @Transactional
    public Watchdog put(Watchdog watchdog) {
        watchdog.persistAndFlush();
        return watchdog;
    }

    @Override
    public Optional<Watchdog> find(UUID id) {
        return Optional.ofNullable(Watchdog.findById(id));
    }

    @Override
    public List<Watchdog> scan(WatchdogQuery q) {
        StringBuilder jpql = new StringBuilder("FROM Watchdog WHERE 1=1");
        List<Object> params = new ArrayList<>();
        int idx = 1;

        if (q.enabled() != null) {
            jpql.append(" AND enabled = ?").append(idx++);
            params.add(q.enabled());
        }
        if (q.conditionType() != null) {
            jpql.append(" AND conditionType = ?").append(idx++);
            params.add(q.conditionType());
        }

        return Watchdog.list(jpql.toString(), params.toArray());
    }

    @Override
    @Transactional
    public void delete(UUID id) {
        Watchdog.deleteById(id);
    }
}
```

- [ ] **Step 4: Run store tests, migrate WatchdogEvaluationService, run full suite, commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaWatchdogStoreTest -q 2>&1 | tail -5
```

Read `WatchdogEvaluationService.java`. Add `@Inject WatchdogStore watchdogStore;`. Replace Panache calls.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | grep -E "Tests run:|BUILD"
```

```bash
git add runtime/src/main/java/io/quarkiverse/qhorus/runtime/store/jpa/JpaWatchdogStore.java \
        runtime/src/main/java/io/quarkiverse/qhorus/runtime/watchdog/WatchdogEvaluationService.java \
        runtime/src/test/java/io/quarkiverse/qhorus/store/JpaWatchdogStoreTest.java
git commit -m "feat(store): JpaWatchdogStore + WatchdogEvaluationService migration

Refs #WATCHDOG_ISSUE"
```

---

## Task 10: In-Memory Stores (testing/ module)

**Files:** Create all 5 in-memory stores and their unit tests.

- [ ] **Step 1: Create InMemoryChannelStore.java**

Create `testing/src/main/java/io/quarkiverse/qhorus/testing/InMemoryChannelStore.java`:

```java
package io.quarkiverse.qhorus.testing;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.store.ChannelStore;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryChannelStore implements ChannelStore {

    private final Map<UUID, Channel> store = new LinkedHashMap<>();

    @Override
    public Channel put(Channel channel) {
        if (channel.id == null) channel.id = UUID.randomUUID();
        store.put(channel.id, channel);
        return channel;
    }

    @Override
    public Optional<Channel> find(UUID id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public Optional<Channel> findByName(String name) {
        return store.values().stream().filter(c -> name.equals(c.name)).findFirst();
    }

    @Override
    public List<Channel> scan(ChannelQuery query) {
        return store.values().stream().filter(query::matches).toList();
    }

    @Override
    public void delete(UUID id) {
        store.remove(id);
    }

    public void clear() { store.clear(); }
}
```

- [ ] **Step 2: Write InMemoryChannelStoreTest**

Create `testing/src/test/java/io/quarkiverse/qhorus/testing/InMemoryChannelStoreTest.java`:

```java
package io.quarkiverse.qhorus.testing;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;

class InMemoryChannelStoreTest {

    private InMemoryChannelStore store;

    @BeforeEach
    void setUp() { store = new InMemoryChannelStore(); }

    @Test
    void put_assignsUuid_whenIdIsNull() {
        Channel ch = new Channel();
        ch.name = "test";
        ch.semantic = ChannelSemantic.APPEND;
        Channel saved = store.put(ch);
        assertThat(saved.id).isNotNull();
    }

    @Test
    void find_returnsEmpty_whenNotFound() {
        assertThat(store.find(UUID.randomUUID())).isEmpty();
    }

    @Test
    void findByName_returnsChannel() {
        Channel ch = new Channel();
        ch.name = "findme";
        ch.semantic = ChannelSemantic.COLLECT;
        store.put(ch);
        assertThat(store.findByName("findme")).isPresent();
    }

    @Test
    void scan_paused_returnsOnlyPausedChannels() {
        Channel active = new Channel();
        active.name = "active";
        active.paused = false;
        active.semantic = ChannelSemantic.APPEND;
        store.put(active);

        Channel paused = new Channel();
        paused.name = "paused";
        paused.paused = true;
        paused.semantic = ChannelSemantic.APPEND;
        store.put(paused);

        List<Channel> results = store.scan(ChannelQuery.paused());
        assertThat(results).hasSize(1);
        assertThat(results.get(0).name).isEqualTo("paused");
    }

    @Test
    void delete_removesChannel() {
        Channel ch = new Channel();
        ch.name = "bye";
        ch.semantic = ChannelSemantic.APPEND;
        store.put(ch);
        store.delete(ch.id);
        assertThat(store.find(ch.id)).isEmpty();
    }

    @Test
    void clear_removesAll() {
        Channel ch = new Channel();
        ch.name = "clear-test";
        ch.semantic = ChannelSemantic.APPEND;
        store.put(ch);
        store.clear();
        assertThat(store.scan(ChannelQuery.all())).isEmpty();
    }
}
```

- [ ] **Step 3: Run InMemoryChannelStoreTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=InMemoryChannelStoreTest -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, 6 tests passing.

- [ ] **Step 4: Create remaining in-memory stores and their tests**

Create the following files, following the same pattern as `InMemoryChannelStore`:

**InMemoryMessageStore.java** — `Map<Long, Message>`. `put()` assigns `id` using an `AtomicLong` counter (messages use `Long` IDs, not UUID). `scan()` applies `MessageQuery.matches()` then applies `limit()` if non-null. Add `void clear()`.

**InMemoryInstanceStore.java** — `Map<UUID, Instance>` for instances, `Map<UUID, List<String>>` for capabilities (keyed by instanceId). `putCapabilities()` replaces the entire list. Add `void clear()`.

**InMemoryDataStore.java** — `Map<UUID, SharedData>` for data, `Map<String, List<ArtefactClaim>>` for claims (keyed by artefactId string). `putClaim()` adds to the list. `deleteClaim()` removes by `artefactId` + `instanceId`. `countClaims()` returns list size. Add `void clear()`.

**InMemoryWatchdogStore.java** — `Map<UUID, Watchdog>`. `scan()` applies `WatchdogQuery.matches()`. Add `void clear()`.

Write a matching `InMemory*StoreTest` for each, following the same pattern as `InMemoryChannelStoreTest` — at minimum: put assigns id, find returns empty when not found, scan filters correctly, clear removes all.

- [ ] **Step 5: Run all testing module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -q 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: all in-memory store tests passing.

- [ ] **Step 6: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 7: Commit**

```bash
git add testing/
git commit -m "feat(testing): InMemory store implementations for all 5 domains

@Alternative @Priority(1) — CDI activates automatically when testing module
is on classpath. No database required; clear() in @BeforeEach for isolation.

Closes #INMEMORY_ISSUE"
```

---

## Task 11: ADR-0002 + DESIGN.md Update

**Files:**
- Create: `adr/0002-persistence-abstraction-store-pattern.md`
- Modify: `adr/INDEX.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Create ADR-0002**

Create `adr/0002-persistence-abstraction-store-pattern.md`:

```markdown
# ADR-0002: Persistence Abstraction — Store + scan(Query) Pattern

**Date:** 2026-04-18
**Status:** Accepted

## Context

Qhorus services called Panache entity statics directly. This made the
JPA/Panache backend impossible to swap without touching services or MCP tools,
and made unit testing consumers of Qhorus require a running database.

## Decision

Introduce per-domain store interfaces (`ChannelStore`, `MessageStore`,
`InstanceStore`, `DataStore`, `WatchdogStore`) following the
`put` / `find` / `scan(Query)` KV pattern established in quarkus-workitems.

Each store has:
- A `*Query` value object with immutable fields, static factories, a builder,
  and a `matches(Entity)` predicate for in-memory implementations.
- A `Jpa*Store` default implementation (`@ApplicationScoped`) using Panache.
- An `InMemory*Store` in the `quarkus-qhorus-testing` module
  (`@Alternative @Priority(1)`) for consumers who want fast tests.

Services inject stores instead of calling Panache statics. Business logic
(BARRIER semantics, rate limiting, observer fanout) remains in services.

Reactive migration (`Uni<T>`) is deferred — store interfaces are blocking;
when reactive is added, only the interfaces and JPA implementations change.

## Rationale

- **quarkus-workitems** validated this pattern in the same ecosystem. The
  `scan(Query)` shape is strictly better than named finders for extensibility.
- **CDI `@Alternative @Priority(1)`** is the Quarkus-native way to swap
  implementations — no builder pattern needed inside a CDI container.
- **No Info record layer** — Panache entities are POJOs; in-memory
  implementations store them in Maps without needing JPA. This saves a
  full mapping layer (validated by quarkus-workitems).

## Consequences

- Consumers can add `quarkus-qhorus-testing` at test scope for instant-boot
  unit tests with no database.
- All 5 services now depend on store interfaces — alternative backends
  (MongoDB, Redis, MVStore) implement those interfaces and activate via CDI.
- Reactive migration path: swap `T` → `Uni<T>` on interfaces + JPA impls.

## References

- Cross-project comparison: `docs/specs/2026-04-17-persistence-abstraction-strategy.md`
- Design spec: `docs/superpowers/specs/2026-04-18-persistence-abstraction-design.md`
- quarkus-workitems: `WorkItemStore`, `WorkItemQuery` (established pattern)
```

- [ ] **Step 2: Update adr/INDEX.md**

Add the new ADR entry to `adr/INDEX.md`.

- [ ] **Step 3: Update docs/DESIGN.md**

In `docs/DESIGN.md`, update the **Services** section to note store injection, add a new **Persistence Abstraction** section, and update the **Build Roadmap** to mark Phase 13 done. Update test count to reflect new tests added.

Add to Services section key invariants:
```
- Services inject `*Store` interfaces — Panache calls are isolated in `Jpa*Store` implementations; alternative backends activate via CDI `@Alternative @Priority(1)`.
```

Add new section after Services:
```markdown
## Persistence Abstraction

Five domain store interfaces under `runtime/store/`, JPA implementations under
`runtime/store/jpa/`, in-memory implementations in `quarkus-qhorus-testing`.

| Interface | Query Object | Key extras |
|---|---|---|
| `ChannelStore` | `ChannelQuery(namePattern, semantic, paused)` | — |
| `MessageStore` | `MessageQuery(channelId, afterId, limit, excludeTypes, sender, target, contentPattern, inReplyTo)` | `countByChannel`, `deleteAll` |
| `InstanceStore` | `InstanceQuery(capability, status, staleOlderThan)` | `putCapabilities` (replace-all), `findCapabilities` |
| `DataStore` | `DataQuery(createdBy, complete)` | `putClaim`, `deleteClaim`, `countClaims` |
| `WatchdogStore` | `WatchdogQuery(enabled, conditionType)` | — |

Consumers add `quarkus-qhorus-testing` at test scope to activate in-memory
stores automatically — no database required for unit tests.
```

- [ ] **Step 4: Commit**

```bash
git add adr/0002-persistence-abstraction-store-pattern.md adr/INDEX.md docs/DESIGN.md
git commit -m "docs: ADR-0002 persistence abstraction + DESIGN.md update

Records Store + scan(Query) decision. DESIGN.md updated with Persistence
Abstraction section and store injection invariant.

Refs #DOCS_ISSUE"
```

---

## Task 12: Examples Module

**Files:**
- Create: `examples/src/main/java/io/quarkiverse/qhorus/examples/StoreUsageExample.java`
- Create: `examples/src/test/java/io/quarkiverse/qhorus/examples/StoreUsageExampleTest.java`

- [ ] **Step 1: Create StoreUsageExample.java**

This class shows how a consuming service injects Qhorus stores:

```java
package io.quarkiverse.qhorus.examples;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.runtime.store.ChannelStore;
import io.quarkiverse.qhorus.runtime.store.MessageStore;
import io.quarkiverse.qhorus.runtime.store.query.ChannelQuery;
import io.quarkiverse.qhorus.runtime.store.query.MessageQuery;

/**
 * Example showing how to use Qhorus store interfaces in a consuming service.
 *
 * In production: JpaChannelStore / JpaMessageStore are the default CDI beans.
 * In tests: add quarkus-qhorus-testing to test scope — InMemory*Store activates
 * automatically via @Alternative @Priority(1), no database required.
 */
@ApplicationScoped
public class StoreUsageExample {

    // Package-private for direct injection in unit tests (see StoreUsageExampleTest)
    @Inject ChannelStore channelStore;
    @Inject MessageStore messageStore;

    /** Create a channel and return it. */
    public Channel createChannel(String name, ChannelSemantic semantic) {
        Channel ch = new Channel();
        ch.name = name;
        ch.semantic = semantic;
        return channelStore.put(ch);
    }

    /** Post a message to a named channel. Returns empty if channel not found. */
    public Optional<Message> postMessage(String channelName, String sender, String content) {
        return channelStore.findByName(channelName).map(ch -> {
            Message m = new Message();
            m.channelId = ch.id;
            m.sender = sender;
            m.messageType = MessageType.REQUEST;
            m.content = content;
            return messageStore.put(m);
        });
    }

    /** Retrieve all non-EVENT messages for a channel after a cursor. */
    public List<Message> pollMessages(UUID channelId, Long afterId) {
        return messageStore.scan(
            MessageQuery.poll(channelId, afterId, 20)
                .toBuilder()
                .excludeTypes(List.of(MessageType.EVENT))
                .build());
    }

    /** List all paused channels. */
    public List<Channel> pausedChannels() {
        return channelStore.scan(ChannelQuery.paused());
    }
}
```

- [ ] **Step 2: Create StoreUsageExampleTest.java**

This test demonstrates using `InMemoryChannelStore` / `InMemoryMessageStore` — no database needed:

```java
package io.quarkiverse.qhorus.examples;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Optional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.qhorus.runtime.channel.Channel;
import io.quarkiverse.qhorus.runtime.channel.ChannelSemantic;
import io.quarkiverse.qhorus.runtime.message.Message;
import io.quarkiverse.qhorus.runtime.message.MessageType;
import io.quarkiverse.qhorus.testing.InMemoryChannelStore;
import io.quarkiverse.qhorus.testing.InMemoryMessageStore;

/**
 * Happy path example test using in-memory stores — no database, no Quarkus boot.
 *
 * This is the recommended pattern for consumers of quarkus-qhorus who want
 * fast unit tests. Add quarkus-qhorus-testing to test scope and wire up
 * InMemory*Store directly (or let CDI do it with @Alternative @Priority(1)).
 */
class StoreUsageExampleTest {

    private InMemoryChannelStore channelStore;
    private InMemoryMessageStore messageStore;
    private StoreUsageExample example;

    @BeforeEach
    void setUp() {
        channelStore = new InMemoryChannelStore();
        messageStore = new InMemoryMessageStore();
        example = new StoreUsageExample();
        // Direct field injection for unit test (CDI not needed here)
        example.channelStore = channelStore;
        example.messageStore = messageStore;
    }

    @Test
    void happyPath_createChannelAndPostMessage() {
        Channel ch = example.createChannel("coordination", ChannelSemantic.APPEND);
        assertThat(ch.id).isNotNull();
        assertThat(ch.name).isEqualTo("coordination");

        Optional<Message> msg = example.postMessage("coordination", "agent-1", "hello");
        assertThat(msg).isPresent();
        assertThat(msg.get().content).isEqualTo("hello");
        assertThat(msg.get().sender).isEqualTo("agent-1");
        assertThat(msg.get().messageType).isEqualTo(MessageType.REQUEST);
    }

    @Test
    void pollMessages_excludesEventMessages() {
        Channel ch = example.createChannel("events-test", ChannelSemantic.APPEND);

        example.postMessage("events-test", "agent-1", "real message");

        // Post an EVENT directly via store (bypassing example method)
        Message evt = new Message();
        evt.channelId = ch.id;
        evt.sender = "agent-1";
        evt.messageType = MessageType.EVENT;
        evt.content = "{\"tool_name\":\"foo\",\"duration_ms\":42}";
        messageStore.put(evt);

        List<Message> polled = example.pollMessages(ch.id, null);
        assertThat(polled).hasSize(1);
        assertThat(polled.get(0).messageType).isEqualTo(MessageType.REQUEST);
    }

    @Test
    void postMessage_returnsEmpty_whenChannelNotFound() {
        Optional<Message> result = example.postMessage("no-such-channel", "agent-1", "hello");
        assertThat(result).isEmpty();
    }

    @Test
    void pausedChannels_returnsOnlyPausedChannels() {
        Channel active = example.createChannel("active-ch", ChannelSemantic.APPEND);
        active.paused = false;
        channelStore.put(active);

        Channel paused = example.createChannel("paused-ch", ChannelSemantic.APPEND);
        paused.paused = true;
        channelStore.put(paused);

        List<Channel> results = example.pausedChannels();
        assertThat(results).anyMatch(c -> c.name.equals("paused-ch"));
        assertThat(results).noneMatch(c -> c.name.equals("active-ch"));
    }
}
```

- [ ] **Step 3: Run examples tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples -q 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: BUILD SUCCESS, 4 tests passing.

- [ ] **Step 4: Run full build — all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add examples/
git commit -m "docs(examples): StoreUsageExample with happy-path unit tests

Shows channel creation, message posting, polling, and paused-channel
listing using InMemoryChannelStore/InMemoryMessageStore — no database needed.

Closes #DOCS_ISSUE"
```

---

## Final Verification

- [ ] **Run full suite across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install 2>&1 | grep -E "Tests run:|BUILD|module"
```

Expected: BUILD SUCCESS. Runtime module: 561+ tests passing (original 561 + new store integration tests + new query unit tests). Testing module: 25+ tests. Examples module: 4 tests.

- [ ] **Close the epic**

```bash
gh issue close #EPIC --repo mdproctor/quarkus-qhorus \
  --comment "All child issues resolved. Store abstraction complete across all 5 domains. testing/ and examples/ modules added."
```

---

## Issue Number Reference

Replace these placeholders with actual GitHub issue numbers from Task 1:

| Placeholder | Description |
|---|---|
| `#EPIC` | Persistence abstraction epic |
| `#SCAFFOLDING_ISSUE` | testing/ + examples/ module scaffold |
| `#INTERFACES_ISSUE` | Store interfaces + Query objects |
| `#CHANNEL_ISSUE` | Channel domain |
| `#MESSAGE_ISSUE` | Message domain |
| `#INSTANCE_ISSUE` | Instance domain |
| `#DATA_ISSUE` | Data domain |
| `#WATCHDOG_ISSUE` | Watchdog domain |
| `#INMEMORY_ISSUE` | In-memory stores |
| `#DOCS_ISSUE` | ADR-0002 + DESIGN.md + examples |

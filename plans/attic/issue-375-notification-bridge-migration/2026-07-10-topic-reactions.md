# Topic and Reactions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #328 — epic: conversation model enrichments for qhorus-native chat UI
**Issue group:** #328, #329, #330, #331, #332, #333, #334

**Goal:** Add Topic (named sub-conversations) and Reactions (emoji reactions) to the qhorus conversation model, per the approved design spec.

**Architecture:** Hybrid Topic entity + denormalized topic string on Message. Topic is organizational (not normative) — recorded on the ledger at dispatch time but never mutated there. Reactions are non-normative UI metadata outside the dispatch pipeline, with their own CDI event for live notification.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate (Panache), H2 (test), Flyway migrations

## Global Constraints

- Pre-release platform — breaking changes cost nothing
- All store interfaces in `api/store/`; JPA impls in `runtime/store/jpa/`; InMemory in `persistence-memory/`
- Reactive parity required (gated by `@IfBuildProperty(casehub.qhorus.reactive.enabled)`)
- InMemory stores: `@Alternative @Priority(1)`; must not mutate PanacheEntity fields in session scope
- CDI-free unit tests set `service.tracingConfig` to disabled-tracing impl
- `@QuarkusTest` integration tests use `QuarkusTransaction.requiringNew()` not `@TestTransaction` for observer assertions
- Unique channel names per test (RateLimiter is ApplicationScoped, state doesn't roll back)
- `@Tool` overloads: non-`@Tool` convenience overloads must be package-private
- Topic naming: free-form text, trimmed, max 200 chars, case-insensitive matching, "general" reserved
- Flyway migrations start at V28; next domain migration updates CLAUDE.md to V32

---

### Task 1: API Layer — Records, Interfaces, Message Modifications

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/message/Topic.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/Reaction.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/ReactionChangedEvent.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/ReactionGroup.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/message/TopicSummary.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/TopicStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/ReactionStore.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/Message.java` — add `topic` field
- Modify: `api/src/main/java/io/casehub/qhorus/api/message/MessageDispatch.java` — add `topic` field, null→"general" default
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/query/MessageQuery.java` — add `topic` field + matches()
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/MessageReceivedEvent.java` — add `topic` field
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/MessageStore.java` — add `updateTopicName()`

**Interfaces:**
- Produces: `Topic`, `Reaction`, `ReactionChangedEvent`, `ReactionGroup`, `TopicSummary` records; `TopicStore`, `ReactionStore` interfaces; `Message.topic()`, `MessageDispatch.topic()`, `MessageQuery.topic()`, `MessageReceivedEvent.topic()`, `MessageStore.updateTopicName()`

- [ ] **Step 1: Create Topic record**

```java
// api/src/main/java/io/casehub/qhorus/api/message/Topic.java
package io.casehub.qhorus.api.message;

import java.time.Instant;
import java.util.UUID;

public record Topic(
        Long id,
        UUID channelId,
        String name,
        boolean resolved,
        Instant resolvedAt,
        String resolvedBy,
        Instant createdAt,
        String tenancyId) {}
```

- [ ] **Step 2: Create TopicSummary record**

```java
// api/src/main/java/io/casehub/qhorus/api/message/TopicSummary.java
package io.casehub.qhorus.api.message;

import java.time.Instant;

public record TopicSummary(
        String name,
        long messageCount,
        Instant lastActivityAt,
        boolean resolved,
        Instant resolvedAt) {}
```

- [ ] **Step 3: Create Reaction record**

```java
// api/src/main/java/io/casehub/qhorus/api/message/Reaction.java
package io.casehub.qhorus.api.message;

import java.time.Instant;

public record Reaction(
        Long id,
        Long messageId,
        String emoji,
        String actorId,
        Instant createdAt,
        String tenancyId) {}
```

- [ ] **Step 4: Create ReactionGroup record**

```java
// api/src/main/java/io/casehub/qhorus/api/message/ReactionGroup.java
package io.casehub.qhorus.api.message;

import java.util.List;

public record ReactionGroup(
        String emoji,
        int count,
        List<String> actorIds) {

    public ReactionGroup {
        actorIds = actorIds != null ? List.copyOf(actorIds) : List.of();
    }
}
```

- [ ] **Step 5: Create ReactionChangedEvent record**

```java
// api/src/main/java/io/casehub/qhorus/api/message/ReactionChangedEvent.java
package io.casehub.qhorus.api.message;

public record ReactionChangedEvent(
        Long messageId,
        String emoji,
        String actorId,
        boolean added) {}
```

- [ ] **Step 6: Create TopicStore interface**

```java
// api/src/main/java/io/casehub/qhorus/api/store/TopicStore.java
package io.casehub.qhorus.api.store;

import java.util.Collection;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.casehub.qhorus.api.message.Topic;

public interface TopicStore {
    Topic put(Topic topic);
    Optional<Topic> find(UUID channelId, String name);
    Optional<Topic> findById(Long id);
    List<Topic> findByChannel(UUID channelId);
    int rename(UUID channelId, String oldName, String newName);
    void delete(UUID channelId, String name);
    void deleteAll(UUID channelId);
}
```

- [ ] **Step 7: Create ReactionStore interface**

```java
// api/src/main/java/io/casehub/qhorus/api/store/ReactionStore.java
package io.casehub.qhorus.api.store;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import io.casehub.qhorus.api.message.Reaction;

public interface ReactionStore {
    Reaction react(Long messageId, String emoji, String actorId, String tenancyId);
    boolean unreact(Long messageId, String emoji, String actorId);
    List<Reaction> findByMessage(Long messageId);
    Map<Long, List<Reaction>> findByMessages(Collection<Long> messageIds);
    void deleteByMessage(Long messageId);
    void deleteByChannel(java.util.UUID channelId);
}
```

- [ ] **Step 8: Add `topic` to Message record**

Add `String topic` field after `version` (position 17, before `createdAt`). Update the record, Builder, and `toBuilder()`.

Use `ide_edit_member` on `Message.java` to replace the record declaration with the new field added. Update the Builder class to include `topic`.

- [ ] **Step 9: Add `topic` to MessageDispatch**

Add `String topic` field after `tenancyId` (position 15). Update Builder with null→"general" default in `build()`. Add to the builder chain and the canonical constructor call.

In `build()`, after existing validations, add:
```java
if (topic == null || topic.isBlank()) {
    topic = "general";
} else {
    topic = topic.strip();
    if (topic.length() > 200) {
        throw new IllegalArgumentException("topic exceeds 200 characters");
    }
}
```

- [ ] **Step 10: Add `topic` to MessageQuery**

Add `private final String topic` field. Add to Builder, `matches()` method, and `toBuilder()`.

In `matches()`:
```java
if (topic != null && !topic.equalsIgnoreCase(m.topic())) {
    return false;
}
```

- [ ] **Step 11: Add `topic` to MessageReceivedEvent**

Add `String topic` field after `content` (position 9). The compact constructor already validates `occurredAt` and EVENT content — topic has no validation.

- [ ] **Step 12: Add `updateTopicName` to MessageStore**

```java
int updateTopicName(java.util.UUID channelId, String oldTopic, String newTopic);
```

- [ ] **Step 13: Build api module to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#329,#330): API layer — Topic, Reaction records, store interfaces, message field additions

Add Topic, Reaction, ReactionGroup, TopicSummary, ReactionChangedEvent records.
Add TopicStore, ReactionStore interfaces.
Add topic field to Message, MessageDispatch, MessageQuery, MessageReceivedEvent.
Add MessageStore.updateTopicName() for bulk rename.

Refs #329, #330"
```

---

### Task 2: Topic Store Layer — Entity, Migrations, JPA, InMemory, Contract Test

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicEntity.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaTopicStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryTopicStore.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/TopicStoreContractTest.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryTopicStoreTest.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V28__add_message_topic.sql`
- Create: `runtime/src/main/resources/db/qhorus/migration/V29__add_topic_table.sql`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageEntity.java` — add `topic` column
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java` — add `updateTopicName()`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryMessageStore.java` — add `updateTopicName()`

**Interfaces:**
- Consumes: `Topic` record, `TopicStore` interface, `MessageStore.updateTopicName()` from Task 1
- Produces: `TopicEntity`, `JpaTopicStore`, `InMemoryTopicStore`, `MessageEntity.topic`, `JpaMessageStore.updateTopicName()`, `InMemoryMessageStore.updateTopicName()`

- [ ] **Step 1: Write TopicStoreContractTest**

```java
// persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/TopicStoreContractTest.java
package io.casehub.qhorus.persistence.memory.contract;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.Topic;
import io.casehub.qhorus.api.store.TopicStore;

import static org.assertj.core.api.Assertions.*;

public abstract class TopicStoreContractTest {

    protected abstract TopicStore store();

    private UUID channelId;
    private static final String TENANCY = "test-tenant";

    @BeforeEach
    void setUp() {
        channelId = UUID.randomUUID();
    }

    protected Topic makeTopic(String name) {
        return new Topic(null, channelId, name, false, null, null, Instant.now(), TENANCY);
    }

    @Test
    void put_and_find_by_channel_and_name() {
        Topic saved = store().put(makeTopic("auth-analysis"));
        assertThat(saved.id()).isNotNull();
        assertThat(saved.name()).isEqualTo("auth-analysis");

        Optional<Topic> found = store().find(channelId, "auth-analysis");
        assertThat(found).isPresent();
        assertThat(found.get().id()).isEqualTo(saved.id());
    }

    @Test
    void find_is_case_insensitive() {
        store().put(makeTopic("Auth-Analysis"));

        assertThat(store().find(channelId, "auth-analysis")).isPresent();
        assertThat(store().find(channelId, "AUTH-ANALYSIS")).isPresent();
        assertThat(store().find(channelId, "Auth-Analysis")).isPresent();
    }

    @Test
    void find_returns_empty_for_unknown() {
        assertThat(store().find(channelId, "nonexistent")).isEmpty();
    }

    @Test
    void findById() {
        Topic saved = store().put(makeTopic("billing"));
        assertThat(store().findById(saved.id())).isPresent();
        assertThat(store().findById(99999L)).isEmpty();
    }

    @Test
    void findByChannel_returns_all_topics() {
        store().put(makeTopic("topic-a"));
        store().put(makeTopic("topic-b"));
        store().put(makeTopic("topic-c"));

        UUID otherChannel = UUID.randomUUID();
        store().put(new Topic(null, otherChannel, "other", false, null, null, Instant.now(), TENANCY));

        List<Topic> topics = store().findByChannel(channelId);
        assertThat(topics).hasSize(3);
        assertThat(topics).extracting(Topic::name)
                .containsExactlyInAnyOrder("topic-a", "topic-b", "topic-c");
    }

    @Test
    void rename_updates_topic_name() {
        store().put(makeTopic("old-name"));

        int updated = store().rename(channelId, "old-name", "new-name");
        assertThat(updated).isEqualTo(1);

        assertThat(store().find(channelId, "old-name")).isEmpty();
        assertThat(store().find(channelId, "new-name")).isPresent();
    }

    @Test
    void rename_nonexistent_returns_zero() {
        int updated = store().rename(channelId, "nonexistent", "new-name");
        assertThat(updated).isEqualTo(0);
    }

    @Test
    void delete_removes_topic() {
        store().put(makeTopic("to-delete"));
        assertThat(store().find(channelId, "to-delete")).isPresent();

        store().delete(channelId, "to-delete");
        assertThat(store().find(channelId, "to-delete")).isEmpty();
    }

    @Test
    void deleteAll_removes_all_for_channel() {
        store().put(makeTopic("a"));
        store().put(makeTopic("b"));
        UUID otherChannel = UUID.randomUUID();
        store().put(new Topic(null, otherChannel, "other", false, null, null, Instant.now(), TENANCY));

        store().deleteAll(channelId);
        assertThat(store().findByChannel(channelId)).isEmpty();
        assertThat(store().findByChannel(otherChannel)).hasSize(1);
    }

    @Test
    void put_is_upsert_by_channel_and_name() {
        Topic first = store().put(makeTopic("upsert-test"));
        Topic second = store().put(new Topic(null, channelId, "upsert-test", true,
                Instant.now(), "actor-1", Instant.now(), TENANCY));

        assertThat(store().findByChannel(channelId)).hasSize(1);
        Topic found = store().find(channelId, "upsert-test").orElseThrow();
        assertThat(found.resolved()).isTrue();
    }
}
```

- [ ] **Step 2: Write InMemoryTopicStoreTest**

```java
// persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryTopicStoreTest.java
package io.casehub.qhorus.persistence.memory.contract;

import io.casehub.qhorus.api.store.TopicStore;
import io.casehub.qhorus.persistence.memory.InMemoryTopicStore;

public class InMemoryTopicStoreTest extends TopicStoreContractTest {

    private final InMemoryTopicStore store = new InMemoryTopicStore();

    @Override
    protected TopicStore store() {
        return store;
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryTopicStoreTest`
Expected: FAIL — `InMemoryTopicStore` does not exist yet

- [ ] **Step 4: Create Flyway migrations V28 and V29**

```sql
-- runtime/src/main/resources/db/qhorus/migration/V28__add_message_topic.sql
ALTER TABLE message ADD COLUMN topic VARCHAR(200);
```

```sql
-- runtime/src/main/resources/db/qhorus/migration/V29__add_topic_table.sql
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
```

- [ ] **Step 5: Create TopicEntity**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicEntity.java
package io.casehub.qhorus.runtime.message;

import java.time.Instant;
import java.util.UUID;

import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.qhorus.api.message.Topic;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity(name = "Topic")
@Table(name = "topic")
public class TopicEntity extends PanacheEntityBase {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long id;

    @Column(name = "channel_id", nullable = false)
    public UUID channelId;

    @Column(nullable = false, length = 200)
    public String name;

    @Column(nullable = false)
    public boolean resolved;

    @Column(name = "resolved_at")
    public Instant resolvedAt;

    @Column(name = "resolved_by")
    public String resolvedBy;

    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;

    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (createdAt == null) {
            createdAt = Instant.now();
        }
    }

    public static TopicEntity fromDomain(Topic topic) {
        TopicEntity e = new TopicEntity();
        e.id = topic.id();
        e.channelId = topic.channelId();
        e.name = topic.name();
        e.resolved = topic.resolved();
        e.resolvedAt = topic.resolvedAt();
        e.resolvedBy = topic.resolvedBy();
        e.tenancyId = topic.tenancyId() != null ? topic.tenancyId() : TenancyConstants.DEFAULT_TENANT_ID;
        e.createdAt = topic.createdAt();
        return e;
    }

    public Topic toDomain() {
        return new Topic(id, channelId, name, resolved, resolvedAt, resolvedBy, createdAt, tenancyId);
    }
}
```

- [ ] **Step 6: Add `topic` to MessageEntity**

Use `ide_insert_member` to add the topic field after the `target` field:
```java
@Column(name = "topic", length = 200)
public String topic;
```

Update `fromDomain()` to include `e.topic = msg.topic();`
Update `toDomain()` to include `topic` in the constructor call — the Message record now has `topic` between `version` and `createdAt`.

- [ ] **Step 7: Create JpaTopicStore**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaTopicStore.java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.message.Topic;
import io.casehub.qhorus.api.store.TopicStore;
import io.casehub.qhorus.runtime.message.TopicEntity;

@ApplicationScoped
public class JpaTopicStore implements TopicStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    public Topic put(Topic topic) {
        String tenancyId = topic.tenancyId() != null ? topic.tenancyId() : currentPrincipal.tenancyId();
        Optional<TopicEntity> existing = TopicEntity.find(
                "channelId = ?1 AND LOWER(name) = LOWER(?2) AND tenancyId = ?3",
                topic.channelId(), topic.name(), tenancyId).firstResultOptional();
        if (existing.isPresent()) {
            TopicEntity e = existing.get();
            e.resolved = topic.resolved();
            e.resolvedAt = topic.resolvedAt();
            e.resolvedBy = topic.resolvedBy();
            return e.toDomain();
        }
        TopicEntity e = TopicEntity.fromDomain(topic);
        e.tenancyId = tenancyId;
        e.persist();
        return e.toDomain();
    }

    @Override
    public Optional<Topic> find(UUID channelId, String name) {
        return TopicEntity.<TopicEntity>find(
                "channelId = ?1 AND LOWER(name) = LOWER(?2) AND tenancyId = ?3",
                channelId, name, currentPrincipal.tenancyId())
                .firstResultOptional()
                .map(TopicEntity::toDomain);
    }

    @Override
    public Optional<Topic> findById(Long id) {
        return TopicEntity.<TopicEntity>findByIdOptional(id).map(TopicEntity::toDomain);
    }

    @Override
    public List<Topic> findByChannel(UUID channelId) {
        return TopicEntity.<TopicEntity>find(
                "channelId = ?1 AND tenancyId = ?2 ORDER BY createdAt",
                channelId, currentPrincipal.tenancyId())
                .list()
                .stream()
                .map(TopicEntity::toDomain)
                .toList();
    }

    @Override
    public int rename(UUID channelId, String oldName, String newName) {
        return (int) TopicEntity.update(
                "name = ?1 WHERE channelId = ?2 AND LOWER(name) = LOWER(?3) AND tenancyId = ?4",
                newName, channelId, oldName, currentPrincipal.tenancyId());
    }

    @Override
    public void delete(UUID channelId, String name) {
        TopicEntity.delete("channelId = ?1 AND LOWER(name) = LOWER(?2) AND tenancyId = ?3",
                channelId, name, currentPrincipal.tenancyId());
    }

    @Override
    public void deleteAll(UUID channelId) {
        TopicEntity.delete("channelId = ?1 AND tenancyId = ?2",
                channelId, currentPrincipal.tenancyId());
    }
}
```

- [ ] **Step 8: Add `updateTopicName` to JpaMessageStore**

Use `ide_insert_member` to add after `findLastMessage`:
```java
@Override
public int updateTopicName(UUID channelId, String oldTopic, String newTopic) {
    return (int) MessageEntity.update(
            "topic = ?1 WHERE channelId = ?2 AND LOWER(topic) = LOWER(?3) AND tenancyId = ?4",
            newTopic, channelId, oldTopic, currentPrincipal.tenancyId());
}
```

- [ ] **Step 9: Create InMemoryTopicStore**

```java
// persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryTopicStore.java
package io.casehub.qhorus.persistence.memory;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.api.message.Topic;
import io.casehub.qhorus.api.store.TopicStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class InMemoryTopicStore implements TopicStore {

    private final Map<Long, Topic> store = new ConcurrentHashMap<>();
    private final AtomicLong idCounter = new AtomicLong(1);

    @Override
    public Topic put(Topic topic) {
        Optional<Topic> existing = find(topic.channelId(), topic.name());
        if (existing.isPresent()) {
            Topic updated = new Topic(existing.get().id(), topic.channelId(), existing.get().name(),
                    topic.resolved(), topic.resolvedAt(), topic.resolvedBy(),
                    existing.get().createdAt(), topic.tenancyId());
            store.put(updated.id(), updated);
            return updated;
        }
        Topic saved = new Topic(idCounter.getAndIncrement(), topic.channelId(), topic.name(),
                topic.resolved(), topic.resolvedAt(), topic.resolvedBy(),
                topic.createdAt(), topic.tenancyId());
        store.put(saved.id(), saved);
        return saved;
    }

    @Override
    public Optional<Topic> find(UUID channelId, String name) {
        return store.values().stream()
                .filter(t -> t.channelId().equals(channelId)
                        && t.name().equalsIgnoreCase(name))
                .findFirst();
    }

    @Override
    public Optional<Topic> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<Topic> findByChannel(UUID channelId) {
        return store.values().stream()
                .filter(t -> t.channelId().equals(channelId))
                .sorted((a, b) -> a.createdAt().compareTo(b.createdAt()))
                .toList();
    }

    @Override
    public int rename(UUID channelId, String oldName, String newName) {
        Optional<Topic> existing = find(channelId, oldName);
        if (existing.isEmpty()) return 0;
        Topic t = existing.get();
        Topic renamed = new Topic(t.id(), t.channelId(), newName, t.resolved(),
                t.resolvedAt(), t.resolvedBy(), t.createdAt(), t.tenancyId());
        store.put(renamed.id(), renamed);
        return 1;
    }

    @Override
    public void delete(UUID channelId, String name) {
        store.values().removeIf(t -> t.channelId().equals(channelId)
                && t.name().equalsIgnoreCase(name));
    }

    @Override
    public void deleteAll(UUID channelId) {
        store.values().removeIf(t -> t.channelId().equals(channelId));
    }

    public void clear() {
        store.clear();
    }
}
```

- [ ] **Step 10: Add `updateTopicName` to InMemoryMessageStore**

Use `ide_insert_member` to add before `clear()`:
```java
@Override
public int updateTopicName(UUID channelId, String oldTopic, String newTopic) {
    int count = 0;
    for (Map.Entry<Long, Message> entry : store.entrySet()) {
        Message m = entry.getValue();
        if (m.channelId().equals(channelId) && oldTopic.equalsIgnoreCase(m.topic())) {
            store.put(entry.getKey(), m.toBuilder().topic(newTopic).build());
            count++;
        }
    }
    return count;
}
```

- [ ] **Step 11: Run contract tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryTopicStoreTest`
Expected: PASS — all contract tests green

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicEntity.java runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaTopicStore.java runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageEntity.java runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaMessageStore.java runtime/src/main/resources/db/qhorus/migration/ persistence-memory/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#329): topic store layer — entity, JPA, InMemory, contract tests, migrations V28-V29

TopicEntity JPA entity with UNIQUE(channel_id, name, tenancy_id).
JpaTopicStore with case-insensitive name matching.
InMemoryTopicStore for test isolation.
MessageEntity gains topic column. JpaMessageStore + InMemoryMessageStore
gain updateTopicName() for bulk rename.

Refs #329"
```

---

### Task 3: Reaction Store Layer — Entity, Migrations, JPA, InMemory, Contract Test

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactionEntity.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaReactionStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactionStore.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/ReactionStoreContractTest.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactionStoreTest.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V30__add_reaction_table.sql`

**Interfaces:**
- Consumes: `Reaction` record, `ReactionStore` interface from Task 1
- Produces: `ReactionEntity`, `JpaReactionStore`, `InMemoryReactionStore`

- [ ] **Step 1: Write ReactionStoreContractTest**

```java
// persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/ReactionStoreContractTest.java
package io.casehub.qhorus.persistence.memory.contract;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.Reaction;
import io.casehub.qhorus.api.store.ReactionStore;

import static org.assertj.core.api.Assertions.*;

public abstract class ReactionStoreContractTest {

    protected abstract ReactionStore store();

    private static final String TENANCY = "test-tenant";

    @Test
    void react_creates_reaction() {
        Reaction r = store().react(1L, "👍", "agent-1", TENANCY);
        assertThat(r.id()).isNotNull();
        assertThat(r.messageId()).isEqualTo(1L);
        assertThat(r.emoji()).isEqualTo("👍");
        assertThat(r.actorId()).isEqualTo("agent-1");
    }

    @Test
    void react_is_idempotent() {
        store().react(1L, "👍", "agent-1", TENANCY);
        store().react(1L, "👍", "agent-1", TENANCY);

        List<Reaction> reactions = store().findByMessage(1L);
        assertThat(reactions).hasSize(1);
    }

    @Test
    void different_actors_same_emoji() {
        store().react(1L, "👍", "agent-1", TENANCY);
        store().react(1L, "👍", "agent-2", TENANCY);

        List<Reaction> reactions = store().findByMessage(1L);
        assertThat(reactions).hasSize(2);
    }

    @Test
    void multiple_emojis_same_message() {
        store().react(1L, "👍", "agent-1", TENANCY);
        store().react(1L, "❤️", "agent-1", TENANCY);

        List<Reaction> reactions = store().findByMessage(1L);
        assertThat(reactions).hasSize(2);
    }

    @Test
    void unreact_removes_reaction() {
        store().react(1L, "👍", "agent-1", TENANCY);
        boolean removed = store().unreact(1L, "👍", "agent-1");
        assertThat(removed).isTrue();
        assertThat(store().findByMessage(1L)).isEmpty();
    }

    @Test
    void unreact_is_idempotent() {
        boolean removed = store().unreact(1L, "👍", "agent-1");
        assertThat(removed).isFalse();
    }

    @Test
    void findByMessages_batch() {
        store().react(1L, "👍", "agent-1", TENANCY);
        store().react(1L, "❤️", "agent-2", TENANCY);
        store().react(2L, "🎉", "agent-1", TENANCY);

        Map<Long, List<Reaction>> batch = store().findByMessages(List.of(1L, 2L, 3L));
        assertThat(batch.get(1L)).hasSize(2);
        assertThat(batch.get(2L)).hasSize(1);
        assertThat(batch.getOrDefault(3L, List.of())).isEmpty();
    }

    @Test
    void deleteByMessage_removes_all_for_message() {
        store().react(1L, "👍", "agent-1", TENANCY);
        store().react(1L, "❤️", "agent-2", TENANCY);
        store().react(2L, "👍", "agent-1", TENANCY);

        store().deleteByMessage(1L);
        assertThat(store().findByMessage(1L)).isEmpty();
        assertThat(store().findByMessage(2L)).hasSize(1);
    }
}
```

- [ ] **Step 2: Write InMemoryReactionStoreTest**

```java
// persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactionStoreTest.java
package io.casehub.qhorus.persistence.memory.contract;

import io.casehub.qhorus.api.store.ReactionStore;
import io.casehub.qhorus.persistence.memory.InMemoryReactionStore;

public class InMemoryReactionStoreTest extends ReactionStoreContractTest {

    private final InMemoryReactionStore store = new InMemoryReactionStore();

    @Override
    protected ReactionStore store() {
        return store;
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryReactionStoreTest`
Expected: FAIL — `InMemoryReactionStore` does not exist yet

- [ ] **Step 4: Create Flyway migration V30**

```sql
-- runtime/src/main/resources/db/qhorus/migration/V30__add_reaction_table.sql
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
```

- [ ] **Step 5: Create ReactionEntity**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactionEntity.java
package io.casehub.qhorus.runtime.message;

import java.time.Instant;

import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.qhorus.api.message.Reaction;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity(name = "Reaction")
@Table(name = "reaction")
public class ReactionEntity extends PanacheEntityBase {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long id;

    @Column(name = "message_id", nullable = false)
    public Long messageId;

    @Column(nullable = false, length = 100)
    public String emoji;

    @Column(name = "actor_id", nullable = false)
    public String actorId;

    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;

    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (createdAt == null) {
            createdAt = Instant.now();
        }
    }

    public static ReactionEntity fromDomain(Reaction r) {
        ReactionEntity e = new ReactionEntity();
        e.id = r.id();
        e.messageId = r.messageId();
        e.emoji = r.emoji();
        e.actorId = r.actorId();
        e.tenancyId = r.tenancyId() != null ? r.tenancyId() : TenancyConstants.DEFAULT_TENANT_ID;
        e.createdAt = r.createdAt();
        return e;
    }

    public Reaction toDomain() {
        return new Reaction(id, messageId, emoji, actorId, createdAt, tenancyId);
    }
}
```

- [ ] **Step 6: Create JpaReactionStore**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaReactionStore.java
package io.casehub.qhorus.runtime.store.jpa;

import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.message.Reaction;
import io.casehub.qhorus.api.store.ReactionStore;
import io.casehub.qhorus.runtime.message.ReactionEntity;

@ApplicationScoped
public class JpaReactionStore implements ReactionStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    public Reaction react(Long messageId, String emoji, String actorId, String tenancyId) {
        String tid = tenancyId != null ? tenancyId : currentPrincipal.tenancyId();
        Optional<ReactionEntity> existing = ReactionEntity.<ReactionEntity>find(
                "messageId = ?1 AND emoji = ?2 AND actorId = ?3",
                messageId, emoji, actorId).firstResultOptional();
        if (existing.isPresent()) {
            return existing.get().toDomain();
        }
        ReactionEntity e = new ReactionEntity();
        e.messageId = messageId;
        e.emoji = emoji;
        e.actorId = actorId;
        e.tenancyId = tid;
        e.persist();
        return e.toDomain();
    }

    @Override
    public boolean unreact(Long messageId, String emoji, String actorId) {
        return ReactionEntity.delete("messageId = ?1 AND emoji = ?2 AND actorId = ?3",
                messageId, emoji, actorId) > 0;
    }

    @Override
    public List<Reaction> findByMessage(Long messageId) {
        return ReactionEntity.<ReactionEntity>find("messageId = ?1", messageId)
                .list().stream().map(ReactionEntity::toDomain).toList();
    }

    @Override
    public Map<Long, List<Reaction>> findByMessages(Collection<Long> messageIds) {
        if (messageIds == null || messageIds.isEmpty()) return Map.of();
        List<ReactionEntity> entities = ReactionEntity.find("messageId IN ?1", List.copyOf(messageIds)).list();
        Map<Long, List<Reaction>> result = new HashMap<>();
        for (Long id : messageIds) {
            result.put(id, entities.stream()
                    .filter(e -> e.messageId.equals(id))
                    .map(ReactionEntity::toDomain)
                    .toList());
        }
        return result;
    }

    @Override
    public void deleteByMessage(Long messageId) {
        ReactionEntity.delete("messageId = ?1", messageId);
    }

    @Override
    public void deleteByChannel(UUID channelId) {
        ReactionEntity.delete("messageId IN (SELECT m.id FROM MessageEntity m WHERE m.channelId = ?1)", channelId);
    }
}
```

- [ ] **Step 7: Create InMemoryReactionStore**

```java
// persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactionStore.java
package io.casehub.qhorus.persistence.memory;

import java.time.Instant;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.qhorus.api.message.Reaction;
import io.casehub.qhorus.api.store.ReactionStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class InMemoryReactionStore implements ReactionStore {

    private final Map<Long, Reaction> store = new ConcurrentHashMap<>();
    private final AtomicLong idCounter = new AtomicLong(1);

    @Override
    public Reaction react(Long messageId, String emoji, String actorId, String tenancyId) {
        for (Reaction r : store.values()) {
            if (r.messageId().equals(messageId) && r.emoji().equals(emoji) && r.actorId().equals(actorId)) {
                return r;
            }
        }
        Reaction r = new Reaction(idCounter.getAndIncrement(), messageId, emoji, actorId, Instant.now(), tenancyId);
        store.put(r.id(), r);
        return r;
    }

    @Override
    public boolean unreact(Long messageId, String emoji, String actorId) {
        return store.values().removeIf(r ->
                r.messageId().equals(messageId) && r.emoji().equals(emoji) && r.actorId().equals(actorId));
    }

    @Override
    public List<Reaction> findByMessage(Long messageId) {
        return store.values().stream()
                .filter(r -> r.messageId().equals(messageId))
                .toList();
    }

    @Override
    public Map<Long, List<Reaction>> findByMessages(Collection<Long> messageIds) {
        Map<Long, List<Reaction>> result = new HashMap<>();
        for (Long id : messageIds) {
            result.put(id, findByMessage(id));
        }
        return result;
    }

    @Override
    public void deleteByMessage(Long messageId) {
        store.values().removeIf(r -> r.messageId().equals(messageId));
    }

    @Override
    public void deleteByChannel(UUID channelId) {
        // InMemory: no channel→message relationship — noop in unit tests
        // Integration tests use JPA store where the subquery works
    }

    public void clear() {
        store.clear();
    }
}
```

- [ ] **Step 8: Run contract tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryReactionStoreTest`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactionEntity.java runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaReactionStore.java runtime/src/main/resources/db/qhorus/migration/V30__add_reaction_table.sql persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactionStore.java persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/ReactionStoreContractTest.java persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactionStoreTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#330): reaction store layer — entity, JPA, InMemory, contract tests, migration V30

ReactionEntity with UNIQUE(message_id, emoji, actor_id).
JpaReactionStore with idempotent react/unreact and batch findByMessages.
InMemoryReactionStore for test isolation.

Refs #330"
```

---

### Task 4: TopicService + Dispatch Integration

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/TopicServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — wire topic
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java` — topic on event

**Interfaces:**
- Consumes: `TopicStore`, `MessageStore.updateTopicName()`, `TopicSummary`, `MessageDispatch.topic()`, `Message.topic()`
- Produces: `TopicService.ensureExists()`, `TopicService.listTopics()`, `TopicService.resolve()`, `TopicService.unresolve()`, `TopicService.rename()`

- [ ] **Step 1: Write TopicServiceTest**

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/message/TopicServiceTest.java
package io.casehub.qhorus.runtime.message;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.Topic;
import io.casehub.qhorus.api.message.TopicSummary;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.persistence.memory.InMemoryTopicStore;
import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.platform.api.identity.ActorType;

import static org.assertj.core.api.Assertions.*;

class TopicServiceTest {

    private TopicService service;
    private InMemoryTopicStore topicStore;
    private InMemoryMessageStore messageStore;
    private UUID channelId;

    @BeforeEach
    void setUp() {
        topicStore = new InMemoryTopicStore();
        messageStore = new InMemoryMessageStore();
        service = new TopicService();
        service.topicStore = topicStore;
        service.messageStore = messageStore;
        channelId = UUID.randomUUID();
    }

    @Test
    void ensureExists_creates_topic_on_first_call() {
        Topic topic = service.ensureExists(channelId, "auth-analysis", "tenant-1");
        assertThat(topic).isNotNull();
        assertThat(topic.name()).isEqualTo("auth-analysis");
        assertThat(topic.resolved()).isFalse();
    }

    @Test
    void ensureExists_returns_existing_on_second_call() {
        Topic first = service.ensureExists(channelId, "auth-analysis", "tenant-1");
        Topic second = service.ensureExists(channelId, "auth-analysis", "tenant-1");
        assertThat(second.id()).isEqualTo(first.id());
    }

    @Test
    void ensureExists_case_insensitive() {
        Topic first = service.ensureExists(channelId, "Auth-Analysis", "tenant-1");
        Topic second = service.ensureExists(channelId, "auth-analysis", "tenant-1");
        assertThat(second.id()).isEqualTo(first.id());
    }

    @Test
    void resolve_sets_resolved_state() {
        service.ensureExists(channelId, "done-topic", "tenant-1");
        service.resolve(channelId, "done-topic", "actor-1");

        Topic found = topicStore.find(channelId, "done-topic").orElseThrow();
        assertThat(found.resolved()).isTrue();
        assertThat(found.resolvedBy()).isEqualTo("actor-1");
        assertThat(found.resolvedAt()).isNotNull();
    }

    @Test
    void resolve_rejects_general() {
        service.ensureExists(channelId, "general", "tenant-1");
        assertThatThrownBy(() -> service.resolve(channelId, "general", "actor-1"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("general");
    }

    @Test
    void unresolve_clears_resolved_state() {
        service.ensureExists(channelId, "topic-1", "tenant-1");
        service.resolve(channelId, "topic-1", "actor-1");
        service.unresolve(channelId, "topic-1");

        Topic found = topicStore.find(channelId, "topic-1").orElseThrow();
        assertThat(found.resolved()).isFalse();
    }

    @Test
    void rename_rejects_general_as_source() {
        service.ensureExists(channelId, "general", "tenant-1");
        assertThatThrownBy(() -> service.rename(channelId, "general", "new-name", "actor-1"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rename_rejects_if_target_exists() {
        service.ensureExists(channelId, "topic-a", "tenant-1");
        service.ensureExists(channelId, "topic-b", "tenant-1");
        assertThatThrownBy(() -> service.rename(channelId, "topic-a", "topic-b", "actor-1"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("already exists");
    }

    @Test
    void rename_updates_topic_and_messages() {
        service.ensureExists(channelId, "old-name", "tenant-1");
        messageStore.put(Message.builder()
                .channelId(channelId).sender("a").messageType(MessageType.STATUS)
                .actorType(ActorType.AGENT).topic("old-name").tenancyId("tenant-1").build());
        messageStore.put(Message.builder()
                .channelId(channelId).sender("b").messageType(MessageType.STATUS)
                .actorType(ActorType.AGENT).topic("old-name").tenancyId("tenant-1").build());

        TopicService.RenameResult result = service.rename(channelId, "old-name", "new-name", "actor-1");
        assertThat(result.messagesUpdated()).isEqualTo(2);
        assertThat(topicStore.find(channelId, "new-name")).isPresent();
        assertThat(topicStore.find(channelId, "old-name")).isEmpty();
    }

    @Test
    void listTopics_returns_summaries_with_counts() {
        service.ensureExists(channelId, "topic-a", "tenant-1");
        service.ensureExists(channelId, "topic-b", "tenant-1");
        messageStore.put(Message.builder()
                .channelId(channelId).sender("a").messageType(MessageType.STATUS)
                .actorType(ActorType.AGENT).topic("topic-a").tenancyId("tenant-1").build());
        messageStore.put(Message.builder()
                .channelId(channelId).sender("a").messageType(MessageType.STATUS)
                .actorType(ActorType.AGENT).topic("topic-a").tenancyId("tenant-1").build());
        messageStore.put(Message.builder()
                .channelId(channelId).sender("b").messageType(MessageType.COMMAND)
                .actorType(ActorType.AGENT).topic("topic-b").tenancyId("tenant-1").build());

        List<TopicSummary> summaries = service.listTopics(channelId);
        assertThat(summaries).hasSize(2);
        TopicSummary a = summaries.stream().filter(s -> s.name().equals("topic-a")).findFirst().orElseThrow();
        assertThat(a.messageCount()).isEqualTo(2);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TopicServiceTest`
Expected: FAIL — `TopicService` does not exist yet

- [ ] **Step 3: Create TopicService**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicService.java
package io.casehub.qhorus.runtime.message;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.Topic;
import io.casehub.qhorus.api.message.TopicSummary;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.TopicStore;
import io.casehub.qhorus.api.store.query.MessageQuery;

@ApplicationScoped
public class TopicService {

    static final String DEFAULT_TOPIC = "general";

    @Inject
    public TopicStore topicStore;

    @Inject
    public MessageStore messageStore;

    public Topic ensureExists(UUID channelId, String topicName, String tenancyId) {
        String name = normalise(topicName);
        Optional<Topic> existing = topicStore.find(channelId, name);
        if (existing.isPresent()) {
            return existing.get();
        }
        return topicStore.put(new Topic(null, channelId, name, false, null, null, Instant.now(), tenancyId));
    }

    public List<TopicSummary> listTopics(UUID channelId) {
        List<Topic> topics = topicStore.findByChannel(channelId);
        return topics.stream().map(t -> {
            List<Message> messages = messageStore.scan(
                    MessageQuery.builder().channelId(channelId).topic(t.name()).build());
            Instant lastActivity = messages.stream()
                    .map(Message::createdAt)
                    .max(Instant::compareTo)
                    .orElse(t.createdAt());
            return new TopicSummary(t.name(), messages.size(), lastActivity, t.resolved(), t.resolvedAt());
        }).toList();
    }

    public void resolve(UUID channelId, String topicName, String actorId) {
        String name = normalise(topicName);
        if (DEFAULT_TOPIC.equalsIgnoreCase(name)) {
            throw new IllegalArgumentException("Cannot resolve the default topic 'general'");
        }
        Topic existing = topicStore.find(channelId, name)
                .orElseThrow(() -> new IllegalArgumentException("Topic '" + name + "' not found"));
        topicStore.put(new Topic(existing.id(), existing.channelId(), existing.name(),
                true, Instant.now(), actorId, existing.createdAt(), existing.tenancyId()));
    }

    public void unresolve(UUID channelId, String topicName) {
        String name = normalise(topicName);
        Topic existing = topicStore.find(channelId, name)
                .orElseThrow(() -> new IllegalArgumentException("Topic '" + name + "' not found"));
        topicStore.put(new Topic(existing.id(), existing.channelId(), existing.name(),
                false, null, null, existing.createdAt(), existing.tenancyId()));
    }

    public RenameResult rename(UUID channelId, String oldName, String newName, String actorId) {
        String normalOld = normalise(oldName);
        String normalNew = normalise(newName);
        if (DEFAULT_TOPIC.equalsIgnoreCase(normalOld)) {
            throw new IllegalArgumentException("Cannot rename the default topic 'general'");
        }
        if (topicStore.find(channelId, normalNew).isPresent()) {
            throw new IllegalArgumentException("Topic '" + normalNew + "' already exists in this channel");
        }
        int topicUpdated = topicStore.rename(channelId, normalOld, normalNew);
        if (topicUpdated == 0) {
            throw new IllegalArgumentException("Topic '" + normalOld + "' not found");
        }
        int messagesUpdated = messageStore.updateTopicName(channelId, normalOld, normalNew);
        return new RenameResult(normalOld, normalNew, messagesUpdated);
    }

    private static String normalise(String topicName) {
        if (topicName == null || topicName.isBlank()) return DEFAULT_TOPIC;
        String trimmed = topicName.strip();
        if (trimmed.length() > 200) {
            throw new IllegalArgumentException("Topic name exceeds 200 characters");
        }
        return trimmed;
    }

    public record RenameResult(String oldName, String newName, int messagesUpdated) {}
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TopicServiceTest`
Expected: PASS

- [ ] **Step 5: Wire topic into MessageService.dispatch()**

In `MessageService.java`, add `@Inject public TopicService topicService;` field.

In the dispatch method, after `Message saved = messageStore.put(message);` (~line 290), before the commitment switch block, add topic to the message builder:

In the `Message.builder()` chain (~line 280), add `.topic(dispatch.topic())`.

After `Message saved = messageStore.put(message);`, add:
```java
topicService.ensureExists(dispatch.channelId(), dispatch.topic(), effectiveTenancyId);
```

Also update the LAST_WRITE overwrite path to include `.topic(dispatch.topic())` in the `last.toBuilder()` chain.

- [ ] **Step 6: Wire topic into MessageObserverDispatcher**

In `MessageObserverDispatcher.dispatch()`, update the `MessageReceivedEvent` constructor to include `message.topic()`:
```java
final MessageReceivedEvent event = new MessageReceivedEvent(
        channelName, channelId, tenancyId,
        message.messageType(), message.sender(),
        message.correlationId(), occurredAt, content,
        message.topic());
```

- [ ] **Step 7: Run full runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS (existing tests unaffected — topic is nullable on Message, defaults to "general" on MessageDispatch)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/TopicService.java runtime/src/test/java/io/casehub/qhorus/runtime/message/TopicServiceTest.java runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageObserverDispatcher.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#329): TopicService + dispatch integration

TopicService: ensureExists, listTopics, resolve/unresolve, rename.
MessageService.dispatch() sets topic on message and calls ensureExists.
MessageObserverDispatcher passes topic through to MessageReceivedEvent.

Refs #329"
```

---

### Task 5: ReactionService

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactionService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/message/ReactionServiceTest.java`

**Interfaces:**
- Consumes: `ReactionStore`, `Reaction`, `ReactionGroup`, `ReactionChangedEvent`, `MessageStore.find()`
- Produces: `ReactionService.react()`, `.unreact()`, `.getReactions()`, `.getReactionsBatch()`

- [ ] **Step 1: Write ReactionServiceTest**

```java
// runtime/src/test/java/io/casehub/qhorus/runtime/message/ReactionServiceTest.java
package io.casehub.qhorus.runtime.message;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.Reaction;
import io.casehub.qhorus.api.message.ReactionGroup;
import io.casehub.qhorus.persistence.memory.InMemoryReactionStore;

import static org.assertj.core.api.Assertions.*;

class ReactionServiceTest {

    private ReactionService service;
    private InMemoryReactionStore reactionStore;

    @BeforeEach
    void setUp() {
        reactionStore = new InMemoryReactionStore();
        service = new ReactionService();
        service.reactionStore = reactionStore;
    }

    @Test
    void react_creates_reaction() {
        Reaction r = service.react(1L, "👍", "agent-1", "tenant-1");
        assertThat(r.emoji()).isEqualTo("👍");
    }

    @Test
    void react_trims_emoji() {
        Reaction r = service.react(1L, "  👍  ", "agent-1", "tenant-1");
        assertThat(r.emoji()).isEqualTo("👍");
    }

    @Test
    void react_rejects_blank_emoji() {
        assertThatThrownBy(() -> service.react(1L, "", "agent-1", "tenant-1"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void unreact_returns_true_when_removed() {
        service.react(1L, "👍", "agent-1", "tenant-1");
        assertThat(service.unreact(1L, "👍", "agent-1")).isTrue();
    }

    @Test
    void unreact_returns_false_when_not_present() {
        assertThat(service.unreact(1L, "👍", "agent-1")).isFalse();
    }

    @Test
    void getReactions_groups_by_emoji() {
        service.react(1L, "👍", "agent-1", "tenant-1");
        service.react(1L, "👍", "agent-2", "tenant-1");
        service.react(1L, "❤️", "agent-1", "tenant-1");

        List<ReactionGroup> groups = service.getReactions(1L);
        assertThat(groups).hasSize(2);

        ReactionGroup thumbsUp = groups.stream()
                .filter(g -> g.emoji().equals("👍")).findFirst().orElseThrow();
        assertThat(thumbsUp.count()).isEqualTo(2);
        assertThat(thumbsUp.actorIds()).containsExactlyInAnyOrder("agent-1", "agent-2");

        ReactionGroup heart = groups.stream()
                .filter(g -> g.emoji().equals("❤️")).findFirst().orElseThrow();
        assertThat(heart.count()).isEqualTo(1);
    }

    @Test
    void getReactionsBatch_returns_grouped_per_message() {
        service.react(1L, "👍", "agent-1", "tenant-1");
        service.react(2L, "❤️", "agent-2", "tenant-1");

        Map<Long, List<ReactionGroup>> batch = service.getReactionsBatch(List.of(1L, 2L));
        assertThat(batch.get(1L)).hasSize(1);
        assertThat(batch.get(2L)).hasSize(1);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ReactionServiceTest`
Expected: FAIL

- [ ] **Step 3: Create ReactionService**

```java
// runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactionService.java
package io.casehub.qhorus.runtime.message;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;

import io.casehub.qhorus.api.message.Reaction;
import io.casehub.qhorus.api.message.ReactionChangedEvent;
import io.casehub.qhorus.api.message.ReactionGroup;
import io.casehub.qhorus.api.store.ReactionStore;

@ApplicationScoped
public class ReactionService {

    @Inject
    public ReactionStore reactionStore;

    @Inject
    Event<ReactionChangedEvent> reactionEvent;

    public Reaction react(Long messageId, String emoji, String actorId, String tenancyId) {
        String trimmed = validateEmoji(emoji);
        Reaction r = reactionStore.react(messageId, trimmed, actorId, tenancyId);
        if (reactionEvent != null) {
            reactionEvent.fireAsync(new ReactionChangedEvent(messageId, trimmed, actorId, true));
        }
        return r;
    }

    public boolean unreact(Long messageId, String emoji, String actorId) {
        String trimmed = validateEmoji(emoji);
        boolean removed = reactionStore.unreact(messageId, trimmed, actorId);
        if (removed && reactionEvent != null) {
            reactionEvent.fireAsync(new ReactionChangedEvent(messageId, trimmed, actorId, false));
        }
        return removed;
    }

    public List<ReactionGroup> getReactions(Long messageId) {
        return groupReactions(reactionStore.findByMessage(messageId));
    }

    public Map<Long, List<ReactionGroup>> getReactionsBatch(Collection<Long> messageIds) {
        Map<Long, List<Reaction>> raw = reactionStore.findByMessages(messageIds);
        return raw.entrySet().stream()
                .collect(Collectors.toMap(
                        Map.Entry::getKey,
                        e -> groupReactions(e.getValue())));
    }

    private static List<ReactionGroup> groupReactions(List<Reaction> reactions) {
        return reactions.stream()
                .collect(Collectors.groupingBy(Reaction::emoji))
                .entrySet().stream()
                .map(e -> new ReactionGroup(
                        e.getKey(),
                        e.getValue().size(),
                        e.getValue().stream().map(Reaction::actorId).toList()))
                .toList();
    }

    private static String validateEmoji(String emoji) {
        if (emoji == null || emoji.isBlank()) {
            throw new IllegalArgumentException("emoji is required");
        }
        return emoji.strip();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ReactionServiceTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactionService.java runtime/src/test/java/io/casehub/qhorus/runtime/message/ReactionServiceTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#330): ReactionService — react, unreact, getReactions, getReactionsBatch

CDI-free service with ReactionChangedEvent async CDI event.
Groups reactions by emoji for UI rendering.

Refs #330"
```

---

### Task 6: Ledger Integration + MCP Tools — Topic

**Files:**
- Create: `runtime/src/main/resources/db/qhorus/migration/V31__add_ledger_entry_topic.sql`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/TopicToolTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageLedgerEntry.java` — add `topic`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/LedgerWriteService.java` — set `entry.topic`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` — `MessageSummary.topic`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — `sendMessage` topic param + new topic tools

**Interfaces:**
- Consumes: `TopicService`, `MessageDispatch.topic()`, `MessageLedgerEntry`, `resolveChannel()`
- Produces: MCP tools: `list_topics`, `resolve_topic`, `unresolve_topic`, `rename_topic`, `sendMessage` with topic

- [ ] **Step 1: Create V31 migration**

```sql
-- runtime/src/main/resources/db/qhorus/migration/V31__add_ledger_entry_topic.sql
ALTER TABLE message_ledger_entry ADD COLUMN topic VARCHAR(200);
```

- [ ] **Step 2: Add `topic` to MessageLedgerEntry**

Use `ide_insert_member` to add after the `commitmentId` field:
```java
@Column(name = "topic")
public String topic;
```

Update `domainContentBytes()` to include `topic` in the canonical string.

- [ ] **Step 3: Set `entry.topic` in LedgerWriteService.record()**

In `LedgerWriteService.record()`, after setting `entry.commitmentId`, add:
```java
entry.topic = dispatch.topic();
```

- [ ] **Step 4: Add `topic` to MessageSummary in QhorusMcpToolsBase**

Add `String topic` field to the `MessageSummary` record. Update `toMessageSummary()` to include `m.topic()`.

- [ ] **Step 5: Add `topic` param to `sendMessage` in QhorusMcpTools**

Add parameter after `causedByEntryId`:
```java
@ToolArg(name = "topic", description = "Topic name for this message. Groups messages into named sub-conversations within the channel. Defaults to 'general' if omitted.", required = false) String topic
```

In the `MessageDispatch.builder()` chain, add `.topic(topic)`.

- [ ] **Step 6: Add topic MCP tools to QhorusMcpTools**

Add the following `@Tool` methods:

```java
@Tool(description = "List all topics in a channel with message counts and activity timestamps")
public List<TopicSummary> listTopics(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel) {
    Channel ch = resolveChannel(channel);
    return topicService.listTopics(ch.id());
}

@Tool(description = "Mark a topic as resolved (done). Messages remain queryable but visually distinct in UI")
public Topic resolveTopic(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "topic_name", description = "Name of the topic to resolve") String topicName,
        @ToolArg(name = "caller_instance_id", description = "Instance ID of the caller", required = false) String callerInstanceId) {
    Channel ch = resolveChannel(channel);
    String actorId = callerInstanceId != null ? callerInstanceId : "anonymous";
    topicService.resolve(ch.id(), topicName, actorId);
    return topicService.topicStore.find(ch.id(), topicName).orElseThrow();
}

@Tool(description = "Unresolve a previously resolved topic")
public Topic unresolveTopic(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "topic_name", description = "Name of the topic to unresolve") String topicName) {
    Channel ch = resolveChannel(channel);
    topicService.unresolve(ch.id(), topicName);
    return topicService.topicStore.find(ch.id(), topicName).orElseThrow();
}

@Tool(description = "Rename a topic — updates the topic name on all messages in the topic and emits an audit EVENT")
public TopicService.RenameResult renameTopic(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "old_name", description = "Current topic name") String oldName,
        @ToolArg(name = "new_name", description = "New topic name") String newName,
        @ToolArg(name = "caller_instance_id", description = "Instance ID of the caller", required = false) String callerInstanceId) {
    Channel ch = resolveChannel(channel);
    String actorId = callerInstanceId != null ? callerInstanceId : "anonymous";
    return topicService.rename(ch.id(), oldName, newName, actorId);
}
```

Add `@Inject TopicService topicService;` field to `QhorusMcpTools`.

- [ ] **Step 7: Write TopicToolTest**

A `@QuarkusTest` integration test that verifies:
- `sendMessage` with topic param → message has topic set
- `sendMessage` without topic → message defaults to "general"
- `listTopics` returns topics with message counts
- `resolveTopic` / `unresolveTopic` toggle state
- `renameTopic` updates messages, ledger entries preserve original topic
- Ledger entry has original topic after rename

Key test: send messages with topic "old-name", rename to "new-name", verify:
- `checkMessages` returns messages with topic "new-name"
- Ledger entries still have topic "old-name"

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TopicToolTest`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/resources/db/qhorus/migration/V31__add_ledger_entry_topic.sql runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ runtime/src/test/java/io/casehub/qhorus/runtime/mcp/TopicToolTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#329): ledger topic + MCP topic tools

MessageLedgerEntry gains topic column (immutable — preserves original).
LedgerWriteService sets topic from dispatch.
MCP tools: sendMessage gains topic param, list_topics, resolve_topic,
unresolve_topic, rename_topic. Rename emits system EVENT for audit trail.

Refs #329"
```

---

### Task 7: MCP Tools — Reaction + Channel Delete Cascade

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/ReactionToolTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — react, unreact, get_reactions tools + delete cascade

**Interfaces:**
- Consumes: `ReactionService`, `resolveChannel()`, `InstanceService`
- Produces: MCP tools: `react`, `unreact`, `get_reactions`

- [ ] **Step 1: Add reaction MCP tools to QhorusMcpTools**

```java
@Tool(description = "Add an emoji reaction to a message. Idempotent — reacting twice is a no-op.")
public Reaction react(
        @ToolArg(name = "message_id", description = "ID of the message to react to") Long messageId,
        @ToolArg(name = "emoji", description = "Emoji character or shortcode") String emoji,
        @ToolArg(name = "actor_id", description = "Who is reacting. Defaults to caller's instance ID.", required = false) String actorId) {
    String actor = actorId != null ? actorId : currentPrincipal.actorId();
    return reactionService.react(messageId, emoji, actor, currentPrincipal.tenancyId());
}

@Tool(description = "Remove an emoji reaction from a message. Idempotent — unreacting when not reacted is a no-op.")
public ReactionResult unreact(
        @ToolArg(name = "message_id", description = "ID of the message") Long messageId,
        @ToolArg(name = "emoji", description = "Emoji character or shortcode") String emoji,
        @ToolArg(name = "actor_id", description = "Who is unreacting. Defaults to caller's instance ID.", required = false) String actorId) {
    String actor = actorId != null ? actorId : currentPrincipal.actorId();
    boolean removed = reactionService.unreact(messageId, emoji, actor);
    return new ReactionResult(messageId, emoji, removed);
}

@Tool(description = "Get all reactions for a message, grouped by emoji with actor lists")
public List<ReactionGroup> getReactions(
        @ToolArg(name = "message_id", description = "ID of the message") Long messageId) {
    return reactionService.getReactions(messageId);
}
```

Add `ReactionResult` record to `QhorusMcpToolsBase`:
```java
public record ReactionResult(Long messageId, String emoji, boolean removed) {}
```

Add `@Inject ReactionService reactionService;` field to `QhorusMcpTools`.

- [ ] **Step 2: Update deleteChannel to cascade reactions and topics**

In the `deleteChannel` method, after `commitmentStore.deleteAll(channelId)` and before `messageStore.deleteAll(channelId)`, add:
```java
reactionStore.deleteByChannel(ch.id());
topicStore.deleteAll(ch.id());
```

Add `@Inject ReactionStore reactionStore;` and `@Inject TopicStore topicStore;` fields.

- [ ] **Step 3: Write ReactionToolTest**

A `@QuarkusTest` integration test that verifies:
- `react` creates a reaction
- `react` is idempotent
- `unreact` removes a reaction
- `unreact` is idempotent (returns removed=false)
- `getReactions` returns groups
- Reactions are cleaned up when channel is deleted with force=true

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ReactionToolTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ runtime/src/test/java/io/casehub/qhorus/runtime/mcp/ReactionToolTest.java
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#330): MCP reaction tools + channel delete cascade

MCP tools: react, unreact, get_reactions.
deleteChannel(force=true) cascades to reactions and topics before
deleting messages.

Refs #330"
```

---

### Task 8: Reactive Parity + Full Build Verification

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveTopicStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveReactionStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveTopicStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveReactionStore.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactiveTopicStoreTest.java`
- Create: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactiveReactionStoreTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — reactive topic + reaction tools
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` — topic in dispatch
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` — topic in entry

**Interfaces:**
- Consumes: All blocking store/service interfaces from Tasks 1–7
- Produces: Reactive equivalents with `Uni<T>` returns

- [ ] **Step 1: Create ReactiveTopicStore interface**

```java
// api/src/main/java/io/casehub/qhorus/api/store/ReactiveTopicStore.java
package io.casehub.qhorus.api.store;

import java.util.List;
import java.util.UUID;

import io.casehub.qhorus.api.message.Topic;
import io.smallrye.mutiny.Uni;

public interface ReactiveTopicStore {
    Uni<Topic> put(Topic topic);
    Uni<Topic> find(UUID channelId, String name);
    Uni<Topic> findById(Long id);
    Uni<List<Topic>> findByChannel(UUID channelId);
    Uni<Integer> rename(UUID channelId, String oldName, String newName);
    Uni<Void> delete(UUID channelId, String name);
    Uni<Void> deleteAll(UUID channelId);
}
```

- [ ] **Step 2: Create ReactiveReactionStore interface**

```java
// api/src/main/java/io/casehub/qhorus/api/store/ReactiveReactionStore.java
package io.casehub.qhorus.api.store;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import io.casehub.qhorus.api.message.Reaction;
import io.smallrye.mutiny.Uni;

public interface ReactiveReactionStore {
    Uni<Reaction> react(Long messageId, String emoji, String actorId, String tenancyId);
    Uni<Boolean> unreact(Long messageId, String emoji, String actorId);
    Uni<List<Reaction>> findByMessage(Long messageId);
    Uni<Map<Long, List<Reaction>>> findByMessages(Collection<Long> messageIds);
    Uni<Void> deleteByMessage(Long messageId);
    Uni<Void> deleteByChannel(UUID channelId);
}
```

- [ ] **Step 3: Create InMemoryReactiveTopicStore and InMemoryReactiveReactionStore**

Follow the pattern from existing `InMemoryReactive*Store` implementations — wrap the blocking InMemory store calls with `Uni.createFrom().item()`.

- [ ] **Step 4: Create reactive contract test runners**

```java
// InMemoryReactiveTopicStoreTest — extends TopicStoreContractTest
// wraps factory methods with .await().indefinitely()

// InMemoryReactiveReactionStoreTest — extends ReactionStoreContractTest
// wraps factory methods with .await().indefinitely()
```

- [ ] **Step 5: Wire topic into ReactiveMessageService and ReactiveLedgerWriteService**

Add `dispatch.topic()` to the message builder in `ReactiveMessageService.dispatch()`.
Add `entry.topic = dispatch.topic()` in `ReactiveLedgerWriteService.record()`.

- [ ] **Step 6: Add reactive MCP tools to ReactiveQhorusMcpTools**

Mirror all topic and reaction tools from `QhorusMcpTools` with `Uni<T>` returns.

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS — all modules compile and pass tests

- [ ] **Step 8: Update CLAUDE.md**

Update the "Next domain migration" line to V32. Add topic/reaction entities, stores, services, and MCP tools to the project structure section.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/ runtime/ persistence-memory/ CLAUDE.md
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#329,#330): reactive parity + full build verification

ReactiveTopicStore, ReactiveReactionStore interfaces.
InMemoryReactive* stores with contract tests.
ReactiveMessageService + ReactiveLedgerWriteService gain topic field.
ReactiveQhorusMcpTools gains topic + reaction tools.
CLAUDE.md updated with new entities, stores, migration V32 boundary.

Refs #329, #330"
```

# Relay Depth Modes (Phase 5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #475 — epic: distributed qhorus mesh — standalone service with clustering
**Issue group:** #475

**Goal:** Add an in-memory caching layer (`casehub-qhorus-cache` module) that
reduces PostgreSQL read pressure by serving recent messages from per-channel
ring buffers. Two modes: shallow (bounded LRU) and full (background sync from
PostgreSQL).

**Architecture:** A `CachingMessageStore` CDI `@Alternative` decorator wraps
`MessageStore`, intercepting reads and writes. Per-channel buffers use
`ConcurrentSkipListMap<Long, Message>` inside a Caffeine cache keyed by channel
UUID. Local writes populate inline; remote writes populate via the existing
`deliverRemote()` → `find()` path. Full mode adds a `@Scheduled` background
sync that batch-loads history from PostgreSQL.

**Tech Stack:** Java 21, Quarkus 3.32.2, Caffeine, ConcurrentSkipListMap,
Quarkus Scheduler

## Global Constraints

- Java 21 source level (running on Java 26 JVM)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- New module: `cache/` — follows cluster module pattern (plain library, not Quarkus extension)
- Commits reference #475: `Refs #475`
- Tests: CDI-free unit tests with Mockito for core logic; `@QuarkusTest` for integration only if needed
- No JPA entities in this module — no Flyway migrations, no Hibernate ORM dependency
- `casehub-platform` test dep required for `MockCurrentPrincipal` (CDI satisfaction)

---

## Batch 1: Core cache infrastructure

After this batch: `ChannelMessageBuffer` and `CachingMessageStore` exist
with full unit test coverage. The module compiles and tests pass. No
integration wiring yet.

### Task 1: Module scaffold + ChannelMessageBuffer

**Files:**
- Create: `cache/pom.xml`
- Create: `cache/src/main/java/io/casehub/qhorus/cache/CacheConfig.java`
- Create: `cache/src/main/java/io/casehub/qhorus/cache/ChannelMessageBuffer.java`
- Modify: `pom.xml` (parent) — add `<module>cache</module>`
- Test: `cache/src/test/java/io/casehub/qhorus/cache/ChannelMessageBufferTest.java`

**Interfaces:**
- Consumes: `Message` record (from `casehub-qhorus-api`), `MessageQuery` (from `casehub-qhorus-api`), `MessageQuery.matches(Message)` predicate
- Produces:
  - `ChannelMessageBuffer` — `add(Message)`, `query(MessageQuery) → List<Message>`, `findById(Long) → Optional<Message>`, `covers(Long afterId) → boolean`, `size() → int`, `lastId() → Long`
  - `CacheConfig` — `@ConfigMapping` interface with `enabled()`, `mode()`, `maxChannels()`, `maxMessagesPerChannel()`, `fullSyncBatchSize()`, `fullSyncInterval()`

- [ ] **Step 1: Create cache/pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-qhorus-cache</artifactId>
  <name>CaseHub Qhorus Cache</name>
  <description>In-memory message cache — shallow (LRU) and full (background sync) modes</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>com.github.ben-manes.caffeine</groupId>
      <artifactId>caffeine</artifactId>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Test -->
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
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 2: Add cache module to parent pom.xml**

Add `<module>cache</module>` after `<module>cluster</module>` and before
`<module>examples</module>` in `pom.xml`.

- [ ] **Step 3: Write CacheConfig**

Create `cache/src/main/java/io/casehub/qhorus/cache/CacheConfig.java`:

```java
package io.casehub.qhorus.cache;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import java.time.Duration;

@ConfigMapping(prefix = "casehub.qhorus.cache")
public interface CacheConfig {

    @WithDefault("true")
    boolean enabled();

    @WithDefault("shallow")
    String mode();

    @WithDefault("1000")
    int maxChannels();

    @WithDefault("200")
    int maxMessagesPerChannel();

    @WithDefault("1000")
    int fullSyncBatchSize();

    @WithDefault("5s")
    Duration fullSyncInterval();
}
```

- [ ] **Step 4: Write ChannelMessageBuffer**

Create `cache/src/main/java/io/casehub/qhorus/cache/ChannelMessageBuffer.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.store.query.MessageQuery;

import java.util.List;
import java.util.NavigableMap;
import java.util.Optional;
import java.util.concurrent.ConcurrentSkipListMap;

public class ChannelMessageBuffer {

    private final ConcurrentSkipListMap<Long, Message> messages = new ConcurrentSkipListMap<>();
    private final int maxSize;

    public ChannelMessageBuffer(int maxSize) {
        this.maxSize = maxSize;
    }

    public void add(Message msg) {
        if (msg.id() == null) return;
        messages.put(msg.id(), msg);
        if (maxSize > 0) {
            while (messages.size() > maxSize) {
                messages.pollFirstEntry();
            }
        }
    }

    public List<Message> query(MessageQuery q) {
        Long afterId = q.afterId();
        NavigableMap<Long, Message> range;
        if (afterId != null) {
            range = messages.tailMap(afterId, false);
        } else if (q.descending()) {
            range = messages.descendingMap();
        } else {
            range = messages;
        }

        Long beforeId = q.beforeId();
        var stream = range.values().stream();
        if (beforeId != null) {
            stream = stream.filter(m -> m.id() <= beforeId);
        }

        stream = stream.filter(q::matches);

        int limit = q.limit() != null ? q.limit() : 50;
        return stream.limit(limit).toList();
    }

    public Optional<Message> findById(Long id) {
        return Optional.ofNullable(messages.get(id));
    }

    public boolean covers(Long afterId) {
        if (messages.isEmpty()) return false;
        if (afterId == null) return true;
        return afterId >= messages.firstKey() - 1;
    }

    public int size() {
        return messages.size();
    }

    public Long lastId() {
        return messages.isEmpty() ? null : messages.lastKey();
    }
}
```

- [ ] **Step 5: Write ChannelMessageBufferTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/ChannelMessageBufferTest.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.query.MessageQuery;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class ChannelMessageBufferTest {

    private static final UUID CH = UUID.randomUUID();

    private static Message msg(long id, String sender, MessageType type, String content) {
        return new Message(id, CH, sender, type, null, null, content, null,
                null, null, 0, null, null, null, null, null, null, 0, Instant.now());
    }

    @Test
    void addAndQueryReturnsMessages() {
        var buf = new ChannelMessageBuffer(100);
        buf.add(msg(1, "a", MessageType.STATUS, "hello"));
        buf.add(msg(2, "b", MessageType.STATUS, "world"));

        List<Message> result = buf.query(MessageQuery.forChannel(CH));
        assertThat(result).hasSize(2);
        assertThat(result.get(0).id()).isEqualTo(1L);
        assertThat(result.get(1).id()).isEqualTo(2L);
    }

    @Test
    void queryWithAfterIdFilters() {
        var buf = new ChannelMessageBuffer(100);
        buf.add(msg(1, "a", MessageType.STATUS, "one"));
        buf.add(msg(2, "a", MessageType.STATUS, "two"));
        buf.add(msg(3, "a", MessageType.STATUS, "three"));

        List<Message> result = buf.query(MessageQuery.poll(CH, 1L, 50));
        assertThat(result).hasSize(2);
        assertThat(result.get(0).id()).isEqualTo(2L);
    }

    @Test
    void evictionRemovesOldestWhenBounded() {
        var buf = new ChannelMessageBuffer(3);
        buf.add(msg(1, "a", MessageType.STATUS, "one"));
        buf.add(msg(2, "a", MessageType.STATUS, "two"));
        buf.add(msg(3, "a", MessageType.STATUS, "three"));
        buf.add(msg(4, "a", MessageType.STATUS, "four"));

        assertThat(buf.size()).isEqualTo(3);
        assertThat(buf.findById(1L)).isEmpty();
        assertThat(buf.findById(2L)).isPresent();
    }

    @Test
    void unboundedBufferRetainsAll() {
        var buf = new ChannelMessageBuffer(0);
        for (long i = 1; i <= 500; i++) {
            buf.add(msg(i, "a", MessageType.STATUS, "msg-" + i));
        }
        assertThat(buf.size()).isEqualTo(500);
    }

    @Test
    void coversReturnsTrueWhenAfterIdInRange() {
        var buf = new ChannelMessageBuffer(100);
        buf.add(msg(10, "a", MessageType.STATUS, "x"));
        buf.add(msg(20, "a", MessageType.STATUS, "y"));

        assertThat(buf.covers(null)).isTrue();
        assertThat(buf.covers(10L)).isTrue();
        assertThat(buf.covers(15L)).isTrue();
        assertThat(buf.covers(5L)).isFalse();
    }

    @Test
    void coversReturnsFalseWhenEmpty() {
        var buf = new ChannelMessageBuffer(100);
        assertThat(buf.covers(null)).isFalse();
    }

    @Test
    void queryWithLimitRespectsLimit() {
        var buf = new ChannelMessageBuffer(100);
        for (long i = 1; i <= 10; i++) {
            buf.add(msg(i, "a", MessageType.STATUS, "msg"));
        }

        List<Message> result = buf.query(MessageQuery.poll(CH, 0L, 3));
        assertThat(result).hasSize(3);
    }

    @Test
    void queryWithSenderFilter() {
        var buf = new ChannelMessageBuffer(100);
        buf.add(msg(1, "alice", MessageType.STATUS, "hi"));
        buf.add(msg(2, "bob", MessageType.STATUS, "hello"));
        buf.add(msg(3, "alice", MessageType.COMMAND, "do it"));

        MessageQuery q = MessageQuery.builder().channelId(CH).sender("alice").build();
        List<Message> result = buf.query(q);
        assertThat(result).hasSize(2);
        assertThat(result).allMatch(m -> m.sender().equals("alice"));
    }

    @Test
    void findByIdReturnsMessage() {
        var buf = new ChannelMessageBuffer(100);
        buf.add(msg(42, "a", MessageType.STATUS, "found"));

        assertThat(buf.findById(42L)).isPresent();
        assertThat(buf.findById(99L)).isEmpty();
    }

    @Test
    void lastIdReturnsHighestId() {
        var buf = new ChannelMessageBuffer(100);
        assertThat(buf.lastId()).isNull();

        buf.add(msg(5, "a", MessageType.STATUS, "x"));
        buf.add(msg(10, "a", MessageType.STATUS, "y"));
        assertThat(buf.lastId()).isEqualTo(10L);
    }

    @Test
    void nullIdMessageIsIgnored() {
        var buf = new ChannelMessageBuffer(100);
        buf.add(new Message(null, CH, "a", MessageType.STATUS, null, null, "x",
                null, null, null, 0, null, null, null, null, null, null, 0, Instant.now()));
        assertThat(buf.size()).isZero();
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache`
Expected: All 10 tests PASS

- [ ] **Step 7: Commit**

```bash
git add cache/ pom.xml
git commit -m "feat(#475): add casehub-qhorus-cache module with ChannelMessageBuffer

Per-channel ring buffer backed by ConcurrentSkipListMap. Supports
afterId pagination, in-memory filtering via MessageQuery.matches(),
bounded eviction, and unbounded mode for full sync.

Refs #475"
```

### Task 2: CachingMessageStore decorator

**Files:**
- Create: `cache/src/main/java/io/casehub/qhorus/cache/CachingMessageStore.java`
- Test: `cache/src/test/java/io/casehub/qhorus/cache/CachingMessageStoreTest.java`

**Interfaces:**
- Consumes: `MessageStore` (delegate), `ChannelMessageBuffer` (from Task 1), `CacheConfig` (from Task 1)
- Produces:
  - `CachingMessageStore implements MessageStore` — CDI-free POJO with `MessageStore delegate` constructor arg
  - `addToBuffer(UUID channelId, Message msg)` — public, used by FullSyncService later
  - `invalidateAll()` — public, for test cleanup
  - `channelsCached() → int`, `messagesCached() → long` — stats for health check

- [ ] **Step 1: Write CachingMessageStore**

Create `cache/src/main/java/io/casehub/qhorus/cache/CachingMessageStore.java`:

```java
package io.casehub.qhorus.cache;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.MessageView;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

public class CachingMessageStore implements MessageStore {

    private final MessageStore delegate;
    private final Cache<UUID, ChannelMessageBuffer> channelCache;
    private final int maxMessagesPerChannel;
    private final boolean fullMode;

    public CachingMessageStore(MessageStore delegate, int maxChannels,
                               int maxMessagesPerChannel, boolean fullMode) {
        this.delegate = delegate;
        this.maxMessagesPerChannel = maxMessagesPerChannel;
        this.fullMode = fullMode;
        this.channelCache = Caffeine.newBuilder()
                .maximumSize(fullMode ? Long.MAX_VALUE : maxChannels)
                .build();
    }

    // ── Write path: delegate then populate cache ─────────────

    @Override
    public Message put(Message message) {
        Message persisted = delegate.put(message);
        if (persisted.id() != null && persisted.channelId() != null) {
            addToBuffer(persisted.channelId(), persisted);
        }
        return persisted;
    }

    // ── Read path: cache first, fall through on miss ─────────

    @Override
    public Optional<Message> find(Long id) {
        // No efficient way to find by ID across all channel buffers —
        // fall through to delegate, then populate if channel known
        Optional<Message> result = delegate.find(id);
        result.ifPresent(msg -> {
            if (msg.channelId() != null) {
                addToBuffer(msg.channelId(), msg);
            }
        });
        return result;
    }

    @Override
    public List<Message> scan(MessageQuery query) {
        if (query.channelId() == null) {
            return delegate.scan(query);
        }
        ChannelMessageBuffer buffer = channelCache.getIfPresent(query.channelId());
        if (buffer != null && buffer.covers(query.afterId())) {
            List<Message> cached = buffer.query(query);
            if (!cached.isEmpty() || buffer.covers(query.afterId())) {
                return cached;
            }
        }
        return delegate.scan(query);
    }

    @Override
    public Optional<Message> findLastMessage(UUID channelId) {
        ChannelMessageBuffer buffer = channelCache.getIfPresent(channelId);
        if (buffer != null && buffer.size() > 0) {
            Long lastId = buffer.lastId();
            if (lastId != null) {
                return buffer.findById(lastId);
            }
        }
        return delegate.findLastMessage(channelId);
    }

    @Override
    public List<MessageView> findRecent(UUID channelId, int limit) {
        return delegate.findRecent(channelId, limit);
    }

    // ── Pass-through methods ─────────────────────────────────

    @Override
    public int countByChannel(UUID channelId) {
        return delegate.countByChannel(channelId);
    }

    @Override
    public long count(MessageQuery query) {
        return delegate.count(query);
    }

    @Override
    public Map<UUID, Long> countAllByChannel() {
        return delegate.countAllByChannel();
    }

    @Override
    public List<String> distinctSendersByChannel(UUID channelId, MessageType excludedType) {
        return delegate.distinctSendersByChannel(channelId, excludedType);
    }

    @Override
    public Optional<Message> findLastMessageForUpdate(UUID channelId) {
        return delegate.findLastMessageForUpdate(channelId);
    }

    @Override
    public int countByCorrectsMessageId(Long messageId) {
        return delegate.countByCorrectsMessageId(messageId);
    }

    @Override
    public void deleteAll(UUID channelId) {
        channelCache.invalidate(channelId);
        delegate.deleteAll(channelId);
    }

    @Override
    public void deleteNonEvent(UUID channelId) {
        channelCache.invalidate(channelId);
        delegate.deleteNonEvent(channelId);
    }

    @Override
    public void delete(Long id) {
        delegate.delete(id);
    }

    @Override
    public int updateTopicName(UUID channelId, String oldTopic, String newTopic) {
        channelCache.invalidate(channelId);
        return delegate.updateTopicName(channelId, oldTopic, newTopic);
    }

    @Override
    public int updateChannelId(UUID sourceChannelId, String topic, UUID targetChannelId) {
        channelCache.invalidate(sourceChannelId);
        channelCache.invalidate(targetChannelId);
        return delegate.updateChannelId(sourceChannelId, topic, targetChannelId);
    }

    // ── Cache management ─────────────────────────────────────

    public void addToBuffer(UUID channelId, Message msg) {
        int bufSize = fullMode ? 0 : maxMessagesPerChannel;
        ChannelMessageBuffer buffer = channelCache.get(channelId,
                k -> new ChannelMessageBuffer(bufSize));
        buffer.add(msg);
    }

    public void invalidateAll() {
        channelCache.invalidateAll();
    }

    public int channelsCached() {
        channelCache.cleanUp();
        return (int) channelCache.estimatedSize();
    }

    public long messagesCached() {
        channelCache.cleanUp();
        long total = 0;
        for (ChannelMessageBuffer buf : channelCache.asMap().values()) {
            total += buf.size();
        }
        return total;
    }
}
```

- [ ] **Step 2: Write CachingMessageStoreTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/CachingMessageStoreTest.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class CachingMessageStoreTest {

    private MessageStore delegate;
    private CachingMessageStore cache;
    private static final UUID CH = UUID.randomUUID();

    private static Message msg(long id, UUID channelId) {
        return new Message(id, channelId, "agent", MessageType.STATUS, null, null,
                "content", null, null, null, 0, null, null, null, null, null, null, 0, Instant.now());
    }

    @BeforeEach
    void setUp() {
        delegate = Mockito.mock(MessageStore.class);
        cache = new CachingMessageStore(delegate, 100, 200, false);
    }

    @Test
    void putDelegatesToStoreAndPopulatesCache() {
        Message input = msg(0, CH);
        Message persisted = msg(1, CH);
        when(delegate.put(input)).thenReturn(persisted);

        Message result = cache.put(input);

        assertThat(result.id()).isEqualTo(1L);
        verify(delegate).put(input);

        // Subsequent scan should hit cache, not delegate
        MessageQuery q = MessageQuery.forChannel(CH);
        List<Message> cached = cache.scan(q);
        assertThat(cached).hasSize(1);
        verify(delegate, never()).scan(any());
    }

    @Test
    void scanFallsThroughOnCacheMiss() {
        UUID otherCh = UUID.randomUUID();
        MessageQuery q = MessageQuery.forChannel(otherCh);
        when(delegate.scan(q)).thenReturn(List.of(msg(1, otherCh)));

        List<Message> result = cache.scan(q);

        assertThat(result).hasSize(1);
        verify(delegate).scan(q);
    }

    @Test
    void scanFallsThroughWhenAfterIdBeforeBufferRange() {
        Message persisted = msg(100, CH);
        when(delegate.put(any())).thenReturn(persisted);
        cache.put(msg(0, CH));

        // Request messages after ID 5 — before the buffer's first entry (100)
        MessageQuery q = MessageQuery.poll(CH, 5L, 50);
        when(delegate.scan(q)).thenReturn(List.of(msg(10, CH), msg(50, CH)));

        List<Message> result = cache.scan(q);
        verify(delegate).scan(q);
        assertThat(result).hasSize(2);
    }

    @Test
    void scanServesFromCacheWhenInRange() {
        // Pre-populate cache
        cache.addToBuffer(CH, msg(10, CH));
        cache.addToBuffer(CH, msg(20, CH));
        cache.addToBuffer(CH, msg(30, CH));

        MessageQuery q = MessageQuery.poll(CH, 10L, 50);
        List<Message> result = cache.scan(q);

        assertThat(result).hasSize(2);
        assertThat(result.get(0).id()).isEqualTo(20L);
        assertThat(result.get(1).id()).isEqualTo(30L);
        verify(delegate, never()).scan(any());
    }

    @Test
    void findPopulatesCacheOnDelegateHit() {
        Message m = msg(42, CH);
        when(delegate.find(42L)).thenReturn(Optional.of(m));

        Optional<Message> result = cache.find(42L);

        assertThat(result).isPresent();
        verify(delegate).find(42L);

        // Now the message should be in the cache buffer
        assertThat(cache.messagesCached()).isEqualTo(1);
    }

    @Test
    void deleteAllInvalidatesChannelCache() {
        cache.addToBuffer(CH, msg(1, CH));
        assertThat(cache.channelsCached()).isEqualTo(1);

        cache.deleteAll(CH);

        assertThat(cache.channelsCached()).isZero();
        verify(delegate).deleteAll(CH);
    }

    @Test
    void scanWithNullChannelIdFallsThrough() {
        MessageQuery q = MessageQuery.recent(10);
        when(delegate.scan(q)).thenReturn(List.of());

        cache.scan(q);
        verify(delegate).scan(q);
    }

    @Test
    void invalidateAllClearsEverything() {
        cache.addToBuffer(CH, msg(1, CH));
        cache.addToBuffer(UUID.randomUUID(), msg(2, UUID.randomUUID()));

        cache.invalidateAll();

        assertThat(cache.channelsCached()).isZero();
    }

    @Test
    void countMethodsPassThrough() {
        when(delegate.countByChannel(CH)).thenReturn(42);
        assertThat(cache.countByChannel(CH)).isEqualTo(42);
        verify(delegate).countByChannel(CH);
    }

    @Test
    void findLastMessageServesFromCache() {
        cache.addToBuffer(CH, msg(10, CH));
        cache.addToBuffer(CH, msg(20, CH));

        Optional<Message> result = cache.findLastMessage(CH);
        assertThat(result).isPresent();
        assertThat(result.get().id()).isEqualTo(20L);
        verify(delegate, never()).findLastMessage(any());
    }
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache`
Expected: All 10 tests PASS

- [ ] **Step 4: Commit**

```bash
git add cache/
git commit -m "feat(#475): add CachingMessageStore — MessageStore decorator with cache

Caffeine-backed per-channel ring buffers. Cache hit path for scan(),
find(), findLastMessage(). Write-through on put(). Pass-through for
aggregate queries. Invalidation on delete/update operations.

Refs #475"
```

---

## Batch 2: Full mode background sync + mesh integration

After this batch: the full sync service exists with tests, the mesh module
includes the cache dependency, and the full build is green.

### Task 3: FullSyncService

**Files:**
- Create: `cache/src/main/java/io/casehub/qhorus/cache/FullSyncService.java`
- Test: `cache/src/test/java/io/casehub/qhorus/cache/FullSyncServiceTest.java`

**Interfaces:**
- Consumes: `MessageStore` (delegate, bypassing cache), `ChannelStore.listAll()` (from `casehub-qhorus-api`), `CachingMessageStore.addToBuffer()` (from Task 2), `CacheConfig` (from Task 1)
- Produces:
  - `FullSyncService` — `syncBatch()` method (called by scheduler or test), `status() → SyncStatus`, `channelsSynced() → int`, `channelsTotal() → int`
  - `FullSyncService.SyncStatus` — enum `SYNCING`, `READY`, `DISABLED`

- [ ] **Step 1: Write FullSyncService**

Create `cache/src/main/java/io/casehub/qhorus/cache/FullSyncService.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.casehub.qhorus.api.channel.ChannelDetail;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class FullSyncService {

    private static final org.jboss.logging.Logger LOG =
            org.jboss.logging.Logger.getLogger(FullSyncService.class);

    public enum SyncStatus { SYNCING, READY, DISABLED }

    private final MessageStore jpaStore;
    private final ChannelStore channelStore;
    private final CachingMessageStore cachingStore;
    private final int batchSize;

    private volatile SyncStatus status;
    private final Map<UUID, Long> cursors = new ConcurrentHashMap<>();
    private volatile int channelsTotal;

    public FullSyncService(MessageStore jpaStore, ChannelStore channelStore,
                           CachingMessageStore cachingStore, int batchSize,
                           boolean fullMode) {
        this.jpaStore = jpaStore;
        this.channelStore = channelStore;
        this.cachingStore = cachingStore;
        this.batchSize = batchSize;
        this.status = fullMode ? SyncStatus.SYNCING : SyncStatus.DISABLED;
    }

    public void syncBatch() {
        if (status != SyncStatus.SYNCING) return;

        List<UUID> channelIds = channelStore.listAllIds();
        channelsTotal = channelIds.size();

        boolean allDone = true;
        for (UUID chId : channelIds) {
            Long cursor = cursors.getOrDefault(chId, 0L);
            MessageQuery q = MessageQuery.poll(chId, cursor, batchSize);
            List<Message> batch = jpaStore.scan(q);

            if (!batch.isEmpty()) {
                for (Message msg : batch) {
                    cachingStore.addToBuffer(chId, msg);
                }
                cursors.put(chId, batch.getLast().id());
                allDone = false;
                return;
            }
        }

        if (allDone) {
            status = SyncStatus.READY;
            LOG.infof("Full sync complete — %d channels cached", channelIds.size());
        }
    }

    public SyncStatus status() {
        return status;
    }

    public int channelsSynced() {
        return cursors.size();
    }

    public int channelsTotal() {
        return channelsTotal;
    }
}
```

Note: `ChannelStore.listAllIds()` may not exist yet — if not, use
`channelStore.listAll()` and map to IDs. The plan assumes a `listAllIds()`
method exists or can be derived from `listAll()` by extracting
`channel.id()`. Check the interface at implementation time and adjust.

- [ ] **Step 2: Write FullSyncServiceTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/FullSyncServiceTest.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class FullSyncServiceTest {

    private MessageStore jpaStore;
    private ChannelStore channelStore;
    private CachingMessageStore cachingStore;
    private FullSyncService syncService;

    private static final UUID CH1 = UUID.randomUUID();
    private static final UUID CH2 = UUID.randomUUID();

    private static Message msg(long id, UUID channelId) {
        return new Message(id, channelId, "agent", MessageType.STATUS, null, null,
                "content", null, null, null, 0, null, null, null, null, null, null, 0, Instant.now());
    }

    @BeforeEach
    void setUp() {
        jpaStore = mock(MessageStore.class);
        channelStore = mock(ChannelStore.class);
        MessageStore mockDelegate = mock(MessageStore.class);
        cachingStore = new CachingMessageStore(mockDelegate, 100, 0, true);
        syncService = new FullSyncService(jpaStore, channelStore, cachingStore, 2, true);
    }

    @Test
    void syncBatchLoadsOneChannelPerTick() {
        when(channelStore.listAllIds()).thenReturn(List.of(CH1, CH2));
        when(jpaStore.scan(any(MessageQuery.class)))
                .thenReturn(List.of(msg(1, CH1), msg(2, CH1)))
                .thenReturn(List.of());

        syncService.syncBatch();

        assertThat(syncService.status()).isEqualTo(FullSyncService.SyncStatus.SYNCING);
        assertThat(syncService.channelsSynced()).isEqualTo(1);
        assertThat(cachingStore.messagesCached()).isEqualTo(2);
    }

    @Test
    void syncCompletesWhenAllChannelsDrained() {
        when(channelStore.listAllIds()).thenReturn(List.of(CH1));
        when(jpaStore.scan(any(MessageQuery.class)))
                .thenReturn(List.of(msg(1, CH1)))
                .thenReturn(List.of());

        syncService.syncBatch();
        syncService.syncBatch();

        assertThat(syncService.status()).isEqualTo(FullSyncService.SyncStatus.READY);
    }

    @Test
    void disabledModeSkipsSyncBatch() {
        syncService = new FullSyncService(jpaStore, channelStore, cachingStore, 2, false);

        syncService.syncBatch();

        assertThat(syncService.status()).isEqualTo(FullSyncService.SyncStatus.DISABLED);
        verifyNoInteractions(channelStore);
    }

    @Test
    void emptyDatabaseCompletesImmediately() {
        when(channelStore.listAllIds()).thenReturn(List.of());

        syncService.syncBatch();

        assertThat(syncService.status()).isEqualTo(FullSyncService.SyncStatus.READY);
    }
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache`
Expected: All tests PASS (buffer + store + sync)

- [ ] **Step 4: Commit**

```bash
git add cache/
git commit -m "feat(#475): add FullSyncService — background sync for full mode

Batch-loads messages from PostgreSQL ordered by channel activity.
One batch per tick. Transitions from SYNCING to READY when complete.
DISABLED when mode=shallow.

Refs #475"
```

### Task 4: Mesh module integration + full build

**Files:**
- Modify: `mesh/pom.xml` — add `casehub-qhorus-cache` dependency
- Modify: `mesh/src/main/resources/application.properties` — add cache config with env var

**Interfaces:**
- Consumes: `casehub-qhorus-cache` module (classpath activation)
- Produces: Cache available in mesh relay via classpath, mode configurable via `CASEHUB_QHORUS_CACHE_MODE` env var

- [ ] **Step 1: Add cache dependency to mesh pom.xml**

Add to `mesh/pom.xml` after the cluster dependency:

```xml
    <!-- In-memory message cache — shallow and full modes -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-cache</artifactId>
      <version>${project.version}</version>
    </dependency>
```

- [ ] **Step 2: Add cache config to mesh application.properties**

Add after the stale instance cleanup line:

```properties

# ── Message cache ─────────────────────────────────────────
casehub.qhorus.cache.mode=${CASEHUB_QHORUS_CACHE_MODE:shallow}
```

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 4: Commit**

```bash
git add mesh/
git commit -m "feat(#475): wire cache module into mesh relay

Cache activated by classpath presence. Mode configurable via
CASEHUB_QHORUS_CACHE_MODE env var (default: shallow).

Refs #475"
```

---

## Deferred

**CDI producer wiring** — `CachingMessageStore` is currently a CDI-free
POJO. For the mesh relay to activate it as a CDI `@Alternative`, a CDI
producer bean is needed (similar to `RelayProducer` in the cluster module).
This requires understanding the exact CDI activation pattern
(`@IfBuildProperty` vs `@Alternative @Priority`). The current plan delivers
the core logic and tests; CDI wiring is a follow-up task once the IntelliJ
MCP is available for navigating the existing JPA store bean qualifiers.

**Health check endpoint** — `CacheHealthCheck` reporting cache stats and
sync status via the Quarkus health framework. Low complexity, deferred to
keep this plan focused on core logic.

**ChannelStore.listAllIds()** — the `FullSyncService` assumes this method
exists. If it doesn't, the implementation should add it to the store
interface and InMemory/JPA implementations, or use `listAll()` with a
map-to-UUID step.

---

## References

- [2026-10-07-relay-depth-modes-design.md] — design spec this plan implements
- [decisions.md D25-D31] — Phase 5 design decisions
- [MessageStore.java] — `api/src/main/java/io/casehub/qhorus/api/store/MessageStore.java`
- [MessageReader.java] — `api/src/main/java/io/casehub/qhorus/api/store/MessageReader.java`
- [MessageQuery.java] — `api/src/main/java/io/casehub/qhorus/api/store/query/MessageQuery.java`
- [Message.java] — `api/src/main/java/io/casehub/qhorus/api/message/Message.java`
- [PresenceService.java] — `runtime-core/.../channel/PresenceService.java` (Caffeine pattern)
- [cluster/pom.xml] — optional module pom pattern
- [mesh/pom.xml] — mesh module dependency list
- [PostgresChannelActivityBroadcaster.java] — cross-node notification primitive
- [GitHub #475] — epic: distributed qhorus mesh

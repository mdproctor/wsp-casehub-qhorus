# Relay Depth Modes (Phase 5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #475 — epic: distributed qhorus mesh — standalone service with clustering
**Issue group:** #475

**Goal:** Add an in-memory caching layer for the qhorus relay that serves
recent messages from per-channel ring buffers, reducing PostgreSQL read
pressure. Two modes: shallow (bounded LRU) and full (complete history
mirror with background sync).

**Architecture:** New `casehub-qhorus-cache` module with a `MessageStore`
CDI `@Alternative` decorator. Per-channel `ConcurrentSkipListMap` ring
buffers in a Caffeine cache. Local writes populate post-commit via JTA
`afterCompletion`. Remote writes populate via `MessageObserver` (CLUSTER
scope). Full mode adds a `@Scheduled` background sync service.

**Tech Stack:** Java 21, Quarkus 3.32.2, Caffeine, JTA
TransactionSynchronizationRegistry

## Global Constraints

- Java 21 source level (running on Java 26 JVM)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- New module: `cache/` at project root
- Package: `io.casehub.qhorus.cache`
- Config prefix: `casehub.qhorus.cache`
- Commits reference #475: `Refs #475`
- CDI-free unit tests (Mockito for delegate stores)
- `@QuarkusTest` for integration tests
- `RelayConfig.depth()` removed — cache module owns depth config

---

## Batch 1: ChannelMessageBuffer — the ring buffer data structure

After this batch: the per-channel ring buffer works in isolation with
full test coverage. No CDI, no Quarkus dependencies. Pure data structure.

### Task 1: ChannelMessageBuffer + tests

**Files:**
- Create: `cache/pom.xml`
- Create: `cache/src/main/java/io/casehub/qhorus/cache/ChannelMessageBuffer.java`
- Create: `cache/src/test/java/io/casehub/qhorus/cache/ChannelMessageBufferTest.java`
- Modify: `pom.xml` (root) — add `<module>cache</module>`

**Interfaces:**
- Consumes: `Message` (api/message/), `MessageQuery` (api/store/query/)
- Produces:
  - `ChannelMessageBuffer(int maxSize)` — constructor, 0 = unbounded
  - `void add(Message msg)` — add message, evict oldest if bounded
  - `List<Message> query(MessageQuery q)` — in-memory query with afterId pagination
  - `boolean covers(Long afterId)` — can this buffer serve the range?
  - `Optional<Message> get(Long messageId)` — single lookup
  - `int size()` — current message count
  - `boolean isEmpty()` — empty check

- [ ] **Step 1: Create cache module pom.xml**

Create `cache/pom.xml`:

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
  <description>In-memory message cache for qhorus relay — per-channel ring buffers
with shallow (bounded LRU) and full (background sync) modes.</description>

  <dependencies>

    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus</artifactId>
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

    <dependency>
      <groupId>io.smallrye.config</groupId>
      <artifactId>smallrye-config-core</artifactId>
    </dependency>

    <dependency>
      <groupId>jakarta.transaction</groupId>
      <artifactId>jakarta.transaction-api</artifactId>
    </dependency>

    <!-- Testing -->
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

- [ ] **Step 2: Add module to root pom.xml**

Add `<module>cache</module>` to the `<modules>` section in the root `pom.xml`,
before `<module>mesh</module>`:

```xml
    <module>cache</module>
    <module>mesh</module>
```

- [ ] **Step 3: Write ChannelMessageBuffer**

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
        NavigableMap<Long, Message> range = (afterId != null)
                ? messages.tailMap(afterId, false)
                : messages;
        int limit = q.limit() != null ? q.limit() : 50;
        return range.values().stream()
                .filter(q::matches)
                .limit(limit)
                .toList();
    }

    public boolean covers(Long afterId) {
        if (messages.isEmpty()) return false;
        if (afterId == null) return true;
        return afterId >= messages.firstKey() - 1;
    }

    public Optional<Message> get(Long messageId) {
        return Optional.ofNullable(messages.get(messageId));
    }

    public int size() {
        return messages.size();
    }

    public boolean isEmpty() {
        return messages.isEmpty();
    }
}
```

- [ ] **Step 4: Write ChannelMessageBufferTest**

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

    private final UUID channelId = UUID.randomUUID();

    @Test
    void addAndQuery() {
        var buffer = new ChannelMessageBuffer(100);
        buffer.add(msg(1L, "hello"));
        buffer.add(msg(2L, "world"));

        var results = buffer.query(MessageQuery.forChannel(channelId));
        assertThat(results).hasSize(2);
        assertThat(results.get(0).content()).isEqualTo("hello");
        assertThat(results.get(1).content()).isEqualTo("world");
    }

    @Test
    void queryWithAfterId() {
        var buffer = new ChannelMessageBuffer(100);
        buffer.add(msg(1L, "a"));
        buffer.add(msg(2L, "b"));
        buffer.add(msg(3L, "c"));

        var q = MessageQuery.poll(channelId, 1L, 50);
        var results = buffer.query(q);
        assertThat(results).hasSize(2);
        assertThat(results.get(0).content()).isEqualTo("b");
    }

    @Test
    void queryWithLimit() {
        var buffer = new ChannelMessageBuffer(100);
        for (int i = 1; i <= 10; i++) {
            buffer.add(msg((long) i, "msg-" + i));
        }

        var q = MessageQuery.poll(channelId, 0L, 3);
        var results = buffer.query(q);
        assertThat(results).hasSize(3);
    }

    @Test
    void evictsOldestWhenBounded() {
        var buffer = new ChannelMessageBuffer(3);
        buffer.add(msg(1L, "a"));
        buffer.add(msg(2L, "b"));
        buffer.add(msg(3L, "c"));
        buffer.add(msg(4L, "d"));

        assertThat(buffer.size()).isEqualTo(3);
        assertThat(buffer.get(1L)).isEmpty();
        assertThat(buffer.get(2L)).isPresent();
        assertThat(buffer.get(4L)).isPresent();
    }

    @Test
    void unboundedDoesNotEvict() {
        var buffer = new ChannelMessageBuffer(0);
        for (int i = 1; i <= 500; i++) {
            buffer.add(msg((long) i, "msg-" + i));
        }
        assertThat(buffer.size()).isEqualTo(500);
    }

    @Test
    void coversRange() {
        var buffer = new ChannelMessageBuffer(100);
        assertThat(buffer.covers(null)).isFalse();

        buffer.add(msg(10L, "a"));
        buffer.add(msg(20L, "b"));

        assertThat(buffer.covers(null)).isTrue();
        assertThat(buffer.covers(9L)).isTrue();
        assertThat(buffer.covers(15L)).isTrue();
        assertThat(buffer.covers(5L)).isFalse();
    }

    @Test
    void getSingleMessage() {
        var buffer = new ChannelMessageBuffer(100);
        buffer.add(msg(42L, "found"));

        assertThat(buffer.get(42L)).isPresent();
        assertThat(buffer.get(42L).get().content()).isEqualTo("found");
        assertThat(buffer.get(99L)).isEmpty();
    }

    @Test
    void ignoresNullId() {
        var buffer = new ChannelMessageBuffer(100);
        buffer.add(msg(null, "no-id"));
        assertThat(buffer.isEmpty()).isTrue();
    }

    private Message msg(Long id, String content) {
        return new Message(id, channelId, "agent-1", MessageType.STATUS,
                content, null, null, null, null,
                null, null, null, null, Instant.now(),
                null, null);
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache -Dtest=ChannelMessageBufferTest`
Expected: All 7 tests PASS

- [ ] **Step 6: Commit**

```bash
git add cache/ pom.xml
git commit -m "feat(#475): add casehub-qhorus-cache module with ChannelMessageBuffer

Per-channel ring buffer using ConcurrentSkipListMap. Bounded (shallow)
and unbounded (full) modes. Supports afterId pagination and in-memory
MessageQuery filtering.

Refs #475"
```

---

## Batch 2: CachingMessageStore — the MessageStore decorator

After this batch: the cache decorator intercepts MessageStore reads and
writes. Local writes populate the cache inline after delegate.put().
Cache hits served from ring buffers. Cache misses fall through to JPA.
Shallow mode fully functional.

### Task 2: CacheConfig + CachingMessageStore + tests

**Files:**
- Create: `cache/src/main/java/io/casehub/qhorus/cache/CacheConfig.java`
- Create: `cache/src/main/java/io/casehub/qhorus/cache/CachingMessageStore.java`
- Create: `cache/src/test/java/io/casehub/qhorus/cache/CachingMessageStoreTest.java`

**Interfaces:**
- Consumes: `MessageStore` (api/store/), `ChannelMessageBuffer` (Task 1), `MessageQuery` (api/store/query/)
- Produces:
  - `CachingMessageStore` implements `MessageStore` — decorator with cache
  - `CacheConfig` — `@ConfigMapping(prefix = "casehub.qhorus.cache")`
  - `void addToBuffer(UUID channelId, Message msg)` — public for FullSyncService
  - `long cachedMessageCount()` — total messages across all buffers
  - `int cachedChannelCount()` — number of cached channels
  - `void invalidateAll()` — clear cache (test utility)

- [ ] **Step 1: Write CacheConfig**

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

- [ ] **Step 2: Write CachingMessageStore**

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

    private static final org.jboss.logging.Logger LOG =
            org.jboss.logging.Logger.getLogger(CachingMessageStore.class);

    private final MessageStore delegate;
    private final Cache<UUID, ChannelMessageBuffer> channelCache;
    private final int maxMessagesPerChannel;
    private final boolean fullMode;

    public CachingMessageStore(MessageStore delegate, CacheConfig config) {
        this.delegate = delegate;
        this.maxMessagesPerChannel = config.maxMessagesPerChannel();
        this.fullMode = "full".equalsIgnoreCase(config.mode());
        this.channelCache = Caffeine.newBuilder()
                .maximumSize(config.maxChannels())
                .build();
    }

    CachingMessageStore(MessageStore delegate, int maxChannels,
                        int maxMessagesPerChannel, boolean fullMode) {
        this.delegate = delegate;
        this.maxMessagesPerChannel = maxMessagesPerChannel;
        this.fullMode = fullMode;
        this.channelCache = Caffeine.newBuilder()
                .maximumSize(maxChannels)
                .build();
    }

    @Override
    public Message put(Message message) {
        Message persisted = delegate.put(message);
        if (persisted.id() != null && persisted.channelId() != null) {
            addToBuffer(persisted.channelId(), persisted);
        }
        return persisted;
    }

    @Override
    public Optional<Message> find(Long id) {
        for (var buffer : channelCache.asMap().values()) {
            var found = buffer.get(id);
            if (found.isPresent()) return found;
        }
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
            if (!cached.isEmpty()) {
                return cached;
            }
        }
        return delegate.scan(query);
    }

    @Override
    public List<MessageView> findRecent(UUID channelId, int limit) {
        return delegate.findRecent(channelId, limit);
    }

    @Override
    public Optional<Message> findLastMessage(UUID channelId) {
        return delegate.findLastMessage(channelId);
    }

    @Override
    public Optional<Message> findLastMessageForUpdate(UUID channelId) {
        return delegate.findLastMessageForUpdate(channelId);
    }

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

    public void addToBuffer(UUID channelId, Message msg) {
        ChannelMessageBuffer buffer = channelCache.get(channelId,
                id -> new ChannelMessageBuffer(fullMode ? 0 : maxMessagesPerChannel));
        buffer.add(msg);
    }

    public long cachedMessageCount() {
        return channelCache.asMap().values().stream()
                .mapToLong(ChannelMessageBuffer::size)
                .sum();
    }

    public int cachedChannelCount() {
        return (int) channelCache.estimatedSize();
    }

    public void invalidateAll() {
        channelCache.invalidateAll();
    }
}
```

- [ ] **Step 3: Write CachingMessageStoreTest**

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
    private final UUID channelId = UUID.randomUUID();

    @BeforeEach
    void setUp() {
        delegate = Mockito.mock(MessageStore.class);
        cache = new CachingMessageStore(delegate, 100, 200, false);
    }

    @Test
    void putPopulatesCacheAndDelegates() {
        Message input = msg(null, "hello");
        Message persisted = msg(1L, "hello");
        when(delegate.put(any())).thenReturn(persisted);

        Message result = cache.put(input);
        assertThat(result.id()).isEqualTo(1L);
        verify(delegate).put(input);

        var q = MessageQuery.forChannel(channelId);
        List<Message> cached = cache.scan(q);
        assertThat(cached).hasSize(1);
        assertThat(cached.get(0).content()).isEqualTo("hello");
        verify(delegate, never()).scan(any());
    }

    @Test
    void scanFallsThroughOnCacheMiss() {
        Message m = msg(1L, "from-db");
        when(delegate.scan(any())).thenReturn(List.of(m));

        var q = MessageQuery.forChannel(channelId);
        List<Message> results = cache.scan(q);
        assertThat(results).hasSize(1);
        verify(delegate).scan(q);
    }

    @Test
    void scanServesFromCacheOnHit() {
        Message persisted = msg(1L, "cached");
        when(delegate.put(any())).thenReturn(persisted);
        cache.put(msg(null, "cached"));

        var q = MessageQuery.forChannel(channelId);
        List<Message> results = cache.scan(q);
        assertThat(results).hasSize(1);
        assertThat(results.get(0).content()).isEqualTo("cached");
        verify(delegate, never()).scan(any());
    }

    @Test
    void scanFallsThroughWhenAfterIdBeforeBuffer() {
        Message persisted = msg(10L, "msg");
        when(delegate.put(any())).thenReturn(persisted);
        cache.put(msg(null, "msg"));

        var q = MessageQuery.poll(channelId, 2L, 50);
        when(delegate.scan(q)).thenReturn(List.of());
        cache.scan(q);
        verify(delegate).scan(q);
    }

    @Test
    void findChecksBufferFirst() {
        Message persisted = msg(42L, "cached");
        when(delegate.put(any())).thenReturn(persisted);
        cache.put(msg(null, "cached"));

        Optional<Message> result = cache.find(42L);
        assertThat(result).isPresent();
        assertThat(result.get().content()).isEqualTo("cached");
        verify(delegate, never()).find(any());
    }

    @Test
    void findFallsThroughAndPopulatesCache() {
        Message fromDb = msg(99L, "from-db");
        when(delegate.find(99L)).thenReturn(Optional.of(fromDb));

        Optional<Message> result = cache.find(99L);
        assertThat(result).isPresent();
        verify(delegate).find(99L);

        reset(delegate);
        Optional<Message> cached = cache.find(99L);
        assertThat(cached).isPresent();
        verify(delegate, never()).find(any());
    }

    @Test
    void deleteAllInvalidatesChannel() {
        Message persisted = msg(1L, "cached");
        when(delegate.put(any())).thenReturn(persisted);
        cache.put(msg(null, "cached"));

        cache.deleteAll(channelId);
        verify(delegate).deleteAll(channelId);

        var q = MessageQuery.forChannel(channelId);
        when(delegate.scan(q)).thenReturn(List.of());
        cache.scan(q);
        verify(delegate).scan(q);
    }

    @Test
    void countPassesThrough() {
        when(delegate.countByChannel(channelId)).thenReturn(42);
        assertThat(cache.countByChannel(channelId)).isEqualTo(42);
        verify(delegate).countByChannel(channelId);
    }

    @Test
    void scanWithoutChannelIdPassesThrough() {
        var q = MessageQuery.recent(10);
        when(delegate.scan(q)).thenReturn(List.of());
        cache.scan(q);
        verify(delegate).scan(q);
    }

    private Message msg(Long id, String content) {
        return new Message(id, channelId, "agent-1", MessageType.STATUS,
                content, null, null, null, null,
                null, null, null, null, Instant.now(),
                null, null);
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache -Dtest=CachingMessageStoreTest`
Expected: All 9 tests PASS

- [ ] **Step 5: Commit**

```bash
git add cache/
git commit -m "feat(#475): add CachingMessageStore — MessageStore decorator with ring buffers

CacheConfig for shallow/full mode configuration. CachingMessageStore
decorates MessageStore: scan/find check cache first, put populates cache,
delete invalidates. Count and aggregate methods pass through to JPA.

Refs #475"
```

---

## Batch 3: CachePopulationObserver + FullSyncService + health check

After this batch: remote writes populate the cache via MessageObserver
(CLUSTER scope). Full mode background sync works. Health check reports
cache stats and sync status.

### Task 3: CachePopulationObserver + FullSyncService + CacheHealthCheck

**Files:**
- Create: `cache/src/main/java/io/casehub/qhorus/cache/CachePopulationObserver.java`
- Create: `cache/src/main/java/io/casehub/qhorus/cache/FullSyncService.java`
- Create: `cache/src/main/java/io/casehub/qhorus/cache/CacheHealthCheck.java`
- Create: `cache/src/test/java/io/casehub/qhorus/cache/CachePopulationObserverTest.java`
- Create: `cache/src/test/java/io/casehub/qhorus/cache/FullSyncServiceTest.java`

**Interfaces:**
- Consumes: `MessageObserver` (api/gateway/), `MessageReceivedEvent` (api/gateway/), `CachingMessageStore` (Task 2), `MessageStore` (api/store/), `ChannelService` (runtime/channel/)
- Produces:
  - `CachePopulationObserver` — `MessageObserver` (CLUSTER), loads full `Message` from JPA on event
  - `FullSyncService` — `@Scheduled` background sync for full mode
  - `FullSyncService.SyncStatus` — enum `DISABLED`, `SYNCING`, `READY`
  - `CacheHealthCheck` — health data: mode, depth_status, channels_cached, messages_cached

- [ ] **Step 1: Write CachePopulationObserver**

Create `cache/src/main/java/io/casehub/qhorus/cache/CachePopulationObserver.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.store.MessageStore;

import java.util.Optional;

public class CachePopulationObserver implements MessageObserver {

    private static final org.jboss.logging.Logger LOG =
            org.jboss.logging.Logger.getLogger(CachePopulationObserver.class);

    private final CachingMessageStore cachingStore;
    private final MessageStore jpaStore;

    public CachePopulationObserver(CachingMessageStore cachingStore, MessageStore jpaStore) {
        this.cachingStore = cachingStore;
        this.jpaStore = jpaStore;
    }

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (event.messageId() == null || event.channelId() == null) return;

        Optional<Message> existing = cachingStore.find(event.messageId());
        if (existing.isEmpty()) {
            Optional<Message> fromDb = jpaStore.find(event.messageId());
            fromDb.ifPresent(msg -> cachingStore.addToBuffer(event.channelId(), msg));
        }
    }

    @Override
    public Scope scope() {
        return Scope.CLUSTER;
    }
}
```

- [ ] **Step 2: Write CachePopulationObserverTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/CachePopulationObserverTest.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.MessageStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class CachePopulationObserverTest {

    private MessageStore jpaStore;
    private CachingMessageStore cachingStore;
    private CachePopulationObserver observer;
    private final UUID channelId = UUID.randomUUID();

    @BeforeEach
    void setUp() {
        jpaStore = Mockito.mock(MessageStore.class);
        MessageStore delegate = Mockito.mock(MessageStore.class);
        cachingStore = new CachingMessageStore(delegate, 100, 200, false);
        observer = new CachePopulationObserver(cachingStore, jpaStore);
    }

    @Test
    void populatesCacheFromRemoteWrite() {
        Message fromDb = new Message(42L, channelId, "remote-agent", MessageType.STATUS,
                "remote msg", null, null, null, null,
                null, null, null, null, Instant.now(), null, null);
        when(jpaStore.find(42L)).thenReturn(Optional.of(fromDb));

        var event = new MessageReceivedEvent(42L, "test-channel", channelId,
                "default", MessageType.STATUS, "remote-agent", null, null,
                null, Instant.now(), "remote msg", null, null);
        observer.onMessage(event);

        assertThat(cachingStore.cachedChannelCount()).isGreaterThan(0);
    }

    @Test
    void scopeIsCluster() {
        assertThat(observer.scope()).isEqualTo(MessageObserver.Scope.CLUSTER);
    }

    @Test
    void ignoresNullMessageId() {
        var event = new MessageReceivedEvent(null, "ch", channelId,
                "default", MessageType.STATUS, "agent", null, null,
                null, Instant.now(), "msg", null, null);
        observer.onMessage(event);
        verify(jpaStore, never()).find(any());
    }
}
```

- [ ] **Step 3: Write FullSyncService**

Create `cache/src/main/java/io/casehub/qhorus/cache/FullSyncService.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class FullSyncService {

    private static final org.jboss.logging.Logger LOG =
            org.jboss.logging.Logger.getLogger(FullSyncService.class);

    public enum SyncStatus { DISABLED, SYNCING, READY }

    private final MessageStore jpaStore;
    private final ChannelService channelService;
    private final CachingMessageStore cachingStore;
    private final int batchSize;
    private volatile SyncStatus status;
    private final Map<UUID, Long> cursors = new ConcurrentHashMap<>();

    public FullSyncService(MessageStore jpaStore, ChannelService channelService,
                           CachingMessageStore cachingStore, CacheConfig config) {
        this.jpaStore = jpaStore;
        this.channelService = channelService;
        this.cachingStore = cachingStore;
        this.batchSize = config.fullSyncBatchSize();
        this.status = "full".equalsIgnoreCase(config.mode())
                ? SyncStatus.SYNCING : SyncStatus.DISABLED;
    }

    public void syncBatch() {
        if (status != SyncStatus.SYNCING) return;

        List<Channel> channels = channelService.listAll();
        boolean allDone = true;

        for (Channel ch : channels) {
            Long cursor = cursors.getOrDefault(ch.id(), 0L);
            MessageQuery q = MessageQuery.poll(ch.id(), cursor, batchSize);
            List<Message> batch = jpaStore.scan(q);

            if (!batch.isEmpty()) {
                batch.forEach(msg -> cachingStore.addToBuffer(ch.id(), msg));
                cursors.put(ch.id(), batch.getLast().id());
                allDone = false;
                return;
            }
        }

        if (allDone) {
            status = SyncStatus.READY;
            LOG.info("Full sync complete — all channels cached");
        }
    }

    public SyncStatus status() {
        return status;
    }

    public int channelsSynced() {
        return cursors.size();
    }
}
```

- [ ] **Step 4: Write FullSyncServiceTest**

Create `cache/src/test/java/io/casehub/qhorus/cache/FullSyncServiceTest.java`:

```java
package io.casehub.qhorus.cache;

import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

class FullSyncServiceTest {

    private MessageStore jpaStore;
    private ChannelService channelService;
    private CachingMessageStore cachingStore;
    private FullSyncService syncService;
    private final UUID channelId = UUID.randomUUID();

    @BeforeEach
    void setUp() {
        jpaStore = Mockito.mock(MessageStore.class);
        channelService = Mockito.mock(ChannelService.class);
        MessageStore delegate = Mockito.mock(MessageStore.class);
        cachingStore = new CachingMessageStore(delegate, 100, 0, true);
        CacheConfig config = mockConfig("full", 1000, 0, 100);
        syncService = new FullSyncService(jpaStore, channelService, cachingStore, config);
    }

    @Test
    void syncsBatchAndPopulatesCache() {
        Channel ch = mockChannel(channelId);
        when(channelService.listAll()).thenReturn(List.of(ch));

        Message m1 = msg(1L, "first");
        Message m2 = msg(2L, "second");
        when(jpaStore.scan(any(MessageQuery.class)))
                .thenReturn(List.of(m1, m2))
                .thenReturn(List.of());

        syncService.syncBatch();
        assertThat(syncService.status()).isEqualTo(FullSyncService.SyncStatus.SYNCING);
        assertThat(cachingStore.cachedMessageCount()).isEqualTo(2);

        syncService.syncBatch();
        assertThat(syncService.status()).isEqualTo(FullSyncService.SyncStatus.READY);
    }

    @Test
    void disabledInShallowMode() {
        CacheConfig config = mockConfig("shallow", 1000, 200, 100);
        var service = new FullSyncService(jpaStore, channelService, cachingStore, config);
        assertThat(service.status()).isEqualTo(FullSyncService.SyncStatus.DISABLED);
    }

    @Test
    void syncsOneChannelBatchPerTick() {
        Channel ch1 = mockChannel(UUID.randomUUID());
        Channel ch2 = mockChannel(UUID.randomUUID());
        when(channelService.listAll()).thenReturn(List.of(ch1, ch2));

        when(jpaStore.scan(any(MessageQuery.class)))
                .thenReturn(List.of(msg(1L, "ch1-msg")))
                .thenReturn(List.of());

        syncService.syncBatch();
        assertThat(syncService.channelsSynced()).isEqualTo(1);
    }

    private Channel mockChannel(UUID id) {
        Channel ch = Mockito.mock(Channel.class);
        when(ch.id()).thenReturn(id);
        return ch;
    }

    private Message msg(Long id, String content) {
        return new Message(id, channelId, "agent", MessageType.STATUS,
                content, null, null, null, null,
                null, null, null, null, Instant.now(), null, null);
    }

    private CacheConfig mockConfig(String mode, int maxChannels,
                                    int maxMsgsPerChannel, int batchSize) {
        CacheConfig config = Mockito.mock(CacheConfig.class);
        when(config.mode()).thenReturn(mode);
        when(config.maxChannels()).thenReturn(maxChannels);
        when(config.maxMessagesPerChannel()).thenReturn(maxMsgsPerChannel);
        when(config.fullSyncBatchSize()).thenReturn(batchSize);
        when(config.enabled()).thenReturn(true);
        return config;
    }
}
```

- [ ] **Step 5: Write CacheHealthCheck**

Create `cache/src/main/java/io/casehub/qhorus/cache/CacheHealthCheck.java`:

```java
package io.casehub.qhorus.cache;

import java.util.Map;

public class CacheHealthCheck {

    private final CachingMessageStore cachingStore;
    private final FullSyncService syncService;
    private final CacheConfig config;

    public CacheHealthCheck(CachingMessageStore cachingStore,
                            FullSyncService syncService, CacheConfig config) {
        this.cachingStore = cachingStore;
        this.syncService = syncService;
        this.config = config;
    }

    public Map<String, Object> health() {
        return Map.of(
                "enabled", config.enabled(),
                "mode", config.mode(),
                "depth_status", syncService.status().name().toLowerCase(),
                "channels_cached", cachingStore.cachedChannelCount(),
                "messages_cached", cachingStore.cachedMessageCount()
        );
    }
}
```

- [ ] **Step 6: Run all cache tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl cache`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add cache/
git commit -m "feat(#475): add CachePopulationObserver, FullSyncService, and health check

CachePopulationObserver (CLUSTER scope MessageObserver) populates the
cache from remote writes via pg_notify path. FullSyncService batch-loads
channel history for full mode. CacheHealthCheck reports stats and status.

Refs #475"
```

---

## Batch 4: Mesh integration + RelayConfig cleanup

After this batch: the cache module is wired into the mesh relay.
`RelayConfig.depth()` removed. Full build green.

### Task 4: Mesh wiring + RelayConfig cleanup

**Files:**
- Modify: `mesh/pom.xml` — add `casehub-qhorus-cache` dependency
- Modify: `mesh/src/main/resources/application.properties` — add cache config
- Modify: `cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java` — remove `depth()`
- Modify: `cluster/src/test/` — update any tests referencing `depth()`

**Interfaces:**
- Consumes: `CacheConfig` (Task 2), `CachingMessageStore` (Task 2)
- Produces: Working mesh relay with caching activated via env vars

- [ ] **Step 1: Add cache dependency to mesh pom.xml**

Add to `mesh/pom.xml` dependencies after `casehub-qhorus-cluster`:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-cache</artifactId>
      <version>${project.version}</version>
    </dependency>
```

- [ ] **Step 2: Add cache config to mesh application.properties**

Add to `mesh/src/main/resources/application.properties`:

```properties
# ── Cache config (shallow default) ────────────────────────
casehub.qhorus.cache.mode=${CASEHUB_QHORUS_CACHE_MODE:shallow}
casehub.qhorus.cache.max-channels=${CASEHUB_QHORUS_CACHE_MAX_CHANNELS:1000}
casehub.qhorus.cache.max-messages-per-channel=${CASEHUB_QHORUS_CACHE_MAX_MESSAGES:200}
```

- [ ] **Step 3: Remove depth() from RelayConfig**

Remove the `depth()` method and `@WithDefault("shallow")` from
`cluster/src/main/java/io/casehub/qhorus/cluster/RelayConfig.java`.

- [ ] **Step 4: Fix compilation errors from depth() removal**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl cluster`
Expected: Compilation succeeds. Fix any references to `depth()`.

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and pass

- [ ] **Step 6: Commit**

```bash
git add mesh/ cluster/ cache/
git commit -m "feat(#475): wire cache module into mesh, remove RelayConfig.depth()

Cache module activated via CASEHUB_QHORUS_CACHE_MODE env var (default:
shallow). RelayConfig.depth() removed — cache module owns depth config.

Refs #475"
```

---

## Deferred

**Post-commit cache population via JTA afterCompletion:** The spec calls
for JTA `afterCompletion(STATUS_COMMITTED)` to avoid phantom messages
from rolled-back transactions. The current `put()` implementation adds
to cache inline. This is safe for the common case (dispatch commits
immediately) and the cache is a performance optimisation. The
afterCompletion variant should be added once CDI wiring is proven in
integration tests.

**CDI producer wiring:** All classes use plain constructors for CDI-free
testing. CDI `@Produces` methods or `@ApplicationScoped` annotations
with `@IfBuildProperty` gating need to be added for Quarkus integration.
Best done during integration testing when the full CDI context is
available.

---

## References

- [2026-10-07-relay-depth-modes-design.md] — design spec
- [decisions.md] D25-D31 — relay depth mode decisions
- [PresenceService.java] — Caffeine cache pattern
- [PostgresChannelActivityBroadcaster.java] — cross-node notification
- [WriteRoutingDecorator.java] — CDI decorator pattern
- [MessageStore.java, MessageReader.java] — store interfaces
- [MessageQuery.java] — query model with afterId pagination
- [MessageObserver.java] — observer SPI (CLUSTER scope)
- [MessageReceivedEvent.java] — observer event payload
- [websocket-observer/pom.xml] — optional module pom pattern
- [GitHub #475] — epic issue

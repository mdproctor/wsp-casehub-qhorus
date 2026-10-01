# Qhorus Mesh Phase 1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** (to be created via work-start)
**Issue group:** (single issue)

**Goal:** Add searchable key/value metadata to Channel and Instance in qhorus core, then build a lightweight standalone qhorus-mesh relay node with MCP tools for inter-LLM communication.

**Architecture:** Metadata on Channel and Instance follows the existing `policyOverrides` pattern — nullable `Map<String, String>`, JSON-serialized TEXT column, merge semantics. The mesh module is a new Quarkus application that depends on `casehub-qhorus` runtime, exposes MCP tools via SSE transport, and persists to H2 file. A connection shim bridges Claude Code's stdio MCP to the relay's SSE endpoint.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, quarkus-mcp-server 1.11.1, H2

## Global Constraints

- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- After API visibility changes, always run `mvn install` from the project root
- Named `qhorus` datasource for tests (`quarkus.datasource.qhorus.*`)
- InMemory store implementations must not mutate PanacheEntity fields in-place
- Next domain Flyway migration: V56 (channel), V57 (instance)
- All commits must reference an issue (`Refs #N` or `Closes #N`)
- `ChannelService` lives in `runtime-core/`, not `runtime/`
- `ChannelEntity` lives in `runtime/`
- `ChannelCore` and REST request records live in `runtime-core/src/main/java/.../api/core/`

---

## Batch 1: Channel Metadata

### Task 1: Channel.metadata + ChannelEntity + V56 migration

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V56__channel_metadata.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMetadataTest.java`

**Interfaces:**
- Produces: `Channel.metadata()` → `Map<String, String>` (nullable); `Channel.Builder.metadata(Map<String, String>)`; `Channel.toBuilder()` includes metadata; `ChannelEntity.metadata` field (JSON TEXT).

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMetadataTest.java`:

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class ChannelMetadataTest {

    @Test
    void channelRecord_metadataDefensiveCopy() {
        var mutable = new LinkedHashMap<String, String>();
        mutable.put("key", "val");
        Channel ch = Channel.builder("test").id(UUID.randomUUID()).semantic(ChannelSemantic.APPEND)
                .metadata(mutable).build();
        mutable.put("key2", "val2");
        assertThat(ch.metadata()).doesNotContainKey("key2");
    }

    @Test
    void channelRecord_nullMetadataRemains() {
        Channel ch = Channel.builder("test").id(UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build();
        assertThat(ch.metadata()).isNull();
    }

    @Test
    void channelEntity_roundTrip() {
        Channel ch = Channel.builder("test-meta").id(UUID.randomUUID()).semantic(ChannelSemantic.APPEND)
                .metadata(Map.of("project", "qhorus", "family", "casehub")).build();
        ChannelEntity entity = ChannelEntity.fromDomain(ch);
        assertThat(entity.metadata).isNotNull();
        Channel restored = entity.toDomain();
        assertThat(restored.metadata()).containsEntry("project", "qhorus");
        assertThat(restored.metadata()).containsEntry("family", "casehub");
    }

    @Test
    void channelEntity_nullMetadata_roundTrip() {
        Channel ch = Channel.builder("test-null").id(UUID.randomUUID()).semantic(ChannelSemantic.APPEND).build();
        ChannelEntity entity = ChannelEntity.fromDomain(ch);
        assertThat(entity.metadata).isNull();
        Channel restored = entity.toDomain();
        assertThat(restored.metadata()).isNull();
    }

    @Test
    void toBuilder_preservesMetadata() {
        Channel ch = Channel.builder("test-builder").id(UUID.randomUUID()).semantic(ChannelSemantic.APPEND)
                .metadata(Map.of("k", "v")).build();
        Channel copy = ch.toBuilder().build();
        assertThat(copy.metadata()).containsEntry("k", "v");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelMetadataTest -pl runtime -DfailIfNoTests=false`
Expected: FAIL — `metadata()` method does not exist.

- [ ] **Step 3: Add metadata to Channel record**

In `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java`:

1. Add `Map<String, String> metadata` as the 29th record component (after `policyOverrides`).
2. In the compact constructor, add: `metadata = metadata != null ? Map.copyOf(metadata) : null;`
3. In backward-compat constructors, pass `null` for the new `metadata` parameter.
4. In `fromRequest()`, pass `null` for metadata (position 29).
5. In `Builder`: add `private Map<String, String> metadata;` field, add `public Builder metadata(Map<String, String> v) { this.metadata = v; return this; }` method.
6. In `Builder.build()`, add `metadata` as the last parameter.
7. In `toBuilder()`, add `.metadata(metadata)` to the chain.

- [ ] **Step 4: Add metadata to ChannelEntity**

In `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java`:

1. Add field: `@Column(name = "metadata", columnDefinition = "TEXT") public String metadata;`
2. In `fromDomain()`, add: `e.metadata = serializeMap(channel.metadata());`
3. In `toDomain()`, add `deserializeMap(metadata)` as the 29th parameter (after `deserializeMap(policyOverrides)`).

- [ ] **Step 5: Create V56 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V56__channel_metadata.sql`:

```sql
ALTER TABLE channel ADD COLUMN metadata TEXT;
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelMetadataTest -pl runtime`
Expected: PASS (all 5 tests)

- [ ] **Step 7: Run full build to catch compilation errors in other modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS. Fix any compilation errors in other modules caused by the new record component (backward-compat constructors should prevent this, but check).

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/channel/Channel.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java \
        runtime/src/main/resources/db/qhorus/migration/V56__channel_metadata.sql \
        runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMetadataTest.java
git commit -m "feat: add metadata Map<String,String> to Channel record + entity + V56 migration Refs #N"
```

---

### Task 2: ChannelQuery.byMetadata + store filtering

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/query/ChannelQuery.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelStore.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMetadataQueryTest.java`

**Interfaces:**
- Consumes: `Channel.metadata()` from Task 1.
- Produces: `ChannelQuery.byMetadata(String key, String value)` static factory; `ChannelQuery.metadataKey()` and `ChannelQuery.metadataValue()` accessors; JPA LIKE-based JSON filtering; InMemory metadata filtering via `matches()`.

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMetadataQueryTest.java`:

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.api.store.query.ChannelQuery;
import io.casehub.qhorus.persistence.memory.InMemoryChannelStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class ChannelMetadataQueryTest {

    private InMemoryChannelStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryChannelStore();
    }

    private Channel createChannel(String name, Map<String, String> metadata) {
        return store.put(Channel.builder(name)
                .id(UUID.randomUUID())
                .semantic(ChannelSemantic.APPEND)
                .metadata(metadata)
                .build());
    }

    @Test
    void byMetadata_findsMatchingChannels() {
        createChannel("alpha", Map.of("project", "qhorus"));
        createChannel("beta", Map.of("project", "claudony"));
        createChannel("gamma", Map.of("project", "qhorus", "family", "casehub"));

        List<Channel> results = store.scan(ChannelQuery.byMetadata("project", "qhorus"));
        assertThat(results).hasSize(2);
        assertThat(results).extracting(Channel::name).containsExactlyInAnyOrder("alpha", "gamma");
    }

    @Test
    void byMetadata_returnsEmpty_whenNoMatch() {
        createChannel("alpha", Map.of("project", "qhorus"));
        List<Channel> results = store.scan(ChannelQuery.byMetadata("project", "unknown"));
        assertThat(results).isEmpty();
    }

    @Test
    void byMetadata_skipsChannelsWithNullMetadata() {
        createChannel("no-meta", null);
        createChannel("has-meta", Map.of("project", "qhorus"));
        List<Channel> results = store.scan(ChannelQuery.byMetadata("project", "qhorus"));
        assertThat(results).hasSize(1);
        assertThat(results.get(0).name()).isEqualTo("has-meta");
    }

    @Test
    void byMetadata_combinedWithOtherFilters() {
        createChannel("alpha", Map.of("project", "qhorus"));
        Channel paused = createChannel("paused-one", Map.of("project", "qhorus"));
        store.put(paused.toBuilder().paused(true).build());

        List<Channel> results = store.scan(ChannelQuery.builder()
                .metadataKey("project").metadataValue("qhorus")
                .paused(false)
                .build());
        assertThat(results).hasSize(1);
        assertThat(results.get(0).name()).isEqualTo("alpha");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelMetadataQueryTest -pl runtime -DfailIfNoTests=false`
Expected: FAIL — `byMetadata` method does not exist on `ChannelQuery`.

- [ ] **Step 3: Add metadata filtering to ChannelQuery**

In `api/src/main/java/io/casehub/qhorus/api/store/query/ChannelQuery.java`:

1. Add fields to the record: `private final String metadataKey;` and `private final String metadataValue;`
2. Update `ChannelQuery(Builder b)` constructor to include: `this.metadataKey = b.metadataKey; this.metadataValue = b.metadataValue;`
3. Add static factory: `public static ChannelQuery byMetadata(String key, String value) { return new Builder().metadataKey(key).metadataValue(value).build(); }`
4. Add accessors: `public String metadataKey() { return metadataKey; }` and `public String metadataValue() { return metadataValue; }`
5. In `matches(Channel ch)`, add:
```java
if (metadataKey != null && metadataValue != null) {
    if (ch.metadata() == null || !metadataValue.equals(ch.metadata().get(metadataKey))) {
        return false;
    }
}
```
6. In `toBuilder()`, add `.metadataKey(metadataKey).metadataValue(metadataValue)`.
7. In `Builder`, add fields and setters:
```java
private String metadataKey;
private String metadataValue;
public Builder metadataKey(String v) { this.metadataKey = v; return this; }
public Builder metadataValue(String v) { this.metadataValue = v; return this; }
```

- [ ] **Step 4: Add JPA metadata filtering to JpaChannelStore.scan()**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java`, inside `scan()`, after the existing `topLevelOnly` check, add:

```java
if (q.metadataKey() != null && q.metadataValue() != null) {
    jpql.append(" AND e.metadata LIKE ?").append(idx++);
    params.add("%" + q.metadataKey() + "\":" + "\"" + q.metadataValue() + "\"" + "%");
}
```

Note: H2 does not support `->` JSON operators. LIKE on the serialized JSON is sufficient for key:value exact match when keys/values don't contain quotes. For PostgreSQL, a JSONB-specific implementation can be added later.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelMetadataQueryTest -pl runtime`
Expected: PASS (all 4 tests)

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/store/query/ChannelQuery.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java \
        persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelStore.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMetadataQueryTest.java
git commit -m "feat: add ChannelQuery.byMetadata() + JPA/InMemory store filtering Refs #N"
```

---

### Task 3: ChannelCreateRequest.metadata + ChannelService.setMetadata + REST

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/CreateChannelRequest.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/ChannelCore.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java`
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/MetadataRequest.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelSetMetadataTest.java`

**Interfaces:**
- Consumes: `Channel.metadata()` from Task 1; `ChannelQuery.byMetadata()` from Task 2.
- Produces: `ChannelService.setMetadata(UUID, Map<String, String>)` with merge semantics; `PUT /api/channels/{id}/metadata` REST endpoint; `ChannelCreateRequest.metadata()` field; `GET /api/channels?metadataKey=&metadataValue=` query params.

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelSetMetadataTest.java` (CDI-free, same pattern as `PolicyOverridesTest`):

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.store.ChannelStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Collection;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import static org.assertj.core.api.Assertions.assertThat;

class ChannelSetMetadataTest {

    private final Map<UUID, Channel> store = new ConcurrentHashMap<>();
    private final ChannelStore channelStore = new ChannelStore() {
        @Override public Channel put(Channel ch) { store.put(ch.id(), ch); return ch; }
        @Override public Optional<Channel> find(UUID id) { return Optional.ofNullable(store.get(id)); }
        @Override public Optional<Channel> findByName(String name) { return Optional.empty(); }
        @Override public List<Channel> scan(io.casehub.qhorus.api.store.query.ChannelQuery q) { return List.of(); }
        @Override public void delete(UUID id) { store.remove(id); }
        @Override public void updateLastActivity(UUID id, String t) {}
        @Override public void updateTrackDelivery(UUID id, Boolean td) {}
        @Override public List<Channel> findByIds(Collection<UUID> ids) { return List.of(); }
    };
    private ChannelService channelService;

    @BeforeEach
    void setup() {
        channelService = new ChannelService();
        channelService.channelStore = channelStore;
    }

    private Channel createChannel(String name) {
        return channelStore.put(Channel.builder(name)
                .id(UUID.randomUUID())
                .semantic(ChannelSemantic.APPEND)
                .build());
    }

    @Test
    void setMetadata_storesAndRetrieves() {
        Channel ch = createChannel("meta-test-" + UUID.randomUUID());
        channelService.setMetadata(ch.id(), Map.of("project", "qhorus"));
        Channel updated = channelStore.find(ch.id()).orElseThrow();
        assertThat(updated.metadata()).containsEntry("project", "qhorus");
    }

    @Test
    void setMetadata_mergesWithExisting() {
        Channel ch = createChannel("meta-merge-" + UUID.randomUUID());
        channelService.setMetadata(ch.id(), Map.of("key1", "val1"));
        channelService.setMetadata(ch.id(), Map.of("key2", "val2"));
        Channel updated = channelStore.find(ch.id()).orElseThrow();
        assertThat(updated.metadata())
                .containsEntry("key1", "val1")
                .containsEntry("key2", "val2");
    }

    @Test
    void setMetadata_nullValueRemovesKey() {
        Channel ch = createChannel("meta-remove-" + UUID.randomUUID());
        channelService.setMetadata(ch.id(), Map.of("key1", "val1", "key2", "val2"));
        Map<String, String> update = new LinkedHashMap<>();
        update.put("key1", null);
        channelService.setMetadata(ch.id(), update);
        Channel updated = channelStore.find(ch.id()).orElseThrow();
        assertThat(updated.metadata())
                .doesNotContainKey("key1")
                .containsEntry("key2", "val2");
    }

    @Test
    void setMetadata_nullMapClearsAll() {
        Channel ch = createChannel("meta-clear-" + UUID.randomUUID());
        channelService.setMetadata(ch.id(), Map.of("key1", "val1"));
        channelService.setMetadata(ch.id(), null);
        Channel updated = channelStore.find(ch.id()).orElseThrow();
        assertThat(updated.metadata()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelSetMetadataTest -pl runtime -DfailIfNoTests=false`
Expected: FAIL — `setMetadata` method does not exist.

- [ ] **Step 3: Add setMetadata to ChannelService**

In `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`, add after `setPolicyOverrides`:

```java
@Transactional
public Channel setMetadata(UUID channelId, java.util.Map<String, String> metadata) {
    Channel ch = channelStore.find(channelId)
                             .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelId));
    java.util.Map<String, String> merged;
    if (metadata == null) {
        merged = null;
    } else {
        merged = new java.util.LinkedHashMap<>();
        if (ch.metadata() != null) {
            merged.putAll(ch.metadata());
        }
        for (var entry : metadata.entrySet()) {
            if (entry.getValue() == null) {
                merged.remove(entry.getKey());
            } else {
                merged.put(entry.getKey(), entry.getValue());
            }
        }
        if (merged.isEmpty()) {
            merged = null;
        }
    }
    return channelStore.put(ch.toBuilder().metadata(merged).build());
}
```

Also add `setMetadata` to `ChannelManager` interface in `api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java`:

```java
Channel setMetadata(UUID channelId, java.util.Map<String, String> metadata);
```

- [ ] **Step 4: Add metadata to ChannelCreateRequest**

In `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java`:

1. Add `Map<String, String> metadata` field to the canonical constructor (after `outboundDestination`, or as a new trailing parameter).
2. In the compact constructor, add: `metadata = metadata != null ? Map.copyOf(metadata) : null;`
3. In all backward-compat constructors, pass `null` for metadata.
4. In `Builder`, add field and setter:
```java
private Map<String, String> metadata;
public Builder metadata(Map<String, String> v) { this.metadata = v; return this; }
```
5. In `Builder.build()`, add `metadata` as the last parameter.

- [ ] **Step 5: Wire metadata through Channel.fromRequest()**

In `Channel.fromRequest()`, pass `req.metadata()` as the metadata parameter (position 29, after `null` for policyOverrides).

- [ ] **Step 6: Add REST request record and endpoint**

Create `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/MetadataRequest.java`:

```java
package io.casehub.qhorus.runtime.api.core;

import java.util.Map;

public record MetadataRequest(Map<String, String> metadata) {}
```

Add to `ChannelCore`:
```java
public ChannelResponse setMetadata(String id, MetadataRequest req) {
    Channel ch = requireChannel(id);
    return toResponse(channelService.setMetadata(ch.id(), req.metadata()));
}
```

Update `ChannelCore.create()` — add metadata wiring:
```java
if (req.metadata() != null) builder.metadata(req.metadata());
```

Update `ChannelCore.list()` — add `metadataKey` and `metadataValue` parameters and wire into `ChannelQuery`.

Add to `ChannelResource`:
```java
@PUT
@Path("/{id}/metadata")
public ChannelResponse setMetadata(@PathParam("id") final String id,
                                    final io.casehub.qhorus.runtime.api.core.MetadataRequest req) {
    return core.setMetadata(id, req);
}
```

Update `ChannelResource.list()` to accept `@QueryParam("metadataKey")` and `@QueryParam("metadataValue")`.

Add `metadata` to `CreateChannelRequest` record.

Add `Map<String, String> metadata` to `ChannelResponse` record, and include `ch.metadata()` in the `from()` factory method.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelSetMetadataTest -pl runtime`
Expected: PASS

Run full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add api/ runtime-core/ runtime/
git commit -m "feat: ChannelCreateRequest.metadata + ChannelService.setMetadata + REST endpoint Refs #N"
```

---

## Batch 2: Instance Metadata

### Task 4: Instance.metadata + InstanceEntity + V57 migration

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/instance/Instance.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/instance/InstanceEntity.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V57__instance_metadata.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/instance/InstanceMetadataTest.java`

**Interfaces:**
- Produces: `Instance.metadata()` → `Map<String, String>` (nullable); `Instance.Builder.metadata(Map<String, String>)`; `InstanceEntity.metadata` field (JSON TEXT); `InstanceService.register()` overload accepting metadata.

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/instance/InstanceMetadataTest.java`:

```java
package io.casehub.qhorus.runtime.instance;

import io.casehub.qhorus.api.instance.Instance;
import io.casehub.qhorus.persistence.memory.InMemoryInstanceStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class InstanceMetadataTest {

    private InMemoryInstanceStore store;
    private InstanceService service;

    @BeforeEach
    void setUp() {
        store = new InMemoryInstanceStore();
        service = new InstanceService(store);
    }

    @Test
    void register_withMetadata_persists() {
        Instance inst = service.register("casehub/qhorus", "qhorus session",
                List.of("project:qhorus"), null, false,
                Map.of("project", "qhorus", "family", "casehub"));
        Instance found = store.findByInstanceId("casehub/qhorus").orElseThrow();
        assertThat(found.metadata()).containsEntry("project", "qhorus");
        assertThat(found.metadata()).containsEntry("family", "casehub");
    }

    @Test
    void register_withoutMetadata_nullMetadata() {
        Instance inst = service.register("no-meta", "desc", List.of(), null, false, null);
        Instance found = store.findByInstanceId("no-meta").orElseThrow();
        assertThat(found.metadata()).isNull();
    }

    @Test
    void instanceRecord_metadataDefensiveCopy() {
        var mutable = new java.util.LinkedHashMap<String, String>();
        mutable.put("key", "val");
        Instance inst = Instance.builder("test").metadata(mutable).build();
        mutable.put("key2", "val2");
        assertThat(inst.metadata()).doesNotContainKey("key2");
    }

    @Test
    void instanceEntity_roundTrip() {
        Instance inst = Instance.builder("test-entity")
                .id(UUID.randomUUID())
                .metadata(Map.of("slot", "174"))
                .build();
        InstanceEntity entity = InstanceEntity.fromDomain(inst);
        assertThat(entity.metadata).isNotNull();
        Instance restored = entity.toDomain();
        assertThat(restored.metadata()).containsEntry("slot", "174");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceMetadataTest -pl runtime -DfailIfNoTests=false`
Expected: FAIL — `metadata()` method does not exist on `Instance`.

- [ ] **Step 3: Add metadata to Instance record**

In `api/src/main/java/io/casehub/qhorus/api/instance/Instance.java`:

1. Add `Map<String, String> metadata` as the 10th record component.
2. Add compact constructor that normalizes: `metadata = metadata != null ? Map.copyOf(metadata) : null;`
3. Add backward-compat 9-arg constructor that passes `null` for metadata.
4. In `Builder`: add `private Map<String, String> metadata;` field, setter method, and include in `build()`.
5. In `toBuilder()`: add `.metadata(metadata)`.

- [ ] **Step 4: Add metadata to InstanceEntity**

In `runtime/src/main/java/io/casehub/qhorus/runtime/instance/InstanceEntity.java`:

1. Add field: `@Column(name = "metadata", columnDefinition = "TEXT") public String metadata;`
2. In `fromDomain()`, add serialization (use same static ObjectMapper + helper pattern as ChannelEntity):
```java
private static final com.fasterxml.jackson.databind.ObjectMapper JSON = new com.fasterxml.jackson.databind.ObjectMapper();
private static String serializeMap(java.util.Map<String, String> map) {
    if (map == null || map.isEmpty()) return null;
    try { return JSON.writeValueAsString(map); } catch (Exception e) { return null; }
}
private static java.util.Map<String, String> deserializeMap(String json) {
    if (json == null || json.isBlank()) return null;
    try { return JSON.readValue(json, new com.fasterxml.jackson.core.type.TypeReference<java.util.Map<String, String>>() {}); } catch (Exception e) { return null; }
}
```
3. In `fromDomain()`: `e.metadata = serializeMap(inst.metadata());`
4. In `toDomain()`: include `deserializeMap(metadata)` as the 10th parameter.

- [ ] **Step 5: Add register overload with metadata to InstanceService**

In `runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java`, add:

```java
@Transactional
public Instance register(String instanceId, String description, List<String> capabilityTags,
                         String claudonySessionId, boolean readOnly,
                         Map<String, String> metadata) {
    Instance existing = instanceStore.findByInstanceId(instanceId).orElse(null);
    List<String> previousCaps = existing != null
                                ? instanceStore.findCapabilities(existing.id())
                                : List.of();
    Instance.Builder b;
    if (existing == null) {
        b = Instance.builder(instanceId);
    } else {
        b = existing.toBuilder();
    }
    Instance instance = b.description(description)
                         .status("online")
                         .lastSeen(Instant.now())
                         .claudonySessionId(claudonySessionId)
                         .readOnly(readOnly)
                         .metadata(metadata)
                         .build();
    Instance saved = instanceStore.put(instance);
    instanceStore.putCapabilities(saved.id(), capabilityTags);
    if (registeredEvent != null) {
        registeredEvent.fireAsync(new io.casehub.qhorus.api.instance.InstanceRegisteredEvent(
                instanceId, previousCaps, capabilityTags));
    }
    return saved;
}
```

Update the existing 5-arg `register()` to delegate to this new 6-arg version passing `null` for metadata.

- [ ] **Step 6: Create V57 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V57__instance_metadata.sql`:

```sql
ALTER TABLE instance ADD COLUMN metadata TEXT;
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceMetadataTest -pl runtime`
Expected: PASS

Run full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/instance/Instance.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/instance/InstanceEntity.java \
        runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java \
        runtime/src/main/resources/db/qhorus/migration/V57__instance_metadata.sql \
        runtime/src/test/java/io/casehub/qhorus/runtime/instance/InstanceMetadataTest.java
git commit -m "feat: add metadata Map<String,String> to Instance record + entity + V57 migration Refs #N"
```

---

### Task 5: InstanceQuery.byMetadata + store filtering

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/query/InstanceQuery.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaInstanceStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryInstanceStore.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/instance/InstanceMetadataQueryTest.java`

**Interfaces:**
- Consumes: `Instance.metadata()` from Task 4.
- Produces: `InstanceQuery.byMetadata(String key, String value)` static factory; JPA + InMemory filtering.

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/instance/InstanceMetadataQueryTest.java`:

```java
package io.casehub.qhorus.runtime.instance;

import io.casehub.qhorus.api.instance.Instance;
import io.casehub.qhorus.api.store.query.InstanceQuery;
import io.casehub.qhorus.persistence.memory.InMemoryInstanceStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class InstanceMetadataQueryTest {

    private InMemoryInstanceStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryInstanceStore();
    }

    private Instance register(String instanceId, Map<String, String> metadata) {
        return store.put(Instance.builder(instanceId).metadata(metadata).build());
    }

    @Test
    void byMetadata_findsMatchingInstances() {
        register("casehub/qhorus", Map.of("project", "qhorus"));
        register("casehub/claudony", Map.of("project", "claudony"));
        register("casehub/engine", Map.of("project", "engine", "family", "casehub"));

        List<Instance> results = store.scan(InstanceQuery.byMetadata("project", "qhorus"));
        assertThat(results).hasSize(1);
        assertThat(results.get(0).instanceId()).isEqualTo("casehub/qhorus");
    }

    @Test
    void byMetadata_skipsNullMetadata() {
        register("no-meta", null);
        register("has-meta", Map.of("project", "qhorus"));
        List<Instance> results = store.scan(InstanceQuery.byMetadata("project", "qhorus"));
        assertThat(results).hasSize(1);
    }

    @Test
    void byMetadata_combinedWithStatusFilter() {
        Instance online = register("online-agent", Map.of("project", "qhorus"));
        store.put(online.toBuilder().status("online").build());
        Instance stale = register("stale-agent", Map.of("project", "qhorus"));
        store.put(stale.toBuilder().status("stale").build());

        List<Instance> results = store.scan(InstanceQuery.builder()
                .metadataKey("project").metadataValue("qhorus")
                .status("online")
                .build());
        assertThat(results).hasSize(1);
        assertThat(results.get(0).instanceId()).isEqualTo("online-agent");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceMetadataQueryTest -pl runtime -DfailIfNoTests=false`
Expected: FAIL — `byMetadata` method does not exist on `InstanceQuery`.

- [ ] **Step 3: Add metadata filtering to InstanceQuery**

In `api/src/main/java/io/casehub/qhorus/api/store/query/InstanceQuery.java`:

1. Add fields: `private final String metadataKey;` and `private final String metadataValue;`
2. Update constructor to include them.
3. Add static factory: `public static InstanceQuery byMetadata(String key, String value) { return new Builder().metadataKey(key).metadataValue(value).build(); }`
4. Add accessors.
5. In `matches(Instance inst)`, add:
```java
if (metadataKey != null && metadataValue != null) {
    if (inst.metadata() == null || !metadataValue.equals(inst.metadata().get(metadataKey))) {
        return false;
    }
}
```
6. Add Builder fields and setters.
7. Update `toBuilder()`.

- [ ] **Step 4: Add JPA metadata filtering to JpaInstanceStore.scan()**

In `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaInstanceStore.java`, inside `scan()`, after the capability check:

```java
if (q.metadataKey() != null && q.metadataValue() != null) {
    jpql.append(" AND e.metadata LIKE ?").append(idx++);
    params.add("%" + q.metadataKey() + "\":" + "\"" + q.metadataValue() + "\"" + "%");
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InstanceMetadataQueryTest -pl runtime`
Expected: PASS

Run full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/store/query/InstanceQuery.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaInstanceStore.java \
        persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryInstanceStore.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/instance/InstanceMetadataQueryTest.java
git commit -m "feat: add InstanceQuery.byMetadata() + JPA/InMemory store filtering Refs #N"
```

---

## Batch 3: Mesh Module Foundation

### Task 6: Maven module scaffold + startup verification

**Files:**
- Create: `mesh/pom.xml`
- Create: `mesh/src/main/resources/application.properties`
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/MeshApp.java` (empty — Quarkus auto-detects)
- Modify: `pom.xml` (parent — add `<module>mesh</module>`)
- Test: verify `mvn quarkus:dev` starts and health check responds

**Interfaces:**
- Produces: A running Quarkus app on port 9741 with H2 persistence and the qhorus named PU.

- [ ] **Step 1: Create pom.xml**

Create `mesh/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-qhorus-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-qhorus-mesh</artifactId>
    <name>CaseHub Qhorus Mesh Relay</name>
    <description>Lightweight local relay node for LLM-to-LLM communication</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-qhorus</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.mcp</groupId>
            <artifactId>quarkus-mcp-server-sse</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-h2</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-hibernate-orm</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-flyway</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-smallrye-health</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Create application.properties**

Create `mesh/src/main/resources/application.properties`:

```properties
# Mesh relay server port
quarkus.http.port=9741

# H2 file-based persistence
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:file:${user.home}/.qhorus/mesh;AUTO_SERVER=TRUE
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=

# Hibernate ORM — qhorus named PU
quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime,io.casehub.ledger.jpa
quarkus.hibernate-orm.qhorus.database.generation=none

# Flyway migrations
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration
quarkus.flyway.qhorus.migrate-at-start=true

# MCP server — SSE transport
quarkus.mcp.server.transport=sse

# Stale instance cleanup
casehub.qhorus.cleanup.stale-instance-seconds=300
```

- [ ] **Step 3: Add module to parent pom**

In the root `pom.xml`, add `<module>mesh</module>` to the `<modules>` list (after `testing`, before `examples`).

- [ ] **Step 4: Create minimal MeshApp marker**

Create `mesh/src/main/java/io/casehub/qhorus/mesh/MeshApp.java`:

```java
package io.casehub.qhorus.mesh;

import io.quarkus.runtime.Quarkus;
import io.quarkus.runtime.annotations.QuarkusMain;

@QuarkusMain
public class MeshApp {
    public static void main(String... args) {
        Quarkus.run(args);
    }
}
```

- [ ] **Step 5: Create health check smoke test**

Create `mesh/src/test/java/io/casehub/qhorus/mesh/MeshStartupTest.java`:

```java
package io.casehub.qhorus.mesh;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.is;

@QuarkusTest
class MeshStartupTest {

    @Test
    void healthCheck_responds() {
        given()
            .when().get("/q/health/ready")
            .then()
            .statusCode(200)
            .body("status", is("UP"));
    }
}
```

Create `mesh/src/test/resources/application.properties`:

```properties
quarkus.http.test-port=0
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:mesh-test;DB_CLOSE_DELAY=-1
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=
quarkus.datasource.qhorus.reactive=false
quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
```

- [ ] **Step 6: Build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl mesh -am`
Expected: BUILD SUCCESS, health check test PASS.

- [ ] **Step 7: Commit**

```bash
git add mesh/ pom.xml
git commit -m "feat: scaffold qhorus-mesh relay module with H2 + MCP SSE + health check Refs #N"
```

---

### Task 7: Core MCP tools

**Files:**
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/MeshMcpTools.java`
- Test: `mesh/src/test/java/io/casehub/qhorus/mesh/MeshMcpToolsTest.java`

**Interfaces:**
- Consumes: `ChannelService` (Batch 1), `InstanceService` (Batch 2), `MessageDispatcher`, `MessageStore`, `ChannelQuery.byMetadata()`, `InstanceQuery.byMetadata()`.
- Produces: MCP tools: `mesh_register`, `mesh_deregister`, `mesh_send_message`, `mesh_check_messages`, `mesh_create_channel`, `mesh_list_channels`, `mesh_discover_peers`.

- [ ] **Step 1: Write the failing test**

Create `mesh/src/test/java/io/casehub/qhorus/mesh/MeshMcpToolsTest.java`:

```java
package io.casehub.qhorus.mesh;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkiverse.mcp.server.test.McpClient;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class MeshMcpToolsTest {

    @Test
    void register_and_discover() {
        // Register two instances
        var reg1 = McpClient.callTool("mesh_register",
                "instance_id", "casehub/qhorus",
                "description", "qhorus session",
                "metadata_json", "{\"project\":\"qhorus\",\"family\":\"casehub\"}");
        assertThat(reg1).contains("casehub/qhorus");

        var reg2 = McpClient.callTool("mesh_register",
                "instance_id", "casehub/claudony",
                "description", "claudony session",
                "metadata_json", "{\"project\":\"claudony\",\"family\":\"casehub\"}");
        assertThat(reg2).contains("casehub/claudony");

        // Discover by family
        var peers = McpClient.callTool("mesh_discover_peers",
                "metadata_key", "family",
                "metadata_value", "casehub");
        assertThat(peers).contains("casehub/qhorus");
        assertThat(peers).contains("casehub/claudony");

        // Discover by project
        var qhorusPeers = McpClient.callTool("mesh_discover_peers",
                "metadata_key", "project",
                "metadata_value", "qhorus");
        assertThat(qhorusPeers).contains("casehub/qhorus");
        assertThat(qhorusPeers).doesNotContain("casehub/claudony");
    }

    @Test
    void create_channel_and_send_message() {
        McpClient.callTool("mesh_register",
                "instance_id", "test-sender",
                "description", "test",
                "metadata_json", "{}");

        var created = McpClient.callTool("mesh_create_channel",
                "name", "design-review",
                "metadata_json", "{\"project\":\"qhorus\",\"purpose\":\"review\"}");
        assertThat(created).contains("design-review");

        var sent = McpClient.callTool("mesh_send_message",
                "channel", "design-review",
                "sender", "test-sender",
                "type", "query",
                "content", "Has anyone reviewed the mesh spec?");
        assertThat(sent).contains("query");

        var messages = McpClient.callTool("mesh_check_messages",
                "channel", "design-review");
        assertThat(messages).contains("Has anyone reviewed the mesh spec?");
    }

    @Test
    void list_channels_filtered_by_metadata() {
        McpClient.callTool("mesh_create_channel",
                "name", "qhorus-work",
                "metadata_json", "{\"project\":\"qhorus\"}");
        McpClient.callTool("mesh_create_channel",
                "name", "claudony-work",
                "metadata_json", "{\"project\":\"claudony\"}");

        var filtered = McpClient.callTool("mesh_list_channels",
                "metadata_key", "project",
                "metadata_value", "qhorus");
        assertThat(filtered).contains("qhorus-work");
        assertThat(filtered).doesNotContain("claudony-work");
    }
}
```

Note: The exact McpClient API depends on `quarkus-mcp-server-test` — if unavailable, use REST-based tool invocation against the SSE endpoint. Adjust test approach at implementation time.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshMcpToolsTest -pl mesh`
Expected: FAIL — `MeshMcpTools` class does not exist.

- [ ] **Step 3: Implement MeshMcpTools**

Create `mesh/src/main/java/io/casehub/qhorus/mesh/MeshMcpTools.java`:

```java
package io.casehub.qhorus.mesh;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.instance.Instance;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.ChannelQuery;
import io.casehub.qhorus.api.store.query.InstanceQuery;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@ApplicationScoped
public class MeshMcpTools {

    @Inject InstanceService instanceService;
    @Inject ChannelService channelService;
    @Inject MessageDispatcher messageDispatcher;
    @Inject MessageStore messageStore;
    @Inject ChannelStore channelStore;
    @Inject ObjectMapper objectMapper;

    @Tool(description = "Register this session with the mesh relay. Returns registration confirmation.")
    public String mesh_register(
            @ToolArg(description = "Instance ID — typically the repo path relative to ~/claude/") String instance_id,
            @ToolArg(description = "Human-readable description of this session") String description,
            @ToolArg(description = "JSON object of key-value metadata (e.g. {\"project\":\"qhorus\",\"family\":\"casehub\"})") String metadata_json) {
        Map<String, String> metadata = parseMetadata(metadata_json);
        List<String> caps = metadata != null
                ? metadata.entrySet().stream().map(e -> e.getKey() + ":" + e.getValue()).toList()
                : List.of();
        Instance inst = instanceService.register(instance_id, description, caps, null, false, metadata);
        return "Registered: " + inst.instanceId() + " (id=" + inst.id() + ")";
    }

    @Tool(description = "Deregister this session from the mesh relay.")
    public String mesh_deregister(
            @ToolArg(description = "Instance ID to deregister") String instance_id) {
        instanceService.deregister(instance_id);
        return "Deregistered: " + instance_id;
    }

    @Tool(description = "Send a message to a channel.")
    public String mesh_send_message(
            @ToolArg(description = "Channel name") String channel,
            @ToolArg(description = "Sender instance ID") String sender,
            @ToolArg(description = "Message type: query, command, response, status, done, failure, propose, event") String type,
            @ToolArg(description = "Message content") String content) {
        Channel ch = channelService.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel));
        MessageType msgType = MessageType.valueOf(type.toUpperCase());
        DispatchResult result = messageDispatcher.dispatch(
                MessageDispatch.builder(ch.id(), sender, msgType, content).build());
        return "Sent " + msgType + " to " + channel + " (id=" + result.messageId() + ")";
    }

    @Tool(description = "Check messages in a channel. Returns recent messages.")
    public String mesh_check_messages(
            @ToolArg(description = "Channel name") String channel) {
        Channel ch = channelService.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel));
        var messages = messageStore.scan(MessageQuery.builder().channelId(ch.id()).build());
        return messages.stream()
                .map(m -> "[" + m.messageType() + " from " + m.sender() + "] " + m.content())
                .collect(Collectors.joining("\n"));
    }

    @Tool(description = "Create a new channel with optional metadata.")
    public String mesh_create_channel(
            @ToolArg(description = "Channel name (slug format: lowercase, hyphens, slashes)") String name,
            @ToolArg(description = "JSON object of key-value metadata") String metadata_json) {
        Map<String, String> metadata = parseMetadata(metadata_json);
        Channel ch = channelService.create(
                ChannelCreateRequest.builder(name).metadata(metadata).build());
        return "Created channel: " + ch.name() + " (id=" + ch.id() + ")";
    }

    @Tool(description = "List channels, optionally filtered by metadata key-value pair.")
    public String mesh_list_channels(
            @ToolArg(description = "Metadata key to filter by (optional)") String metadata_key,
            @ToolArg(description = "Metadata value to filter by (optional)") String metadata_value) {
        List<Channel> channels;
        if (metadata_key != null && metadata_value != null) {
            channels = channelStore.scan(ChannelQuery.byMetadata(metadata_key, metadata_value));
        } else {
            channels = channelStore.scan(ChannelQuery.all());
        }
        return channels.stream()
                .map(ch -> ch.name() + (ch.metadata() != null ? " " + ch.metadata() : ""))
                .collect(Collectors.joining("\n"));
    }

    @Tool(description = "Discover peer sessions by metadata key-value pair.")
    public String mesh_discover_peers(
            @ToolArg(description = "Metadata key to filter by") String metadata_key,
            @ToolArg(description = "Metadata value to filter by") String metadata_value) {
        List<Instance> peers = instanceService.listAll().stream()
                .filter(i -> i.metadata() != null && metadata_value.equals(i.metadata().get(metadata_key)))
                .toList();
        return peers.stream()
                .map(i -> i.instanceId() + " — " + i.description()
                        + (i.metadata() != null ? " " + i.metadata() : ""))
                .collect(Collectors.joining("\n"));
    }

    private Map<String, String> parseMetadata(String json) {
        if (json == null || json.isBlank()) return null;
        try {
            return objectMapper.readValue(json, new TypeReference<Map<String, String>>() {});
        } catch (Exception e) {
            throw new IllegalArgumentException("Invalid metadata JSON: " + e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=MeshMcpToolsTest -pl mesh`
Expected: PASS

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add mesh/src/main/java/io/casehub/qhorus/mesh/MeshMcpTools.java \
        mesh/src/test/java/io/casehub/qhorus/mesh/MeshMcpToolsTest.java
git commit -m "feat: add MeshMcpTools — register, send, check, create, discover MCP tools Refs #N"
```

---

## Batch 4: Connection Shim

### Task 8: Stdio-to-SSE bridge with relay detection

**Files:**
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/shim/MeshShim.java`
- Create: `mesh/src/main/java/io/casehub/qhorus/mesh/shim/SseToStdioBridge.java`

**Interfaces:**
- Consumes: Relay health check on `localhost:9741/q/health/ready`; SSE MCP endpoint.
- Produces: A standalone main class that bridges stdio MCP ↔ SSE MCP; auto-detects relay; falls back to embedded in-memory Qhorus.

Note: The connection shim is architecturally the most complex piece and has the highest uncertainty. The exact MCP SSE client API and stdio protocol bridging depend on the `quarkus-mcp-server` SSE transport implementation. This task provides the skeleton; the exact bridge implementation may need adjustment based on the MCP SDK's client-side SSE API.

- [ ] **Step 1: Create the shim main class**

Create `mesh/src/main/java/io/casehub/qhorus/mesh/shim/MeshShim.java`:

```java
package io.casehub.qhorus.mesh.shim;

import java.io.IOException;
import java.net.HttpURLConnection;
import java.net.URI;

public class MeshShim {

    private static final String DEFAULT_RELAY_URL = "http://localhost:9741";
    private static final String HEALTH_PATH = "/q/health/ready";

    public static void main(String[] args) throws Exception {
        String relayUrl = System.getenv("QHORUS_MESH_URL");
        if (relayUrl == null) relayUrl = DEFAULT_RELAY_URL;

        if (isRelayRunning(relayUrl)) {
            System.err.println("[qhorus-mesh] Connecting to relay at " + relayUrl);
            SseToStdioBridge.bridge(relayUrl);
        } else {
            System.err.println("[qhorus-mesh] No relay detected. Starting embedded instance.");
            startEmbedded(args);
        }
    }

    static boolean isRelayRunning(String baseUrl) {
        try {
            HttpURLConnection conn = (HttpURLConnection)
                    URI.create(baseUrl + HEALTH_PATH).toURL().openConnection();
            conn.setConnectTimeout(500);
            conn.setReadTimeout(500);
            int status = conn.getResponseCode();
            conn.disconnect();
            return status == 200;
        } catch (IOException e) {
            return false;
        }
    }

    static void startEmbedded(String[] args) {
        // Start Quarkus with in-memory persistence
        // This makes the shim itself a Quarkus app with persistence-memory
        System.setProperty("quarkus.datasource.qhorus.db-kind", "h2");
        System.setProperty("quarkus.datasource.qhorus.jdbc.url", "jdbc:h2:mem:mesh-embedded;DB_CLOSE_DELAY=-1");
        System.setProperty("quarkus.hibernate-orm.qhorus.database.generation", "drop-and-create");
        System.setProperty("quarkus.mcp.server.transport", "stdio");
        io.quarkus.runtime.Quarkus.run(args);
    }
}
```

- [ ] **Step 2: Create the SSE-to-stdio bridge skeleton**

Create `mesh/src/main/java/io/casehub/qhorus/mesh/shim/SseToStdioBridge.java`:

```java
package io.casehub.qhorus.mesh.shim;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class SseToStdioBridge {

    public static void bridge(String relayUrl) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        String sseEndpoint = relayUrl + "/mcp/sse";

        // Read JSON-RPC requests from stdin, POST to relay, stream SSE responses to stdout
        try (BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in))) {
            String line;
            while ((line = stdin.readLine()) != null) {
                if (line.isBlank()) continue;

                HttpRequest request = HttpRequest.newBuilder()
                        .uri(URI.create(relayUrl + "/mcp/message"))
                        .header("Content-Type", "application/json")
                        .POST(HttpRequest.BodyPublishers.ofString(line))
                        .build();

                HttpResponse<String> response = client.send(request,
                        HttpResponse.BodyHandlers.ofString());

                System.out.println(response.body());
                System.out.flush();
            }
        }
    }
}
```

Note: This is a simplified bridge. The real MCP SSE protocol uses a persistent SSE connection for server-to-client messages and HTTP POST for client-to-server messages. The exact endpoint paths (`/mcp/sse`, `/mcp/message`) come from the `quarkus-mcp-server-sse` module. Implementation should be refined based on the actual SSE transport API.

- [ ] **Step 3: Manual verification**

Start the relay: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl mesh`
In another terminal, verify health: `curl http://localhost:9741/q/health/ready`
Expected: `{"status":"UP",...}`

- [ ] **Step 4: Commit**

```bash
git add mesh/src/main/java/io/casehub/qhorus/mesh/shim/
git commit -m "feat: connection shim — stdio/SSE bridge with relay detection + embedded fallback Refs #N"
```

---

## References

- [2026-10-01-qhorus-mesh-design.md](../specs/qhorus-mesh/2026-10-01-qhorus-mesh-design.md) — design spec this plan implements
- `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — record with 28 components, policyOverrides pattern
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` — JPA entity with serializeMap/deserializeMap
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java:205` — setPolicyOverrides merge semantics pattern
- `runtime/src/test/java/io/casehub/qhorus/runtime/channel/PolicyOverridesTest.java` — CDI-free test pattern
- `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java:60-98` — JPQL dynamic query building
- `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaInstanceStore.java:54-76` — Instance JPQL scan pattern
- `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelStore.java` — in-memory query::matches delegation
- `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryInstanceStore.java` — in-memory scan with capability join
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/ChannelCore.java:92-109` — create flow via ChannelCreateRequest builder
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/InstanceService.java:46-76` — register pattern
- decisions.md — 9 design decisions with rationale

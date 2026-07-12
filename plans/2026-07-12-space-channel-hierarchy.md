# Space — Organizational Channel Hierarchy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #334 — feat: Space — organizational channel hierarchy
**Issue group:** #334

**Goal:** Add recursive Space containers for channels, replacing naming-convention-based grouping with an explicit hierarchy.

**Architecture:** Space is a first-class entity with adjacency-list nesting (parentSpaceId). Channels gain an optional spaceId FK. Substantial partial implementation exists — this plan modifies existing code more than it creates new code. The metadata field on Space is deliberately removed (no production consumers).

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, H2 (tests), PostgreSQL (production)

## Global Constraints

- Pre-release: breaking changes cost nothing. Fix the design, don't protect callers.
- IntelliJ MCP mandatory for all .java file operations.
- All methods use `CurrentPrincipal.tenancyId()` for tenancy scoping — never pass tenancyId as a parameter from external callers.
- Space names: free-form text (not slug-constrained), max 200 chars, UUID-shaped rejected, unique within parent scope.
- MAX_DEPTH = 10 for recursive nesting.
- Delete guard: spaces with children or channels cannot be deleted.
- Next Flyway version: V33 already exists (replace in-place). V34 already exists (replace in-place).

---

### Task 1: API contracts + store implementations + migration

This task breaks the existing API (removes `metadata` from Space), renames store methods for consistency, adds new store methods, and updates all implementations (JPA, InMemory, reactive) to match. It also replaces the Flyway migrations with the correct partial-index schema.

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Space.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/SpaceCreateRequest.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/SpaceStore.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveSpaceStore.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ChannelStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/SpaceEntity.java`
- Delete: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/MetadataMapConverter.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaSpaceStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaSpaceStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemorySpaceStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveSpaceStore.java`
- Modify: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/SpaceStoreContractTest.java`
- Modify: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemorySpaceStoreTest.java`
- Modify: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactiveSpaceStoreTest.java`
- Modify: `runtime/src/main/resources/db/qhorus/migration/V33__space.sql`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/SpaceStoreContractTest.java`

**Interfaces:**
- Produces: `Space(UUID id, String name, String description, UUID parentSpaceId, String tenancyId, Instant createdAt)` — 6-field record (was 7 with metadata)
- Produces: `SpaceCreateRequest(String name, String description, UUID parentSpaceId)` — validated request record
- Produces: `SpaceStore` — `put`, `find(UUID)`, `findByName(String)→List<Space>`, `listByParent(UUID)`, `listRoots()`, `hasChildren(UUID)`, `delete(UUID)`, `findByIds(Collection<UUID>)`
- Produces: `ChannelStore.hasChannelsInSpace(UUID spaceId)` — existence check
- Produces: `ChannelDetail` — gains `spaceName: String?` (field 16, before connectorBinding)

- [ ] **Step 1: Update Space record — remove metadata**

Use `ide_edit_member` to replace the `Space` record declaration, removing the `metadata` field and its compact constructor defensive copy.

```java
public record Space(
        UUID id,
        String name,
        String description,
        UUID parentSpaceId,
        String tenancyId,
        Instant createdAt) {

    public Space {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("name is required");
    }
}
```

Remove the `import java.util.Map;` that is no longer needed.

- [ ] **Step 2: Create SpaceCreateRequest**

Create new file `api/src/main/java/io/casehub/qhorus/api/channel/SpaceCreateRequest.java`:

```java
package io.casehub.qhorus.api.channel;

import java.util.UUID;

public record SpaceCreateRequest(
        String name,
        String description,
        UUID parentSpaceId
) {
    public SpaceCreateRequest {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Space name must not be blank");
        }
        name = name.trim();
        if (name.length() > 200) {
            throw new IllegalArgumentException("Space name exceeds 200 chars: '" + name + "'");
        }
        boolean isUuid;
        try { UUID.fromString(name); isUuid = true; }
        catch (IllegalArgumentException ignored) { isUuid = false; }
        if (isUuid) {
            throw new IllegalArgumentException("Space name must not be UUID-shaped: '" + name + "'");
        }
        if (description != null) description = description.trim();
    }
}
```

- [ ] **Step 3: Update SpaceStore interface**

Use `ide_edit_member` to replace the `SpaceStore` interface declaration:

```java
public interface SpaceStore {
    Space put(Space space);
    Optional<Space> find(UUID id);
    List<Space> findByName(String name);
    List<Space> listByParent(UUID parentSpaceId);
    List<Space> listRoots();
    boolean hasChildren(UUID spaceId);
    void delete(UUID id);

    default List<Space> findByIds(Collection<UUID> ids) {
        if (ids == null || ids.isEmpty()) return List.of();
        return ids.stream()
                .map(this::find)
                .filter(Optional::isPresent)
                .map(Optional::get)
                .toList();
    }
}
```

Add `import java.util.Collection;` to imports.

- [ ] **Step 4: Update ReactiveSpaceStore interface**

Use `ide_edit_member` to replace the full interface:

```java
public interface ReactiveSpaceStore {
    Uni<Space> put(Space space);
    Uni<Optional<Space>> find(UUID id);
    Uni<List<Space>> findByName(String name);
    Uni<List<Space>> listByParent(UUID parentSpaceId);
    Uni<List<Space>> listRoots();
    Uni<Boolean> hasChildren(UUID spaceId);
    Uni<Void> delete(UUID id);
    Uni<List<Space>> findByIds(Collection<UUID> ids);
}
```

Add `import java.util.Collection;`.

- [ ] **Step 5: Add hasChannelsInSpace to ChannelStore**

Use `ide_insert_member` to add after `updateLastActivity`:

```java
default boolean hasChannelsInSpace(UUID spaceId) {
    if (spaceId == null) return false;
    return !scan(io.casehub.qhorus.api.store.query.ChannelQuery.bySpaceId(spaceId)).isEmpty();
}
```

- [ ] **Step 6: Add spaceName to ChannelDetail**

Use `ide_edit_member` to replace the `ChannelDetail` record declaration, adding `String spaceName` as field 16 (after `spaceId`, before `connectorBinding`):

```java
public record ChannelDetail(
        UUID channelId,
        String name,
        String description,
        String semantic,
        String barrierContributors,
        long messageCount,
        String lastActivityAt,
        boolean paused,
        String allowedWriters,
        String adminInstances,
        Integer rateLimitPerChannel,
        Integer rateLimitPerInstance,
        String allowedTypes,
        String deniedTypes,
        UUID spaceId,
        String spaceName,
        ConnectorBinding connectorBinding) {

    public record ConnectorBinding(
            String inboundConnectorId,
            String externalKey,
            String outboundConnectorId,
            String outboundDestination) {}
}
```

- [ ] **Step 7: Update QhorusEntityMapper.toChannelDetail**

Use `ide_edit_member` to update `toChannelDetail` to accept `spaceName`:

```java
public ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                     Optional<ChannelConnectorBinding> binding,
                                     String spaceName) {
    ChannelDetail.ConnectorBinding detailBinding = binding
                                                           .map(b -> new ChannelDetail.ConnectorBinding(
                                                                   b.inboundConnectorId(), b.externalKey(),
                                                                   b.outboundConnectorId(), b.outboundDestination()))
                                                           .orElse(null);
    return new ChannelDetail(
            ch.id(), ch.name(), ch.description(),
            ch.semantic() != null ? ch.semantic().name() : null,
            joinCsv(ch.barrierContributors()), messageCount,
            ch.lastActivityAt() != null ? ch.lastActivityAt().toString() : null,
            ch.paused(), joinCsv(ch.allowedWriters()), joinCsv(ch.adminInstances()),
            ch.rateLimitPerChannel(), ch.rateLimitPerInstance(),
            ch.allowedTypes() != null ? MessageType.serializeTypes(ch.allowedTypes()) : null,
            ch.deniedTypes() != null ? MessageType.serializeTypes(ch.deniedTypes()) : null,
            ch.spaceId(),
            spaceName,
            detailBinding);
}
```

Also add backward-compatible 3-arg overload (no spaceName) that passes null:

```java
public ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                     Optional<ChannelConnectorBinding> binding) {
    return toChannelDetail(ch, messageCount, binding, null);
}
```

- [ ] **Step 8: Update SpaceEntity — remove metadata**

Use `ide_edit_member` to replace the `SpaceEntity` class. Remove the `metadata` field, `MetadataMapConverter` usage, and update `fromDomain()`/`toDomain()`. Drop the `@UniqueConstraint` annotation (partial indexes in migration):

```java
@Entity(name = "Space")
@Table(name = "space")
public class SpaceEntity extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(nullable = false)
    public String name;

    public String description;

    @Column(name = "parent_space_id")
    public UUID parentSpaceId;

    @Column(name = "tenancy_id", nullable = false, updatable = false)
    public String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;

    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
        if (createdAt == null) createdAt = Instant.now();
    }

    public static SpaceEntity fromDomain(Space space) {
        SpaceEntity e = new SpaceEntity();
        e.id            = space.id();
        e.name          = space.name();
        e.description   = space.description();
        e.parentSpaceId = space.parentSpaceId();
        e.tenancyId     = space.tenancyId() != null ? space.tenancyId() : TenancyConstants.DEFAULT_TENANT_ID;
        e.createdAt     = space.createdAt();
        return e;
    }

    public Space toDomain() {
        return new Space(id, name, description, parentSpaceId, tenancyId, createdAt);
    }
}
```

Delete `MetadataMapConverter.java` using `ide_refactor_safe_delete`.

- [ ] **Step 9: Update JpaSpaceStore — rename methods, add new**

Use `ide_edit_member` to rewrite each method. Key changes:
- `findById` → `find`
- `findByParent` → `listByParent`
- `findRoots(String)` → `listRoots()` (uses `currentPrincipal.tenancyId()`)
- Add `findByName(String)` — returns `List<Space>` (names not globally unique)
- Add `findByIds(Collection<UUID>)` — batch IN query
- Override `hasChannelsInSpace` in `JpaChannelStore` with EXISTS query

For `JpaChannelStore`, add:
```java
@Override
public boolean hasChannelsInSpace(UUID spaceId) {
    if (spaceId == null) return false;
    return ChannelEntity.count("spaceId = ?1 AND tenancyId = ?2",
            spaceId, currentPrincipal.tenancyId()) > 0;
}
```

- [ ] **Step 10: Update InMemorySpaceStore — rename methods, add new**

Rename methods to match new interface. Add:
- `findByName(String name)` — stream filter on name
- `findByIds(Collection<UUID>)` — batch get
- Change `findRoots()` to not take tenancyId parameter

- [ ] **Step 11: Update InMemoryReactiveSpaceStore**

Match all blocking store changes with Uni wrappers.

- [ ] **Step 12: Update ReactiveJpaSpaceStore**

Match JpaSpaceStore renames. Add `findByName`, `findByIds`.

- [ ] **Step 13: Replace V33 migration**

Replace `runtime/src/main/resources/db/qhorus/migration/V33__space.sql`:

```sql
CREATE TABLE space (
    id          UUID         NOT NULL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    description TEXT,
    parent_space_id UUID,
    tenancy_id  VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
    created_at  TIMESTAMP    NOT NULL,
    CONSTRAINT fk_space_parent FOREIGN KEY (parent_space_id) REFERENCES space(id)
);

CREATE UNIQUE INDEX uq_space_name_under_parent
    ON space(tenancy_id, parent_space_id, name)
    WHERE parent_space_id IS NOT NULL;
CREATE UNIQUE INDEX uq_space_name_root
    ON space(tenancy_id, name)
    WHERE parent_space_id IS NULL;

CREATE INDEX idx_space_parent ON space(parent_space_id);
CREATE INDEX idx_space_tenancy ON space(tenancy_id);
```

V34 (`channel_space_id`) is unchanged — keep as-is.

- [ ] **Step 14: Update SpaceStoreContractTest**

Rewrite the contract test for the new API:
- Remove `metadata` from all `Space` constructors (now 6 fields, not 7)
- Remove `put_preservesMetadata` and `put_emptyMetadata` tests
- Rename abstract methods: `findById` → `find`, `findByParent` → `listByParent`, `findRoots(String)` → `listRoots()`
- Add abstract methods: `findByName(String)`, `findByIds(Collection<UUID>)`
- Add tests: `findByName_returnsMatchingSpaces`, `findByName_noMatch_returnsEmpty`, `findByIds_returnsBatch`, `findByIds_empty_returnsEmpty`

- [ ] **Step 15: Update test runners**

Update `InMemorySpaceStoreTest` and `InMemoryReactiveSpaceStoreTest` to implement the renamed/new abstract methods.

- [ ] **Step 16: Run store contract tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest="InMemorySpaceStoreTest,InMemoryReactiveSpaceStoreTest" -Dno-format
```

Expected: all tests pass with the new API.

- [ ] **Step 17: Verify api module compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api -Dno-format
```

- [ ] **Step 18: Commit**

```bash
git add -A
git commit -m "feat(#334): Space API overhaul — remove metadata, rename store methods, add SpaceCreateRequest

- Remove metadata field from Space record and SpaceEntity
- Delete MetadataMapConverter
- Rename SpaceStore methods for ChannelStore consistency (find, listByParent, listRoots)
- Add SpaceStore.findByName, findByIds for dual-identity and batch lookup
- Add ChannelStore.hasChannelsInSpace for delete guard
- Add ChannelDetail.spaceName for MCP enrichment
- Replace V33 migration with partial indexes for parent-scoped uniqueness
- Update all store implementations (JPA, InMemory, Reactive)
- Update SpaceStoreContractTest for new API

Refs #334"
```

---

### Task 2: SpaceService overhaul + tests

Rewrites SpaceService to use SpaceCreateRequest, adds MAX_DEPTH enforcement, cycle detection, rename, updateDescription, moveSpace, moveChannelToSpace, and findByName with ambiguity resolution.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/SpaceService.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/SpaceServiceTest.java`

**Interfaces:**
- Consumes: `SpaceStore` (from Task 1), `ChannelStore` (from Task 1), `SpaceCreateRequest` (from Task 1)
- Produces: `SpaceService` — `create(SpaceCreateRequest)`, `findById(UUID)`, `findByName(String)→Optional<Space>` (ambiguity throws), `listChildren(UUID)`, `listRoots()`, `listChannels(UUID)`, `delete(UUID)`, `rename(UUID, String)`, `updateDescription(UUID, String)`, `moveSpace(UUID, UUID)`, `moveChannelToSpace(UUID, UUID)`

- [ ] **Step 1: Write SpaceServiceTest with new API**

Rewrite `SpaceServiceTest` using the new API. The test uses CDI-free wiring with inline store implementations. Key test methods:

```java
// Existing tests updated for new API (SpaceCreateRequest instead of 4 args):
void create_rootSpace()
void create_childSpace()
void create_withNonExistentParent_throws()
void get_existing()
void get_nonExistent_throws()
void listRoots_returnsOnlyRoots()
void listChildren_returnsDirectChildren()
void listChannels_returnsBySpaceId()
void delete_emptySpace_succeeds()
void delete_withChildSpaces_throws()
void delete_withChannels_throws()
void recursiveNesting_threeLevels()

// New tests for new functionality:
void create_exceedingMaxDepth_throws()
void findByName_singleMatch_returnsSpace()
void findByName_noMatch_returnsEmpty()
void findByName_multipleMatches_throws()
void rename_updatesName()
void rename_blankName_throws()
void updateDescription_updatesDescription()
void updateDescription_null_clearsDescription()
void moveSpace_validReparent()
void moveSpace_toRoot()
void moveSpace_cycleDetection_throws()
void moveSpace_selfMove_throws()
void moveSpace_depthExceeded_throws()
void moveSpace_nonExistentParent_throws()
void moveChannelToSpace_assignsSpace()
void moveChannelToSpace_nullRemovesSpace()
void moveChannelToSpace_crossTenancy_throws()
void moveChannelToSpace_nonExistentSpace_throws()
void moveChannelToSpace_nonExistentChannel_throws()
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SpaceServiceTest -Dno-format
```

Expected: compilation failure or test failures (service API doesn't match yet).

- [ ] **Step 3: Rewrite SpaceService**

Use `ide_edit_member` to replace the full `SpaceService` class:

```java
@ApplicationScoped
public class SpaceService {

    static final int MAX_DEPTH = 10;

    @Inject SpaceStore spaceStore;
    @Inject ChannelStore channelStore;
    @Inject CurrentPrincipal currentPrincipal;

    @Transactional
    public Space create(SpaceCreateRequest request) {
        if (request.parentSpaceId() != null) {
            Space parent = spaceStore.find(request.parentSpaceId())
                    .orElseThrow(() -> new IllegalArgumentException(
                            "Parent space not found: " + request.parentSpaceId()));
            int depth = computeDepth(parent) + 1;
            if (depth >= MAX_DEPTH) {
                throw new IllegalStateException(
                        "Maximum nesting depth (" + MAX_DEPTH + ") exceeded");
            }
        }
        Space space = new Space(UUID.randomUUID(), request.name(), request.description(),
                request.parentSpaceId(), currentPrincipal.tenancyId(), Instant.now());
        return spaceStore.put(space);
    }

    public Optional<Space> findById(UUID id) {
        return spaceStore.find(id);
    }

    public Optional<Space> findByName(String name) {
        List<Space> matches = spaceStore.findByName(name);
        if (matches.isEmpty()) return Optional.empty();
        if (matches.size() == 1) return Optional.of(matches.get(0));
        throw new IllegalStateException(
                "Ambiguous space name '" + name + "' — " + matches.size()
                + " matches. Use UUID instead.");
    }

    public List<Space> listChildren(UUID parentSpaceId) {
        return spaceStore.listByParent(parentSpaceId);
    }

    public List<Space> listRoots() {
        return spaceStore.listRoots();
    }

    public List<Channel> listChannels(UUID spaceId) {
        return channelStore.scan(ChannelQuery.bySpaceId(spaceId));
    }

    @Transactional
    public void delete(UUID spaceId) {
        spaceStore.find(spaceId)
                .orElseThrow(() -> new IllegalArgumentException("Space not found: " + spaceId));
        if (spaceStore.hasChildren(spaceId)) {
            throw new IllegalStateException(
                    "Cannot delete space with child spaces: " + spaceId);
        }
        if (channelStore.hasChannelsInSpace(spaceId)) {
            throw new IllegalStateException(
                    "Cannot delete space with channels: " + spaceId);
        }
        spaceStore.delete(spaceId);
    }

    @Transactional
    public Space rename(UUID spaceId, String newName) {
        if (newName == null || newName.isBlank()) {
            throw new IllegalArgumentException("Space name must not be blank");
        }
        newName = newName.trim();
        if (newName.length() > 200) {
            throw new IllegalArgumentException("Space name exceeds 200 chars");
        }
        Space space = spaceStore.find(spaceId)
                .orElseThrow(() -> new IllegalArgumentException("Space not found: " + spaceId));
        Space updated = new Space(space.id(), newName, space.description(),
                space.parentSpaceId(), space.tenancyId(), space.createdAt());
        return spaceStore.put(updated);
    }

    @Transactional
    public Space updateDescription(UUID spaceId, String newDescription) {
        Space space = spaceStore.find(spaceId)
                .orElseThrow(() -> new IllegalArgumentException("Space not found: " + spaceId));
        String desc = newDescription != null ? newDescription.trim() : null;
        Space updated = new Space(space.id(), space.name(), desc,
                space.parentSpaceId(), space.tenancyId(), space.createdAt());
        return spaceStore.put(updated);
    }

    @Transactional
    public Space moveSpace(UUID spaceId, UUID newParentSpaceId) {
        Space space = spaceStore.find(spaceId)
                .orElseThrow(() -> new IllegalArgumentException("Space not found: " + spaceId));
        if (newParentSpaceId != null) {
            if (newParentSpaceId.equals(spaceId)) {
                throw new IllegalArgumentException("Cannot move space into itself");
            }
            Space newParent = spaceStore.find(newParentSpaceId)
                    .orElseThrow(() -> new IllegalArgumentException(
                            "Target parent space not found: " + newParentSpaceId));
            // Cycle detection: walk ancestors of newParent
            UUID ancestor = newParent.parentSpaceId();
            int walked = 0;
            while (ancestor != null && walked < MAX_DEPTH) {
                if (ancestor.equals(spaceId)) {
                    throw new IllegalArgumentException(
                            "Moving space " + spaceId + " under " + newParentSpaceId
                            + " would create a cycle");
                }
                ancestor = spaceStore.find(ancestor).map(Space::parentSpaceId).orElse(null);
                walked++;
            }
            // Depth check: subtree depth of space + depth of newParent must not exceed MAX_DEPTH
            int parentDepth = computeDepth(newParent);
            int subtreeDepth = computeSubtreeDepth(spaceId);
            if (parentDepth + 1 + subtreeDepth > MAX_DEPTH) {
                throw new IllegalStateException(
                        "Moving space would exceed maximum nesting depth (" + MAX_DEPTH + ")");
            }
        }
        Space updated = new Space(space.id(), space.name(), space.description(),
                newParentSpaceId, space.tenancyId(), space.createdAt());
        return spaceStore.put(updated);
    }

    @Transactional
    public Channel moveChannelToSpace(UUID channelId, UUID spaceId) {
        Channel channel = channelStore.find(channelId)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelId));
        if (spaceId != null) {
            Space space = spaceStore.find(spaceId)
                    .orElseThrow(() -> new IllegalArgumentException("Space not found: " + spaceId));
            if (!channel.tenancyId().equals(space.tenancyId())) {
                throw new IllegalArgumentException(
                        "Cannot move channel to space in different tenancy");
            }
        }
        return channelStore.put(channel.toBuilder().spaceId(spaceId).build());
    }

    private int computeDepth(Space space) {
        int depth = 0;
        UUID parentId = space.parentSpaceId();
        while (parentId != null && depth < MAX_DEPTH) {
            parentId = spaceStore.find(parentId).map(Space::parentSpaceId).orElse(null);
            depth++;
        }
        return depth;
    }

    private int computeSubtreeDepth(UUID spaceId) {
        List<Space> children = spaceStore.listByParent(spaceId);
        if (children.isEmpty()) return 0;
        int maxChildDepth = 0;
        for (Space child : children) {
            maxChildDepth = Math.max(maxChildDepth, computeSubtreeDepth(child.id()));
        }
        return 1 + maxChildDepth;
    }
}
```

- [ ] **Step 4: Run SpaceServiceTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SpaceServiceTest -Dno-format
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#334): SpaceService overhaul — SpaceCreateRequest, MAX_DEPTH, cycle detection, move/rename

- Rewrite SpaceService.create() to use SpaceCreateRequest
- Add MAX_DEPTH=10 enforcement on create and moveSpace
- Add moveSpace with cycle detection and subtree depth check
- Add moveChannelToSpace with same-tenancy enforcement
- Add rename and updateDescription
- Add findByName with ambiguity resolution
- Full test coverage: 24 tests covering all paths

Refs #334"
```

---

### Task 3: MCP tools + spaceName enrichment + full build

Adds `resolveSpace()` dual-identity resolution, updates 4 existing space tools, adds 5 new tools, updates `create_channel` with `space_id`, adds spaceName batch enrichment to `list_channels`, mirrors everything in `ReactiveQhorusMcpTools`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`

**Interfaces:**
- Consumes: `SpaceService` (from Task 2), `SpaceStore.findByIds` (from Task 1)
- Produces: 9 MCP tools (create_space, get_space, list_spaces, list_space_channels, rename_space, update_space_description, move_space, move_channel_to_space, delete_space) + create_channel space_id parameter + spaceName enrichment

- [ ] **Step 1: Add resolveSpace() to QhorusMcpToolsBase**

Use `ide_insert_member` to add after `resolveChannel`:

```java
Space resolveSpace(final String space) {
    final UUID parsedUuid = ChannelSlugValidator.tryParseUuid(space);
    if (parsedUuid != null) {
        return spaceService.findById(parsedUuid)
                .orElseThrow(() -> new IllegalArgumentException("Space not found: " + space));
    }
    return spaceService.findByName(space)
            .orElseThrow(() -> new IllegalArgumentException("Space not found: " + space));
}
```

Add import for `io.casehub.qhorus.api.channel.Space` if not already present.

- [ ] **Step 2: Add spaceName batch enrichment helper to QhorusMcpToolsBase**

Use `ide_insert_member` to add a helper for batch space name lookup:

```java
protected Map<UUID, String> buildSpaceNameMap(List<Channel> channels) {
    java.util.Set<UUID> spaceIds = channels.stream()
            .map(Channel::spaceId)
            .filter(java.util.Objects::nonNull)
            .collect(java.util.stream.Collectors.toSet());
    if (spaceIds.isEmpty()) return Map.of();
    return spaceService.spaceStore.findByIds(spaceIds).stream()
            .collect(java.util.stream.Collectors.toMap(Space::id, Space::name));
}
```

Add overloaded `toChannelDetail` that accepts the space name map:

```java
protected ChannelDetail toChannelDetail(Channel ch, long messageCount,
                                        Map<UUID, ChannelConnectorBinding> allBindings,
                                        Map<UUID, String> spaceNames) {
    return entityMapper.toChannelDetail(ch, messageCount,
            Optional.ofNullable(allBindings.get(ch.id())),
            ch.spaceId() != null ? spaceNames.get(ch.spaceId()) : null);
}
```

- [ ] **Step 3: Update existing 4 space tools in QhorusMcpTools**

Replace the 4 existing space tools:

`create_space` — remove `metadata` parameter, use `SpaceCreateRequest`:
```java
@Tool(name = "create_space", description = "Create an organizational space to group related channels. "
        + "Spaces can nest recursively (project → case → channels). "
        + "Channels are assigned to spaces via create_channel(space_id) or move_channel_to_space.")
@Transactional
public Space createSpace(
        @ToolArg(name = "name", description = "Space name (free-form text, max 200 chars)") String name,
        @ToolArg(name = "description", description = "Space purpose description", required = false) String description,
        @ToolArg(name = "parent_space_id", description = "Parent space UUID or name for nesting. Null = root space.", required = false) String parentSpaceId) {
    UUID parentId = parentSpaceId != null ? resolveSpace(parentSpaceId).id() : null;
    return spaceService.create(new SpaceCreateRequest(name, description, parentId));
}
```

`list_spaces` — use `resolveSpace` for parent lookup:
```java
@Tool(name = "list_spaces", description = "List spaces. Without parent_space_id, returns root spaces. "
        + "With parent_space_id, returns direct children of that space.")
public List<Space> listSpaces(
        @ToolArg(name = "parent_space_id", description = "Parent space UUID or name. Null = list root spaces.", required = false) String parentSpaceId) {
    if (parentSpaceId == null) return spaceService.listRoots();
    return spaceService.listChildren(resolveSpace(parentSpaceId).id());
}
```

`list_space_channels` — use `resolveSpace`:
```java
@Tool(name = "list_space_channels", description = "List all channels belonging to a space.")
public List<ChannelDetail> listSpaceChannels(
        @ToolArg(name = "space", description = "Space UUID or name") String space) {
    UUID sid = resolveSpace(space).id();
    List<Channel> channels = spaceService.listChannels(sid);
    return channels.stream().map(ch -> toChannelDetail(ch, messageStore.count(
            MessageQuery.builder().channelId(ch.id()).build()))).toList();
}
```

`delete_space` — use `resolveSpace`:
```java
@Tool(name = "delete_space", description = "Delete a space. Fails if the space contains channels or child spaces.")
@Transactional
public DeleteSpaceResult deleteSpace(
        @ToolArg(name = "space", description = "Space UUID or name to delete") String space) {
    Space s = resolveSpace(space);
    spaceService.delete(s.id());
    return new DeleteSpaceResult(s.id().toString(), true);
}
```

- [ ] **Step 4: Add 5 new space tools in QhorusMcpTools**

Use `ide_insert_member` to add before the projection tools section:

```java
@Tool(name = "get_space", description = "Get a space by UUID or name.")
public Space getSpace(
        @ToolArg(name = "space", description = "Space UUID or name") String space) {
    return resolveSpace(space);
}

@Tool(name = "rename_space", description = "Rename a space.")
@Transactional
public Space renameSpace(
        @ToolArg(name = "space", description = "Space UUID or name") String space,
        @ToolArg(name = "new_name", description = "New space name") String newName) {
    return spaceService.rename(resolveSpace(space).id(), newName);
}

@Tool(name = "update_space_description", description = "Update a space's description. Null clears the description.")
@Transactional
public Space updateSpaceDescription(
        @ToolArg(name = "space", description = "Space UUID or name") String space,
        @ToolArg(name = "description", description = "New description. Null clears it.", required = false) String description) {
    return spaceService.updateDescription(resolveSpace(space).id(), description);
}

@Tool(name = "move_space", description = "Move a space to a new parent. Null parent_space_id makes it a root space. "
        + "Fails if the move would create a cycle or exceed the maximum nesting depth.")
@Transactional
public Space moveSpace(
        @ToolArg(name = "space", description = "Space UUID or name to move") String space,
        @ToolArg(name = "parent_space_id", description = "New parent space UUID or name. Null = make root.", required = false) String parentSpaceId) {
    UUID parentId = parentSpaceId != null ? resolveSpace(parentSpaceId).id() : null;
    return spaceService.moveSpace(resolveSpace(space).id(), parentId);
}

@Tool(name = "move_channel_to_space", description = "Move a channel into a space. Null space_id removes the channel from its space (makes it top-level). "
        + "Channel and space must be in the same tenancy.")
@Transactional
public ChannelDetail moveChannelToSpace(
        @ToolArg(name = "channel", description = "Channel UUID or name") String channel,
        @ToolArg(name = "space_id", description = "Target space UUID or name. Null = remove from space.", required = false) String spaceId) {
    UUID sid = spaceId != null ? resolveSpace(spaceId).id() : null;
    Channel updated = spaceService.moveChannelToSpace(resolveChannel(channel).id(), sid);
    return toChannelDetail(updated, messageStore.count(
            MessageQuery.builder().channelId(updated.id()).build()));
}
```

- [ ] **Step 5: Update create_channel to accept space_id**

In `QhorusMcpTools.createChannel()`, add a new `@ToolArg` parameter and set it on the builder:

Add parameter: `@ToolArg(name = "space_id", description = "Space UUID or name to assign this channel to.", required = false) String spaceId`

In the builder chain, add: `.spaceId(spaceId != null ? resolveSpace(spaceId).id() : null)`

- [ ] **Step 6: Update list_channels for spaceName enrichment**

In `QhorusMcpTools.listChannels()`, add space name batch lookup alongside the existing binding batch lookup. Use `buildSpaceNameMap(channels)` and pass to the 4-arg `toChannelDetail`.

- [ ] **Step 7: Mirror all changes in ReactiveQhorusMcpTools**

Update existing 4 tools (same pattern changes as blocking). Add 5 new tools wrapped in `Uni.createFrom().item(() -> ...).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`. Update `createChannel` with `space_id`. Update `listChannels` for spaceName.

- [ ] **Step 8: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format
```

Expected: all 13 modules compile and test successfully. Fix any cross-module compilation errors (examples modules may reference old Space constructor with metadata).

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#334): Space MCP tools — resolveSpace, 5 new tools, spaceName enrichment

- Add resolveSpace() dual-identity resolution (UUID or name)
- Update 4 existing space tools: remove metadata, use resolveSpace
- Add get_space, rename_space, update_space_description, move_space, move_channel_to_space
- Update create_channel with optional space_id parameter
- Add spaceName batch enrichment to list_channels via findByIds
- Mirror all changes in ReactiveQhorusMcpTools
- Full build green across all modules

Refs #334"
```

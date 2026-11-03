# Channel Visibility Semantics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #509 — Channel Visibility Semantics
**Issue group:** #509

**Goal:** Add a typed visibility dimension (PUBLIC, PRIVATE, DIRECT) to
Qhorus channels, enabling the agent society to use channels as agent
mail inboxes, group communication, and broadcast.

**Architecture:** New `ChannelVisibility` enum + nullable field on
`Channel`. A focused `VisibilityManager` interface for composite
DIRECT/GROUP lifecycle operations. Visibility guards in
`ChannelMembershipService` for DIRECT immutability. `boundGroupId` as
a first-class indexed field for group-channel binding. All existing
normative infrastructure (speech acts, commitments, ledger, trust)
applies unchanged.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, H2 (tests),
PostgreSQL (production), Flyway migrations

## Global Constraints

- Flyway domain migrations: V61 is next (V59, V60 taken)
- Channel names must pass `ChannelSlugValidator.validateSlugPath()`
- `ChannelCreateRequest.builder("name")` is the conventional construction API
- InMemory stores in `persistence-memory/` must match all new store methods
- Tests use `@QuarkusTest @TestTransaction` unless observer dispatch needed
- CDI-free unit tests: set `service.tracingConfig` to disabled impl
- All commits reference #509: `Refs #509`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- After API changes, run full `mvn install` from project root

---

## Batch 1: Foundation — Enum, Record Extension, Migration

After this batch: `ChannelVisibility` enum exists, `Channel` and
`ChannelCreateRequest` carry the new fields, `ChannelEntity` persists
them, `ChannelQuery` filters by them, migration is in place. The build
compiles and all existing tests pass.

### Task 1: ChannelVisibility enum + Channel/ChannelCreateRequest extension + Migration

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelVisibility.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/query/ChannelQuery.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelStore.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V61__channel_visibility.sql`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelVisibilityTest.java`

**Interfaces:**
- Produces: `ChannelVisibility { PUBLIC, PRIVATE, DIRECT }` enum
- Produces: `Channel.visibility()` → `ChannelVisibility` (nullable)
- Produces: `Channel.boundGroupId()` → `UUID` (nullable)
- Produces: `ChannelCreateRequest.visibility()` → `ChannelVisibility` (nullable)
- Produces: `ChannelCreateRequest.participationPolicy()` → `ParticipationPolicyMode` (nullable)
- Produces: `ChannelCreateRequest.boundGroupId()` → `UUID` (nullable)
- Produces: `ChannelQuery.byVisibility(ChannelVisibility)` → `ChannelQuery`
- Produces: `ChannelQuery.byBoundGroupId(UUID)` → `ChannelQuery`

- [ ] **Step 1: Create ChannelVisibility enum**

Create `api/src/main/java/io/casehub/qhorus/api/channel/ChannelVisibility.java`:

```java
package io.casehub.qhorus.api.channel;

public enum ChannelVisibility {
    PUBLIC,
    PRIVATE,
    DIRECT
}
```

- [ ] **Step 2: Add fields to Channel record**

In `Channel.java`, add two new fields to the canonical constructor:
`ChannelVisibility visibility` and `UUID boundGroupId` — both after
`participationPolicy` (last two fields).

Add a backward-compatible constructor that delegates with
`visibility = null, boundGroupId = null` — matching the existing pattern
of the 29-param constructor at line 58.

Update `fromRequest()` to apply visibility-based defaults:

```java
public static Channel fromRequest(ChannelCreateRequest req, String tenancyId) {
    Instant now = Instant.now();
    ChannelSemantic semantic = req.semantic();
    ParticipationPolicyMode policy = req.participationPolicy();

    if (req.visibility() != null) {
        if (semantic == null) {
            semantic = switch (req.visibility()) {
                case PUBLIC -> ChannelSemantic.BROADCAST;
                case PRIVATE, DIRECT -> ChannelSemantic.APPEND;
            };
        }
        if (policy == null) {
            policy = switch (req.visibility()) {
                case PUBLIC -> ParticipationPolicyMode.OPEN;
                case PRIVATE, DIRECT -> ParticipationPolicyMode.ENFORCE;
            };
        }
    } else {
        if (semantic == null) semantic = ChannelSemantic.APPEND;
    }

    return new Channel(
            req.preAssignedId() != null ? req.preAssignedId() : UUID.randomUUID(),
            req.name(), req.description(), semantic,
            req.barrierContributors(), req.allowedWriters(), req.adminInstances(),
            req.rateLimitPerChannel(), req.rateLimitPerInstance(),
            req.allowedTypes(), req.deniedTypes(),
            false, false, req.spaceId(), req.reviewerInstances(),
            req.protocols(), req.protocolParticipants(), req.trackDelivery(),
            req.enforcementMode(), req.enforcementExclusions(),
            req.routingTrustThreshold(), tenancyId, now, now, null, null, null,
            null, req.metadata(), policy,
            req.visibility(), req.boundGroupId());
}
```

Update `toBuilder()` to include `visibility` and `boundGroupId`.
Update `Builder` class with both new fields and setters.

- [ ] **Step 3: Add fields to ChannelCreateRequest**

Add three new fields to `ChannelCreateRequest` canonical constructor:
`ChannelVisibility visibility`, `ParticipationPolicyMode participationPolicy`,
`UUID boundGroupId` — after `preAssignedId`.

Add builder methods: `.visibility()`, `.participationPolicy()`, `.boundGroupId()`.

Add backward-compatible constructor(s) that delegate with the new fields
as `null`.

- [ ] **Step 4: Update ChannelEntity**

Add to `ChannelEntity`:

```java
@Column(name = "visibility")
public String visibility;

@Column(name = "bound_group_id")
public UUID boundGroupId;
```

Update `fromDomain()`:
```java
e.visibility = channel.visibility() != null ? channel.visibility().name() : null;
e.boundGroupId = channel.boundGroupId();
```

Update `toDomain()` to pass both new fields to the Channel constructor:
```java
visibility != null ? ChannelVisibility.valueOf(visibility) : null,
boundGroupId
```

- [ ] **Step 5: Update ChannelQuery**

Add two new fields to `ChannelQuery`:

```java
private final ChannelVisibility visibility;
private final UUID boundGroupId;
```

Add factory methods:
```java
public static ChannelQuery byVisibility(ChannelVisibility v) {
    return new Builder().visibility(v).build();
}
public static ChannelQuery byBoundGroupId(UUID groupId) {
    return new Builder().boundGroupId(groupId).build();
}
```

Update `matches()`:
```java
if (visibility != null && visibility != ch.visibility()) return false;
if (boundGroupId != null && !boundGroupId.equals(ch.boundGroupId())) return false;
```

Update `Builder` and `toBuilder()` with both fields.

- [ ] **Step 6: Update InMemoryChannelStore**

The `InMemoryChannelStore.scan()` delegates to `ChannelQuery.matches()` —
no changes needed in the store itself. The new query fields are handled
by the `matches()` update in Step 5.

Verify by reading the `scan()` method to confirm it uses `query.matches()`.

- [ ] **Step 7: Create Flyway migration V61**

Create `runtime/src/main/resources/db/qhorus/migration/V61__channel_visibility.sql`:

```sql
ALTER TABLE channel ADD COLUMN visibility VARCHAR(20);
ALTER TABLE channel ADD COLUMN bound_group_id UUID;
CREATE INDEX idx_channel_visibility ON channel(visibility) WHERE visibility IS NOT NULL;
CREATE INDEX idx_channel_bound_group_id ON channel(bound_group_id) WHERE bound_group_id IS NOT NULL;
```

- [ ] **Step 8: Write tests**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelVisibilityTest.java`:

```java
@QuarkusTest
@TestTransaction
class ChannelVisibilityTest {

    @Inject ChannelService channelService;
    @Inject ChannelStore channelStore;

    @Test
    void createChannelWithVisibility() {
        var req = ChannelCreateRequest.builder("test-public-ch")
                .visibility(ChannelVisibility.PUBLIC)
                .build();
        Channel ch = channelService.create(req);
        assertThat(ch.visibility()).isEqualTo(ChannelVisibility.PUBLIC);
        assertThat(ch.semantic()).isEqualTo(ChannelSemantic.BROADCAST);
        assertThat(ch.participationPolicy()).isEqualTo(ParticipationPolicyMode.OPEN);
    }

    @Test
    void createDirectChannelDefaults() {
        var req = ChannelCreateRequest.builder("test-direct-ch")
                .visibility(ChannelVisibility.DIRECT)
                .build();
        Channel ch = channelService.create(req);
        assertThat(ch.visibility()).isEqualTo(ChannelVisibility.DIRECT);
        assertThat(ch.semantic()).isEqualTo(ChannelSemantic.APPEND);
        assertThat(ch.participationPolicy()).isEqualTo(ParticipationPolicyMode.ENFORCE);
    }

    @Test
    void createPrivateChannelDefaults() {
        var req = ChannelCreateRequest.builder("test-private-ch")
                .visibility(ChannelVisibility.PRIVATE)
                .build();
        Channel ch = channelService.create(req);
        assertThat(ch.visibility()).isEqualTo(ChannelVisibility.PRIVATE);
        assertThat(ch.semantic()).isEqualTo(ChannelSemantic.APPEND);
        assertThat(ch.participationPolicy()).isEqualTo(ParticipationPolicyMode.ENFORCE);
    }

    @Test
    void callerCanOverrideDefaults() {
        var req = ChannelCreateRequest.builder("test-override-ch")
                .visibility(ChannelVisibility.PRIVATE)
                .semantic(ChannelSemantic.BARRIER)
                .participationPolicy(ParticipationPolicyMode.OPEN)
                .build();
        Channel ch = channelService.create(req);
        assertThat(ch.visibility()).isEqualTo(ChannelVisibility.PRIVATE);
        assertThat(ch.semantic()).isEqualTo(ChannelSemantic.BARRIER);
        assertThat(ch.participationPolicy()).isEqualTo(ParticipationPolicyMode.OPEN);
    }

    @Test
    void nullVisibilityPreservesLegacyBehavior() {
        var req = ChannelCreateRequest.builder("test-legacy-ch").build();
        Channel ch = channelService.create(req);
        assertThat(ch.visibility()).isNull();
        assertThat(ch.semantic()).isEqualTo(ChannelSemantic.APPEND);
    }

    @Test
    void channelWithBoundGroupId() {
        UUID groupId = UUID.randomUUID();
        var req = ChannelCreateRequest.builder("test-group-ch")
                .visibility(ChannelVisibility.PRIVATE)
                .boundGroupId(groupId)
                .build();
        Channel ch = channelService.create(req);
        assertThat(ch.boundGroupId()).isEqualTo(groupId);
    }

    @Test
    void queryByVisibility() {
        channelService.create(ChannelCreateRequest.builder("vis-pub")
                .visibility(ChannelVisibility.PUBLIC).build());
        channelService.create(ChannelCreateRequest.builder("vis-priv")
                .visibility(ChannelVisibility.PRIVATE).build());
        channelService.create(ChannelCreateRequest.builder("vis-none").build());

        var publicChannels = channelStore.scan(ChannelQuery.byVisibility(ChannelVisibility.PUBLIC));
        assertThat(publicChannels).hasSize(1);
        assertThat(publicChannels.get(0).name()).isEqualTo("vis-pub");
    }

    @Test
    void queryByBoundGroupId() {
        UUID gid = UUID.randomUUID();
        channelService.create(ChannelCreateRequest.builder("grp-bound")
                .boundGroupId(gid).build());
        channelService.create(ChannelCreateRequest.builder("grp-none").build());

        var found = channelStore.scan(ChannelQuery.byBoundGroupId(gid));
        assertThat(found).hasSize(1);
        assertThat(found.get(0).name()).isEqualTo("grp-bound");
    }
}
```

- [ ] **Step 9: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelVisibilityTest`
Expected: all tests PASS

- [ ] **Step 10: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile with the new fields, all
existing tests pass (backward-compat constructors prevent breakage).

- [ ] **Step 11: Commit**

```bash
git add api/ runtime/ persistence-memory/
git commit -m "feat(#509): add ChannelVisibility enum, Channel/ChannelCreateRequest fields, V61 migration

Adds ChannelVisibility (PUBLIC, PRIVATE, DIRECT) enum and nullable
visibility + boundGroupId fields to Channel. Applies creation-time
semantic/participationPolicy defaults per visibility. ChannelQuery
gains visibility and boundGroupId filters.

Refs #509

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: DIRECT Channel Mechanics — Membership Guards + VisibilityManager

After this batch: DIRECT channels can be created via `getOrCreateDirect`,
membership is immutable (exactly 2 members), inbox queries work
efficiently, `ChannelMembershipService` enforces visibility guards.

### Task 2: DIRECT membership guards in ChannelMembershipService

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelMembershipService.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ChannelMembershipStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelMembershipStore.java`
- Test: `testing/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMembershipServiceDirectGuardTest.java`

**Interfaces:**
- Consumes: `Channel.visibility()` from Task 1
- Consumes: `ChannelStore.find(UUID)` (existing)
- Consumes: `ChannelMembershipStore.findByChannel(UUID)` (existing)
- Produces: `ChannelMembershipService.join()` with DIRECT guard (count >= 2 → throw)
- Produces: `ChannelMembershipService.leave()` with DIRECT guard (visibility == DIRECT → throw)
- Produces: `ChannelMembershipStore.findChannelsByMemberAndVisibility(String, ChannelVisibility, String)` → `List<Channel>`

- [ ] **Step 1: Write failing test for DIRECT join guard**

Create `testing/src/test/java/io/casehub/qhorus/runtime/channel/ChannelMembershipServiceDirectGuardTest.java`:

```java
class ChannelMembershipServiceDirectGuardTest {

    private ChannelMembershipService service;
    private InMemoryChannelMembershipStore membershipStore;
    private InMemoryChannelStore channelStore;
    private InMemoryMessageStore messageStore;

    @BeforeEach
    void setUp() {
        membershipStore = new InMemoryChannelMembershipStore();
        channelStore = new InMemoryChannelStore();
        messageStore = new InMemoryMessageStore();
        CurrentPrincipal principal = new FixedCurrentPrincipal();
        service = new ChannelMembershipService(membershipStore, messageStore, principal, channelStore);
    }

    @Test
    void directChannelAllowsFirstTwoJoins() {
        Channel ch = createDirectChannel();
        channelStore.put(ch);

        service.join(ch.id(), "agent-a", MemberRole.PARTICIPANT, "tenant");
        service.join(ch.id(), "agent-b", MemberRole.PARTICIPANT, "tenant");

        assertThat(membershipStore.findByChannel(ch.id())).hasSize(2);
    }

    @Test
    void directChannelRejectsThirdJoin() {
        Channel ch = createDirectChannel();
        channelStore.put(ch);

        service.join(ch.id(), "agent-a", MemberRole.PARTICIPANT, "tenant");
        service.join(ch.id(), "agent-b", MemberRole.PARTICIPANT, "tenant");

        assertThatThrownBy(() ->
                service.join(ch.id(), "agent-c", MemberRole.PARTICIPANT, "tenant"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("DIRECT channels have immutable membership");
    }

    @Test
    void directChannelRejectsLeave() {
        Channel ch = createDirectChannel();
        channelStore.put(ch);
        service.join(ch.id(), "agent-a", MemberRole.PARTICIPANT, "tenant");

        assertThatThrownBy(() -> service.leave(ch.id(), "agent-a"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Cannot leave a DIRECT channel");
    }

    @Test
    void nonDirectChannelAllowsUnlimitedJoins() {
        Channel ch = createChannel(null); // no visibility
        channelStore.put(ch);

        service.join(ch.id(), "agent-a", MemberRole.PARTICIPANT, "tenant");
        service.join(ch.id(), "agent-b", MemberRole.PARTICIPANT, "tenant");
        service.join(ch.id(), "agent-c", MemberRole.PARTICIPANT, "tenant");

        assertThat(membershipStore.findByChannel(ch.id())).hasSize(3);
    }

    private Channel createDirectChannel() {
        return createChannel(ChannelVisibility.DIRECT);
    }

    private Channel createChannel(ChannelVisibility visibility) {
        return Channel.builder("test-ch-" + UUID.randomUUID().toString().substring(0, 8))
                .id(UUID.randomUUID())
                .semantic(ChannelSemantic.APPEND)
                .visibility(visibility)
                .tenancyId("tenant")
                .createdAt(Instant.now())
                .lastActivityAt(Instant.now())
                .build();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=ChannelMembershipServiceDirectGuardTest`
Expected: FAIL — `ChannelMembershipService` constructor doesn't accept `ChannelStore`

- [ ] **Step 3: Implement DIRECT guards**

Modify `ChannelMembershipService`:

1. Add `ChannelStore channelStore` field
2. Add new constructor:
   ```java
   public ChannelMembershipService(ChannelMembershipStore membershipStore,
                                    MessageStore messageStore,
                                    CurrentPrincipal currentPrincipal,
                                    ChannelStore channelStore) {
       this.membershipStore = membershipStore;
       this.messageStore = messageStore;
       this.currentPrincipal = currentPrincipal;
       this.channelStore = channelStore;
   }
   ```
3. In `join(UUID channelId, String memberId, MemberRole role, String tenancyId)`, add guard before existing logic:
   ```java
   if (channelStore != null) {
       channelStore.find(channelId).ifPresent(ch -> {
           if (ch.visibility() == ChannelVisibility.DIRECT) {
               long count = membershipStore.findByChannel(channelId).size();
               if (count >= 2) {
                   throw new IllegalStateException(
                       "DIRECT channels have immutable membership — exactly 2 members");
               }
           }
       });
   }
   ```
4. In `leave(UUID channelId, String memberId)`, add guard:
   ```java
   if (channelStore != null) {
       channelStore.find(channelId).ifPresent(ch -> {
           if (ch.visibility() == ChannelVisibility.DIRECT) {
               throw new IllegalStateException("Cannot leave a DIRECT channel");
           }
       });
   }
   ```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=ChannelMembershipServiceDirectGuardTest`
Expected: PASS

- [ ] **Step 5: Add findChannelsByMemberAndVisibility to store**

Add to `ChannelMembershipStore`:
```java
default List<Channel> findChannelsByMemberAndVisibility(
        String memberId, ChannelVisibility visibility, String tenancyId) {
    throw new UnsupportedOperationException("Not implemented");
}
```

Implement in `InMemoryChannelMembershipStore` — this requires the store
to have access to a `ChannelStore` reference. Since InMemory stores are
test-only, add a `setChannelStore(InMemoryChannelStore)` method:

```java
private InMemoryChannelStore channelStore;

public void setChannelStore(InMemoryChannelStore cs) { this.channelStore = cs; }

@Override
public List<Channel> findChannelsByMemberAndVisibility(
        String memberId, ChannelVisibility visibility, String tenancyId) {
    if (channelStore == null) return List.of();
    Set<UUID> memberChannelIds = store.values().stream()
        .filter(m -> m.memberId().equals(memberId))
        .map(ChannelMembership::channelId)
        .collect(Collectors.toSet());
    return memberChannelIds.stream()
        .map(channelStore::find)
        .filter(Optional::isPresent)
        .map(Optional::get)
        .filter(ch -> ch.visibility() == visibility)
        .toList();
}
```

- [ ] **Step 6: Update CDI wrapper for ChannelMembershipService**

Find and update the CDI wrapper (`CdiChannelMembershipService` in
`runtime/cdi/`) to inject `ChannelStore` and pass it to the POJO
constructor.

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add api/ runtime/ runtime-core/ persistence-memory/ testing/
git commit -m "feat(#509): add DIRECT membership guards to ChannelMembershipService

join() rejects when DIRECT channel already has 2 members.
leave() rejects on DIRECT channels entirely.
Adds findChannelsByMemberAndVisibility() store method for inbox queries.

Refs #509

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: VisibilityManager interface + VisibilityService (DIRECT + GROUP)

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/VisibilityManager.java`
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/VisibilityService.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/cdi/CdiVisibilityService.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java` (add setVisibility)
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelReader.java` (add findByVisibility)
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` (implement setVisibility, findByVisibility)
- Test: `testing/src/test/java/io/casehub/qhorus/runtime/channel/VisibilityServiceTest.java`

**Interfaces:**
- Consumes: `ChannelMembershipService.join()` with guards from Task 2
- Consumes: `ChannelService.findOrCreate()` (existing)
- Consumes: `ChannelMembershipStore.findChannelsByMemberAndVisibility()` from Task 2
- Produces: `VisibilityManager.getOrCreateDirect(String, String)` → `FindOrCreateResult`
- Produces: `VisibilityManager.findDirectChannelsFor(String)` → `List<Channel>`
- Produces: `VisibilityManager.createGroupChannel(UUID, String, List<String>, String)` → `Channel`
- Produces: `VisibilityManager.findGroupChannel(UUID)` → `Optional<Channel>`
- Produces: `ChannelManager.setVisibility(UUID, ChannelVisibility)` → `Channel`
- Produces: `ChannelReader.findByVisibility(ChannelVisibility)` → `List<Channel>`

- [ ] **Step 1: Create VisibilityManager interface**

Create `api/src/main/java/io/casehub/qhorus/api/channel/VisibilityManager.java`:

```java
package io.casehub.qhorus.api.channel;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface VisibilityManager {

    FindOrCreateResult getOrCreateDirect(String participantA, String participantB);

    List<Channel> findDirectChannelsFor(String participantId);

    Channel createGroupChannel(UUID groupId, String groupName,
                               List<String> memberIds, String coordinatorId);

    Optional<Channel> findGroupChannel(UUID groupId);
}
```

- [ ] **Step 2: Add setVisibility to ChannelManager, findByVisibility to ChannelReader**

In `ChannelManager.java` add:
```java
Channel setVisibility(UUID channelId, ChannelVisibility visibility);
```

In `ChannelReader.java` add:
```java
List<Channel> findByVisibility(ChannelVisibility visibility);
```

- [ ] **Step 3: Implement setVisibility and findByVisibility in ChannelService**

In `ChannelService.java`:

```java
@Override
public Channel setVisibility(UUID channelId, ChannelVisibility visibility) {
    Channel ch = channelStore.find(channelId)
            .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channelId));
    if (visibility == ChannelVisibility.DIRECT) {
        long memberCount = membershipStore.findByChannel(channelId).size();
        if (memberCount != 2) {
            throw new IllegalStateException(
                "Cannot set visibility to DIRECT — channel must have exactly 2 members (has " + memberCount + ")");
        }
    }
    Channel updated = ch.toBuilder().visibility(visibility).build();
    return channelStore.put(updated);
}

@Override
public List<Channel> findByVisibility(ChannelVisibility visibility) {
    return channelStore.scan(ChannelQuery.byVisibility(visibility));
}
```

- [ ] **Step 4: Write failing tests for VisibilityService**

Create `testing/src/test/java/io/casehub/qhorus/runtime/channel/VisibilityServiceTest.java`:

```java
class VisibilityServiceTest {

    private VisibilityService service;
    private InMemoryChannelStore channelStore;
    private InMemoryChannelMembershipStore membershipStore;
    private InMemoryMessageStore messageStore;
    private ChannelMembershipService membershipService;
    private ChannelService channelService;

    @BeforeEach
    void setUp() {
        channelStore = new InMemoryChannelStore();
        membershipStore = new InMemoryChannelMembershipStore();
        messageStore = new InMemoryMessageStore();
        membershipStore.setChannelStore(channelStore);
        CurrentPrincipal principal = new FixedCurrentPrincipal();
        membershipService = new ChannelMembershipService(
            membershipStore, messageStore, principal, channelStore);
        // ChannelService construction — CDI-free requires mocks/stubs for
        // ChannelGateway, ChannelCreateHelper, etc.
        // Use InMemory stores + minimal stubs
        channelService = buildTestChannelService(channelStore, messageStore,
            membershipStore, principal);
        service = new VisibilityService(channelService, membershipService,
            membershipStore, channelStore, principal);
    }

    @Test
    void getOrCreateDirectCreatesChannel() {
        var result = service.getOrCreateDirect("bob", "alice");
        assertThat(result.wasCreated()).isTrue();
        assertThat(result.channel().name()).isEqualTo("dm/alice/bob");
        assertThat(result.channel().visibility()).isEqualTo(ChannelVisibility.DIRECT);
        assertThat(membershipStore.findByChannel(result.channel().id())).hasSize(2);
    }

    @Test
    void getOrCreateDirectIsDeterministic() {
        var r1 = service.getOrCreateDirect("bob", "alice");
        var r2 = service.getOrCreateDirect("alice", "bob");
        assertThat(r2.wasCreated()).isFalse();
        assertThat(r2.channel().id()).isEqualTo(r1.channel().id());
    }

    @Test
    void getOrCreateDirectRejectsInvalidSlug() {
        assertThatThrownBy(() -> service.getOrCreateDirect("Agent_1", "bob"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("not a valid slug segment");
    }

    @Test
    void findDirectChannelsFor() {
        service.getOrCreateDirect("alice", "bob");
        service.getOrCreateDirect("alice", "carol");

        var channels = service.findDirectChannelsFor("alice");
        assertThat(channels).hasSize(2);
    }

    @Test
    void createGroupChannel() {
        UUID groupId = UUID.randomUUID();
        Channel ch = service.createGroupChannel(groupId, "research-team",
            List.of("alice", "bob", "carol"), "alice");

        assertThat(ch.name()).isEqualTo("group/research-team");
        assertThat(ch.visibility()).isEqualTo(ChannelVisibility.PRIVATE);
        assertThat(ch.boundGroupId()).isEqualTo(groupId);

        var members = membershipStore.findByChannel(ch.id());
        assertThat(members).hasSize(3);
        var moderator = members.stream()
            .filter(m -> m.role() == MemberRole.MODERATOR).findFirst();
        assertThat(moderator).isPresent();
        assertThat(moderator.get().memberId()).isEqualTo("alice");
    }

    @Test
    void createGroupChannelIdempotent() {
        UUID groupId = UUID.randomUUID();
        Channel ch1 = service.createGroupChannel(groupId, "team",
            List.of("alice"), "alice");
        Channel ch2 = service.createGroupChannel(groupId, "team",
            List.of("alice"), "alice");
        assertThat(ch2.id()).isEqualTo(ch1.id());
    }

    @Test
    void createGroupChannelRejectsBindingMismatch() {
        UUID g1 = UUID.randomUUID();
        UUID g2 = UUID.randomUUID();
        service.createGroupChannel(g1, "shared-name", List.of("alice"), "alice");

        assertThatThrownBy(() ->
            service.createGroupChannel(g2, "shared-name", List.of("bob"), "bob"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("bound to a different group");
    }

    @Test
    void findGroupChannel() {
        UUID groupId = UUID.randomUUID();
        service.createGroupChannel(groupId, "team-x", List.of("alice"), "alice");

        var found = service.findGroupChannel(groupId);
        assertThat(found).isPresent();
        assertThat(found.get().boundGroupId()).isEqualTo(groupId);
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=VisibilityServiceTest`
Expected: FAIL — `VisibilityService` class doesn't exist

- [ ] **Step 6: Implement VisibilityService**

Create `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/VisibilityService.java`:

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.*;
import io.casehub.qhorus.api.store.ChannelMembershipStore;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.api.store.query.ChannelQuery;
import io.casehub.platform.api.identity.CurrentPrincipal;

import java.util.*;

public class VisibilityService implements VisibilityManager {

    private final ChannelService channelService;
    private final ChannelMembershipService membershipService;
    private final ChannelMembershipStore membershipStore;
    private final ChannelStore channelStore;
    private final CurrentPrincipal currentPrincipal;

    public VisibilityService(ChannelService channelService,
                             ChannelMembershipService membershipService,
                             ChannelMembershipStore membershipStore,
                             ChannelStore channelStore,
                             CurrentPrincipal currentPrincipal) {
        this.channelService = channelService;
        this.membershipService = membershipService;
        this.membershipStore = membershipStore;
        this.channelStore = channelStore;
        this.currentPrincipal = currentPrincipal;
    }

    @Override
    public FindOrCreateResult getOrCreateDirect(String participantA, String participantB) {
        if (!ChannelSlugValidator.isValidSegment(participantA)) {
            throw new IllegalArgumentException(
                "Participant ID '" + participantA + "' is not a valid slug segment — "
                + "must match [a-z][a-z0-9]*(-[a-z0-9]+)*");
        }
        if (!ChannelSlugValidator.isValidSegment(participantB)) {
            throw new IllegalArgumentException(
                "Participant ID '" + participantB + "' is not a valid slug segment — "
                + "must match [a-z][a-z0-9]*(-[a-z0-9]+)*");
        }
        if (participantA.equals(participantB)) {
            throw new IllegalArgumentException("Cannot create a DIRECT channel with the same participant");
        }

        String first = participantA.compareTo(participantB) < 0 ? participantA : participantB;
        String second = participantA.compareTo(participantB) < 0 ? participantB : participantA;
        String name = "dm/" + first + "/" + second;

        var req = ChannelCreateRequest.builder(name)
                .visibility(ChannelVisibility.DIRECT)
                .semantic(ChannelSemantic.APPEND)
                .participationPolicy(ParticipationPolicyMode.ENFORCE)
                .build();
        FindOrCreateResult result = channelService.findOrCreate(req);
        Channel channel = result.channel();

        if (result.wasCreated()) {
            String tenancyId = currentPrincipal.tenancyId();
            membershipService.join(channel.id(), first, MemberRole.PARTICIPANT, tenancyId);
            membershipService.join(channel.id(), second, MemberRole.PARTICIPANT, tenancyId);
        } else {
            if (channel.visibility() != ChannelVisibility.DIRECT) {
                throw new IllegalStateException(
                    "Channel '" + name + "' exists but is not a DIRECT channel");
            }
        }
        return result;
    }

    @Override
    public List<Channel> findDirectChannelsFor(String participantId) {
        String tenancyId = currentPrincipal.tenancyId();
        return membershipStore.findChannelsByMemberAndVisibility(
            participantId, ChannelVisibility.DIRECT, tenancyId);
    }

    @Override
    public Channel createGroupChannel(UUID groupId, String groupName,
                                       List<String> memberIds, String coordinatorId) {
        String name = "group/" + groupName;
        var req = ChannelCreateRequest.builder(name)
                .visibility(ChannelVisibility.PRIVATE)
                .semantic(ChannelSemantic.APPEND)
                .participationPolicy(ParticipationPolicyMode.ENFORCE)
                .boundGroupId(groupId)
                .build();
        FindOrCreateResult result = channelService.findOrCreate(req);
        Channel channel = result.channel();

        if (result.wasCreated()) {
            String tenancyId = currentPrincipal.tenancyId();
            membershipService.join(channel.id(), coordinatorId, MemberRole.MODERATOR, tenancyId);
            for (String memberId : memberIds) {
                if (!memberId.equals(coordinatorId)) {
                    membershipService.join(channel.id(), memberId, MemberRole.PARTICIPANT, tenancyId);
                }
            }
        } else {
            if (channel.boundGroupId() == null) {
                throw new IllegalStateException(
                    "Channel '" + name + "' exists but is not bound to any group — "
                    + "cannot bind to group (requested: " + groupId + ")");
            }
            if (!channel.boundGroupId().equals(groupId)) {
                throw new IllegalStateException(
                    "Channel '" + name + "' is bound to a different group "
                    + "(existing: " + channel.boundGroupId() + ", requested: " + groupId + ")");
            }
        }
        return channel;
    }

    @Override
    public Optional<Channel> findGroupChannel(UUID groupId) {
        return channelStore.scan(ChannelQuery.byBoundGroupId(groupId))
                .stream().findFirst();
    }
}
```

- [ ] **Step 7: Create CDI wrapper**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/cdi/CdiVisibilityService.java`:

```java
package io.casehub.qhorus.runtime.cdi;

import io.casehub.qhorus.api.store.ChannelMembershipStore;
import io.casehub.qhorus.api.store.ChannelStore;
import io.casehub.qhorus.runtime.channel.ChannelMembershipService;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.channel.VisibilityService;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class CdiVisibilityService extends VisibilityService {

    @Inject
    public CdiVisibilityService(ChannelService channelService,
                                 ChannelMembershipService membershipService,
                                 ChannelMembershipStore membershipStore,
                                 ChannelStore channelStore,
                                 CurrentPrincipal currentPrincipal) {
        super(channelService, membershipService, membershipStore, channelStore, currentPrincipal);
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=VisibilityServiceTest`
Expected: PASS

- [ ] **Step 9: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add api/ runtime/ runtime-core/ persistence-memory/ testing/
git commit -m "feat(#509): add VisibilityManager interface and VisibilityService

getOrCreateDirect creates deterministic dm/{sorted-a}/{sorted-b} channels.
createGroupChannel creates PRIVATE channels with boundGroupId binding.
findDirectChannelsFor and findGroupChannel provide efficient lookups.
ChannelManager.setVisibility and ChannelReader.findByVisibility added.

Refs #509

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: REST API + BroadcastMembershipManager Integration

After this batch: REST endpoints for DIRECT/GROUP channel operations are
live. BroadcastMembershipManager stamps new broadcast channels with
`visibility = PUBLIC`. Feature is fully usable.

### Task 4: REST endpoints + BroadcastMembershipManager update

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/api/VisibilityResource.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java` (add visibility filter to list)
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/broadcast/BroadcastMembershipManager.java` (add visibility=PUBLIC)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResponse.java` (add visibility + boundGroupId)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/api/VisibilityResourceTest.java`

**Interfaces:**
- Consumes: `VisibilityManager` from Task 3
- Consumes: `ChannelManager.setVisibility()` from Task 3
- Consumes: `ChannelReader.findByVisibility()` from Task 3

- [ ] **Step 1: Update ChannelResponse to include visibility and boundGroupId**

In `ChannelResponse.java`, add `ChannelVisibility visibility` and
`UUID boundGroupId` fields. Update the `from()` static factory to map
from `Channel`.

- [ ] **Step 2: Create VisibilityResource**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/api/VisibilityResource.java`:

```java
package io.casehub.qhorus.runtime.api;

import io.casehub.qhorus.api.channel.*;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Path("/api/channels")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class VisibilityResource {

    @Inject VisibilityManager visibilityManager;
    @Inject ChannelManager channelManager;

    @POST @Path("/direct")
    public Response createDirect(DirectChannelRequest req) {
        var result = visibilityManager.getOrCreateDirect(req.participantA(), req.participantB());
        return Response.status(result.wasCreated() ? 201 : 200)
                .entity(ChannelResponse.from(result.channel(), 0, null))
                .build();
    }

    @GET @Path("/direct/{participantId}")
    public List<ChannelResponse> findDirect(@PathParam("participantId") String participantId) {
        return visibilityManager.findDirectChannelsFor(participantId).stream()
                .map(ch -> ChannelResponse.from(ch, 0, null))
                .toList();
    }

    @POST @Path("/group")
    public Response createGroup(GroupChannelRequest req) {
        Channel ch = visibilityManager.createGroupChannel(
            req.groupId(), req.groupName(), req.memberIds(), req.coordinatorId());
        return Response.status(201).entity(ChannelResponse.from(ch, 0, null)).build();
    }

    @GET @Path("/group/{groupId}")
    public Response findGroup(@PathParam("groupId") UUID groupId) {
        Optional<Channel> ch = visibilityManager.findGroupChannel(groupId);
        return ch.map(c -> Response.ok(ChannelResponse.from(c, 0, null)).build())
                .orElse(Response.status(404).build());
    }

    @PUT @Path("/{id}/visibility")
    public ChannelResponse setVisibility(@PathParam("id") UUID id, VisibilityRequest req) {
        Channel ch = channelManager.setVisibility(id, req.visibility());
        return ChannelResponse.from(ch, 0, null);
    }

    public record DirectChannelRequest(String participantA, String participantB) {}
    public record GroupChannelRequest(UUID groupId, String groupName,
                                       List<String> memberIds, String coordinatorId) {}
    public record VisibilityRequest(ChannelVisibility visibility) {}
}
```

- [ ] **Step 3: Add visibility filter to ChannelResource list endpoint**

In `ChannelResource.java`, add `@QueryParam("visibility") String visibility`
to the existing list/scan endpoint. If present, parse as `ChannelVisibility`
and add to the query.

- [ ] **Step 4: Update BroadcastMembershipManager**

In `BroadcastMembershipManager.joinBroadcastChannel()`, add `.visibility(ChannelVisibility.PUBLIC)` to the `ChannelCreateRequest.builder()` call:

```java
ChannelCreateRequest req = ChannelCreateRequest.builder(BROADCAST_PREFIX + capability)
        .semantic(ChannelSemantic.BROADCAST)
        .visibility(ChannelVisibility.PUBLIC)     // ← new
        .description("Broadcast channel for capability: " + capability)
        .allowedTypes(Set.of(MessageType.STATUS, MessageType.EVENT))
        .build();
```

Add import for `ChannelVisibility`.

- [ ] **Step 5: Write REST integration tests**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/api/VisibilityResourceTest.java`:

```java
@QuarkusTest
class VisibilityResourceTest {

    @Test
    void createDirectChannel() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"participantA": "alice", "participantB": "bob"}
                """)
        .when()
            .post("/api/channels/direct")
        .then()
            .statusCode(201)
            .body("name", equalTo("dm/alice/bob"))
            .body("visibility", equalTo("DIRECT"));
    }

    @Test
    void createDirectChannelIdempotent() {
        given().contentType(ContentType.JSON)
            .body("""{"participantA": "carol", "participantB": "dave"}""")
            .post("/api/channels/direct");

        given().contentType(ContentType.JSON)
            .body("""{"participantA": "dave", "participantB": "carol"}""")
        .when()
            .post("/api/channels/direct")
        .then()
            .statusCode(200)
            .body("name", equalTo("dm/carol/dave"));
    }

    @Test
    void findDirectChannelsForParticipant() {
        given().contentType(ContentType.JSON)
            .body("""{"participantA": "eve", "participantB": "frank"}""")
            .post("/api/channels/direct");

        given()
        .when()
            .get("/api/channels/direct/eve")
        .then()
            .statusCode(200)
            .body("size()", greaterThanOrEqualTo(1));
    }

    @Test
    void createGroupChannel() {
        String groupId = UUID.randomUUID().toString();
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"groupId": "%s", "groupName": "test-team",
                 "memberIds": ["grace", "heidi"], "coordinatorId": "grace"}
                """.formatted(groupId))
        .when()
            .post("/api/channels/group")
        .then()
            .statusCode(201)
            .body("name", equalTo("group/test-team"))
            .body("visibility", equalTo("PRIVATE"));
    }

    @Test
    void setVisibility() {
        // Create a channel first
        String chId = given().contentType(ContentType.JSON)
            .body("""{"participantA": "ivan", "participantB": "judy"}""")
            .post("/api/channels/direct")
            .then().extract().path("id");

        // Change visibility to PRIVATE (removes DIRECT immutability)
        given()
            .contentType(ContentType.JSON)
            .body("""{"visibility": "PRIVATE"}""")
        .when()
            .put("/api/channels/" + chId + "/visibility")
        .then()
            .statusCode(200)
            .body("visibility", equalTo("PRIVATE"));
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=VisibilityResourceTest`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add api/ runtime/ runtime-core/
git commit -m "feat(#509): add REST endpoints for visibility operations + BroadcastMembershipManager update

POST /api/channels/direct, GET /api/channels/direct/{id},
POST /api/channels/group, GET /api/channels/group/{groupId},
PUT /api/channels/{id}/visibility, GET /api/channels?visibility=X.
BroadcastMembershipManager now stamps broadcast channels with visibility=PUBLIC.

Refs #509

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-02-channel-visibility-semantics-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — 30-field record, fromRequest(), Builder
- `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java` — builder pattern
- `api/src/main/java/io/casehub/qhorus/api/channel/ChannelSlugValidator.java` — slug validation, isValidSegment()
- `api/src/main/java/io/casehub/qhorus/api/channel/ChannelManager.java` — channel lifecycle mutations
- `api/src/main/java/io/casehub/qhorus/api/channel/ChannelReader.java` — channel queries
- `api/src/main/java/io/casehub/qhorus/api/channel/MembershipManager.java` — membership API
- `api/src/main/java/io/casehub/qhorus/api/store/ChannelMembershipStore.java` — membership persistence
- `api/src/main/java/io/casehub/qhorus/api/store/query/ChannelQuery.java` — store query builder
- `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` — JPA entity
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelMembershipService.java` — join/leave
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` — ChannelManager+ChannelReader impl
- `runtime-core/src/main/java/io/casehub/qhorus/runtime/broadcast/BroadcastMembershipManager.java` — broadcast lifecycle
- `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelStore.java` — InMemory store
- `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelMembershipStore.java` — InMemory store
- GitHub casehubio/qhorus#509 — focal issue

# ChannelService.create() Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate 13 convenience overloads across ChannelService, ReactiveChannelService, and QhorusMcpTools into a single `create(ChannelCreateRequest)` entry point, powered by a builder on the request record.

**Architecture:** Add a builder to `ChannelCreateRequest` (name required, semantic defaults to APPEND). Delete all convenience overloads. Extract duplicated entity construction to `Channel.fromRequest()`. Migrate all callers to the builder.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Panache, Mutiny

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test a single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
- After API changes: always `mvn install` from root (cross-module visibility)
- Use `mvn` not `./mvnw`
- All commits reference `Refs #218`
- The 14-param `@Tool` method signatures on QhorusMcpTools/ReactiveQhorusMcpTools are MCP framework constraints — do not change their parameter lists
- `ToolOverloadDiscoverabilityTest` guards against public non-@Tool overloads sharing a `@Tool` method name — any new convenience methods must have different names or be package-private

---

### Task 1: Add Builder to ChannelCreateRequest

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequest.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelCreateRequestBuilderTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `ChannelCreateRequest.builder(String name)` → `Builder`, `Builder.build()` → `ChannelCreateRequest`. All subsequent tasks depend on this.

- [ ] **Step 1: Write builder tests**

```java
package io.casehub.qhorus.runtime.channel;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.Set;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;

class ChannelCreateRequestBuilderTest {

    @Test
    void builder_minimalArgs_defaultsSemanticToAppend() {
        ChannelCreateRequest req = ChannelCreateRequest.builder("test-ch").build();
        assertThat(req.name()).isEqualTo("test-ch");
        assertThat(req.semantic()).isEqualTo(ChannelSemantic.APPEND);
        assertThat(req.description()).isNull();
        assertThat(req.allowedTypes()).isNull();
        assertThat(req.deniedTypes()).isNull();
        assertThat(req.hasConnectorBinding()).isFalse();
    }

    @Test
    void builder_allFields_roundTrips() {
        ChannelCreateRequest req = ChannelCreateRequest.builder("full-ch")
                .description("Full channel")
                .semantic(ChannelSemantic.BARRIER)
                .barrierContributors("alice,bob")
                .allowedWriters("alice")
                .adminInstances("admin-1")
                .rateLimitPerChannel(100)
                .rateLimitPerInstance(10)
                .allowedTypes(Set.of(MessageType.QUERY, MessageType.COMMAND))
                .deniedTypes(Set.of(MessageType.EVENT))
                .inboundConnectorId("slack-in")
                .externalKey("C123")
                .outboundConnectorId("slack-out")
                .outboundDestination("#general")
                .build();

        assertThat(req.name()).isEqualTo("full-ch");
        assertThat(req.description()).isEqualTo("Full channel");
        assertThat(req.semantic()).isEqualTo(ChannelSemantic.BARRIER);
        assertThat(req.barrierContributors()).isEqualTo("alice,bob");
        assertThat(req.allowedWriters()).isEqualTo("alice");
        assertThat(req.adminInstances()).isEqualTo("admin-1");
        assertThat(req.rateLimitPerChannel()).isEqualTo(100);
        assertThat(req.rateLimitPerInstance()).isEqualTo(10);
        assertThat(req.allowedTypes()).containsExactlyInAnyOrder(MessageType.QUERY, MessageType.COMMAND);
        assertThat(req.deniedTypes()).containsExactly(MessageType.EVENT);
        assertThat(req.hasConnectorBinding()).isTrue();
        assertThat(req.inboundConnectorId()).isEqualTo("slack-in");
        assertThat(req.externalKey()).isEqualTo("C123");
        assertThat(req.outboundConnectorId()).isEqualTo("slack-out");
        assertThat(req.outboundDestination()).isEqualTo("#general");
    }

    @Test
    void builder_overlappingTypes_throwsViaCompactConstructor() {
        assertThatThrownBy(() -> ChannelCreateRequest.builder("overlap-ch")
                .allowedTypes(Set.of(MessageType.QUERY))
                .deniedTypes(Set.of(MessageType.QUERY))
                .build())
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("intersect");
    }

    @Test
    void builder_partialConnectorBinding_throwsViaCompactConstructor() {
        assertThatThrownBy(() -> ChannelCreateRequest.builder("partial-bind")
                .inboundConnectorId("slack-in")
                .build())
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Connector binding requires all four");
    }

    @Test
    void builder_invalidSlug_throwsViaCompactConstructor() {
        assertThatThrownBy(() -> ChannelCreateRequest.builder("INVALID").build())
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelCreateRequestBuilderTest -pl runtime`
Expected: FAIL — `builder` method not found

- [ ] **Step 3: Implement the Builder**

Add to `ChannelCreateRequest.java` inside the record body, after `hasConnectorBinding()`:

```java
public static Builder builder(String name) {
    return new Builder(name);
}

public static final class Builder {
    private final String name;
    private String description;
    private ChannelSemantic semantic = ChannelSemantic.APPEND;
    private String barrierContributors;
    private String allowedWriters;
    private String adminInstances;
    private Integer rateLimitPerChannel;
    private Integer rateLimitPerInstance;
    private Set<MessageType> allowedTypes;
    private Set<MessageType> deniedTypes;
    private String inboundConnectorId;
    private String externalKey;
    private String outboundConnectorId;
    private String outboundDestination;

    private Builder(String name) {
        this.name = name;
    }

    public Builder description(String description) {
        this.description = description;
        return this;
    }

    public Builder semantic(ChannelSemantic semantic) {
        this.semantic = semantic;
        return this;
    }

    public Builder barrierContributors(String barrierContributors) {
        this.barrierContributors = barrierContributors;
        return this;
    }

    public Builder allowedWriters(String allowedWriters) {
        this.allowedWriters = allowedWriters;
        return this;
    }

    public Builder adminInstances(String adminInstances) {
        this.adminInstances = adminInstances;
        return this;
    }

    public Builder rateLimitPerChannel(Integer rateLimitPerChannel) {
        this.rateLimitPerChannel = rateLimitPerChannel;
        return this;
    }

    public Builder rateLimitPerInstance(Integer rateLimitPerInstance) {
        this.rateLimitPerInstance = rateLimitPerInstance;
        return this;
    }

    public Builder allowedTypes(Set<MessageType> allowedTypes) {
        this.allowedTypes = allowedTypes;
        return this;
    }

    public Builder deniedTypes(Set<MessageType> deniedTypes) {
        this.deniedTypes = deniedTypes;
        return this;
    }

    public Builder inboundConnectorId(String inboundConnectorId) {
        this.inboundConnectorId = inboundConnectorId;
        return this;
    }

    public Builder externalKey(String externalKey) {
        this.externalKey = externalKey;
        return this;
    }

    public Builder outboundConnectorId(String outboundConnectorId) {
        this.outboundConnectorId = outboundConnectorId;
        return this;
    }

    public Builder outboundDestination(String outboundDestination) {
        this.outboundDestination = outboundDestination;
        return this;
    }

    public ChannelCreateRequest build() {
        return new ChannelCreateRequest(name, description, semantic,
                barrierContributors, allowedWriters, adminInstances,
                rateLimitPerChannel, rateLimitPerInstance,
                allowedTypes, deniedTypes,
                inboundConnectorId, externalKey,
                outboundConnectorId, outboundDestination);
    }
}
```

Delete the `simple()` factory method from the same file.

- [ ] **Step 4: Migrate simple() callers (5 sites)**

All in `runtime/src/test/java/`:

Pattern — before:
```java
ChannelCreateRequest.simple("name", ChannelSemantic.APPEND)
```
After:
```java
ChannelCreateRequest.builder("name").build()
```

Files (use IntelliJ find-references on `simple` to locate all 5):
- `io/casehub/qhorus/runtime/channel/ChannelCreateRequestSlugTest.java` (2 sites)
- `io/casehub/qhorus/runtime/channel/ChannelCreateRequestTest.java` (1 site)
- `io/casehub/qhorus/runtime/channel/ChannelBindingUpdateTest.java` (1 site)
- `io/casehub/qhorus/runtime/channel/ChannelServiceFindOrCreateTest.java` (1 site)

- [ ] **Step 5: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all PASS

- [ ] **Step 6: Commit**

```
feat(#218): add Builder to ChannelCreateRequest; delete simple()

Builder with name required, semantic defaults to APPEND. Individual
named setters for all 14 fields including connector binding. build()
delegates to compact constructor for validation.

Refs #218
```

---

### Task 2: Extract Channel.fromRequest()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/Channel.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelFromRequestTest.java`

**Interfaces:**
- Consumes: `ChannelCreateRequest` (from Task 1)
- Produces: `Channel.fromRequest(ChannelCreateRequest req, String tenancyId)` → `Channel`

- [ ] **Step 1: Write Channel.fromRequest() test**

```java
package io.casehub.qhorus.runtime.channel;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Set;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageType;

class ChannelFromRequestTest {

    @Test
    void fromRequest_mapsAllFields() {
        ChannelCreateRequest req = ChannelCreateRequest.builder("from-req-ch")
                .description("Test channel")
                .semantic(ChannelSemantic.BARRIER)
                .barrierContributors("alice,bob")
                .allowedWriters("alice")
                .adminInstances("admin-1")
                .rateLimitPerChannel(100)
                .rateLimitPerInstance(10)
                .allowedTypes(Set.of(MessageType.QUERY, MessageType.COMMAND))
                .deniedTypes(Set.of(MessageType.EVENT))
                .build();

        Channel ch = Channel.fromRequest(req, "tenant-42");

        assertThat(ch.name).isEqualTo("from-req-ch");
        assertThat(ch.description).isEqualTo("Test channel");
        assertThat(ch.semantic).isEqualTo(ChannelSemantic.BARRIER);
        assertThat(ch.barrierContributors).isEqualTo("alice,bob");
        assertThat(ch.allowedWriters).isEqualTo("alice");
        assertThat(ch.adminInstances).isEqualTo("admin-1");
        assertThat(ch.rateLimitPerChannel).isEqualTo(100);
        assertThat(ch.rateLimitPerInstance).isEqualTo(10);
        assertThat(ch.allowedTypes).isEqualTo("COMMAND,EVENT,QUERY");
        assertThat(ch.deniedTypes).isEqualTo("EVENT");
        assertThat(ch.tenancyId).isEqualTo("tenant-42");
    }

    @Test
    void fromRequest_blankWritersNormalisedToNull() {
        ChannelCreateRequest req = ChannelCreateRequest.builder("blank-ch")
                .allowedWriters("  ")
                .adminInstances("")
                .build();

        Channel ch = Channel.fromRequest(req, "t1");

        assertThat(ch.allowedWriters).isNull();
        assertThat(ch.adminInstances).isNull();
    }

    @Test
    void fromRequest_nullTypesSerialiseToNull() {
        ChannelCreateRequest req = ChannelCreateRequest.builder("null-types-ch").build();
        Channel ch = Channel.fromRequest(req, "t1");

        assertThat(ch.allowedTypes).isNull();
        assertThat(ch.deniedTypes).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelFromRequestTest -pl runtime`
Expected: FAIL — `fromRequest` method not found

- [ ] **Step 3: Implement Channel.fromRequest()**

Add to `Channel.java`:

```java
public static Channel fromRequest(ChannelCreateRequest req, String tenancyId) {
    Channel ch = new Channel();
    ch.name = req.name();
    ch.description = req.description();
    ch.semantic = req.semantic();
    ch.barrierContributors = req.barrierContributors();
    ch.allowedWriters = blankToNull(req.allowedWriters());
    ch.adminInstances = blankToNull(req.adminInstances());
    ch.rateLimitPerChannel = req.rateLimitPerChannel();
    ch.rateLimitPerInstance = req.rateLimitPerInstance();
    ch.allowedTypes = MessageType.serializeTypes(req.allowedTypes());
    ch.deniedTypes = MessageType.serializeTypes(req.deniedTypes());
    ch.tenancyId = tenancyId;
    return ch;
}

private static String blankToNull(String s) {
    return (s == null || s.isBlank()) ? null : s;
}
```

Add import: `import io.casehub.qhorus.api.message.MessageType;`

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ChannelFromRequestTest -pl runtime`
Expected: PASS

- [ ] **Step 5: Replace populateChannel() in ChannelService**

In `ChannelService.java`:
- In `create(ChannelCreateRequest)`: replace `Channel channel = populateChannel(req);` with `Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());`
- In `findOrCreateWithBinding()`: replace `Channel channel = populateChannel(req);` with `Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());`
- Delete `private Channel populateChannel(ChannelCreateRequest req)` method
- Delete `private static String blankToNull(String s)` method

- [ ] **Step 6: Replace populateChannel() in ReactiveChannelService**

In `ReactiveChannelService.java`:
- In `create(ChannelCreateRequest)`: replace `Channel channel = populateChannel(req);` with `Channel channel = Channel.fromRequest(req, currentPrincipal.tenancyId());`
- Delete `private Channel populateChannel(ChannelCreateRequest req)` method
- Delete `private static String blankToNull(String s)` method

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all PASS

- [ ] **Step 8: Commit**

```
refactor(#218): extract Channel.fromRequest() — eliminate populateChannel duplication

Static factory on Channel replaces identical private populateChannel()
in ChannelService and ReactiveChannelService. tenancyId passed as
parameter — no CDI dependency on the entity.

Refs #218
```

---

### Task 3: Delete ChannelService + ReactiveChannelService convenience overloads

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java`
- Modify: ~76 test files calling `channelService.create(4-param)` (all in `runtime/src/test/`)
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/ChannelServiceContractTest.java` (abstract helper stays, concrete impls change)
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/ChannelServiceTest.java` (8-param contract helper)
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/ReactiveChannelServiceTest.java` (8-param contract helper)
- Modify: `runtime/src/test/java/io/casehub/qhorus/service/ReactiveChannelServiceDeniedTypesTest.java`
- Modify: test files in `examples/normative-layout/`, `examples/type-system/`, `connector-backend/`

**Interfaces:**
- Consumes: `ChannelCreateRequest.builder()` (from Task 1), `Channel.fromRequest()` (from Task 2)
- Produces: `ChannelService.create(ChannelCreateRequest)` as sole entry point

- [ ] **Step 1: Delete 4 convenience overloads from ChannelService**

In `ChannelService.java`, delete lines 42-67: the 4-param, 5-param, 6-param, and 8-param `create()` methods.

- [ ] **Step 2: Delete 4 convenience overloads from ReactiveChannelService**

In `ReactiveChannelService.java`, delete the 4-param, 5-param, 6-param, and 8-param `create()` methods.

- [ ] **Step 3: Migrate all test callers**

Use IntelliJ find-references on the deleted methods to find all compile errors. Apply this mechanical migration:

**Pattern A — 4-param `channelService.create(name, desc, semantic, barrierContributors)`:**
```java
// Before
channelService.create("ch-name", "Test", ChannelSemantic.APPEND, null);

// After
channelService.create(ChannelCreateRequest.builder("ch-name").description("Test").build());

// With barrier:
// Before
channelService.create("ch-name", "Test", ChannelSemantic.BARRIER, "alice,bob");

// After
channelService.create(ChannelCreateRequest.builder("ch-name")
        .description("Test").semantic(ChannelSemantic.BARRIER)
        .barrierContributors("alice,bob").build());
```

Add import `import io.casehub.qhorus.runtime.channel.ChannelCreateRequest;` to each modified test file. `ChannelSemantic` is already imported at most sites — add `import io.casehub.qhorus.api.channel.ChannelSemantic;` where missing.

**Pattern B — 8-param contract test helpers:**

In `ChannelServiceTest.java`:
```java
// Before
@Override
protected Channel create(String name, String desc, ChannelSemantic sem) {
    return svc.create(name, desc, sem, null, null, null, null, null);
}

// After
@Override
protected Channel create(String name, String desc, ChannelSemantic sem) {
    return svc.create(ChannelCreateRequest.builder(name)
            .description(desc).semantic(sem).build());
}
```

In `ReactiveChannelServiceTest.java`:
```java
// Before
@Override
protected Channel create(String name, String desc, ChannelSemantic sem) {
    return svc.create(name, desc, sem, null, null, null, null, null).await().indefinitely();
}

// After
@Override
protected Channel create(String name, String desc, ChannelSemantic sem) {
    return svc.create(ChannelCreateRequest.builder(name)
            .description(desc).semantic(sem).build()).await().indefinitely();
}
```

**Pattern C — `new ChannelCreateRequest(14 args)` → builder:**

```java
// Before
channelService.create(new ChannelCreateRequest(
        name, desc, ChannelSemantic.APPEND,
        null, null, null, null, null,
        Set.of(MessageType.EVENT), null,
        null, null, null, null));

// After
channelService.create(ChannelCreateRequest.builder(name)
        .description(desc)
        .allowedTypes(Set.of(MessageType.EVENT))
        .build());
```

Apply Pattern C to all `new ChannelCreateRequest(...)` sites found in test files. Use IntelliJ find-references on the ChannelCreateRequest canonical constructor to locate them all.

**Pattern D — reactive `new ChannelCreateRequest(...)` in ReactiveChannelServiceDeniedTypesTest:**
```java
// Before
svc.create(new ChannelCreateRequest(
        "ch", null, ChannelSemantic.APPEND,
        null, null, null, null, null,
        Set.of(MessageType.QUERY), Set.of(MessageType.QUERY),
        null, null, null, null))

// After
svc.create(ChannelCreateRequest.builder("ch")
        .allowedTypes(Set.of(MessageType.QUERY))
        .deniedTypes(Set.of(MessageType.QUERY))
        .build())
```

- [ ] **Step 4: Run full build (cross-module)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all PASS. This catches examples/ and connector-backend/ breakage.

- [ ] **Step 5: Commit**

```
refactor(#218): delete ChannelService + ReactiveChannelService convenience overloads

Removes 4-param, 5-param, 6-param, 8-param create() from both services.
create(ChannelCreateRequest) is the sole entry point. All callers
migrated to ChannelCreateRequest.builder().

Refs #218
```

---

### Task 4: Delete QhorusMcpTools convenience overloads + migrate internal @Tool bodies

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/CommitmentToolTest.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/CommitmentLifecycleTest.java`

**Interfaces:**
- Consumes: `ChannelCreateRequest.builder()` (Task 1), `ChannelService.create(ChannelCreateRequest)` (Task 3)
- Produces: `QhorusMcpTools.createChannel` with only the 14-param @Tool method remaining

- [ ] **Step 1: Delete 4 pkg-private convenience overloads from QhorusMcpTools**

In `QhorusMcpTools.java`, delete the 4-param, 5-param, 6-param, and 8-param `createChannel()` methods (lines 199-222).

- [ ] **Step 2: Migrate @Tool body to builder**

In `QhorusMcpTools.java`, inside the 14-param `createChannel` @Tool method, replace the `new ChannelCreateRequest(...)` construction with the builder:

```java
// Before (inside @Tool method, after semantic parsing)
Channel ch = channelService.create(new ChannelCreateRequest(
        name, description, sem, barrierContributors,
        allowedWriters, adminInstances, rateLimitPerChannel, rateLimitPerInstance,
        parsedAllowed, parsedDenied,
        inboundConnectorId, externalKey, outboundConnectorId, outboundDestination));

// After
Channel ch = channelService.create(ChannelCreateRequest.builder(name)
        .description(description)
        .semantic(sem)
        .barrierContributors(barrierContributors)
        .allowedWriters(allowedWriters)
        .adminInstances(adminInstances)
        .rateLimitPerChannel(rateLimitPerChannel)
        .rateLimitPerInstance(rateLimitPerInstance)
        .allowedTypes(parsedAllowed)
        .deniedTypes(parsedDenied)
        .inboundConnectorId(inboundConnectorId)
        .externalKey(externalKey)
        .outboundConnectorId(outboundConnectorId)
        .outboundDestination(outboundDestination)
        .build());
```

Read the actual @Tool method body first to get the exact variable names — the above shows the pattern.

- [ ] **Step 3: Migrate ReactiveQhorusMcpTools @Tool body**

Same pattern — read the reactive `createChannel` @Tool body and replace `new ChannelCreateRequest(...)` with the builder. The reactive method may delegate to `blockingChannelService.create(req)` — ensure the builder is used to construct `req`.

- [ ] **Step 4: Migrate 9 commitment test callers**

These tests use `tools.createChannel(4-param)` purely as setup. Switch to `channelService.create(builder)`:

In `CommitmentToolTest.java` — add `@Inject ChannelService channelService;` and add import for `ChannelCreateRequest`:

```java
// Before (each call site)
tools.createChannel(ch, "APPEND", null, null);

// After
channelService.create(ChannelCreateRequest.builder(ch).build());
```

In `CommitmentLifecycleTest.java` — same pattern. Add `@Inject ChannelService channelService;` and migrate each `tools.createChannel(ch, "APPEND", null, null)`.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all PASS

- [ ] **Step 6: Commit**

```
refactor(#218): delete QhorusMcpTools convenience overloads; @Tool bodies use builder

Removes 4 pkg-private createChannel overloads. @Tool method bodies in
QhorusMcpTools and ReactiveQhorusMcpTools now construct via builder.
Commitment tests use channelService.create(builder) for setup.

Refs #218
```

---

### Task 5: Migrate ConnectorChannelBackend + remaining positional constructors to builder

**Files:**
- Modify: `connector-backend/src/main/java/io/casehub/qhorus/connector/backend/ConnectorChannelBackend.java`
- Modify: remaining test files still using `new ChannelCreateRequest(14 args)` (use IntelliJ find-usages on the canonical constructor to locate them)

**Interfaces:**
- Consumes: `ChannelCreateRequest.builder()` (Task 1)
- Produces: no positional `new ChannelCreateRequest(...)` call sites remain outside `Builder.build()`

- [ ] **Step 1: Migrate ConnectorChannelBackend.tryAutoCreate()**

In `ConnectorChannelBackend.java` at line 146:

```java
// Before
ChannelCreateRequest req = new ChannelCreateRequest(
        spec.channelName(),
        spec.description(),
        spec.semantic(),
        null, null, null, null, null,
        spec.allowedTypes(),
        spec.deniedTypes(),
        msg.connectorId(),
        lookupKey,
        spec.outboundConnectorId(),
        spec.outboundDestination());

// After
ChannelCreateRequest req = ChannelCreateRequest.builder(spec.channelName())
        .description(spec.description())
        .semantic(spec.semantic())
        .allowedTypes(spec.allowedTypes())
        .deniedTypes(spec.deniedTypes())
        .inboundConnectorId(msg.connectorId())
        .externalKey(lookupKey)
        .outboundConnectorId(spec.outboundConnectorId())
        .outboundDestination(spec.outboundDestination())
        .build();
```

- [ ] **Step 2: Find and migrate remaining test-file positional constructors**

Use IntelliJ find-references on the `ChannelCreateRequest` canonical constructor. Any remaining `new ChannelCreateRequest(...)` calls outside of `Builder.build()` should be migrated to the builder using Pattern C from Task 3.

Expected locations: connector-backend test files, any runtime test files not caught in Task 3.

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all PASS

- [ ] **Step 4: Verify no positional constructors remain**

Use IntelliJ find-references on the `ChannelCreateRequest` canonical constructor. The only remaining caller should be `Builder.build()`.

- [ ] **Step 5: Commit**

```
refactor(#218): migrate all positional ChannelCreateRequest construction to builder

ConnectorChannelBackend.tryAutoCreate() and remaining test sites now
use ChannelCreateRequest.builder(). Only Builder.build() calls the
canonical constructor.

Refs #218
```

---

### Task 6: Final verification + CLAUDE.md update

**Files:**
- Modify: project `CLAUDE.md` (update ChannelCreateRequest documentation)

**Interfaces:**
- Consumes: all previous tasks
- Produces: updated documentation

- [ ] **Step 1: Full build from root**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all PASS including examples/type-system (CI), connector-backend, slack-channel

- [ ] **Step 2: Verify examples/agent-communication compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -Pwith-llm-examples -f examples/agent-communication/pom.xml`
Expected: PASS (catches profile-gated stale call sites)

- [ ] **Step 3: Update CLAUDE.md**

Update the `ChannelCreateRequest` compact constructor documentation to mention the builder:

- Add to the `ChannelService` section: `create(ChannelCreateRequest)` is the sole entry point — convenience overloads removed in #218
- Add note: `ChannelCreateRequest.builder("name")` is the conventional construction API — builder defaults semantic to APPEND, all other fields null
- Add: `Channel.fromRequest(ChannelCreateRequest, String tenancyId)` — static factory for entity construction
- Remove any references to `ChannelCreateRequest.simple()` or `channelService.create(4-param)` overloads

- [ ] **Step 4: Commit**

```
docs(#218): update CLAUDE.md for ChannelCreateRequest builder consolidation

Refs #218
```

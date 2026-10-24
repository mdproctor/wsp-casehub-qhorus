# Enforce Mesh Participation on Message Posting — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/qhorus#495 — design: enforce mesh participation on message posting
**Issue group:** #495

**Goal:** Add a configurable participation policy that determines what happens when an unregistered sender posts a message — OPEN (no check), AUTO_REGISTER (auto-enroll), or ENFORCE (reject).

**Architecture:** Three-level configuration (global default + per-channel override + SPI for custom auto-registration logic). The check sits in `MessageService.dispatch()` after tenant/paused validation, before `AllowedWritersPolicy`. System senders (containing `:`) are exempt. A `MeshAutoRegistrar` SPI with `@DefaultBean` creates minimal instances; consumers override for enriched registration.

**Tech Stack:** Java 21, Quarkus 3.32.2, Flyway, Panache JPA, SmallRye Config

## Global Constraints

- Java 21 API surface (release=21, compiled on Java 26)
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` to run tests
- Use `mvn` not `./mvnw`
- Flyway migration next version: V58
- `ParticipationPolicyMode` enum in `api/channel/` alongside `EnforcementMode`
- `MeshAutoRegistrar` SPI in `api/spi/` following `CommitmentAttestationPolicy` pattern
- System senders (containing `:`) always exempt from participation check
- EVENT messages always exempt from participation check
- Global default is `OPEN` for backward compatibility

---

## Batch 1: API Types and SPI

### Task 1: ParticipationPolicyMode enum + MeshAutoRegistrar SPI + Channel record update

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/ParticipationPolicyMode.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/MeshAutoRegistrar.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — add `participationPolicy` field to record + builder
- Test: `runtime/src/test/java/io/casehub/qhorus/channel/ParticipationPolicyModeTest.java`

**Interfaces:**
- Produces: `ParticipationPolicyMode` enum with `OPEN`, `AUTO_REGISTER`, `ENFORCE` and `boolean requiresLookup()`
- Produces: `MeshAutoRegistrar` interface with `Optional<InstanceRegistration> autoRegister(MessageDispatch dispatch)` and `InstanceRegistration` record
- Produces: `Channel.participationPolicy()` accessor (nullable `ParticipationPolicyMode`)
- Produces: `Channel.Builder.participationPolicy(ParticipationPolicyMode)` setter

- [ ] **Step 1: Write the failing test for ParticipationPolicyMode**

```java
// runtime/src/test/java/io/casehub/qhorus/channel/ParticipationPolicyModeTest.java
package io.casehub.qhorus.channel;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;
import io.casehub.qhorus.api.channel.ParticipationPolicyMode;

class ParticipationPolicyModeTest {

    @Test
    void openDoesNotRequireLookup() {
        assertFalse(ParticipationPolicyMode.OPEN.requiresLookup());
    }

    @Test
    void autoRegisterRequiresLookup() {
        assertTrue(ParticipationPolicyMode.AUTO_REGISTER.requiresLookup());
    }

    @Test
    void enforceRequiresLookup() {
        assertTrue(ParticipationPolicyMode.ENFORCE.requiresLookup());
    }

    @Test
    void threeValuesExist() {
        assertEquals(3, ParticipationPolicyMode.values().length);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ParticipationPolicyModeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Create ParticipationPolicyMode enum**

```java
// api/src/main/java/io/casehub/qhorus/api/channel/ParticipationPolicyMode.java
package io.casehub.qhorus.api.channel;

public enum ParticipationPolicyMode {
    OPEN,
    AUTO_REGISTER,
    ENFORCE;

    public boolean requiresLookup() {
        return this != OPEN;
    }
}
```

- [ ] **Step 4: Create MeshAutoRegistrar SPI**

```java
// api/src/main/java/io/casehub/qhorus/api/spi/MeshAutoRegistrar.java
package io.casehub.qhorus.api.spi;

import io.casehub.qhorus.api.message.MessageDispatch;

import java.util.List;
import java.util.Map;
import java.util.Optional;

public interface MeshAutoRegistrar {

    Optional<InstanceRegistration> autoRegister(MessageDispatch dispatch);

    record InstanceRegistration(
        String instanceId,
        String description,
        List<String> capabilityTags,
        boolean readOnly,
        Map<String, String> metadata
    ) {}
}
```

- [ ] **Step 5: Add participationPolicy to Channel record and Builder**

Add `ParticipationPolicyMode participationPolicy` as a new field on the `Channel` record — after `metadata`. Update the canonical constructor, the `toBuilder()` method, and the `Builder` class (new field, setter method, include in `build()`). Update `fromRequest()` to pass `null` for `participationPolicy` (channels start with no override — inherit global default).

The telescoping constructors do NOT need updating — they already pass `null` for trailing fields. The canonical constructor is the only one that must include the new parameter.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ParticipationPolicyModeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (all 4 tests)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/channel/ParticipationPolicyMode.java
git add api/src/main/java/io/casehub/qhorus/api/spi/MeshAutoRegistrar.java
git add api/src/main/java/io/casehub/qhorus/api/channel/Channel.java
git add runtime/src/test/java/io/casehub/qhorus/channel/ParticipationPolicyModeTest.java
git commit -m "feat: add ParticipationPolicyMode enum, MeshAutoRegistrar SPI, Channel field

Refs casehubio/qhorus#495"
```

---

## Batch 2: Persistence and Configuration

### Task 2: ChannelEntity JPA mapping + Flyway migration + QhorusConfig

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` — add `participationPolicy` String column, update `fromDomain()`/`toDomain()`
- Create: `runtime/src/main/resources/db/qhorus/migration/V58__channel_participation_policy.sql`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` — add `Mesh` nested interface with `participationPolicy()`, add `ChannelConfig.participationPolicy()`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelStore.java` — update if `Channel` record constructor changed (verify after Task 1)
- Test: `runtime/src/test/java/io/casehub/qhorus/channel/ChannelEntityParticipationPolicyTest.java`

**Interfaces:**
- Consumes: `ParticipationPolicyMode` enum (Task 1)
- Consumes: `Channel.participationPolicy()` accessor (Task 1)
- Produces: `QhorusConfig.mesh().participationPolicy()` → `ParticipationPolicyMode` (default `OPEN`)
- Produces: `QhorusConfig.ChannelConfig.participationPolicy()` → `Optional<ParticipationPolicyMode>`
- Produces: `ChannelEntity.participationPolicy` String column with `fromDomain()`/`toDomain()` conversion

- [ ] **Step 1: Write the failing test for ChannelEntity round-trip**

```java
// runtime/src/test/java/io/casehub/qhorus/channel/ChannelEntityParticipationPolicyTest.java
package io.casehub.qhorus.channel;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ParticipationPolicyMode;
import io.casehub.qhorus.runtime.channel.ChannelEntity;

class ChannelEntityParticipationPolicyTest {

    @Test
    void fromDomain_toDomain_roundTrips_autoRegister() {
        Channel ch = Channel.builder("test-ch")
                .participationPolicy(ParticipationPolicyMode.AUTO_REGISTER)
                .build();
        ChannelEntity entity = ChannelEntity.fromDomain(ch);
        assertEquals("AUTO_REGISTER", entity.participationPolicy);
        Channel roundTripped = entity.toDomain();
        assertEquals(ParticipationPolicyMode.AUTO_REGISTER, roundTripped.participationPolicy());
    }

    @Test
    void fromDomain_toDomain_roundTrips_null() {
        Channel ch = Channel.builder("test-ch").build();
        ChannelEntity entity = ChannelEntity.fromDomain(ch);
        assertNull(entity.participationPolicy);
        Channel roundTripped = entity.toDomain();
        assertNull(roundTripped.participationPolicy());
    }

    @Test
    void fromDomain_toDomain_roundTrips_enforce() {
        Channel ch = Channel.builder("test-ch")
                .participationPolicy(ParticipationPolicyMode.ENFORCE)
                .build();
        ChannelEntity entity = ChannelEntity.fromDomain(ch);
        assertEquals("ENFORCE", entity.participationPolicy);
        Channel roundTripped = entity.toDomain();
        assertEquals(ParticipationPolicyMode.ENFORCE, roundTripped.participationPolicy());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelEntityParticipationPolicyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `participationPolicy` field not found on `ChannelEntity`

- [ ] **Step 3: Add participationPolicy to ChannelEntity**

Add to `ChannelEntity.java` after the `enforcementExclusions` field:

```java
@Column(name = "participation_policy")
public String participationPolicy;
```

Update `fromDomain()`:
```java
e.participationPolicy = channel.participationPolicy() != null ? channel.participationPolicy().name() : null;
```

Update `toDomain()` — add the new parameter to the canonical constructor call. Follow the existing `enforcementMode` pattern:
```java
participationPolicy != null ? ParticipationPolicyMode.valueOf(participationPolicy) : null
```

- [ ] **Step 4: Create Flyway migration**

```sql
-- runtime/src/main/resources/db/qhorus/migration/V58__channel_participation_policy.sql
ALTER TABLE channel ADD COLUMN participation_policy VARCHAR(20);
```

- [ ] **Step 5: Add Mesh and ChannelConfig entries to QhorusConfig**

Add to `QhorusConfig.java`:

```java
// Add method accessor alongside the existing ones (cleanup, agentCard, etc.)
Mesh mesh();

// Add new nested interface
interface Mesh {
    @WithDefault("OPEN")
    ParticipationPolicyMode participationPolicy();
}
```

Add to `ChannelConfig` interface:

```java
Optional<ParticipationPolicyMode> participationPolicy();
```

Import `ParticipationPolicyMode` at the top.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ChannelEntityParticipationPolicyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (all 3 tests)

- [ ] **Step 7: Run full test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime-core,runtime,persistence-memory`
Expected: All existing tests pass. The new `Channel` record field is nullable, so existing constructors pass `null` and nothing breaks.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java
git add runtime/src/main/resources/db/qhorus/migration/V58__channel_participation_policy.sql
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java
git add runtime/src/test/java/io/casehub/qhorus/channel/ChannelEntityParticipationPolicyTest.java
git commit -m "feat: add participation_policy column, ChannelEntity mapping, QhorusConfig

V58 Flyway migration adds nullable participation_policy column.
QhorusConfig.Mesh.participationPolicy() defaults to OPEN.
ChannelConfig gains optional participationPolicy().

Refs casehubio/qhorus#495"
```

---

## Batch 3: Core Enforcement Logic

### Task 3: DefaultMeshAutoRegistrar + MessageService participation gate

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/DefaultMeshAutoRegistrar.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` — add participation check in `dispatch()`, add `MeshAutoRegistrar` + `ParticipationPolicyMode` fields, update constructor
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/cdi/CdiMessageService.java` — add `MeshAutoRegistrar` + `QhorusConfig` mesh config to constructor
- Test: `runtime/src/test/java/io/casehub/qhorus/message/MessageServiceParticipationTest.java`

**Interfaces:**
- Consumes: `ParticipationPolicyMode` enum (Task 1)
- Consumes: `MeshAutoRegistrar` SPI (Task 1)
- Consumes: `Channel.participationPolicy()` (Task 1)
- Consumes: `QhorusConfig.mesh().participationPolicy()` (Task 2)
- Consumes: `InstanceService.findByInstanceId(String)` → `Optional<Instance>` (existing)
- Consumes: `InstanceService.register(String, String, List<String>, String, boolean, Map<String, String>)` (existing)
- Produces: participation gate in `MessageService.dispatch()` — enforces OPEN/AUTO_REGISTER/ENFORCE

- [ ] **Step 1: Write the failing tests for participation enforcement**

```java
// runtime/src/test/java/io/casehub/qhorus/message/MessageServiceParticipationTest.java
package io.casehub.qhorus.message;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.channel.ParticipationPolicyMode;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.MeshAutoRegistrar;
import io.casehub.qhorus.api.spi.MeshAutoRegistrar.InstanceRegistration;
import io.casehub.qhorus.persistence.memory.InMemoryChannelStore;
import io.casehub.qhorus.persistence.memory.InMemoryInstanceStore;
import io.casehub.qhorus.persistence.memory.InMemoryMessageStore;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.channel.RateLimiter;
import io.casehub.qhorus.runtime.config.QhorusConfig;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.message.MessageTypePolicy;

class MessageServiceParticipationTest {

    private InMemoryInstanceStore instanceStore;
    private InstanceService instanceService;
    private MessageService messageService;
    private MeshAutoRegistrar autoRegistrar;
    private ParticipationPolicyMode globalDefault;

    @BeforeEach
    void setUp() {
        instanceStore = new InMemoryInstanceStore();
        instanceService = new InstanceService(instanceStore);
        globalDefault = ParticipationPolicyMode.OPEN;
        autoRegistrar = dispatch -> Optional.of(new InstanceRegistration(
                dispatch.sender(), "Auto-registered", List.of(), false, Map.of()));
        // MessageService is constructed with the new fields — see Step 3 for wiring
    }

    // --- OPEN mode tests ---

    @Test
    void open_allowsUnregisteredSender() {
        // Channel with OPEN policy (default)
        // Unregistered sender should be allowed
        // Assert: dispatch succeeds, no instance registered
    }

    // --- ENFORCE mode tests ---

    @Test
    void enforce_rejectsUnregisteredSender() {
        // Channel with ENFORCE policy
        // Unregistered sender should get IllegalStateException
    }

    @Test
    void enforce_allowsRegisteredSender() {
        // Channel with ENFORCE policy
        // Pre-registered sender should be allowed
    }

    // --- AUTO_REGISTER mode tests ---

    @Test
    void autoRegister_registersUnknownSender() {
        // Channel with AUTO_REGISTER policy
        // Unregistered sender should be auto-registered and dispatch succeeds
        // Assert: instance now exists in store
    }

    @Test
    void autoRegister_skipsAlreadyRegistered() {
        // Channel with AUTO_REGISTER policy
        // Pre-registered sender should not trigger auto-registration
    }

    @Test
    void autoRegister_spiReturningEmpty_skipsRegistration() {
        // SPI returns Optional.empty()
        // Dispatch should proceed without registration
    }

    @Test
    void autoRegister_spiException_failsDispatch() {
        // SPI throws RuntimeException
        // Dispatch should fail with that exception
    }

    // --- Exemptions ---

    @Test
    void systemSender_bypassesAllModes() {
        // Channel with ENFORCE policy
        // Sender "system:watchdog" (contains ':') should be allowed without registration
    }

    @Test
    void eventType_bypassesAllModes() {
        // Channel with ENFORCE policy
        // EVENT type message from unregistered sender should be allowed
    }

    // --- Per-channel override ---

    @Test
    void perChannelOverride_takePrecedenceOverGlobal() {
        // Global = OPEN, channel = ENFORCE
        // Unregistered sender should be rejected
    }

    @Test
    void nullChannelOverride_inheritsGlobal() {
        // Global = ENFORCE, channel = null
        // Unregistered sender should be rejected (inherits global)
    }
}
```

Note: The actual test bodies require constructing `MessageService` with the new fields. The full test implementations will use `InMemory*Store` instances and mock/stub the dependencies that aren't relevant to participation testing (ledger recorder, observer callbacks, etc.), following the pattern in `MessageServiceContractTest`. The exact wiring depends on the constructor shape established in Step 3.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageServiceParticipationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — constructor/method not found

- [ ] **Step 3: Create DefaultMeshAutoRegistrar**

```java
// runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/DefaultMeshAutoRegistrar.java
package io.casehub.qhorus.runtime.instance;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.spi.MeshAutoRegistrar;

import java.util.List;
import java.util.Map;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class DefaultMeshAutoRegistrar implements MeshAutoRegistrar {

    @Override
    public Optional<InstanceRegistration> autoRegister(MessageDispatch dispatch) {
        return Optional.of(new InstanceRegistration(
                dispatch.sender(),
                "Auto-registered by mesh participation policy",
                List.of(),
                false,
                Map.of()));
    }
}
```

- [ ] **Step 4: Add participation check to MessageService.dispatch()**

In `MessageService.java`:

1. Add two new fields: `MeshAutoRegistrar meshAutoRegistrar` and `ParticipationPolicyMode globalParticipationPolicy`
2. Add them as new constructor parameters (append after `maxCorrectionsPerMessage`)
3. Update the protected no-arg constructor to initialize both to `null`

Add the participation check after the paused check (line ~224) and before the AllowedWritersPolicy check (line ~251):

```java
if (ch != null && dispatch.type() != MessageType.EVENT && !dispatch.sender().contains(":")) {
    ParticipationPolicyMode mode = ch.participationPolicy() != null
            ? ch.participationPolicy() : globalParticipationPolicy;
    if (mode != null && mode.requiresLookup()) {
        boolean registered = instanceService.findByInstanceId(dispatch.sender()).isPresent();
        if (!registered) {
            if (mode == ParticipationPolicyMode.ENFORCE) {
                throw new IllegalStateException(
                        "Sender '" + dispatch.sender() + "' is not a registered mesh instance. "
                        + "Register via meshRegister or POST /api/instances before posting.");
            }
            meshAutoRegistrar.autoRegister(dispatch).ifPresent(reg ->
                    instanceService.register(reg.instanceId(), reg.description(),
                            reg.capabilityTags(), null, reg.readOnly(), reg.metadata()));
        }
    }
}
```

- [ ] **Step 5: Update CdiMessageService constructor**

In `CdiMessageService.java`, add two new `@Inject` parameters:

```java
MeshAutoRegistrar meshAutoRegistrar,
QhorusConfig config  // already partially available — use config.mesh().participationPolicy()
```

Pass `meshAutoRegistrar` and `config.mesh().participationPolicy()` to the `super(...)` call as the two new trailing parameters.

Note: `QhorusConfig config` is already injected — add `config.mesh().participationPolicy()` in the super call. Add `MeshAutoRegistrar meshAutoRegistrar` as a new constructor parameter.

- [ ] **Step 6: Fill in test bodies and run**

Complete the test method bodies using `InMemory*Store` instances, constructing `MessageService` with the new parameters. Use the pattern from `MessageServiceContractTest` for constructing channels with specific policies.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageServiceParticipationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (all 11 tests)

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass. Existing `MessageService` tests work because the global default is `OPEN` (no behavioral change).

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/instance/DefaultMeshAutoRegistrar.java
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/cdi/CdiMessageService.java
git add runtime/src/test/java/io/casehub/qhorus/message/MessageServiceParticipationTest.java
git commit -m "feat: implement mesh participation enforcement gate in MessageService

OPEN/AUTO_REGISTER/ENFORCE modes with system sender and EVENT exemptions.
DefaultMeshAutoRegistrar creates minimal instances. Gate runs before
AllowedWritersPolicy so auto-registered capabilities are visible to ACL.

Refs casehubio/qhorus#495"
```

---

## Batch 4: Channel Configuration Bootstrap and Integration

### Task 4: ChannelConfigBootstrap support + ChannelsApi MCP tools + integration test

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrap.java` — read `participationPolicy` from config, set on channel
- Modify: `api/src/main/java/io/casehub/qhorus/api/spi/channels/ChannelsApi.java` — add participation policy query/mutation
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/ChannelCore.java` — add `setParticipationPolicy()` method
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java` — add REST endpoint for participation policy
- Test: `runtime/src/test/java/io/casehub/qhorus/message/MessageServiceParticipationIntegrationTest.java`

**Interfaces:**
- Consumes: `ParticipationPolicyMode` enum (Task 1)
- Consumes: `QhorusConfig.ChannelConfig.participationPolicy()` (Task 2)
- Consumes: Participation gate in `MessageService.dispatch()` (Task 3)
- Produces: `ChannelsApi.getParticipationPolicy(UUID)` query
- Produces: `ChannelsApi.setParticipationPolicy(UUID, ParticipationPolicyMode)` mutation
- Produces: `ChannelResource` REST endpoint `PUT /api/channels/{id}/participation-policy`

- [ ] **Step 1: Write the failing integration test**

```java
// runtime/src/test/java/io/casehub/qhorus/message/MessageServiceParticipationIntegrationTest.java
package io.casehub.qhorus.message;

import static org.junit.jupiter.api.Assertions.*;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ParticipationPolicyMode;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MessageServiceParticipationIntegrationTest {

    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject InstanceService instanceService;

    @Test
    @TestTransaction
    void autoRegister_createsInstanceAndDispatchSucceeds() {
        Channel ch = channelService.create(ChannelCreateRequest.builder("participation-test")
                .build());
        channelService.setParticipationPolicy(ch.id(), ParticipationPolicyMode.AUTO_REGISTER);

        messageService.dispatch(MessageDispatch.builder()
                .channelId(ch.id())
                .sender("new-agent")
                .type(MessageType.STATUS)
                .content("Hello")
                .actorType(ActorType.AGENT)
                .build());

        assertTrue(instanceService.findByInstanceId("new-agent").isPresent(),
                "Agent should be auto-registered after dispatch");
    }

    @Test
    @TestTransaction
    void enforce_rejectsUnregisteredAndAllowsRegistered() {
        Channel ch = channelService.create(ChannelCreateRequest.builder("enforce-test")
                .build());
        channelService.setParticipationPolicy(ch.id(), ParticipationPolicyMode.ENFORCE);

        assertThrows(IllegalStateException.class, () ->
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(ch.id())
                        .sender("unknown-agent")
                        .type(MessageType.STATUS)
                        .content("Hello")
                        .actorType(ActorType.AGENT)
                        .build()));

        instanceService.register("known-agent", "Test agent", java.util.List.of());

        assertDoesNotThrow(() ->
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(ch.id())
                        .sender("known-agent")
                        .type(MessageType.STATUS)
                        .content("Hello from known")
                        .actorType(ActorType.AGENT)
                        .build()));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageServiceParticipationIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `setParticipationPolicy` method not found

- [ ] **Step 3: Add setParticipationPolicy to ChannelService**

Add method to `ChannelService.java` following the existing `setTypeConstraints`/`setAllowedWriters` pattern — takes `UUID channelId` and `ParticipationPolicyMode policy`, updates the channel entity.

- [ ] **Step 4: Add ChannelsApi query and mutation**

In `ChannelsApi.java`, add a new section:

```java
// --- Mesh Participation ---

@PlatformQuery("Get the participation policy for a channel")
ParticipationPolicyMode getParticipationPolicy(@Description("Channel ID") UUID channelId);

@PlatformMutation("Set the participation policy for a channel")
void setParticipationPolicy(
        @Description("Channel ID") UUID channelId,
        @Description("Policy: OPEN, AUTO_REGISTER, or ENFORCE") ParticipationPolicyMode policy);
```

- [ ] **Step 5: Add REST endpoint to ChannelResource**

Following the existing `PUT /api/channels/{id}/enforcement-mode` pattern:

```java
@PUT
@Path("{id}/participation-policy")
public Response setParticipationPolicy(@PathParam("id") String id,
        ParticipationPolicyRequest request) {
    Channel ch = channelCore.resolveChannel(id);
    channelCore.setParticipationPolicy(ch.id(), request.policy());
    return Response.noContent().build();
}
```

Add `ChannelCore.setParticipationPolicy()` to delegate to `ChannelService`.

- [ ] **Step 6: Update ChannelConfigBootstrap**

In `ChannelConfigBootstrap.java`, in the channel creation/drift-detection loop, add:

```java
channelConfig.participationPolicy().ifPresent(policy -> {
    channelService.setParticipationPolicy(channel.id(), policy);
});
```

Follow the existing pattern used for `enforcementMode`.

- [ ] **Step 7: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MessageServiceParticipationIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (both tests)

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelConfigBootstrap.java
git add api/src/main/java/io/casehub/qhorus/api/spi/channels/ChannelsApi.java
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/api/core/ChannelCore.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/api/ChannelResource.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java
git add runtime/src/test/java/io/casehub/qhorus/message/MessageServiceParticipationIntegrationTest.java
git commit -m "feat: add participation policy config bootstrap, ChannelsApi, REST endpoint

ChannelConfigBootstrap reads participationPolicy from config.
ChannelsApi exposes query/mutation MCP tools.
ChannelResource serves PUT /api/channels/{id}/participation-policy.
Integration tests verify full AUTO_REGISTER and ENFORCE pipeline.

Refs casehubio/qhorus#495"
```

---

## References

- [2026-10-24-enforce-mesh-participation-design.md] — design spec this plan implements
- `runtime-core/.../message/MessageService.java:208-623` — dispatch enforcement pipeline
- `runtime-core/.../instance/InstanceService.java` — instance registration API
- `runtime-core/.../config/QhorusConfig.java` — config mapping pattern
- `runtime/.../cdi/CdiMessageService.java:39-83` — CDI constructor wiring
- `runtime/.../channel/ChannelEntity.java:100-101` — enforcementMode String↔enum pattern
- `runtime/.../channel/ChannelConfigBootstrap.java` — config-file per-channel bootstrap
- `api/.../channel/EnforcementMode.java` — enum pattern reference
- `api/.../spi/CommitmentAttestationPolicy.java` — SPI @DefaultBean pattern reference
- `docs/protocols/casehub/system-sender-trust-exemption.md` — system sender convention
- casehubio/qhorus#495 — focal issue

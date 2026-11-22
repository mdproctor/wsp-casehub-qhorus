# Playbook Protocol Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #528 — playbook-executor-backed ChannelProtocol bridge
**Issue group:** #528, #529

**Goal:** Connect the platform playbook executor to qhorus channel
protocol evaluation, enabling YAML-defined protocols that evaluate on
every dispatch and fire deadline events when time elapses without
messages.

**Architecture:** ProtocolResolver SPI as a fallback in ProtocolRegistry
(api/spi/), PlaybookProtocolResolver in a new optional bridge module
(casehub-qhorus-playbook-bridge), ProtocolDefinitionStore for YAML
persistence, compilation cache, in-memory deadline scheduler with
lifecycle recovery.

**Tech Stack:** Java 21 (JVM 26), Quarkus 3.32.2, Panache JPA (H2 test
/ PostgreSQL prod), casehub-platform-yaml-step-runtime

## Global Constraints

- Quarkus 3.32.2 — do not upgrade
- Java 21 source (JVM 26) — do not use Java 22+ language features
- `${casehub.version}` = `0.4-SNAPSHOT` for all casehub deps
- Flyway domain migrations: V62+ (V59–V61 taken)
- Module naming: `casehub-qhorus-playbook-bridge` / artifactId same
- Package: `io.casehub.qhorus.playbook.bridge`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- Jandex required for new modules (`jandex-maven-plugin`)
- SPI interfaces → `api/spi/`; store interfaces → bridge module (not api/store/)
- Tests: H2 in-memory, `quarkus.http.test-port=0`, named PU `qhorus`

---

## Batch 1: Foundation — ProtocolResolver SPI + ProtocolRegistry fallback

After this batch: `ProtocolRegistry` supports dynamic protocol
resolution via an optional fallback SPI. All existing tests pass.
No bridge module yet — just the extension point.

### Task 1: ProtocolResolver SPI and ProtocolRegistry fallback

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/ProtocolResolver.java`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/protocol/ProtocolRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/RuntimeBeans.java:306-310`
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java:364-366`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/ProtocolRegistryTest.java`

**Interfaces:**
- Produces: `ProtocolResolver.resolve(List<String>, UUID)` → `List<ChannelProtocol>`
- Produces: `ProtocolRegistry.forProtocols(List<String>, UUID)` (new overload)

- [ ] **Step 1: Write failing test for resolver fallback**

Add to `ProtocolRegistryTest.java`:

```java
@Test
void forProtocols_delegatesToResolver_forUnknownNames() {
    ChannelProtocol dynamicProtocol = stub("DYNAMIC");
    ProtocolResolver resolver = (names, channelId) -> {
        if (names.contains("DYNAMIC")) return List.of(dynamicProtocol);
        return List.of();
    };
    ProtocolRegistry registry = new ProtocolRegistry(stubs("STATIC"), resolver);

    UUID channelId = UUID.randomUUID();
    List<ChannelProtocol> result = registry.forProtocols(
            List.of("STATIC", "DYNAMIC"), channelId);

    assertThat(result).hasSize(2);
    assertThat(result.get(0).protocolName()).isEqualTo("STATIC");
    assertThat(result.get(1).protocolName()).isEqualTo("DYNAMIC");
}

@Test
void forProtocols_noResolver_unknownNamesSkipped() {
    ProtocolRegistry registry = new ProtocolRegistry(stubs("ALPHA"));
    UUID channelId = UUID.randomUUID();
    List<ChannelProtocol> result = registry.forProtocols(
            List.of("ALPHA", "MISSING"), channelId);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).protocolName()).isEqualTo("ALPHA");
}

@Test
void forProtocols_staticTakesPrecedence_overResolver() {
    ChannelProtocol staticP = stub("SHARED");
    ChannelProtocol dynamicP = stub("SHARED");
    ProtocolResolver resolver = (names, channelId) -> List.of(dynamicP);
    ProtocolRegistry registry = new ProtocolRegistry(List.of(staticP), resolver);

    UUID channelId = UUID.randomUUID();
    List<ChannelProtocol> result = registry.forProtocols(
            List.of("SHARED"), channelId);

    assertThat(result).hasSize(1);
    assertThat(result.get(0)).isSameAs(staticP);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProtocolRegistryTest -pl runtime`
Expected: Compilation error — `ProtocolResolver` does not exist, `ProtocolRegistry` has no 2-arg constructor or `forProtocols(List, UUID)`.

- [ ] **Step 3: Create ProtocolResolver interface**

Create `api/src/main/java/io/casehub/qhorus/api/spi/ProtocolResolver.java`:

```java
package io.casehub.qhorus.api.spi;

import java.util.List;
import java.util.UUID;

public interface ProtocolResolver {

    List<ChannelProtocol> resolve(List<String> unresolvedNames, UUID channelId);
}
```

- [ ] **Step 4: Add resolver support to ProtocolRegistry**

Add a `resolver` field and new constructor to `ProtocolRegistry`:

```java
private final ProtocolResolver resolver;

public ProtocolRegistry(List<? extends ChannelProtocol> protocols,
                        ProtocolResolver resolver) {
    this(buildMap(protocols));
    // Can't assign in delegating constructor — use a different pattern
}
```

Actually, since the class uses a private constructor with the map, refactor to:

```java
private final Map<String, ChannelProtocol> registry;
private final ProtocolResolver resolver;

public ProtocolRegistry(List<? extends ChannelProtocol> protocols) {
    this(protocols, null);
}

public ProtocolRegistry(List<? extends ChannelProtocol> protocols,
                        ProtocolResolver resolver) {
    this.registry = buildMap(protocols);
    this.resolver = resolver;
}
```

Add the new `forProtocols` overload:

```java
public List<ChannelProtocol> forProtocols(List<String> names, UUID channelId) {
    List<ChannelProtocol> result = new ArrayList<>(names.size());
    List<String> unresolved = new ArrayList<>();
    for (String name : names) {
        ChannelProtocol p = registry.get(name);
        if (p != null) {
            result.add(p);
        } else {
            unresolved.add(name);
        }
    }
    if (!unresolved.isEmpty()) {
        if (resolver != null) {
            result.addAll(resolver.resolve(unresolved, channelId));
        } else {
            for (String name : unresolved) {
                LOG.warnf("Unknown protocol '%s' — skipped. Registered: %s",
                           name, registry.keySet());
            }
        }
    }
    return result;
}
```

Update existing `forProtocols(List<String>)` to delegate:

```java
public List<ChannelProtocol> forProtocols(List<String> names) {
    return forProtocols(names, null);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProtocolRegistryTest -pl runtime`
Expected: All tests PASS (new + existing).

- [ ] **Step 6: Update RuntimeBeans producer to accept optional resolver**

In `RuntimeBeans.java`, update the `protocolRegistry` producer method:

```java
@Produces @ApplicationScoped
public ProtocolRegistry protocolRegistry(
        @Any Instance<ChannelProtocol> protocols,
        Instance<ProtocolResolver> resolverInstance) {
    List<ChannelProtocol> list = StreamSupport.stream(
            protocols.spliterator(), false).toList();
    ProtocolResolver resolver = resolverInstance.isUnsatisfied()
            ? null : resolverInstance.get();
    return new ProtocolRegistry(list, resolver);
}
```

- [ ] **Step 7: Update MessageService dispatch call**

In `MessageService.java` at line 366, change:

```java
// Before:
protocolRegistry.forProtocols(ch.protocols());
// After:
protocolRegistry.forProtocols(ch.protocols(), ch.id());
```

- [ ] **Step 8: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All existing tests PASS. No regressions.

- [ ] **Step 9: Compile api module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api`
Expected: BUILD SUCCESS (ProtocolResolver compiles with existing api types).

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/spi/ProtocolResolver.java
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/message/protocol/ProtocolRegistry.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/RuntimeBeans.java
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java
git add runtime/src/test/java/io/casehub/qhorus/runtime/message/protocol/ProtocolRegistryTest.java
git commit -m "feat: add ProtocolResolver SPI and fallback on ProtocolRegistry

ProtocolRegistry.forProtocols(names, channelId) delegates unresolved
protocol names to an optional ProtocolResolver. Static CDI-registered
protocols take precedence. Existing single-arg overload preserved for
backward compatibility.

Refs #528

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Bridge module scaffold — definition store + migration

After this batch: The `playbook-bridge` module exists with
`ProtocolDefinitionStore`, JPA entity, Flyway migration, and
InMemory test impl. Definitions can be stored and queried. No
protocol evaluation yet.

### Task 2: Module scaffold and ProtocolDefinitionStore

**Files:**
- Create: `playbook-bridge/pom.xml`
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinition.java`
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinitionStore.java`
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinitionEntity.java`
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/JpaProtocolDefinitionStore.java`
- Create: `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/InMemoryProtocolDefinitionStore.java`
- Create: `playbook-bridge/src/main/resources/db/qhorus/migration/V62__protocol_definition.sql`
- Create: `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinitionStoreTest.java`
- Modify: `pom.xml` (parent — add `<module>playbook-bridge</module>`)

**Interfaces:**
- Produces: `ProtocolDefinition(UUID id, UUID channelId, String protocolName, String yamlContent, String tenancyId, Instant createdAt)`
- Produces: `ProtocolDefinitionStore` — `findByChannelAndProtocol`, `findByChannel`, `existsByProtocolName`, `put`, `delete`, `deleteByChannel`

- [ ] **Step 1: Create module pom.xml**

Create `playbook-bridge/pom.xml` following the a2a-push-notification pattern:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-qhorus-playbook-bridge</artifactId>
  <name>CaseHub Qhorus - Playbook Bridge</name>
  <description>Optional playbook-executor-backed ChannelProtocol bridge.
Activates by classpath presence.</description>

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
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-scheduler</artifactId>
    </dependency>

    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>

    <!-- Test -->
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
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

Add `<module>playbook-bridge</module>` to the parent pom.xml module list (before `persistence-memory`).

- [ ] **Step 2: Create ProtocolDefinition record**

Create `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinition.java`:

```java
package io.casehub.qhorus.playbook.bridge;

import java.time.Instant;
import java.util.UUID;

public record ProtocolDefinition(
        UUID id,
        UUID channelId,
        String protocolName,
        String yamlContent,
        String tenancyId,
        Instant createdAt) {}
```

- [ ] **Step 3: Create ProtocolDefinitionStore interface**

Create `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinitionStore.java`:

```java
package io.casehub.qhorus.playbook.bridge;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface ProtocolDefinitionStore {

    Optional<ProtocolDefinition> findByChannelAndProtocol(
            UUID channelId, String protocolName);

    List<ProtocolDefinition> findByChannel(UUID channelId);

    boolean existsByProtocolName(String protocolName);

    void put(ProtocolDefinition definition);

    void delete(UUID channelId, String protocolName);

    void deleteByChannel(UUID channelId);
}
```

- [ ] **Step 4: Create InMemoryProtocolDefinitionStore**

Create `../../playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/InMemoryProtocolDefinitionStore.java`:

```java
package io.casehub.qhorus.playbook.bridge;

import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryProtocolDefinitionStore implements ProtocolDefinitionStore {

    private final ConcurrentHashMap<String, ProtocolDefinition> store = new ConcurrentHashMap<>();

    private static String key(UUID channelId, String protocolName) {
        return channelId + ":" + protocolName;
    }

    @Override
    public Optional<ProtocolDefinition> findByChannelAndProtocol(
            UUID channelId, String protocolName) {
        return Optional.ofNullable(store.get(key(channelId, protocolName)));
    }

    @Override
    public List<ProtocolDefinition> findByChannel(UUID channelId) {
        String prefix = channelId + ":";
        return store.values().stream()
                .filter(d -> d.channelId().equals(channelId))
                .toList();
    }

    @Override
    public boolean existsByProtocolName(String protocolName) {
        return store.values().stream()
                .anyMatch(d -> d.protocolName().equals(protocolName));
    }

    @Override
    public void put(ProtocolDefinition definition) {
        store.put(key(definition.channelId(), definition.protocolName()),
                  definition);
    }

    @Override
    public void delete(UUID channelId, String protocolName) {
        store.remove(key(channelId, protocolName));
    }

    @Override
    public void deleteByChannel(UUID channelId) {
        store.entrySet().removeIf(e ->
                e.getValue().channelId().equals(channelId));
    }
}
```

- [ ] **Step 5: Write store contract tests**

Create `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinitionStoreTest.java`:

```java
package io.casehub.qhorus.playbook.bridge;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ProtocolDefinitionStoreTest {

    private InMemoryProtocolDefinitionStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryProtocolDefinitionStore();
    }

    private ProtocolDefinition def(UUID channelId, String name, String yaml) {
        return new ProtocolDefinition(UUID.randomUUID(), channelId, name,
                yaml, "DEFAULT", Instant.now());
    }

    @Test
    void put_and_find() {
        UUID ch = UUID.randomUUID();
        store.put(def(ch, "approval-gate", "version: 1.0"));
        assertThat(store.findByChannelAndProtocol(ch, "approval-gate"))
                .isPresent()
                .get().extracting(ProtocolDefinition::yamlContent)
                .isEqualTo("version: 1.0");
    }

    @Test
    void findByChannel_returnsAll() {
        UUID ch = UUID.randomUUID();
        store.put(def(ch, "gate-a", "a"));
        store.put(def(ch, "gate-b", "b"));
        store.put(def(UUID.randomUUID(), "gate-c", "c"));
        assertThat(store.findByChannel(ch)).hasSize(2);
    }

    @Test
    void existsByProtocolName_true() {
        store.put(def(UUID.randomUUID(), "quorum", "q"));
        assertThat(store.existsByProtocolName("quorum")).isTrue();
    }

    @Test
    void existsByProtocolName_false() {
        assertThat(store.existsByProtocolName("missing")).isFalse();
    }

    @Test
    void delete_removesEntry() {
        UUID ch = UUID.randomUUID();
        store.put(def(ch, "gate", "g"));
        store.delete(ch, "gate");
        assertThat(store.findByChannelAndProtocol(ch, "gate")).isEmpty();
    }

    @Test
    void deleteByChannel_removesAll() {
        UUID ch = UUID.randomUUID();
        store.put(def(ch, "a", "1"));
        store.put(def(ch, "b", "2"));
        store.deleteByChannel(ch);
        assertThat(store.findByChannel(ch)).isEmpty();
    }

    @Test
    void put_overwrites() {
        UUID ch = UUID.randomUUID();
        store.put(def(ch, "gate", "v1"));
        store.put(def(ch, "gate", "v2"));
        assertThat(store.findByChannelAndProtocol(ch, "gate"))
                .get().extracting(ProtocolDefinition::yamlContent)
                .isEqualTo("v2");
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl playbook-bridge`
Expected: All 7 tests PASS.

- [ ] **Step 7: Create JPA entity and migration**

Create `ProtocolDefinitionEntity.java` (Panache entity matching the
`protocol_definition` table). Create migration
`V62__protocol_definition.sql` in
`playbook-bridge/src/main/resources/db/qhorus/migration/`.

The JPA entity follows the PushNotificationConfigEntity pattern:
`@Entity`, `@Table(name = "protocol_definition")`, named PU `qhorus`,
with `toDomain()` and `fromDomain()` mappers.

Create `JpaProtocolDefinitionStore.java` implementing
`ProtocolDefinitionStore` with `@Transactional` methods.

- [ ] **Step 8: Commit**

```bash
git add playbook-bridge/ pom.xml
git commit -m "feat: scaffold playbook-bridge module with ProtocolDefinitionStore

New optional module casehub-qhorus-playbook-bridge with:
- ProtocolDefinition record and store interface
- InMemoryProtocolDefinitionStore for CDI-free tests
- JPA entity + V62 migration
- 7 store contract tests

Refs #528

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Evaluation bridge — resolver + protocol + output validation

After this batch: Playbook-backed protocols evaluate during dispatch.
A channel with a playbook definition in the store gets its YAML compiled
and run on every message, producing DispatchAdvisory results through the
standard enforcement pipeline.

### Task 3: PlaybookProtocolOutput validator

**Files:**
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/PlaybookProtocolOutput.java`
- Create: `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/PlaybookProtocolOutputTest.java`

**Interfaces:**
- Produces: `PlaybookProtocolOutput.parse(Map<String, Object>, String protocolName)` → `List<DispatchAdvisory>`

- [ ] **Step 1: Write failing tests for output parsing**

```java
package io.casehub.qhorus.playbook.bridge;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.spi.DispatchAdvisory;
import io.casehub.qhorus.api.spi.Severity;
import io.casehub.qhorus.api.spi.SuggestedAction;

class PlaybookProtocolOutputTest {

    @Test
    void validAdvisory_mapsCorrectly() {
        Map<String, Object> output = Map.of("advisories", List.of(
                Map.of("severity", "WARNING", "message", "too many open queries",
                       "action", "LOG")));
        List<DispatchAdvisory> result =
                PlaybookProtocolOutput.parse(output, "approval-gate");
        assertThat(result).hasSize(1);
        assertThat(result.get(0).source()).isEqualTo("approval-gate");
        assertThat(result.get(0).severity()).isEqualTo(Severity.WARNING);
        assertThat(result.get(0).message()).isEqualTo("too many open queries");
        assertThat(result.get(0).suggestedAction()).isEqualTo(SuggestedAction.LOG);
    }

    @Test
    void missingAdvisoriesKey_returnsEmpty() {
        List<DispatchAdvisory> result =
                PlaybookProtocolOutput.parse(Map.of(), "gate");
        assertThat(result).isEmpty();
    }

    @Test
    void invalidSeverity_producesValidationAdvisory() {
        Map<String, Object> output = Map.of("advisories", List.of(
                Map.of("severity", "INVALID", "message", "test")));
        List<DispatchAdvisory> result =
                PlaybookProtocolOutput.parse(output, "gate");
        assertThat(result).hasSize(1);
        assertThat(result.get(0).source())
                .isEqualTo("playbook-bridge:validation");
        assertThat(result.get(0).severity()).isEqualTo(Severity.WARNING);
    }

    @Test
    void missingMessage_producesValidationAdvisory() {
        Map<String, Object> output = Map.of("advisories", List.of(
                Map.of("severity", "WARNING")));
        List<DispatchAdvisory> result =
                PlaybookProtocolOutput.parse(output, "gate");
        assertThat(result).hasSize(1);
        assertThat(result.get(0).source())
                .isEqualTo("playbook-bridge:validation");
    }

    @Test
    void defaultAction_isLOG() {
        Map<String, Object> output = Map.of("advisories", List.of(
                Map.of("severity", "ADVISORY", "message", "info")));
        List<DispatchAdvisory> result =
                PlaybookProtocolOutput.parse(output, "gate");
        assertThat(result).hasSize(1);
        assertThat(result.get(0).suggestedAction())
                .isEqualTo(SuggestedAction.LOG);
    }

    @Test
    void evidenceMap_passedThrough() {
        Map<String, Object> evidence = Map.of("count", 5);
        Map<String, Object> output = Map.of("advisories", List.of(
                Map.of("severity", "WARNING", "message", "over limit",
                       "evidence", evidence)));
        List<DispatchAdvisory> result =
                PlaybookProtocolOutput.parse(output, "gate");
        assertThat(result.get(0).evidence()).containsEntry("count", 5);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PlaybookProtocolOutputTest -pl playbook-bridge`
Expected: Compilation error.

- [ ] **Step 3: Implement PlaybookProtocolOutput**

Create `PlaybookProtocolOutput.java` — a utility class with a static
`parse(Map<String, Object>, String)` method. Reads the `advisories` key,
validates each entry, maps valid entries to `DispatchAdvisory`, produces
validation advisories for invalid entries.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PlaybookProtocolOutputTest -pl playbook-bridge`
Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add playbook-bridge/src/
git commit -m "feat: add PlaybookProtocolOutput validator

Parses playbook run output into List<DispatchAdvisory> with validation.
Invalid entries produce diagnostic advisories instead of silent failures.

Refs #528

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: PlaybookProtocolResolver and PlaybookChannelProtocol

**Files:**
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/CompilationCache.java`
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/PlaybookChannelProtocol.java`
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/PlaybookProtocolResolver.java`
- Create: `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/PlaybookProtocolResolverTest.java`

**Interfaces:**
- Consumes: `ProtocolDefinitionStore.findByChannelAndProtocol(UUID, String)` (Task 2)
- Consumes: `PlaybookProtocolOutput.parse(Map, String)` (Task 3)
- Produces: `PlaybookProtocolResolver implements ProtocolResolver` (CDI bean)
- Produces: `PlaybookChannelProtocol implements ChannelProtocol`
- Produces: `CompilationCache` — `get(UUID, String)`, `put(UUID, String, String)`, `evict(UUID, String)`

**Note:** This task depends on the prerequisite platform change
(`PlaybookRunnerCore.run(String, PlaybookRunConfig)`). For initial
development and testing, mock the `PlaybookRunnerCore` via Mockito.
The actual platform dependency is wired when the platform issue lands.

- [ ] **Step 1: Write failing tests**

Test `PlaybookProtocolResolver` with mocked store and executor.
Test `CompilationCache` eviction. Test that cached entries are reused.
Test that unknown definitions produce WARN and empty results.

- [ ] **Step 2: Run tests to verify fail**

- [ ] **Step 3: Implement CompilationCache, PlaybookChannelProtocol, PlaybookProtocolResolver**

`CompilationCache`: `ConcurrentHashMap<String, String>` keyed by
`channelId:protocolName`, stores compiled YAML content (or in future,
compiled playbook document). `evict()` removes the entry.

`PlaybookChannelProtocol`: Takes protocol name and YAML content. Maps
`ProtocolContext` to `PlaybookRunConfig.vars`, invokes executor, parses
output via `PlaybookProtocolOutput`.

`PlaybookProtocolResolver`: `@ApplicationScoped`, implements
`ProtocolResolver`. Injects `ProtocolDefinitionStore` and
`CompilationCache`. For each unresolved name, looks up definition,
checks cache, constructs `PlaybookChannelProtocol`.

- [ ] **Step 4: Run tests**

- [ ] **Step 5: Commit**

```bash
git add playbook-bridge/src/
git commit -m "feat: add PlaybookProtocolResolver with compilation cache

Implements ProtocolResolver SPI. Resolves unmatched protocol names by
looking up playbook definitions from ProtocolDefinitionStore, caching
compiled results, and delegating to PlaybookRunnerCore for evaluation.

Refs #528

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Deadline timer (#529)

After this batch: Playbook protocols with deadline metadata get
their deadlines tracked. When time elapses, a synthetic EVENT fires
and triggers protocol evaluation.

### Task 5: ProtocolDeadlineScheduler

**Files:**
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/DeadlineKey.java`
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/ProtocolDeadlineScheduler.java`
- Create: `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/ProtocolDeadlineSchedulerTest.java`

**Interfaces:**
- Consumes: `MessageDispatcher.dispatch(MessageDispatch)` (from qhorus api)
- Produces: `DeadlineKey(UUID channelId, String protocolName, String deadlineName)`
- Produces: `ProtocolDeadlineScheduler` — `register(DeadlineKey, Instant)`, `clear(UUID, String)`, `clearAll(UUID)`, `cancel(DeadlineKey)`

- [ ] **Step 1: Write failing tests**

CDI-free tests with `Clock.fixed()` and mocked `MessageDispatcher`.
Tests: register + sweep fires EVENT; register in future doesn't fire;
cancel removes; clear removes by channel+protocol; sweep removes
atomically; past-due fires immediately on next tick.

- [ ] **Step 2: Run to verify fail**

- [ ] **Step 3: Implement DeadlineKey and ProtocolDeadlineScheduler**

`DeadlineKey`: simple record.
`ProtocolDeadlineScheduler`: `ConcurrentHashMap`, `Clock` field (CDI
`@Inject` or test constructor), `@Scheduled(every = "60s")` sweep
method. Dispatches EVENT via `MessageDispatcher` with
`sender=system:protocol-timer`, telemetry JSON.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

### Task 6: PlaybookProtocolLifecycleHandler with deadline registration

**Files:**
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/PlaybookProtocolLifecycleHandler.java`
- Create: `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/PlaybookProtocolLifecycleHandlerTest.java`

**Interfaces:**
- Consumes: `ProtocolDefinitionStore.findByChannelAndProtocol(UUID, String)` (Task 2)
- Consumes: `ProtocolDefinitionStore.existsByProtocolName(String)` (Task 2)
- Consumes: `ProtocolDeadlineScheduler.register(DeadlineKey, Instant)` (Task 5)
- Consumes: `ProtocolRegistry.allNames()` — for precedence guard
- Produces: `PlaybookProtocolLifecycleHandler implements ProtocolLifecycleHandler`

- [ ] **Step 1: Write failing tests**

Tests: `supports()` returns true for known playbook names, false for
CDI-registered names (precedence), false for unknown names.
`onInit()` parses deadline metadata, registers with scheduler.
`onInit(recovered=true)` recomputes firesAt from message history.
`onDestroy()` clears scheduler registrations.

- [ ] **Step 2: Run to verify fail**

- [ ] **Step 3: Implement PlaybookProtocolLifecycleHandler**

`@ApplicationScoped`. In-memory `Set<String>` of known playbook
protocol names, populated at `@PostConstruct`. `supports()` checks
the set after the precedence guard. `onInit()` queries definition
store, parses YAML frontmatter metadata, extracts `deadlines` map,
parses durations, registers with scheduler. Recovery path uses
message history timestamps.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Add deadline action processing to PlaybookChannelProtocol**

Update `PlaybookChannelProtocol.evaluate()` to read `deadline_actions`
from playbook output after advisories. Process `reset` and `cancel`
actions. Write tests for both actions.

- [ ] **Step 6: Commit**

```bash
git add playbook-bridge/src/
git commit -m "feat: add deadline timer and lifecycle handler (#529)

ProtocolDeadlineScheduler tracks in-memory deadlines with @Scheduled
sweep. PlaybookProtocolLifecycleHandler parses deadline metadata from
playbook YAML and registers/clears deadlines on init/destroy.
Deadline actions (reset/cancel) supported via playbook output.

Refs #529

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 5: REST surface + full build verification

After this batch: Protocol definitions can be created via REST.
Full build passes including all existing modules.

### Task 7: ProtocolDefinitionResource REST endpoint

**Files:**
- Create: `playbook-bridge/src/main/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinitionResource.java`
- Create: `playbook-bridge/src/test/java/io/casehub/qhorus/playbook/bridge/ProtocolDefinitionResourceTest.java`

**Interfaces:**
- Consumes: `ProtocolDefinitionStore` (Task 2)
- Consumes: `ChannelService.setProtocols()` (from qhorus runtime)
- Consumes: `CompilationCache.evict()` (Task 4)
- Produces: REST endpoints at `/api/channels/{channelId}/protocol-definitions/*`

- [ ] **Step 1: Write failing tests**

Test PUT (store + validate), GET (retrieve), DELETE (remove + evict
cache), list (all for channel). Test validation rejects invalid YAML
(400). Test validation rejects invalid deadline durations (400).

- [ ] **Step 2: Run to verify fail**

- [ ] **Step 3: Implement ProtocolDefinitionResource**

`@Path("/api/channels/{channelId}/protocol-definitions")`,
`@ApplicationScoped`, `@Produces(APPLICATION_JSON)`. PUT validates
YAML structure and deadline durations before storing. On PUT, adds
protocol name to channel.protocols if not present. On DELETE, removes
from channel.protocols. Cache eviction on PUT and DELETE.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules including playbook-bridge.
Existing module tests pass with no regressions.

- [ ] **Step 6: Commit**

```bash
git add playbook-bridge/src/
git commit -m "feat: add REST endpoint for protocol definitions

PUT/GET/DELETE at /api/channels/{channelId}/protocol-definitions/{name}.
Validates YAML structure and deadline durations at storage time.
Manages channel.protocols list and compilation cache on mutation.

Refs #528

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-20-playbook-protocol-bridge-design.md] — design spec
- ProtocolRegistry.java:14-68 — existing registry (runtime-core)
- ProtocolRegistryTest.java:15-109 — existing tests
- RuntimeBeans.java:306-310 — ProtocolRegistry producer
- MessageService.java:364-382 — protocol evaluation dispatch site
- ProtocolLifecycleManager.java:22-164 — lifecycle orchestrator
- ProtocolLifecycleHandler.java:5-12 — lifecycle SPI
- ProtocolLifecycleContext.java:8-22 — lifecycle context record
- ChannelProtocol.java:5-10 — protocol evaluation SPI
- ProtocolContext.java:11-28 — evaluation context record
- RequestResponseProtocol.java — built-in protocol example
- PushNotificationConfigEntity.java — bridge-owned JPA pattern
- a2a-push-notification/pom.xml — bridge module pom pattern
- consumer-spi-placement protocol
- message-service-dispatch-enforcement-gate protocol
- cross-repo-optional-dep-table-registration protocol
- casehubio/qhorus#528, casehubio/qhorus#529

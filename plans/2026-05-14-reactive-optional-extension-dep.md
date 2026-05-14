# qhorus #141 — Gate quarkus-hibernate-reactive-panache via Capabilities Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `quarkus-hibernate-reactive-panache` an optional dep so consumers without a reactive datasource driver need zero workaround properties.

**Architecture:** Mark the dep `<optional>true</optional>` in `runtime/pom.xml`. Replace the user-facing `casehub.qhorus.reactive.enabled` flag with `Capabilities.isPresent(Capability.HIBERNATE_REACTIVE) && isReactiveDriverPresent()` in `QhorusProcessor`. Use `ExcludedTypeBuildItem` to gate reactive vs blocking beans at build time. Add `@Priority(1)` to reactive `@Alternative` beans so CDI auto-selects them when not excluded.

**Tech Stack:** Java 21, Quarkus 3.32.2, `io.quarkus.arc.deployment.ExcludedTypeBuildItem`, `io.quarkus.deployment.Capabilities`

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl runtime,deployment,examples/normative-layout,examples/type-system -am`

---

## File Map

| File | Change |
|---|---|
| `runtime/pom.xml` | Add `<optional>true</optional>` to reactive Panache dep |
| `deployment/src/.../QhorusBuildConfig.java` | Remove `reactive()` method and `Reactive` inner interface |
| `deployment/src/.../QhorusProcessor.java` | Replace `ReactiveEnabled`, add two `ExcludedTypeBuildItem` steps, remove `markReactiveBeans()` |
| `runtime/src/.../mcp/ReactiveQhorusMcpTools.java` | Remove `@IfBuildProperty`; add `@Priority(1)` |
| `runtime/src/.../api/ReactiveA2AResource.java` | Remove `@IfBuildProperty`; add `@Priority(1)` |
| `runtime/src/.../api/ReactiveAgentCardResource.java` | Remove `@IfBuildProperty`; add `@Priority(1)` |
| `runtime/src/.../mcp/QhorusMcpTools.java` | Remove `@UnlessBuildProperty` |
| `runtime/src/.../api/A2AResource.java` | Remove `@UnlessBuildProperty` |
| `runtime/src/.../api/AgentCardResource.java` | Remove `@UnlessBuildProperty` |
| `runtime/src/.../api/A2AChannelBackend.java` | Remove `@UnlessBuildProperty` |
| `runtime/src/.../api/A2AActorResolver.java` | Remove `@UnlessBuildProperty` |
| 20× reactive `@Alternative` service/store/repo classes | Add `@Priority(1)` |
| `examples/normative-layout/.../application.properties` | Remove reactive workaround properties |
| `examples/type-system/.../application.properties` | Remove reactive workaround properties |
| `runtime/src/test/.../ReactiveCapabilityExclusionTest.java` | New test — asserts reactive beans excluded when no reactive driver present |

---

## Task 1: TDD RED — remove workarounds from examples, write capability exclusion test

**Files:**
- Modify: `examples/normative-layout/src/test/resources/application.properties`
- Modify: `examples/type-system/src/test/resources/application.properties`
- Create: `runtime/src/test/java/io/casehub/qhorus/ReactiveCapabilityExclusionTest.java`

- [ ] **Step 1: Remove reactive workaround from normative-layout**

In `examples/normative-layout/src/test/resources/application.properties`, delete:
```
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.reactive=false
```

- [ ] **Step 2: Remove reactive workaround from type-system**

In `examples/type-system/src/test/resources/application.properties`, delete:
```
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.reactive=false
```

- [ ] **Step 3: Write ReactiveCapabilityExclusionTest**

Create `runtime/src/test/java/io/casehub/qhorus/ReactiveCapabilityExclusionTest.java`:

```java
package io.casehub.qhorus;

import io.casehub.qhorus.runtime.api.A2AActorResolver;
import io.casehub.qhorus.runtime.api.A2AChannelBackend;
import io.casehub.qhorus.runtime.api.A2AResource;
import io.casehub.qhorus.runtime.api.AgentCardResource;
import io.casehub.qhorus.runtime.api.ReactiveA2AResource;
import io.casehub.qhorus.runtime.api.ReactiveAgentCardResource;
import io.casehub.qhorus.runtime.message.ReactiveMessageService;
import io.casehub.qhorus.runtime.mcp.QhorusMcpTools;
import io.casehub.qhorus.runtime.mcp.ReactiveQhorusMcpTools;
import io.quarkus.arc.Arc;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Verifies that when HIBERNATE_REACTIVE capability is present but no reactive
 * datasource driver is available (H2 test environment), reactive beans are
 * excluded from the CDI container and blocking beans are active.
 *
 * This test runs in the runtime module itself where quarkus-hibernate-reactive-panache
 * is still on the classpath (needed for compilation). The ReactiveEnabled BooleanSupplier
 * returns false because no reactive datasource driver capability is present.
 */
@QuarkusTest
class ReactiveCapabilityExclusionTest {

    @Test
    void blockingMcpTools_isActiveWhenNoReactiveDriver() {
        var instance = Arc.container().select(QhorusMcpTools.class);
        assertFalse(instance.isUnsatisfied(),
                "QhorusMcpTools must be active when no reactive datasource driver is present");
    }

    @Test
    void reactiveMcpTools_isExcludedWhenNoReactiveDriver() {
        var instance = Arc.container().select(ReactiveQhorusMcpTools.class);
        assertTrue(instance.isUnsatisfied(),
                "ReactiveQhorusMcpTools must not be in CDI when no reactive datasource driver is present");
    }

    @Test
    void blockingA2AResource_isActive() {
        assertFalse(Arc.container().select(A2AResource.class).isUnsatisfied());
    }

    @Test
    void reactiveA2AResource_isExcluded() {
        assertTrue(Arc.container().select(ReactiveA2AResource.class).isUnsatisfied());
    }

    @Test
    void blockingAgentCardResource_isActive() {
        assertFalse(Arc.container().select(AgentCardResource.class).isUnsatisfied());
    }

    @Test
    void reactiveAgentCardResource_isExcluded() {
        assertTrue(Arc.container().select(ReactiveAgentCardResource.class).isUnsatisfied());
    }

    @Test
    void a2aChannelBackend_isActive() {
        assertFalse(Arc.container().select(A2AChannelBackend.class).isUnsatisfied());
    }

    @Test
    void a2aActorResolver_isActive() {
        assertFalse(Arc.container().select(A2AActorResolver.class).isUnsatisfied());
    }

    @Test
    void reactiveMessageService_isExcluded() {
        assertTrue(Arc.container().select(ReactiveMessageService.class).isUnsatisfied());
    }
}
```

- [ ] **Step 4: Run normative-layout tests — confirm startup failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep -E "BUILD|ERROR|reactive|Vert"
```

Expected: `BUILD FAILURE` — Quarkus fails to start due to reactive datasource not configured. This is the RED state confirming the problem exists.

- [ ] **Step 5: Run ReactiveCapabilityExclusionTest — confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ReactiveCapabilityExclusionTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | tail -20
```

Expected: `BUILD FAILURE` — tests fail because reactive beans ARE in CDI (the current broken state) or because startup fails.

---

## Task 2: Mark quarkus-hibernate-reactive-panache as optional

**Files:**
- Modify: `runtime/pom.xml`

- [ ] **Step 1: Add `<optional>true</optional>` to the reactive dep**

In `runtime/pom.xml`, change:
```xml
    <!-- Persistence: Reactive Panache — for ReactiveLedgerEntryRepository implementation -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-reactive-panache</artifactId>
    </dependency>
```
to:
```xml
    <!-- Persistence: Reactive Panache — optional; activates when consumer adds it + a reactive driver -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-reactive-panache</artifactId>
      <optional>true</optional>
    </dependency>
```

- [ ] **Step 2: Verify compilation still works**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS` — reactive classes still compile because optional doesn't affect the module's own compilation.

---

## Task 3: Remove `Reactive` from QhorusBuildConfig

**Files:**
- Modify: `deployment/src/main/java/io/casehub/qhorus/deployment/QhorusBuildConfig.java`

- [ ] **Step 1: Replace QhorusBuildConfig with empty shell**

The entire `reactive()` method and `Reactive` inner interface are removed. If `QhorusBuildConfig` has no remaining methods after removal, delete the class entirely and remove its reference from `QhorusProcessor`. If other config exists, keep only that.

Current file has only `reactive()`. Replace the entire file with a minimal shell — or delete it if it becomes empty. Since `QhorusProcessor` injects it via `QhorusBuildConfig config`, and we're replacing that injection with `Capabilities capabilities`, we remove the class entirely:

```bash
rm deployment/src/main/java/io/casehub/qhorus/deployment/QhorusBuildConfig.java
```

---

## Task 4: Rewrite QhorusProcessor

**Files:**
- Modify: `deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java`

- [ ] **Step 1: Replace the entire file**

`deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java`:

```java
package io.casehub.qhorus.deployment;

import java.util.function.BooleanSupplier;

import io.quarkus.arc.deployment.ExcludedTypeBuildItem;
import io.quarkus.deployment.Capabilities;
import io.quarkus.deployment.Capability;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

/**
 * Quarkus build-time processor for the Qhorus extension.
 *
 * Reactive stack activation is governed entirely by classpath presence:
 * - HIBERNATE_REACTIVE capability present AND a reactive datasource driver present
 *   → reactive beans active, blocking beans excluded
 * - Either absent → blocking beans active, reactive beans excluded
 *
 * No user-facing config flag is required. See PP-20260514-f41258.
 */
class QhorusProcessor {

    private static final String FEATURE = "qhorus";

    @BuildStep
    FeatureBuildItem feature() {
        return new FeatureBuildItem(FEATURE);
    }

    /** Exclude reactive beans when HIBERNATE_REACTIVE + a reactive driver are NOT both present. */
    @BuildStep(onlyIfNot = ReactiveEnabled.class)
    void excludeReactiveBeans(io.quarkus.arc.deployment.BuildProducer<ExcludedTypeBuildItem> excluded) {
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.mcp.ReactiveQhorusMcpTools"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.api.ReactiveA2AResource"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.api.ReactiveAgentCardResource"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.message.ReactiveMessageService"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.channel.ReactiveChannelService"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.instance.ReactiveInstanceService"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.watchdog.ReactiveWatchdogService"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.data.ReactiveDataService"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.ledger.ReactiveLedgerWriteService"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.ledger.ReactiveMessageLedgerEntryRepository"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.ledger.MessageReactivePanacheRepo"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.ReactiveJpaMessageStore"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.ReactiveJpaChannelStore"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.ReactiveJpaInstanceStore"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.ReactiveJpaDataStore"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.ReactiveJpaWatchdogStore"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.MessageReactivePanacheRepo"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.ChannelReactivePanacheRepo"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.InstanceReactivePanacheRepo"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.SharedDataReactivePanacheRepo"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.WatchdogReactivePanacheRepo"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.CapabilityReactivePanacheRepo"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.store.jpa.ArtefactClaimReactivePanacheRepo"));
    }

    /** Exclude blocking beans when HIBERNATE_REACTIVE + a reactive driver ARE both present. */
    @BuildStep(onlyIf = ReactiveEnabled.class)
    void excludeBlockingBeans(io.quarkus.arc.deployment.BuildProducer<ExcludedTypeBuildItem> excluded) {
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.mcp.QhorusMcpTools"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.api.A2AResource"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.api.AgentCardResource"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.api.A2AChannelBackend"));
        excluded.produce(new ExcludedTypeBuildItem("io.casehub.qhorus.runtime.api.A2AActorResolver"));
    }

    /**
     * Reactive stack is active when BOTH:
     * 1. quarkus-hibernate-reactive-panache is on the classpath (HIBERNATE_REACTIVE capability), AND
     * 2. A reactive datasource driver is present (REACTIVE_PG_CLIENT or another reactive client).
     *
     * Checking for a reactive driver prevents activation when the dep is present for
     * compilation (in the runtime module itself) but no reactive datasource is wired.
     */
    static final class ReactiveEnabled implements BooleanSupplier {

        Capabilities capabilities;

        @Override
        public boolean getAsBoolean() {
            if (!capabilities.isPresent(Capability.HIBERNATE_REACTIVE)) {
                return false;
            }
            return capabilities.isPresent(Capability.REACTIVE_PG_CLIENT)
                    || capabilities.isPresent("io.quarkus.REACTIVE_MYSQL_CLIENT")
                    || capabilities.isPresent("io.quarkus.REACTIVE_MSSQL_CLIENT")
                    || capabilities.isPresent("io.quarkus.REACTIVE_DB2_CLIENT")
                    || capabilities.isPresent("io.quarkus.REACTIVE_ORACLE_CLIENT");
        }
    }
}
```

- [ ] **Step 2: Fix the import for BuildProducer — collapse inline to top-level import**

Replace the inline `io.quarkus.arc.deployment.BuildProducer` references with a proper import block:

```java
import io.quarkus.arc.deployment.BuildProducer;
import io.quarkus.arc.deployment.ExcludedTypeBuildItem;
import io.quarkus.deployment.Capabilities;
import io.quarkus.deployment.Capability;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;
```

And update method signatures to use `BuildProducer<ExcludedTypeBuildItem>` without the package prefix.

- [ ] **Step 3: Verify deployment module compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl deployment -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep -E "BUILD|ERROR|cannot find"
```

Expected: `BUILD SUCCESS`.

---

## Task 5: Add `@Priority(1)` to all reactive `@Alternative` service/store/repo beans

These 20 classes are `@Alternative` — CDI selects them over their blocking counterparts when not excluded. `@Priority(1)` makes the selection automatic without needing `quarkus.arc.selected-alternatives`.

**Files (all in `runtime/src/main/java/io/casehub/qhorus/runtime/`):**

- [ ] **Step 1: Add to all service beans**

For each of these files, add `import jakarta.annotation.Priority;` and `@Priority(1)` immediately after `@Alternative`:

- `message/ReactiveMessageService.java`
- `channel/ReactiveChannelService.java`
- `instance/ReactiveInstanceService.java`
- `watchdog/ReactiveWatchdogService.java`
- `data/ReactiveDataService.java`
- `ledger/ReactiveLedgerWriteService.java`
- `ledger/ReactiveMessageLedgerEntryRepository.java`
- `ledger/MessageReactivePanacheRepo.java`

Pattern — before:
```java
@Alternative
@ApplicationScoped
public class ReactiveMessageService {
```
After:
```java
@Alternative
@Priority(1)
@ApplicationScoped
public class ReactiveMessageService {
```

- [ ] **Step 2: Add to all reactive JPA store beans**

Same `@Alternative` → `@Alternative @Priority(1)` pattern for:

- `store/jpa/ReactiveJpaMessageStore.java`
- `store/jpa/ReactiveJpaChannelStore.java`
- `store/jpa/ReactiveJpaInstanceStore.java`
- `store/jpa/ReactiveJpaDataStore.java`
- `store/jpa/ReactiveJpaWatchdogStore.java`

- [ ] **Step 3: Add to all reactive Panache repo beans**

- `store/jpa/MessageReactivePanacheRepo.java`
- `store/jpa/ChannelReactivePanacheRepo.java`
- `store/jpa/InstanceReactivePanacheRepo.java`
- `store/jpa/SharedDataReactivePanacheRepo.java`
- `store/jpa/WatchdogReactivePanacheRepo.java`
- `store/jpa/CapabilityReactivePanacheRepo.java`
- `store/jpa/ArtefactClaimReactivePanacheRepo.java`

- [ ] **Step 4: Verify runtime compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS`.

---

## Task 6: Remove `@IfBuildProperty` / `@UnlessBuildProperty` from all classes

These annotations are replaced by `ExcludedTypeBuildItem` in `QhorusProcessor`.

**Files:**

- [ ] **Step 1: Strip `@IfBuildProperty` from three reactive REST/MCP classes**

In `mcp/ReactiveQhorusMcpTools.java` — remove:
```java
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
```
and its import `import io.quarkus.arc.properties.IfBuildProperty;`.

Repeat for `api/ReactiveA2AResource.java` and `api/ReactiveAgentCardResource.java`.

- [ ] **Step 2: Strip `@UnlessBuildProperty` from five blocking classes**

In `mcp/QhorusMcpTools.java` — remove:
```java
@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)
```
and its import `import io.quarkus.arc.properties.UnlessBuildProperty;`.

Repeat for:
- `api/A2AResource.java`
- `api/AgentCardResource.java`
- `api/A2AChannelBackend.java`
- `api/A2AActorResolver.java`

- [ ] **Step 3: Verify full runtime compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep -E "BUILD|ERROR"
```

Expected: `BUILD SUCCESS`.

---

## Task 7: TDD GREEN — run all tests

- [ ] **Step 1: Run ReactiveCapabilityExclusionTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ReactiveCapabilityExclusionTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`, 9 tests passing. Reactive beans absent from CDI, blocking beans active.

- [ ] **Step 2: Run normative-layout tests (no workaround in application.properties)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/normative-layout -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | tail -10
```

Expected: `BUILD SUCCESS` — Quarkus starts cleanly without reactive workaround because `quarkus-hibernate-reactive-panache` is no longer transitively included.

- [ ] **Step 3: Run type-system tests (no workaround)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/type-system -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4: Run ToolOverloadDiscoverabilityTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ToolOverloadDiscoverabilityTest -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | tail -10
```

Expected: `BUILD SUCCESS` — `@Tool` name conflict guard still passes.

- [ ] **Step 5: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAILURE|ERROR" | tail -20
```

Expected: All 1035 previously passing tests + new 9 capability exclusion tests pass. 44 `@Disabled` reactive integration tests remain skipped (Docker dependency, unchanged).

---

## Task 8: Commit

- [ ] **Step 1: Stage and commit**

```bash
cd /Users/mdproctor/claude/casehub/qhorus
git add runtime/pom.xml \
  deployment/src/main/java/io/casehub/qhorus/deployment/QhorusBuildConfig.java \
  deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/api/ \
  runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/instance/ReactiveInstanceService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ReactiveWatchdogService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/data/ReactiveDataService.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ \
  runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ \
  runtime/src/test/java/io/casehub/qhorus/ReactiveCapabilityExclusionTest.java \
  examples/normative-layout/src/test/resources/application.properties \
  examples/type-system/src/test/resources/application.properties
git status
```

- [ ] **Step 2: Create the commit**

```bash
git commit -m "$(cat <<'EOF'
fix(#141): gate quarkus-hibernate-reactive-panache via Capabilities + ExcludedTypeBuildItem

Replace user-facing casehub.qhorus.reactive.enabled flag with Quarkus-native
capability detection. Reactive stack activates when HIBERNATE_REACTIVE and a
reactive datasource driver are both present; absent either → blocking stack.

- runtime/pom.xml: quarkus-hibernate-reactive-panache → <optional>true</optional>
- QhorusBuildConfig: remove Reactive inner interface (flag replaced by capability)
- QhorusProcessor: ReactiveEnabled checks Capability.HIBERNATE_REACTIVE + reactive driver;
  excludeReactiveBeans/excludeBlockingBeans via ExcludedTypeBuildItem; remove markReactiveBeans
- 20 reactive @Alternative beans: add @Priority(1) for automatic CDI selection
- 8 classes: remove @IfBuildProperty / @UnlessBuildProperty (replaced by build step)
- examples: remove quarkus.datasource.reactive=false workarounds (no longer needed)
- new: ReactiveCapabilityExclusionTest verifies reactive beans excluded in H2 environment

Consumers on JDBC-only datasources need zero workaround properties.
Refs casehubio/qhorus#141. Protocol PP-20260514-f41258.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

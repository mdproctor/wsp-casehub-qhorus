# Reactive Service Tier Gating — Implementation Plan A

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace per-bean `@IfBuildProperty`/`@UnlessBuildProperty` gating with centralised `ExcludedTypeBuildItem` in `QhorusProcessor`, driven by a `BUILD_TIME @ConfigRoot` property `casehub.qhorus.reactive.enabled`.

**Architecture:** New `QhorusBuildTimeConfig` interface in the `deployment/` module declares the build-time property. `QhorusProcessor` gains a `@BuildStep` that produces `ExcludedTypeBuildItem` for the appropriate bean set based on that property. All `@IfBuildProperty`/`@UnlessBuildProperty` annotations are removed from runtime beans. Structural `BlockingTierPurityTest` enforces the tier boundary going forward.

**Tech Stack:** Java 21, Quarkus 3.32.2, `quarkus-arc-deployment` (`ExcludedTypeBuildItem`, `BuildProducer`, `BuildStep`), SmallRye Config (`@ConfigMapping`, `@ConfigRoot`, `ConfigPhase.BUILD_TIME`), JUnit 5 (reflection-based purity test).

**Reference:** casehub-ledger — `LedgerBuildTimeConfig.java`, `LedgerProcessor.java`, `BlockingTierPurityTest.java`.

---

## File Map

| Action | File |
|--------|------|
| **Create** | `deployment/src/main/java/io/casehub/qhorus/deployment/QhorusBuildTimeConfig.java` |
| **Modify** | `deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AActorResolver.java` — remove `@UnlessBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java` — remove `@UnlessBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java` — remove `@UnlessBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java` — remove `@UnlessBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveAgentCardResource.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java` — remove `@IfBuildProperty`, `@Alternative` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/data/ReactiveDataService.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/instance/ReactiveInstanceService.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` — remove `@IfBuildProperty`, `@Alternative` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveMessageLedgerEntryRepository.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` — remove `@UnlessBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` — remove `@IfBuildProperty`, `@Alternative` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ArtefactClaimReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/CapabilityReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ChannelReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/InstanceReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaChannelStore.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaDataStore.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaInstanceStore.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaWatchdogStore.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/SharedDataReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/WatchdogReactivePanacheRepo.java` — remove `@IfBuildProperty` |
| **Modify** | `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ReactiveWatchdogService.java` — remove `@IfBuildProperty` |
| **Create** | `runtime/src/test/java/io/casehub/qhorus/service/BlockingTierPurityTest.java` |
| **Modify** | `runtime/src/test/resources/application.properties` — add `casehub.qhorus.reactive.enabled=true` |

---

## Task 1: `QhorusBuildTimeConfig` and updated `QhorusProcessor`

**Files:**
- Create: `deployment/src/main/java/io/casehub/qhorus/deployment/QhorusBuildTimeConfig.java`
- Modify: `deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java`

- [ ] **Step 1.1: Create `QhorusBuildTimeConfig`**

```java
package io.casehub.qhorus.deployment;

import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

/**
 * Build-time configuration for the Qhorus extension.
 *
 * <p>
 * Properties declared here are evaluated during Quarkus augmentation and are not
 * available at runtime. Changes require a rebuild.
 */
@ConfigMapping(prefix = "casehub.qhorus")
@ConfigRoot(phase = ConfigPhase.BUILD_TIME)
public interface QhorusBuildTimeConfig {

    /** Reactive service tier configuration. */
    ReactiveConfig reactive();

    interface ReactiveConfig {
        /**
         * Whether to activate the reactive service tier.
         *
         * <p>
         * Set to {@code true} in deployments that provide a reactive datasource
         * (e.g. Hibernate Reactive + reactive PostgreSQL client). JDBC-only consumers
         * must leave this unset — the default {@code false} excludes all reactive
         * beans from CDI augmentation, preventing unsatisfied-dependency failures.
         *
         * <p>
         * Corresponds to {@code casehub.qhorus.reactive.enabled} in
         * {@code application.properties}.
         */
        @WithDefault("false")
        boolean enabled();
    }
}
```

- [ ] **Step 1.2: Rewrite `QhorusProcessor`**

```java
package io.casehub.qhorus.deployment;

import io.quarkus.arc.deployment.ExcludedTypeBuildItem;
import io.quarkus.deployment.annotations.BuildProducer;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

/**
 * Quarkus build-time processor for the Qhorus extension.
 *
 * <p>
 * Registers the "qhorus" feature and performs build-time stack selection:
 * when {@code casehub.qhorus.reactive.enabled=false} (default), all reactive
 * beans are excluded from CDI augmentation so JDBC-only consumers build cleanly.
 * When {@code true}, conflicting blocking entry-point beans (MCP tools, REST
 * resources) are excluded instead — only one stack is active at a time.
 *
 * <p>
 * Non-conflicting blocking service beans (ChannelService, MessageService, etc.)
 * are always present regardless of the property — they carry no reactive
 * dependencies and are safe to include in both stacks.
 */
class QhorusProcessor {

    private static final String FEATURE = "qhorus";

    @BuildStep
    FeatureBuildItem feature() {
        return new FeatureBuildItem(FEATURE);
    }

    /**
     * Selects the active stack at build time.
     *
     * <p>
     * When reactive is disabled (default): exclude all reactive CDI beans.
     * When reactive is enabled: exclude blocking entry-point beans that would
     * conflict with their reactive counterparts (same {@code @Tool} names or
     * REST paths).
     */
    @BuildStep
    void selectStack(
            final QhorusBuildTimeConfig config,
            final BuildProducer<ExcludedTypeBuildItem> excluded) {

        if (!config.reactive().enabled()) {
            // Reactive stack disabled — exclude all reactive beans.
            // Blocking service beans (ChannelService etc.) remain always present.
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.mcp.ReactiveQhorusMcpTools"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.api.ReactiveA2AResource"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.api.ReactiveAgentCardResource"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.channel.ReactiveChannelService"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.data.ReactiveDataService"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.instance.ReactiveInstanceService"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.message.ReactiveMessageService"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.watchdog.ReactiveWatchdogService"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.ledger.ReactiveLedgerWriteService"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.ledger.ReactiveMessageLedgerEntryRepository"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.ledger.MessageReactivePanacheRepo"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.ReactiveJpaChannelStore"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.ReactiveJpaMessageStore"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.ReactiveJpaInstanceStore"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.ReactiveJpaDataStore"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.ReactiveJpaWatchdogStore"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.ChannelReactivePanacheRepo"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.MessageReactivePanacheRepo"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.InstanceReactivePanacheRepo"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.CapabilityReactivePanacheRepo"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.SharedDataReactivePanacheRepo"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.WatchdogReactivePanacheRepo"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.store.jpa.ArtefactClaimReactivePanacheRepo"));
        } else {
            // Reactive stack enabled — exclude blocking entry-point beans that
            // conflict with reactive counterparts (same @Tool names / REST paths).
            // Non-conflicting blocking service beans remain active in both stacks.
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.mcp.QhorusMcpTools"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.api.A2AResource"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.api.ReactiveA2AResource")); // replaced by reactive
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.api.AgentCardResource"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.api.A2AActorResolver"));
            excluded.produce(new ExcludedTypeBuildItem(
                    "io.casehub.qhorus.runtime.api.A2AChannelBackend"));
        }
    }
}
```

- [ ] **Step 1.3: Build the deployment module to verify no compile errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl deployment -q
```

Expected: `BUILD SUCCESS` with no errors.

---

## Task 2: Remove gating annotations from all runtime beans

Each bean below has one or both of these annotations to remove. The annotation import lines also go away.

**Annotations to remove:**
- `@IfBuildProperty(name = "quarkus.datasource.qhorus.reactive", stringValue = "true")` and its import
- `@UnlessBuildProperty(name = "quarkus.datasource.qhorus.reactive", stringValue = "true", enableIfMissing = true)` and its import
- `@Alternative` — only on beans that use it solely for the gating pattern (i.e., `ReactiveChannelService`, `ReactiveMessageService`, `ReactiveLedgerWriteService`). Do NOT remove `@Alternative` from beans where it's used for CDI producer selection unrelated to reactive gating.

**Files — reactive side (remove `@IfBuildProperty` + import):**

- [ ] **Step 2.1:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveA2AResource.java`
- [ ] **Step 2.2:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/ReactiveAgentCardResource.java`
- [ ] **Step 2.3:** `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelService.java` — also remove `@Alternative` and its import
- [ ] **Step 2.4:** `runtime/src/main/java/io/casehub/qhorus/runtime/data/ReactiveDataService.java`
- [ ] **Step 2.5:** `runtime/src/main/java/io/casehub/qhorus/runtime/instance/ReactiveInstanceService.java`
- [ ] **Step 2.6:** `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/MessageReactivePanacheRepo.java`
- [ ] **Step 2.7:** `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveLedgerWriteService.java` — also remove `@Alternative` and its import
- [ ] **Step 2.8:** `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/ReactiveMessageLedgerEntryRepository.java`
- [ ] **Step 2.9:** `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- [ ] **Step 2.10:** `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` — also remove `@Alternative` and its import
- [ ] **Step 2.11:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ArtefactClaimReactivePanacheRepo.java`
- [ ] **Step 2.12:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/CapabilityReactivePanacheRepo.java`
- [ ] **Step 2.13:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ChannelReactivePanacheRepo.java`
- [ ] **Step 2.14:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/InstanceReactivePanacheRepo.java`
- [ ] **Step 2.15:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/MessageReactivePanacheRepo.java`
- [ ] **Step 2.16:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaChannelStore.java`
- [ ] **Step 2.17:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaDataStore.java`
- [ ] **Step 2.18:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaInstanceStore.java`
- [ ] **Step 2.19:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaMessageStore.java`
- [ ] **Step 2.20:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaWatchdogStore.java`
- [ ] **Step 2.21:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/SharedDataReactivePanacheRepo.java`
- [ ] **Step 2.22:** `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/WatchdogReactivePanacheRepo.java`
- [ ] **Step 2.23:** `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/ReactiveWatchdogService.java`

**Files — blocking side (remove `@UnlessBuildProperty` + import):**

- [ ] **Step 2.24:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AActorResolver.java`
- [ ] **Step 2.25:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AChannelBackend.java`
- [ ] **Step 2.26:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java`
- [ ] **Step 2.27:** `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java`
- [ ] **Step 2.28:** `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

- [ ] **Step 2.29: Verify no remaining gating annotations in runtime/**

```bash
grep -rn "IfBuildProperty\|UnlessBuildProperty" \
  /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/java \
  --include="*.java"
```

Expected: no output.

- [ ] **Step 2.30: Compile runtime to catch any import errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: `BUILD SUCCESS`.

---

## Task 3: Update test `application.properties`

**Files:**
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 3.1: Add `casehub.qhorus.reactive.enabled=true`**

Add after the `# casehub-ledger` block:

```properties
# Reactive stack — enabled for tests (matches blocking behaviour via @DefaultBean shims)
casehub.qhorus.reactive.enabled=true
```

The existing `quarkus.datasource.qhorus.reactive=false` line stays — it suppresses `FastBootHibernateReactivePersistenceProvider` and is a separate Hibernate concern, not a bean-gating concern.

- [ ] **Step 3.2: Check any `@TestProfile` classes that restart the Quarkus context**

```bash
grep -rn "getConfigOverrides\|QuarkusTestProfile" \
  /Users/mdproctor/claude/casehub/qhorus/runtime/src/test/java \
  --include="*.java" -l
```

For each file found: open it and verify `getConfigOverrides()` includes `casehub.qhorus.reactive.enabled=true`. Any profile that restarts the context and omits it will fail CDI startup because the processor uses the build-time property value from augmentation.

---

## Task 4: `BlockingTierPurityTest`

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/service/BlockingTierPurityTest.java`

- [ ] **Step 4.1: Write the failing test**

```java
package io.casehub.qhorus.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.data.DataService;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.runtime.ledger.LedgerWriteService;
import io.casehub.qhorus.runtime.message.CommitmentService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.watchdog.WatchdogEvaluationService;
import io.smallrye.mutiny.Uni;

/**
 * Structural verification that blocking-tier service beans contain no
 * {@code Uni<T>}-returning methods.
 *
 * <p>
 * Enforces PP-20260519-f2e160 (reactive-blocking-tier-separation) going forward.
 * Pure reflection — no Quarkus context, no CDI. Fast and cheap to run on every build.
 */
class BlockingTierPurityTest {

    @Test
    void channelService_hasNoUniMethods() {
        assertNoUniMethods(ChannelService.class);
    }

    @Test
    void messageService_hasNoUniMethods() {
        assertNoUniMethods(MessageService.class);
    }

    @Test
    void instanceService_hasNoUniMethods() {
        assertNoUniMethods(InstanceService.class);
    }

    @Test
    void dataService_hasNoUniMethods() {
        assertNoUniMethods(DataService.class);
    }

    @Test
    void commitmentService_hasNoUniMethods() {
        assertNoUniMethods(CommitmentService.class);
    }

    @Test
    void ledgerWriteService_hasNoUniMethods() {
        assertNoUniMethods(LedgerWriteService.class);
    }

    @Test
    void watchdogEvaluationService_hasNoUniMethods() {
        assertNoUniMethods(WatchdogEvaluationService.class);
    }

    private static void assertNoUniMethods(final Class<?> cls) {
        final List<String> uniMethods = Arrays.stream(cls.getDeclaredMethods())
                .filter(m -> Uni.class.isAssignableFrom(m.getReturnType()))
                .map(Method::getName)
                .toList();
        assertThat(uniMethods)
                .as("%s must contain no Uni<T>-returning methods — reactive variants belong in Reactive%s",
                        cls.getSimpleName(), cls.getSimpleName())
                .isEmpty();
    }
}
```

- [ ] **Step 4.2: Run the test to confirm it passes (all blocking services are already pure)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -pl runtime -Dtest=BlockingTierPurityTest -q
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`. If any fail, the service has a `Uni<T>` method that must be moved to its reactive counterpart before continuing.

---

## Task 5: Full test suite — verify gating works

- [ ] **Step 5.1: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, same pass/skip counts as before (40 passing, 44 skipped).

If CDI startup fails with `UnsatisfiedResolutionException`, the likely cause is a reactive bean that was excluded by `@IfBuildProperty` before but now has an unsatisfied injection point visible at augmentation time. Check `StubReactiveLedgerEntryRepository` is still in test sources and satisfying `ReactiveLedgerEntryRepository`.

- [ ] **Step 5.2: Run the full project build including examples**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q
```

Expected: `BUILD SUCCESS`. This compiles all modules including `examples/type-system/` and `examples/normative-layout/` which depend on the runtime jar.

---

## Task 6: Commit

- [ ] **Step 6.1: Stage all changes**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  deployment/src/main/java/io/casehub/qhorus/deployment/QhorusBuildTimeConfig.java \
  deployment/src/main/java/io/casehub/qhorus/deployment/QhorusProcessor.java \
  runtime/src/main/java/io/casehub/qhorus/runtime/ \
  runtime/src/test/java/io/casehub/qhorus/service/BlockingTierPurityTest.java \
  runtime/src/test/resources/application.properties
```

- [ ] **Step 6.2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "$(cat <<'EOF'
feat(#172): ExcludedTypeBuildItem gating — QhorusBuildTimeConfig + QhorusProcessor

Replaces per-bean @IfBuildProperty/@UnlessBuildProperty with centralised
ExcludedTypeBuildItem in QhorusProcessor, driven by BUILD_TIME @ConfigRoot
property casehub.qhorus.reactive.enabled (default false). Aligns with
casehub-ledger#92 and PP-20260519-39a9a5.

Reactive beans carry no gating annotation. Blocking entry-point beans
(QhorusMcpTools, A2AResource, AgentCardResource, A2AActorResolver,
A2AChannelBackend) are excluded when reactive is enabled. Non-conflicting
blocking service beans (ChannelService, MessageService etc.) are always
present. BlockingTierPurityTest enforces the tier boundary going forward.

Refs #172

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review Checklist

- [x] **Spec Section 1 coverage:** `QhorusBuildTimeConfig` ✓, two-sided `QhorusProcessor` ✓, all 28 annotation removals ✓, test properties ✓
- [x] **Spec Section 3 (partial):** `BlockingTierPurityTest` ✓ — `ReactiveTierPurityTest` deferred to Plan B
- [x] **No placeholders:** All steps have exact commands or complete code
- [x] **Type consistency:** Class names in `QhorusProcessor` match the fully qualified names of the files in the annotation removal steps
- [x] **`@Alternative` removal:** Only `ReactiveChannelService`, `ReactiveMessageService`, `ReactiveLedgerWriteService` — these used `@Alternative` as part of the old gating pattern, not for independent CDI producer selection
- [x] **`quarkus.datasource.qhorus.reactive=false` preserved** — separate Hibernate concern, not a bean-gating signal
- [x] **`@TestProfile` caveat** — Step 3.2 explicitly checks for restarting profiles

**Note:** Plan B (Category B reactive conversion + `ReactiveTierPurityTest`) is a separate plan to be written once Plan A is merged and passing. It covers `ReactiveQhorusMcpTools` Category B tool conversion from `@Blocking @Transactional` to pure `Uni<T>`.

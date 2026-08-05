# Channel Summary & Circular Delegation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #355 — channel context summary slot with SPI hook
**Issue group:** #355, #368

**Goal:** Add per-channel maintained summaries with an SPI hook for consumer-provided summarisation, and circular delegation detection as a watchdog condition.

**Architecture:** #368 extends the existing watchdog pattern (new enum value, AlertContext record, evaluation method). #355 adds a new entity/store/service/scheduler stack following the ChannelMembership precedent, plus an SPI hook in `api/spi/` following the CommitmentAttestationPolicy pattern.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, H2 (test), Flyway

## Global Constraints

- All store interfaces follow blocking + reactive + cross-tenant triad
- InMemory stores: `@Alternative @Priority(1) @ApplicationScoped`
- JPA stores: `@ApplicationScoped`, inject `CurrentPrincipal` for tenant scoping
- Cross-tenant JPA stores: `@ApplicationScoped`, no tenant filter
- Reactive stores: `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")`
- Blocking stores: `@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true", enableIfMissing = true)`
- SPI: `@DefaultBean @ApplicationScoped` for no-op implementations
- Watchdog scheduler: `scheduled-service-cross-tenant-stores` protocol
- MCP tools: resolve channel at boundary via `resolveChannel(String)` per `mcp-tool-channel-resolution-boundary` protocol
- Contract tests: abstract base in `persistence-memory/src/test/.../contract/`, two runners (blocking + reactive)
- CDI-free unit tests: set `service.tracingConfig` to disabled-tracing implementation
- Flyway: `runtime/src/main/resources/db/qhorus/migration/V37__channel_summary.sql`

---

### Task 1: Circular Delegation — API Types + Store Methods

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/watchdog/CircularDelegationContext.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogConditionType.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/AlertContext.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CommitmentStore.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantCommitmentStore.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveCommitmentStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCommitmentStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveCommitmentStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantCommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaCommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantCommitmentStore.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/message/StubReactiveCommitmentStore.java`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/CommitmentStoreContractTest.java`

**Interfaces:**
- Produces: `WatchdogConditionType.CIRCULAR_DELEGATION`, `CircularDelegationContext`, `CommitmentStore.findAllByCorrelationId(String) → List<Commitment>`, `CrossTenantCommitmentStore.findAllByCorrelationId(String) → List<Commitment>`

- [ ] **Step 1: Write contract test for `findAllByCorrelationId`**

Add to `CommitmentStoreContractTest`:

```java
protected abstract List<Commitment> findAllByCorrelationId(String correlationId);

@Test
void findAllByCorrelationId_returnsDelegationChain_orderedChronologically() {
    UUID ch = UUID.randomUUID();
    // Parent commitment: A is obligor
    Commitment parent = save(openCommitment("corr-chain-1", "requester", "agent-a", ch));
    // Delegate to B: parent transitions to DELEGATED
    save(parent.toBuilder().state(CommitmentState.DELEGATED)
            .delegatedTo("agent-b").resolvedAt(Instant.now()).build());
    // Child commitment: B is obligor
    Commitment child = save(Commitment.builder()
            .correlationId("corr-chain-1").channelId(ch)
            .messageType(MessageType.COMMAND).requester("requester")
            .obligor("agent-b").state(CommitmentState.OPEN)
            .parentCommitmentId(parent.id()).build());

    List<Commitment> chain = findAllByCorrelationId("corr-chain-1");
    assertThat(chain).hasSize(2);
    assertThat(chain.get(0).obligor()).isEqualTo("agent-a");
    assertThat(chain.get(1).obligor()).isEqualTo("agent-b");
}

@Test
void findAllByCorrelationId_returnsEmpty_whenAbsent() {
    assertThat(findAllByCorrelationId("ghost-corr")).isEmpty();
}

@Test
void findAllByCorrelationId_includesTerminalCommitments() {
    UUID ch = UUID.randomUUID();
    Commitment c = save(openCommitment("corr-all-states", "req", "obl", ch));
    save(c.toBuilder().state(CommitmentState.FULFILLED).resolvedAt(Instant.now()).build());

    List<Commitment> result = findAllByCorrelationId("corr-all-states");
    assertThat(result).hasSize(1);
    assertThat(result.get(0).state()).isEqualTo(CommitmentState.FULFILLED);
}
```

Wire abstract method in `InMemoryCommitmentStoreTest` and `InMemoryReactiveCommitmentStoreTest`.

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest="InMemoryCommitmentStoreTest#findAllByCorrelationId*,InMemoryReactiveCommitmentStoreTest#findAllByCorrelationId*"
```

Expected: compilation failure — `findAllByCorrelationId` does not exist on the interfaces.

- [ ] **Step 3: Add `findAllByCorrelationId` to store interfaces**

`CommitmentStore.java`:
```java
/** All commitments sharing a correlationId, ordered by createdAt ASC. */
List<Commitment> findAllByCorrelationId(String correlationId);
```

`CrossTenantCommitmentStore.java`:
```java
/** All commitments sharing a correlationId (any tenancy), ordered by createdAt ASC. */
List<Commitment> findAllByCorrelationId(String correlationId);
```

`ReactiveCommitmentStore.java`:
```java
/** All commitments sharing a correlationId, ordered by createdAt ASC. */
Uni<List<Commitment>> findAllByCorrelationId(String correlationId);
```

- [ ] **Step 4: Implement in InMemory stores**

`InMemoryCommitmentStore`:
```java
@Override
public List<Commitment> findAllByCorrelationId(String correlationId) {
    return byId.values().stream()
            .filter(c -> correlationId.equals(c.correlationId()))
            .sorted(java.util.Comparator.comparing(c -> c.createdAt() != null ? c.createdAt() : Instant.MIN))
            .toList();
}
```

`InMemoryReactiveCommitmentStore`: wrap blocking store with `Uni.createFrom().item(...)`.

`InMemoryCrossTenantCommitmentStore`: delegate to `delegate.findAllByCorrelationId(correlationId)`.

- [ ] **Step 5: Implement in JPA stores**

`JpaCommitmentStore`:
```java
@Override
public List<Commitment> findAllByCorrelationId(String correlationId) {
    return repo.<CommitmentEntity>list(
            "correlationId = ?1 AND tenancyId = ?2 ORDER BY createdAt ASC",
            correlationId, currentPrincipal.tenancyId())
            .stream().map(CommitmentEntity::toDomain).toList();
}
```

`JpaCrossTenantCommitmentStore`:
```java
@Override
public List<Commitment> findAllByCorrelationId(String correlationId) {
    return repo.<CommitmentEntity>list(
            "correlationId = ?1 ORDER BY createdAt ASC", correlationId)
            .stream().map(CommitmentEntity::toDomain).toList();
}
```

`ReactiveJpaCommitmentStore`: reactive equivalent using `repo.find(...).list()` returning `Uni<List<Commitment>>`.

`StubReactiveCommitmentStore`: add `throw new UnsupportedOperationException()` stub.

- [ ] **Step 6: Create API types**

`WatchdogConditionType.java` — add `CIRCULAR_DELEGATION` after `ECHO_CHAMBER`:
```java
LOOP_DETECTED, OBLIGATION_FAN_OUT, CONVERSATION_STALL, ECHO_CHAMBER,
CIRCULAR_DELEGATION;
```

`CircularDelegationContext.java`:
```java
package io.casehub.qhorus.api.watchdog;

import java.util.List;
import java.util.UUID;

public record CircularDelegationContext(
        UUID channelId, String channelName,
        String correlationId, List<String> cycle, int chainDepth
) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.CIRCULAR_DELEGATION; }
}
```

`AlertContext.java` — add `CircularDelegationContext` to permits list.

- [ ] **Step 7: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory
```

- [ ] **Step 8: Verify with `ide_diagnostics`**

Check `api/` and `persistence-memory/` for compilation errors.

- [ ] **Step 9: Commit**

```bash
git add -A && git commit -m "feat(#368): circular delegation API types + findAllByCorrelationId store method

Adds CIRCULAR_DELEGATION to WatchdogConditionType, CircularDelegationContext
record, findAllByCorrelationId on all commitment store interfaces with
blocking, reactive, cross-tenant, InMemory, and JPA implementations.

Refs #368"
```

---

### Task 2: Circular Delegation — Evaluation Logic + Integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Modify: `connectors/src/main/java/io/casehub/qhorus/connectors/ConnectorAlertBridge.java`
- Modify: `connectors/src/test/java/io/casehub/qhorus/connectors/ConnectorAlertBridgeTest.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationServiceTest.java`

**Interfaces:**
- Consumes: `WatchdogConditionType.CIRCULAR_DELEGATION`, `CircularDelegationContext`, `CrossTenantCommitmentStore.findAllByCorrelationId(String)`
- Produces: `WatchdogEvaluationService.evaluateCircularDelegation(Watchdog, Instant) → boolean`

- [ ] **Step 1: Write unit test for cycle detection**

Add to `WatchdogEvaluationServiceTest` (CDI-free, uses InMemory stores):

```java
@Test
void circularDelegation_detectsCycle_A_to_B_to_A() {
    // Setup: channel + watchdog
    UUID chId = UUID.randomUUID();
    Channel ch = Channel.builder("cycle-ch").id(chId).tenancyId(DEFAULT_TENANT).build();
    channelStore.put(ch);
    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CIRCULAR_DELEGATION, ch.name())
            .id(UUID.randomUUID()).thresholdCount(10)
            .notificationChannel(notifChannel.name()).tenancyId(DEFAULT_TENANT)
            .createdBy("test").build());

    // Create delegation chain: A → B → A (cycle)
    String corrId = UUID.randomUUID().toString();
    Commitment parent = commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("requester").obligor("agent-a").state(CommitmentState.DELEGATED)
            .delegatedTo("agent-b").resolvedAt(Instant.now()).build());
    Commitment child = commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("requester").obligor("agent-b").state(CommitmentState.DELEGATED)
            .delegatedTo("agent-a").resolvedAt(Instant.now())
            .parentCommitmentId(parent.id()).build());
    commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("requester").obligor("agent-a").state(CommitmentState.OPEN)
            .parentCommitmentId(child.id()).build());

    service.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.builder()
            .channelId(notifChannel.id()).build());
    assertThat(alerts).isNotEmpty();
    assertThat(alerts.get(0).content()).contains("CIRCULAR_DELEGATION");
}

@Test
void circularDelegation_noCycle_linearChain() {
    UUID chId = UUID.randomUUID();
    Channel ch = Channel.builder("linear-ch").id(chId).tenancyId(DEFAULT_TENANT).build();
    channelStore.put(ch);
    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CIRCULAR_DELEGATION, ch.name())
            .id(UUID.randomUUID()).thresholdCount(10)
            .notificationChannel(notifChannel.name()).tenancyId(DEFAULT_TENANT)
            .createdBy("test").build());

    // A → B → C (no cycle)
    String corrId = UUID.randomUUID().toString();
    Commitment p = commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("req").obligor("a").state(CommitmentState.DELEGATED)
            .delegatedTo("b").resolvedAt(Instant.now()).build());
    Commitment c = commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("req").obligor("b").state(CommitmentState.DELEGATED)
            .delegatedTo("c").resolvedAt(Instant.now())
            .parentCommitmentId(p.id()).build());
    commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("req").obligor("c").state(CommitmentState.OPEN)
            .parentCommitmentId(c.id()).build());

    service.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.builder()
            .channelId(notifChannel.id()).build());
    assertThat(alerts).isEmpty();
}

@Test
void circularDelegation_skipsChains_exceedingMaxDepth() {
    // thresholdCount = 2, chain length = 3 → skipped
    UUID chId = UUID.randomUUID();
    Channel ch = Channel.builder("depth-ch").id(chId).tenancyId(DEFAULT_TENANT).build();
    channelStore.put(ch);
    watchdogStore.put(Watchdog.builder(WatchdogConditionType.CIRCULAR_DELEGATION, ch.name())
            .id(UUID.randomUUID()).thresholdCount(2)
            .notificationChannel(notifChannel.name()).tenancyId(DEFAULT_TENANT)
            .createdBy("test").build());

    String corrId = UUID.randomUUID().toString();
    Commitment p = commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("req").obligor("a").state(CommitmentState.DELEGATED)
            .delegatedTo("b").resolvedAt(Instant.now()).build());
    Commitment c = commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("req").obligor("b").state(CommitmentState.DELEGATED)
            .delegatedTo("a").resolvedAt(Instant.now())
            .parentCommitmentId(p.id()).build());
    commitmentStore.save(Commitment.builder()
            .correlationId(corrId).channelId(chId).messageType(MessageType.COMMAND)
            .requester("req").obligor("a").state(CommitmentState.OPEN)
            .parentCommitmentId(c.id()).build());

    service.evaluateAll();

    List<Message> alerts = messageStore.scan(MessageQuery.builder()
            .channelId(notifChannel.id()).build());
    assertThat(alerts).isEmpty();
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WatchdogEvaluationServiceTest#circularDelegation*"
```

Expected: compilation failure — `evaluateCircularDelegation` not in the switch.

- [ ] **Step 3: Implement `evaluateCircularDelegation` in `WatchdogEvaluationService`**

```java
private boolean evaluateCircularDelegation(Watchdog w, Instant now) {
    int maxDepth = w.thresholdCount() != null ? w.thresholdCount() : 10;

    List<Channel> channels = crossTenantChannelStore.listAll().stream()
            .filter(ch -> "*".equals(w.targetName()) || ch.name().equals(w.targetName()))
            .toList();

    boolean fired = false;
    for (Channel ch : channels) {
        List<Commitment> open = crossTenantCommitmentStore.findOpenByChannel(ch.id()).stream()
                .filter(c -> c.parentCommitmentId() != null)
                .toList();

        Set<String> checked = new HashSet<>();
        for (Commitment c : open) {
            if (!checked.add(c.correlationId())) continue;

            List<Commitment> chain = crossTenantCommitmentStore.findAllByCorrelationId(c.correlationId());
            if (chain.size() > maxDepth) continue;

            Set<String> seen = new LinkedHashSet<>();
            List<String> cycle = null;
            for (Commitment link : chain) {
                if (link.obligor() == null) continue;
                if (!seen.add(link.obligor())) {
                    List<String> ordered = new ArrayList<>(seen);
                    int start = ordered.indexOf(link.obligor());
                    cycle = ordered.subList(start, ordered.size());
                    cycle.add(link.obligor());
                    break;
                }
            }

            if (cycle != null) {
                String summary = "CIRCULAR_DELEGATION: cycle detected on '"
                        + ch.name() + "' — " + String.join(" → ", cycle);
                fireAlert(w, summary,
                        new CircularDelegationContext(ch.id(), ch.name(),
                                c.correlationId(), List.copyOf(cycle), chain.size()), now);
                fired = true;
            }
        }
    }
    return fired;
}
```

Add to the `evaluateAll()` switch:
```java
case CIRCULAR_DELEGATION -> evaluateCircularDelegation(w, now);
```

Add imports: `CircularDelegationContext`, `LinkedHashSet`, `ArrayList`.

- [ ] **Step 4: Update `ConnectorAlertBridge`**

Add new case to `buildBody()` switch:
```java
case CircularDelegationContext c -> event.summary()
        + "\nChannel: " + c.channelName()
        + "\nCorrelation ID: " + c.correlationId()
        + "\nCycle: " + String.join(" → ", c.cycle())
        + "\nChain depth: " + c.chainDepth();
```

Add import for `CircularDelegationContext`. Update `ConnectorAlertBridgeTest` if it has an exhaustive enum test.

- [ ] **Step 5: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WatchdogEvaluationServiceTest"
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl connectors
```

- [ ] **Step 6: Verify with `ide_diagnostics`**

Check `runtime/` and `connectors/` modules.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat(#368): circular delegation watchdog evaluation + ConnectorAlertBridge

Adds evaluateCircularDelegation() to WatchdogEvaluationService. Detects
cycles by finding repeated obligors across commitments sharing a
correlationId. Updates ConnectorAlertBridge for exhaustive switch.

Refs #368"
```

---

### Task 3: ChannelInfo Rename

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` (rename record)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` (update usages)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java` (update usages)

**Interfaces:**
- Produces: `QhorusMcpToolsBase.ChannelInfo` (replaces `ChannelSummary`), `RegisterResponse` with `List<ChannelInfo>`

- [ ] **Step 1: Rename via IntelliJ**

Use `ide_refactor_rename` on `QhorusMcpToolsBase.ChannelSummary` → `ChannelInfo`. This updates all references across `QhorusMcpTools.java` and `ReactiveQhorusMcpTools.java` automatically.

- [ ] **Step 2: Verify with `ide_diagnostics`**

Check `runtime/` for compilation errors.

- [ ] **Step 3: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(#355): rename ChannelSummary DTO to ChannelInfo

Resolves naming clash with the new ChannelSummary domain type.
The existing DTO in QhorusMcpToolsBase (name, description, semantic)
is a channel info record, not a summary.

Refs #355"
```

---

### Task 4: Channel Summary — Domain Type + Stores + Flyway

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelSummary.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelSummaryUpdatedEvent.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/ChannelSummaryStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/ReactiveChannelSummaryStore.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantChannelSummaryStore.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelSummaryEntity.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ChannelSummaryPanacheRepo.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelSummaryStore.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantChannelSummaryStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelSummaryStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryReactiveChannelSummaryStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantChannelSummaryStore.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V37__channel_summary.sql`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/ChannelSummaryStoreContractTest.java`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryChannelSummaryStoreTest.java`
- Test: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/InMemoryReactiveChannelSummaryStoreTest.java`

**Interfaces:**
- Produces: `ChannelSummary` record, `ChannelSummaryStore`, `ReactiveChannelSummaryStore`, `CrossTenantChannelSummaryStore`, `ChannelSummaryUpdatedEvent`

- [ ] **Step 1: Write contract tests**

`ChannelSummaryStoreContractTest.java`:
```java
package io.casehub.qhorus.persistence.memory.contract;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.*;
import org.junit.jupiter.api.*;
import io.casehub.qhorus.api.channel.ChannelSummary;

public abstract class ChannelSummaryStoreContractTest {

    protected abstract ChannelSummary save(ChannelSummary s);
    protected abstract Optional<ChannelSummary> findByChannelId(UUID channelId);
    protected abstract void deleteByChannelId(UUID channelId);
    protected abstract void reset();

    @BeforeEach void beforeEach() { reset(); }

    @Test void save_assignsId_whenNull() {
        ChannelSummary s = summary(UUID.randomUUID());
        assertThat(save(s).id()).isNotNull();
    }

    @Test void findByChannelId_afterSave() {
        UUID chId = UUID.randomUUID();
        save(summary(chId));
        assertThat(findByChannelId(chId)).isPresent();
    }

    @Test void findByChannelId_returnsEmpty_whenAbsent() {
        assertThat(findByChannelId(UUID.randomUUID())).isEmpty();
    }

    @Test void save_updatesExisting() {
        UUID chId = UUID.randomUUID();
        ChannelSummary s = save(summary(chId));
        save(s.toBuilder().content("updated").build());
        assertThat(findByChannelId(chId).get().content()).isEqualTo("updated");
    }

    @Test void deleteByChannelId_removes() {
        UUID chId = UUID.randomUUID();
        save(summary(chId));
        deleteByChannelId(chId);
        assertThat(findByChannelId(chId)).isEmpty();
    }

    protected ChannelSummary summary(UUID channelId) {
        return ChannelSummary.builder(channelId)
                .content("test summary").updatedBy("test")
                .tenancyId("278776f9-e1b0-46fb-9032-8bddebdcf9ce")
                .build();
    }
}
```

`InMemoryChannelSummaryStoreTest`:
```java
package io.casehub.qhorus.persistence.memory.contract;

import io.casehub.qhorus.api.channel.ChannelSummary;
import io.casehub.qhorus.persistence.memory.InMemoryChannelSummaryStore;
import java.util.*;

public class InMemoryChannelSummaryStoreTest extends ChannelSummaryStoreContractTest {
    private final InMemoryChannelSummaryStore store = new InMemoryChannelSummaryStore();
    @Override protected ChannelSummary save(ChannelSummary s) { return store.save(s); }
    @Override protected Optional<ChannelSummary> findByChannelId(UUID channelId) { return store.findByChannelId(channelId); }
    @Override protected void deleteByChannelId(UUID channelId) { store.deleteByChannelId(channelId); }
    @Override protected void reset() { store.clear(); }
}
```

`InMemoryReactiveChannelSummaryStoreTest`: same pattern, wrapping each call with `.await().indefinitely()`.

- [ ] **Step 2: Create domain type `ChannelSummary`**

```java
package io.casehub.qhorus.api.channel;

import java.time.Instant;
import java.util.UUID;

public record ChannelSummary(
        UUID id, UUID channelId, String content,
        Instant updatedAt, String updatedBy,
        Long lastUpdatedMessageId,
        Integer updateAfterMessages, Integer updateAfterSeconds,
        String tenancyId) {

    public Builder toBuilder() { /* copy all fields */ }
    public static Builder builder(UUID channelId) { return new Builder(channelId); }

    public static final class Builder {
        private final UUID channelId;
        private UUID id;
        private String content;
        private Instant updatedAt;
        private String updatedBy;
        private Long lastUpdatedMessageId;
        private Integer updateAfterMessages;
        private Integer updateAfterSeconds;
        private String tenancyId;
        private Builder(UUID channelId) { this.channelId = channelId; }
        // standard builder methods...
        public ChannelSummary build() { return new ChannelSummary(id, channelId, content, updatedAt, updatedBy, lastUpdatedMessageId, updateAfterMessages, updateAfterSeconds, tenancyId); }
    }
}
```

`ChannelSummaryUpdatedEvent`:
```java
package io.casehub.qhorus.api.channel;
import java.util.UUID;
public record ChannelSummaryUpdatedEvent(UUID channelId, String channelName, String updatedBy) {}
```

- [ ] **Step 3: Create store interfaces**

`ChannelSummaryStore`, `ReactiveChannelSummaryStore`, `CrossTenantChannelSummaryStore` — as specified in the design spec. `CrossTenantChannelSummaryStore` includes both `findAll()` and `findWithAutoUpdateConfigured()`.

- [ ] **Step 4: Create `ChannelSummaryEntity`**

Follow `ChannelMembershipEntity` pattern. `@Entity(name = "ChannelSummary")`, `@Table(name = "channel_summary")`, FK to `channel(id)` via `@JoinColumn` with ON DELETE CASCADE. `@PrePersist` for id generation and tenancyId default.

- [ ] **Step 5: Create `ChannelSummaryPanacheRepo`**

```java
@ApplicationScoped
@PersistenceUnit("qhorus")
public class ChannelSummaryPanacheRepo extends PanacheRepositoryBase<ChannelSummaryEntity, UUID> {}
```

- [ ] **Step 6: Create JPA store implementations**

`JpaChannelSummaryStore`: `@ApplicationScoped`, injects `ChannelSummaryPanacheRepo` and `CurrentPrincipal`. Standard tenant-scoped queries.

`JpaCrossTenantChannelSummaryStore`: `@ApplicationScoped`, no tenant filter. `findWithAutoUpdateConfigured()` uses `"updateAfterMessages IS NOT NULL OR updateAfterSeconds IS NOT NULL"` JPQL.

- [ ] **Step 7: Create InMemory store implementations**

`InMemoryChannelSummaryStore`: `@Alternative @Priority(1) @ApplicationScoped`, `ConcurrentHashMap<UUID, ChannelSummary>` keyed by `channelId`.

`InMemoryReactiveChannelSummaryStore`: wraps blocking store.

`InMemoryCrossTenantChannelSummaryStore`: delegates to `InMemoryChannelSummaryStore`.

- [ ] **Step 8: Create Flyway migration**

`V37__channel_summary.sql`:
```sql
CREATE TABLE channel_summary (
    id UUID PRIMARY KEY,
    channel_id UUID NOT NULL UNIQUE REFERENCES channel(id) ON DELETE CASCADE,
    content TEXT,
    updated_at TIMESTAMP,
    updated_by VARCHAR(255),
    last_updated_message_id BIGINT,
    update_after_messages INTEGER,
    update_after_seconds INTEGER,
    tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'
);
```

- [ ] **Step 9: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest="InMemoryChannelSummaryStoreTest,InMemoryReactiveChannelSummaryStoreTest"
```

- [ ] **Step 10: Verify and commit**

```bash
git add -A && git commit -m "feat(#355): ChannelSummary domain type, stores, and V37 migration

Adds ChannelSummary record in api/channel/, store interfaces (blocking,
reactive, cross-tenant), JPA and InMemory implementations, contract
tests, and V37 Flyway migration for the channel_summary table.

Refs #355"
```

---

### Task 5: Channel Summary — SPI Hook + Service + Config

**Files:**
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/SummaryUpdateHook.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/ReactiveSummaryUpdateHook.java`
- Create: `api/src/main/java/io/casehub/qhorus/api/spi/SummaryUpdateContext.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/NoOpSummaryUpdateHook.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/DefaultReactiveSummaryUpdateHook.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelSummaryService.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ReactiveChannelSummaryService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelSummaryServiceTest.java`

**Interfaces:**
- Consumes: `ChannelSummary`, `ChannelSummaryStore`, `CrossTenantChannelSummaryStore`, `ChannelSummaryUpdatedEvent`
- Produces: `SummaryUpdateHook`, `SummaryUpdateContext`, `ChannelSummaryService` (getSummary, setSummary, configureSummary, triggerUpdate, deleteSummary)

- [ ] **Step 1: Write service unit tests** (CDI-free)

```java
package io.casehub.qhorus.runtime.channel;

import static org.assertj.core.api.Assertions.*;
import io.casehub.qhorus.api.channel.*;
import io.casehub.qhorus.api.spi.*;
import io.casehub.qhorus.persistence.memory.*;
import java.util.*;
import org.junit.jupiter.api.*;

class ChannelSummaryServiceTest {
    InMemoryChannelSummaryStore summaryStore = new InMemoryChannelSummaryStore();
    InMemoryChannelStore channelStore = new InMemoryChannelStore();
    InMemoryMessageStore messageStore = new InMemoryMessageStore();
    SummaryUpdateHook hook = ctx -> "generated summary for " + ctx.channelName();
    ChannelSummaryService service; // constructed with stores + hook in @BeforeEach

    @BeforeEach void setUp() {
        summaryStore.clear(); channelStore.clear(); messageStore.clear();
        service = new ChannelSummaryService();
        service.summaryStore = summaryStore;
        service.channelStore = channelStore;
        service.messageStore = messageStore;
        service.hook = hook;
        service.summaryEvents = /* no-op Event stub */;
    }

    @Test void getSummary_returnsEmpty_whenNone() {
        assertThat(service.getSummary(UUID.randomUUID())).isEmpty();
    }

    @Test void setSummary_createsNewSummary() {
        UUID chId = createTestChannel("test-ch");
        ChannelSummary s = service.setSummary(chId, "manual summary", "operator");
        assertThat(s.content()).isEqualTo("manual summary");
        assertThat(s.updatedBy()).isEqualTo("operator");
    }

    @Test void setSummary_advancesCursor() {
        UUID chId = createTestChannel("cursor-ch");
        // Send a message to the channel so there's a max message ID
        dispatchMessage(chId, "test message");
        ChannelSummary s = service.setSummary(chId, "manual", "op");
        assertThat(s.lastUpdatedMessageId()).isNotNull();
    }

    @Test void configureSummary_setsThresholds() {
        UUID chId = createTestChannel("config-ch");
        ChannelSummary s = service.configureSummary(chId, 10, 300);
        assertThat(s.updateAfterMessages()).isEqualTo(10);
        assertThat(s.updateAfterSeconds()).isEqualTo(300);
    }

    @Test void configureSummary_rejectsZero() {
        UUID chId = createTestChannel("reject-ch");
        assertThatThrownBy(() -> service.configureSummary(chId, 0, null))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void configureSummary_rejectsNegative() {
        UUID chId = createTestChannel("neg-ch");
        assertThatThrownBy(() -> service.configureSummary(chId, -1, null))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void triggerUpdate_invokesHookAndStores() {
        UUID chId = createTestChannel("trigger-ch");
        service.configureSummary(chId, null, null);
        Optional<ChannelSummary> result = service.triggerUpdate(chId);
        assertThat(result).isPresent();
        assertThat(result.get().content()).contains("generated summary");
    }

    @Test void triggerUpdate_returnsEmpty_whenNoSummaryConfigured() {
        assertThat(service.triggerUpdate(UUID.randomUUID())).isEmpty();
    }

    @Test void deleteSummary_removes() {
        UUID chId = createTestChannel("del-ch");
        service.setSummary(chId, "to delete", "op");
        service.deleteSummary(chId);
        assertThat(service.getSummary(chId)).isEmpty();
    }

    // helper methods: createTestChannel(), dispatchMessage()
}
```

- [ ] **Step 2: Create SPI types**

`SummaryUpdateHook.java`, `ReactiveSummaryUpdateHook.java`, `SummaryUpdateContext.java` — as specified in design spec.

- [ ] **Step 3: Create `NoOpSummaryUpdateHook` and `DefaultReactiveSummaryUpdateHook`**

`NoOpSummaryUpdateHook`: `@DefaultBean @ApplicationScoped`, returns `context.currentSummary()`.

`DefaultReactiveSummaryUpdateHook`: `@DefaultBean @ApplicationScoped`, wraps blocking hook on `Infrastructure.getDefaultWorkerPool()`.

- [ ] **Step 4: Add `Summary` config interface to `QhorusConfig`**

```java
Summary summary();

interface Summary {
    @WithDefault("true")
    boolean enabled();

    @WithDefault("60")
    int checkIntervalSeconds();
}
```

- [ ] **Step 5: Implement `ChannelSummaryService`**

Key methods:
- `getSummary` — delegates to `summaryStore.findByChannelId()`
- `setSummary` — creates or updates, advances `lastUpdatedMessageId` to channel's max message ID
- `configureSummary` — validates thresholds (>= 1 or null), creates/updates entity
- `triggerUpdate` — calls `hook.update(context)`, stores result, fires `ChannelSummaryUpdatedEvent`
- `deleteSummary` — delegates to store

Inject: `ChannelSummaryStore`, `ChannelStore`, `MessageStore`, `SummaryUpdateHook`, `Event<ChannelSummaryUpdatedEvent>`.

- [ ] **Step 6: Implement `ReactiveChannelSummaryService`**

Reactive mirror with `@IfBuildProperty`. Uses `ReactiveSummaryUpdateHook`.

- [ ] **Step 7: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ChannelSummaryServiceTest"
```

- [ ] **Step 8: Commit**

```bash
git add -A && git commit -m "feat(#355): SummaryUpdateHook SPI + ChannelSummaryService

Adds SummaryUpdateHook and ReactiveSummaryUpdateHook SPI interfaces
with @DefaultBean no-op implementations. ChannelSummaryService provides
get/set/configure/trigger/delete operations. setSummary advances the
cursor to prevent auto-update overwrite.

Refs #355"
```

---

### Task 6: Channel Summary — Scheduler + MCP Tools + Integration

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelSummaryScheduler.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/ReactiveQhorusMcpTools.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/ChannelSummarySchedulerTest.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/mcp/ChannelSummaryToolTest.java`

**Interfaces:**
- Consumes: `ChannelSummaryService`, `CrossTenantChannelSummaryStore`, `CrossTenantChannelStore`, `CrossTenantMessageStore`, `SummaryUpdateHook`, `QhorusConfig.Summary`
- Produces: MCP tools `get_channel_summary`, `update_channel_summary`, `configure_channel_summary`, `trigger_channel_summary_update`

- [ ] **Step 1: Write scheduler unit test** (CDI-free)

```java
class ChannelSummarySchedulerTest {
    @Test void sweep_triggersUpdate_whenMessageThresholdExceeded() {
        // Configure summary with updateAfterMessages=5, lastUpdatedMessageId=10
        // Add 6 messages after ID 10
        // Run sweep → hook is invoked, summary updated
    }

    @Test void sweep_skipsUpdate_whenBelowThreshold() {
        // Configure with updateAfterMessages=10, only 3 new messages
        // Run sweep → no update
    }

    @Test void sweep_triggersUpdate_whenTimeThresholdExceeded() {
        // Configure with updateAfterSeconds=60, updatedAt=2 minutes ago
        // Run sweep → hook invoked
    }

    @Test void sweep_skipsChannels_withoutAutoUpdateConfig() {
        // Summary with null thresholds → not in findWithAutoUpdateConfigured result
    }

    @Test void sweep_continuesOnHookFailure() {
        // Hook throws exception on channel-1 → channel-2 still processed
    }
}
```

- [ ] **Step 2: Implement `ChannelSummaryScheduler`**

```java
@ApplicationScoped
public class ChannelSummaryScheduler {
    @Inject QhorusConfig config;
    @Inject CrossTenantChannelSummaryStore summaryStore;
    @Inject CrossTenantChannelStore channelStore;
    @Inject CrossTenantMessageStore messageStore;
    @Inject SummaryUpdateHook hook;
    @Inject ChannelSummaryStore tenantSummaryStore;

    @Scheduled(every = "${casehub.qhorus.summary.check-interval-seconds:60}s",
               identity = "summary-update-check")
    public void sweep() {
        if (!config.summary().enabled()) return;
        List<ChannelSummary> candidates = summaryStore.findWithAutoUpdateConfigured();
        Instant now = Instant.now();
        for (ChannelSummary s : candidates) {
            try {
                if (shouldUpdate(s, now)) {
                    updateSummary(s);
                }
            } catch (Exception e) {
                LOG.warnf("Summary update failed for channel %s: %s", s.channelId(), e.getMessage());
            }
        }
    }
    // shouldUpdate() checks message count and time thresholds
    // updateSummary() calls hook, stores result, updates cursor
}
```

- [ ] **Step 3: Add MCP tools to `QhorusMcpToolsBase`**

Add response record:
```java
record ChannelSummaryResult(String channelName, String content, String updatedAt,
                            String updatedBy, Integer updateAfterMessages, Integer updateAfterSeconds) {}
```

- [ ] **Step 4: Add blocking MCP tools to `QhorusMcpTools`**

```java
@Tool(description = "Get the maintained summary for a channel")
public ChannelSummaryResult get_channel_summary(String channel) { ... }

@Tool(description = "Set or update a channel's summary text (manual override)")
public ChannelSummaryResult update_channel_summary(String channel, String summary) { ... }

@Tool(description = "Configure auto-update thresholds for a channel's summary")
public ChannelSummaryResult configure_channel_summary(String channel,
        Integer update_after_messages, Integer update_after_seconds) { ... }

@Tool(description = "Trigger an immediate summary update via the configured hook")
public ChannelSummaryResult trigger_channel_summary_update(String channel) { ... }
```

- [ ] **Step 5: Add reactive MCP tools to `ReactiveQhorusMcpTools`**

Reactive mirrors returning `Uni<ChannelSummaryResult>`.

- [ ] **Step 6: Write integration test for MCP tools**

`ChannelSummaryToolTest.java` — `@QuarkusTest` with standard profile:

```java
@QuarkusTest
class ChannelSummaryToolTest {
    @Inject QhorusMcpTools tools;

    @Test void getChannelSummary_returnsNotConfigured_whenNoSummary() { ... }

    @Test void updateChannelSummary_createsSummary() {
        tools.create_channel("summary-test", ...);
        var result = tools.update_channel_summary("summary-test", "Test summary");
        assertThat(result.content()).isEqualTo("Test summary");
    }

    @Test void configureChannelSummary_setsThresholds() {
        tools.create_channel("config-test", ...);
        var result = tools.configure_channel_summary("config-test", 10, 300);
        assertThat(result.updateAfterMessages()).isEqualTo(10);
    }

    @Test void triggerChannelSummaryUpdate_invokesHook() {
        tools.create_channel("trigger-test", ...);
        tools.configure_channel_summary("trigger-test", null, null);
        var result = tools.trigger_channel_summary_update("trigger-test");
        assertThat(result.content()).isNotNull(); // NoOp hook returns null (no prior summary)
    }
}
```

- [ ] **Step 7: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

- [ ] **Step 8: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

This catches cross-module compilation issues (e.g., examples module referencing old `ChannelSummary`).

- [ ] **Step 9: Commit**

```bash
git add -A && git commit -m "feat(#355): channel summary scheduler + MCP tools

Adds ChannelSummaryScheduler with @Scheduled sweep for auto-update.
MCP tools: get_channel_summary, update_channel_summary,
configure_channel_summary, trigger_channel_summary_update.
Blocking and reactive implementations.

Refs #355"
```

---

## Post-Implementation

After all tasks complete:
- Update `CLAUDE.md` with new store interfaces, SPI, MCP tools, Flyway version, and watchdog condition
- Verify `ide_diagnostics` clean on all modules
- Run `mvn clean install` from project root

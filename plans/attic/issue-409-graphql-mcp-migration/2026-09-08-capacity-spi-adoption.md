# Capacity Redistribution SPI Adoption — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #434 — feat: adopt capacity redistribution SPI from platform
**Issue group:** #434

**Goal:** Adopt platform capacity redistribution SPIs — enrich the platform's DefaultRedistributionPolicy, add a commitment-count signal source, per-channel redistribution thresholds, and capacity MCP tools.

**Architecture:** Two-repo change. Platform PR enriches DefaultRedistributionPolicy with obligation/inactivity awareness and adds @DefaultBean. Qhorus adds CommitmentCountCapacitySource (second CapacitySignalSource), Channel.redistributionCapacityThreshold with executor filtering, and capacity MCP tools. Existing ContextPressureCapacitySource and QhorusRedistributionExecutor are extended, not replaced.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api capacity SPI, JPA/Hibernate (qhorus named PU), H2 for tests

## Global Constraints

- Java 21 source (Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` (not `./mvnw`)
- Test port: `quarkus.http.test-port=0`
- Named datasource: `quarkus.datasource.qhorus.*`
- Flyway: `db/qhorus/migration/` namespace, next domain V = 52
- Ledger join table: V2000+ range
- Commits reference issue: `Refs #434`
- `@Tool` overload rule: never add public non-@Tool methods sharing a name with a @Tool method
- IntelliJ MCP: use `mcp__intellij-index__*` for all code navigation and structural editing

---

## Batch 1: Platform enrichment (cross-repo PR to casehub-platform)

### Task 1: Enrich DefaultRedistributionPolicy + @DefaultBean + config consolidation

**Repo:** `/Users/mdproctor/claude/casehub/platform`

**Files:**
- Modify: `platform/src/main/java/io/casehub/platform/capacity/DefaultRedistributionPolicy.java`
- Modify: `platform/src/test/java/io/casehub/platform/capacity/DefaultRedistributionPolicyTest.java`

**Interfaces:**
- Consumes: `RedistributionContext(actorId, capacity, triggerSignalType, openObligationCount, timeSinceLastActivity)` from platform-api (unchanged)
- Produces: Enriched `evaluate()` — obligation-aware branching, inactivity escalation, grace-period Redistribute decisions

- [ ] **Step 1: Write failing tests for enriched policy behavior**

Add tests to `DefaultRedistributionPolicyTest.java`:

```java
@Test
void immediateRedistributeAboveEscalateThresholdWithObligations() {
    var policy = policy(0.7, 0.85, 0.95);
    var ctx = new RedistributionContext("agent-1",
            capacity(0.96), "context_pressure", 3, Duration.ofSeconds(30));
    var decision = policy.evaluate(ctx);
    assertThat(decision).isInstanceOf(RedistributionDecision.Redistribute.class);
    var redistribute = (RedistributionDecision.Redistribute) decision;
    assertThat(redistribute.gracePeriod()).isEqualTo(Duration.ZERO);
    assertThat(redistribute.excludeActors()).contains("agent-1");
}

@Test
void holdAboveRedistributeThresholdWithZeroObligations() {
    var policy = policy(0.7, 0.85, 0.95);
    var ctx = new RedistributionContext("agent-1",
            capacity(0.90), "context_pressure", 0, Duration.ofSeconds(30));
    var decision = policy.evaluate(ctx);
    assertThat(decision).isInstanceOf(RedistributionDecision.Hold.class);
}

@Test
void escalateOnInactivity() {
    var policy = policy(0.7, 0.85, 0.95);
    var ctx = new RedistributionContext("agent-1",
            capacity(0.5), "context_pressure", 2, Duration.ofMinutes(6));
    var decision = policy.evaluate(ctx);
    assertThat(decision).isInstanceOf(RedistributionDecision.Escalate.class);
}

@Test
void redistributeWithGracePeriodBetweenThresholds() {
    var policy = policy(0.7, 0.85, 0.95);
    var ctx = new RedistributionContext("agent-1",
            capacity(0.88), "context_pressure", 2, Duration.ofSeconds(30));
    var decision = policy.evaluate(ctx);
    assertThat(decision).isInstanceOf(RedistributionDecision.Redistribute.class);
    var redistribute = (RedistributionDecision.Redistribute) decision;
    assertThat(redistribute.gracePeriod()).isEqualTo(Duration.ofSeconds(30));
    assertThat(redistribute.excludeActors()).contains("agent-1");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DefaultRedistributionPolicyTest -pl platform -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: 4 failures — current policy returns Escalate at 0.96, Redistribute at 0.90 with 0 obligations, Hold at low pressure with inactivity, and Redistribute without grace period

- [ ] **Step 3: Update DefaultRedistributionPolicy with enriched evaluate()**

Add `@DefaultBean` annotation. Add config for grace period and inactivity escalation. Update `evaluate()`:

```java
@DefaultBean
@ApplicationScoped
public class DefaultRedistributionPolicy implements RedistributionPolicy {

    private final double compressThreshold;
    private final double redistributeThreshold;
    private final double immediateThreshold;
    private final Duration gracePeriod;
    private final Duration inactivityEscalation;

    public DefaultRedistributionPolicy(
            @ConfigProperty(name = "casehub.capacity.redistribution.compress-threshold",
                            defaultValue = "0.7") double compressThreshold,
            @ConfigProperty(name = "casehub.capacity.redistribution.redistribute-threshold",
                            defaultValue = "0.85") double redistributeThreshold,
            @ConfigProperty(name = "casehub.capacity.redistribution.immediate-threshold",
                            defaultValue = "0.95") double immediateThreshold,
            @ConfigProperty(name = "casehub.capacity.redistribution.grace-period",
                            defaultValue = "30s") Duration gracePeriod,
            @ConfigProperty(name = "casehub.capacity.redistribution.inactivity-escalation",
                            defaultValue = "5m") Duration inactivityEscalation) {
        this.compressThreshold = compressThreshold;
        this.redistributeThreshold = redistributeThreshold;
        this.immediateThreshold = immediateThreshold;
        this.gracePeriod = gracePeriod;
        this.inactivityEscalation = inactivityEscalation;
    }

    @Override
    public RedistributionDecision evaluate(RedistributionContext context) {
        double pressure = context.capacity().aggregatePressure();

        if (context.timeSinceLastActivity().compareTo(inactivityEscalation) >= 0) {
            return RedistributionDecision.escalate(
                    "inactive for " + context.timeSinceLastActivity());
        }

        if (pressure >= immediateThreshold && context.openObligationCount() > 0) {
            return new RedistributionDecision.Redistribute(
                    "pressure " + pressure + " exceeds immediate threshold " + immediateThreshold,
                    Duration.ZERO, Set.of(context.actorId()));
        }
        if (pressure >= redistributeThreshold && context.openObligationCount() > 0) {
            return new RedistributionDecision.Redistribute(
                    "pressure " + pressure + " exceeds redistribute threshold " + redistributeThreshold,
                    gracePeriod, Set.of(context.actorId()));
        }
        if (pressure >= redistributeThreshold && context.openObligationCount() == 0) {
            return RedistributionDecision.hold(
                    "pressure " + pressure + " but no movable obligations");
        }
        if (pressure >= compressThreshold) {
            return RedistributionDecision.compress(
                    "pressure " + pressure + " exceeds compress threshold " + compressThreshold);
        }
        return RedistributionDecision.hold("pressure " + pressure + " below all thresholds");
    }
}
```

Update the test helper `policy()` to accept 3 thresholds and construct with defaults for grace/inactivity:

```java
private DefaultRedistributionPolicy policy(double compress, double redistribute, double escalate) {
    return new DefaultRedistributionPolicy(compress, redistribute, escalate,
            Duration.ofSeconds(30), Duration.ofMinutes(5));
}

private static ActorCapacity capacity(double pressure) {
    return new ActorCapacity("agent-1", pressure, Map.of(), Instant.now());
}
```

- [ ] **Step 4: Update CapacityPressureMonitor config key**

`CapacityPressureMonitor` currently uses `casehub.capacity.redistribution.compress-threshold` — verify this matches. If it uses the old `casehub.capacity.threshold.compress`, update it.

- [ ] **Step 5: Run all platform capacity tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="DefaultRedistributionPolicyTest,AggregatingActorCapacityViewTest,CapacityPressureMonitorTest" -pl platform -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform/src/
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#434): enrich DefaultRedistributionPolicy — obligation/inactivity-aware, @DefaultBean, config consolidation

Refs casehubio/qhorus#434"
```

---

## Batch 2: Commitment count signal source

### Task 2: CrossTenantCommitmentStore.findObligorsExceedingCount + CommitmentCountCapacitySource

**Repo:** `/Users/mdproctor/claude/casehub/qhorus`

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantCommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantCommitmentStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantCommitmentStore.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/CommitmentCountCapacitySource.java`
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/capacity/CommitmentCountCapacitySourceTest.java`

**Interfaces:**
- Consumes: `CrossTenantCommitmentStore.findOpenByObligor(String)` (existing), `CapacitySignalSource` SPI (platform-api)
- Produces: `CrossTenantCommitmentStore.findObligorsExceedingCount(int minCount)` → `Map<String, Long>`, `CommitmentCountCapacitySource` (CDI-discovered by AggregatingActorCapacityView)

- [ ] **Step 1: Add countOpenByObligor + findObligorsExceedingCount to CrossTenantCommitmentStore interface**

Add to `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantCommitmentStore.java`:

```java
/**
 * Count open or acknowledged commitments for a single obligor.
 * Efficient single-row COUNT(*) — no entity materialization.
 */
long countOpenByObligor(String obligor);

/**
 * Find obligors with at least {@code minCount} open or acknowledged commitments.
 * Returns one row per qualifying obligor with their commitment count.
 */
Map<String, Long> findObligorsExceedingCount(int minCount);
```

- [ ] **Step 2: Implement in JpaCrossTenantCommitmentStore**

Add both methods to `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantCommitmentStore.java`:

```java
@Override
public long countOpenByObligor(String obligor) {
    return em.createQuery(
                    "SELECT COUNT(c) FROM CommitmentEntity c " +
                    "WHERE c.obligor = :obligor AND c.state IN ('OPEN', 'ACKNOWLEDGED')", Long.class)
            .setParameter("obligor", obligor)
            .getSingleResult();
}

@Override
public Map<String, Long> findObligorsExceedingCount(int minCount) {
    @SuppressWarnings("unchecked")
    List<Object[]> rows = em.createQuery(
                    "SELECT c.obligor, COUNT(c) FROM CommitmentEntity c " +
                    "WHERE c.state IN ('OPEN', 'ACKNOWLEDGED') " +
                    "GROUP BY c.obligor HAVING COUNT(c) >= :minCount")
            .setParameter("minCount", (long) minCount)
            .getResultList();
    Map<String, Long> result = new java.util.LinkedHashMap<>();
    for (Object[] row : rows) {
        result.put((String) row[0], (Long) row[1]);
    }
    return result;
}
```

- [ ] **Step 3: Implement in InMemoryCrossTenantCommitmentStore**

Add both methods to `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantCommitmentStore.java`:

```java
@Override
public long countOpenByObligor(String obligor) {
    return commitments.values().stream()
            .filter(c -> obligor.equals(c.obligor())
                    && (c.state() == CommitmentState.OPEN || c.state() == CommitmentState.ACKNOWLEDGED))
            .count();
}

@Override
public Map<String, Long> findObligorsExceedingCount(int minCount) {
    return commitments.values().stream()
            .filter(c -> c.state() == CommitmentState.OPEN || c.state() == CommitmentState.ACKNOWLEDGED)
            .collect(java.util.stream.Collectors.groupingBy(
                    io.casehub.qhorus.api.message.Commitment::obligor,
                    java.util.stream.Collectors.counting()))
            .entrySet().stream()
            .filter(e -> e.getValue() >= minCount)
            .collect(java.util.stream.Collectors.toMap(
                    Map.Entry::getKey, Map.Entry::getValue,
                    (a, b) -> a, java.util.LinkedHashMap::new));
}
```

- [ ] **Step 4: Write failing tests for CommitmentCountCapacitySource**

Create `runtime/src/test/java/io/casehub/qhorus/runtime/capacity/CommitmentCountCapacitySourceTest.java`:

```java
package io.casehub.qhorus.runtime.capacity;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.capacity.CapacitySignalTypes;
import io.casehub.qhorus.api.message.Commitment;
import io.casehub.qhorus.api.message.CommitmentState;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.CrossTenantCommitmentStore;

class CommitmentCountCapacitySourceTest {

    @Test
    void observeReturnsPressureFromOpenCommitments() {
        var store = mock(CrossTenantCommitmentStore.class);
        when(store.countOpenByObligor("agent-1")).thenReturn(3L);

        var source = new CommitmentCountCapacitySource(store, 20);

        var signals = source.observe("agent-1");
        assertThat(signals).hasSize(1);
        assertThat(signals.getFirst().pressure()).isCloseTo(0.15, within(0.001));
        assertThat(signals.getFirst().signalType()).isEqualTo(CapacitySignalTypes.TASK_COUNT);
        assertThat(signals.getFirst().actorId()).isEqualTo("agent-1");
    }

    @Test
    void observeClampsPressureToOne() {
        var store = mock(CrossTenantCommitmentStore.class);
        when(store.countOpenByObligor("agent-1")).thenReturn(25L);

        var source = new CommitmentCountCapacitySource(store, 20);

        var signals = source.observe("agent-1");
        assertThat(signals.getFirst().pressure()).isCloseTo(1.0, within(0.001));
    }

    @Test
    void observeReturnsZeroForNoCommitments() {
        var store = mock(CrossTenantCommitmentStore.class);
        when(store.countOpenByObligor("agent-1")).thenReturn(0L);

        var source = new CommitmentCountCapacitySource(store, 20);

        var signals = source.observe("agent-1");
        assertThat(signals).hasSize(1);
        assertThat(signals.getFirst().pressure()).isCloseTo(0.0, within(0.001));
    }

    @Test
    void observeOverloadedUsesEfficientQuery() {
        var store = mock(CrossTenantCommitmentStore.class);
        when(store.findObligorsExceedingCount(16))
                .thenReturn(Map.of("agent-1", 18L, "agent-2", 20L));

        var source = new CommitmentCountCapacitySource(store, 20);

        var overloaded = source.observeOverloaded(0.8);
        assertThat(overloaded).hasSize(2);
        assertThat(overloaded.stream().map(s -> s.actorId()).toList())
                .containsExactlyInAnyOrder("agent-1", "agent-2");
    }

    private static Commitment obligation(String obligor) {
        return Commitment.builder()
                .id(UUID.randomUUID()).correlationId(UUID.randomUUID().toString())
                .channelId(UUID.randomUUID()).messageType(MessageType.COMMAND)
                .requester("requester-1").obligor(obligor)
                .state(CommitmentState.OPEN).tenancyId("tenant-1")
                .capabilityTag("analyst").build();
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentCountCapacitySourceTest -pl runtime`
Expected: FAIL — class does not exist

- [ ] **Step 6: Implement CommitmentCountCapacitySource**

Create `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/CommitmentCountCapacitySource.java`:

```java
package io.casehub.qhorus.runtime.capacity;

import io.casehub.platform.api.capacity.CapacitySignal;
import io.casehub.platform.api.capacity.CapacitySignalSource;
import io.casehub.platform.api.capacity.CapacitySignalTypes;
import io.casehub.qhorus.api.store.CrossTenantCommitmentStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class CommitmentCountCapacitySource implements CapacitySignalSource {

    private final CrossTenantCommitmentStore commitmentStore;
    private final int maxObligations;

    @Inject
    public CommitmentCountCapacitySource(
            CrossTenantCommitmentStore commitmentStore,
            @ConfigProperty(name = "casehub.qhorus.capacity.max-obligations",
                            defaultValue = "20") int maxObligations) {
        this.commitmentStore = commitmentStore;
        this.maxObligations = maxObligations;
    }

    CommitmentCountCapacitySource(CrossTenantCommitmentStore commitmentStore, int maxObligations) {
        this.commitmentStore = commitmentStore;
        this.maxObligations = maxObligations;
    }

    @Override
    public List<CapacitySignal> observe(String actorId) {
        long count = commitmentStore.countOpenByObligor(actorId);
        double pressure = Math.min((double) count / maxObligations, 1.0);
        return List.of(new CapacitySignal(
                actorId, CapacitySignalTypes.TASK_COUNT, pressure, Instant.now(),
                Map.of("commitmentCount", String.valueOf(count),
                       "maxObligations", String.valueOf(maxObligations))));
    }

    @Override
    public List<CapacitySignal> observeOverloaded(double threshold) {
        int minCount = (int) Math.ceil(threshold * maxObligations);
        return commitmentStore.findObligorsExceedingCount(minCount).entrySet().stream()
                .map(e -> new CapacitySignal(
                        e.getKey(), CapacitySignalTypes.TASK_COUNT,
                        Math.min((double) e.getValue() / maxObligations, 1.0),
                        Instant.now(),
                        Map.of("commitmentCount", String.valueOf(e.getValue()),
                               "maxObligations", String.valueOf(maxObligations))))
                .toList();
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CommitmentCountCapacitySourceTest -pl runtime`
Expected: ALL PASS

- [ ] **Step 8: Run full build to verify no compile errors across modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests`
Expected: BUILD SUCCESS (verifies API changes compile in all dependent modules)

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/store/CrossTenantCommitmentStore.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCrossTenantCommitmentStore.java
git add persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryCrossTenantCommitmentStore.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/capacity/CommitmentCountCapacitySource.java
git add runtime/src/test/java/io/casehub/qhorus/runtime/capacity/CommitmentCountCapacitySourceTest.java
git commit -m "feat(#434): CommitmentCountCapacitySource — obligation count as capacity signal

Adds findObligorsExceedingCount to CrossTenantCommitmentStore for efficient
fleet scan. CommitmentCountCapacitySource maps open obligations / max to
pressure 0.0-1.0 via CapacitySignalTypes.TASK_COUNT.

Refs #434"
```

---

## Batch 3: Channel-threshold-aware redistribution

### Task 3: Channel.redistributionCapacityThreshold + V52 migration

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java` — add `redistributionCapacityThreshold` field
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java` — add JPA column
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` — add setter method
- Create: `runtime/src/main/resources/db/qhorus/migration/V52__channel_redistribution_capacity_threshold.sql`

**Interfaces:**
- Consumes: existing Channel record pattern (nullable Double field, V46 routingTrustThreshold as precedent)
- Produces: `Channel.redistributionCapacityThreshold()` (nullable Double), `ChannelService.setRedistributionCapacityThreshold(UUID, Double)`, V52 migration

- [ ] **Step 1: Create V52 migration**

Create `runtime/src/main/resources/db/qhorus/migration/V52__channel_redistribution_capacity_threshold.sql`:

```sql
ALTER TABLE channel ADD COLUMN redistribution_capacity_threshold DOUBLE PRECISION;
```

- [ ] **Step 2: Add field to Channel record**

Add `Double redistributionCapacityThreshold` after `routingTrustThreshold` in the Channel record. Update the canonical constructor parameter list. Add to the Builder. Add backward-compatible constructor that passes `null` for the new field.

Append the field at the end of the record (position 26, after `displayOrder` — becoming a 26-param constructor). This follows the established pattern of prior Channel additions where new fields are appended. Follow the same nullable-Double pattern as `routingTrustThreshold`.

- [ ] **Step 3: Add JPA column to ChannelEntity**

Add to `ChannelEntity.java`:

```java
@Column(name = "redistribution_capacity_threshold")
public Double redistributionCapacityThreshold;
```

Update `toDomain()` and `fromDomain()` to map the field.

- [ ] **Step 4: Add setter to ChannelService**

Add `setRedistributionCapacityThreshold(UUID channelId, Double threshold)` following the pattern of `setRateLimits` — find channel by ID, update, persist.

- [ ] **Step 5: Run Flyway migration schema test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FlywayMigrationSchemaTest -pl runtime`
Expected: PASS — V52 applies cleanly

- [ ] **Step 6: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/qhorus/api/channel/Channel.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java
git add runtime/src/main/resources/db/qhorus/migration/V52__channel_redistribution_capacity_threshold.sql
git commit -m "feat(#434): Channel.redistributionCapacityThreshold — per-channel redistribution threshold

Nullable Double, falls back to global casehub.capacity.redistribution.redistribute-threshold.
V52 migration adds the column.

Refs #434"
```

### Task 4: Executor channel-threshold filtering + RedistributionResult enrichment

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/RedistributionResult.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/capacity/RedistributionExecutedEvent.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/RedistributionDelegate.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/QhorusRedistributionExecutor.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/capacity/QhorusRedistributionExecutorTest.java`

**Interfaces:**
- Consumes: `Channel.redistributionCapacityThreshold()` (from Task 3), `ChannelStore.findById()` (existing)
- Produces: `RedistributionResult(successCount, attemptedCount, filteredCount)`, updated executor with compress-fallback on all-filtered

- [ ] **Step 1: Write failing tests for channel-threshold filtering**

Add tests to `QhorusRedistributionExecutorTest.java`:

```java
@Test
void channelThresholdFiltersObligations() {
    var obligations = List.of(obligation("analyst"));
    when(commitmentStore.findOpenByObligor("agent-1")).thenReturn(obligations);
    when(commitmentStore.findLatestDelegatedByObligor("agent-1")).thenReturn(Optional.empty());
    when(policy.evaluate(any())).thenReturn(
            new RedistributionDecision.Redistribute("high", Duration.ZERO, Set.of("agent-1")));
    when(delegate.redistribute(eq("agent-1"), eq(obligations), any(), anyDouble()))
            .thenReturn(new RedistributionResult(0, 0, 1));

    executor.onCapacityPressure(event("agent-1", 0.86));

    verify(delegate).redistribute(eq("agent-1"), eq(obligations), any(), eq(0.86));
    verify(delegate).compress("agent-1", obligations);
    verify(delegate, never()).escalate(anyString(), anyString());
}

@Test
void compressFallbackWhenAllFiltered() {
    var obligations = List.of(obligation("analyst"), obligation("analyst"));
    when(commitmentStore.findOpenByObligor("agent-1")).thenReturn(obligations);
    when(commitmentStore.findLatestDelegatedByObligor("agent-1")).thenReturn(Optional.empty());
    when(policy.evaluate(any())).thenReturn(
            new RedistributionDecision.Redistribute("high", Duration.ZERO, Set.of("agent-1")));
    when(delegate.redistribute(eq("agent-1"), eq(obligations), any(), anyDouble()))
            .thenReturn(new RedistributionResult(0, 0, 2));

    executor.onCapacityPressure(event("agent-1", 0.86));

    verify(delegate).compress("agent-1", obligations);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QhorusRedistributionExecutorTest -pl runtime`
Expected: FAIL — RedistributionResult constructor signature mismatch (2-arg vs 3-arg)

- [ ] **Step 3: Update RedistributionResult**

Replace `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/RedistributionResult.java`:

```java
package io.casehub.qhorus.runtime.capacity;

public record RedistributionResult(int successCount, int attemptedCount, int filteredCount) {}
```

- [ ] **Step 4: Update RedistributionExecutedEvent**

Modify `api/src/main/java/io/casehub/qhorus/api/capacity/RedistributionExecutedEvent.java`:
- Rename `totalCount` → `attemptedCount`
- Add `filteredCount` field (int)
- Update `redistributed()` factory: `redistributed(String actorId, int successCount, int attemptedCount, int filteredCount)`

```java
public record RedistributionExecutedEvent(
    String actorId,
    Outcome outcome,
    int successCount,
    int attemptedCount,
    int filteredCount,
    String reason,
    Instant occurredAt
) {
    public enum Outcome { COMPRESSED, REDISTRIBUTED, ESCALATED }

    public static RedistributionExecutedEvent compressed(String actorId, int channelCount) {
        return new RedistributionExecutedEvent(actorId, Outcome.COMPRESSED,
                channelCount, channelCount, 0, null, Instant.now());
    }

    public static RedistributionExecutedEvent redistributed(String actorId,
                                                             int successCount, int attemptedCount,
                                                             int filteredCount) {
        return new RedistributionExecutedEvent(actorId, Outcome.REDISTRIBUTED,
                successCount, attemptedCount, filteredCount, null, Instant.now());
    }

    public static RedistributionExecutedEvent escalated(String actorId, String reason) {
        return new RedistributionExecutedEvent(actorId, Outcome.ESCALATED,
                0, 0, 0, reason, Instant.now());
    }
}
```

- [ ] **Step 5: Add channel-threshold filtering to RedistributionDelegate.redistribute()**

Inject `globalRedistributeThreshold` config via `@ConfigProperty(name = "casehub.capacity.redistribution.redistribute-threshold", defaultValue = "0.85") double globalRedistributeThreshold` as a field on `RedistributionDelegate`. Update the test constructor to accept the new parameter (8th param). In `QhorusRedistributionExecutorTest`, the delegate is mocked — no constructor change needed there.

Add the threshold check inside the existing per-obligation loop, after `channelStore.findById()`, before the HANDOFF dispatch:

```java
double threshold = channel.redistributionCapacityThreshold() != null
    ? channel.redistributionCapacityThreshold()
    : globalRedistributeThreshold;
if (aggregatePressure < threshold) {
    filteredCount++;
    continue;
}
```

Update the return and event to use `new RedistributionResult(successCount, redistributable.size() - filteredCount, filteredCount)` and the updated `redistributed()` factory.

- [ ] **Step 6: Update QhorusRedistributionExecutor escalation guard**

Update the Redistribute case to handle the compress-fallback:

```java
case RedistributionDecision.Redistribute r -> {
    // ... existing grace period check ...
    RedistributionResult result = delegate.redistribute(actorId, obligations, r, event.capacity().aggregatePressure());
    if (result.successCount() == 0 && result.attemptedCount() > 0) {
        delegate.escalate(actorId, "redistribution requested but no targets available");
    } else if (result.attemptedCount() == 0 && result.filteredCount() > 0) {
        LOG.infof("All obligations filtered by channel thresholds for %s — compressing", actorId);
        delegate.compress(actorId, obligations);
    }
}
```

- [ ] **Step 7: Fix existing tests for new RedistributionResult signature**

Update all `new RedistributionResult(x, y)` calls in `QhorusRedistributionExecutorTest` to `new RedistributionResult(x, y, 0)` (third param = filteredCount, 0 for non-filtering tests).

- [ ] **Step 8: Run all capacity tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="QhorusRedistributionExecutorTest,ContextPressureCapacitySourceTest,CommitmentCountCapacitySourceTest" -pl runtime`
Expected: ALL PASS

- [ ] **Step 9: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/capacity/RedistributionResult.java
git add api/src/main/java/io/casehub/qhorus/api/capacity/RedistributionExecutedEvent.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/capacity/RedistributionDelegate.java
git add runtime/src/main/java/io/casehub/qhorus/runtime/capacity/QhorusRedistributionExecutor.java
git add runtime/src/test/java/io/casehub/qhorus/runtime/capacity/QhorusRedistributionExecutorTest.java
git commit -m "feat(#434): executor channel-threshold filtering + RedistributionResult enrichment

RedistributionDelegate.redistribute() now filters obligations by
Channel.redistributionCapacityThreshold before HANDOFF dispatch.
RedistributionResult gains attemptedCount + filteredCount.
Executor falls back to compress when all obligations are channel-filtered.

Refs #434"
```

---

## Batch 4: MCP tools + integration tests

### Task 5: Capacity MCP tools

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java`

**Interfaces:**
- Consumes: `ActorCapacityView.getCapacity()`, `ActorCapacityView.getOverloaded()`, `MessageLedgerEntryRepository`, `ChannelService.setRedistributionCapacityThreshold()`, `Channel.redistributionCapacityThreshold()`
- Produces: 5 `@Tool` methods: `get_actor_capacity`, `list_overloaded_actors`, `get_redistribution_history`, `set_channel_redistribution_threshold`, `get_channel_redistribution_threshold`

- [ ] **Step 1: Add ActorCapacityView injection to QhorusMcpTools**

Add `@Inject Instance<ActorCapacityView> capacityView;` to the tools class. Use `Instance<>` for optional availability (no hard dependency when platform capacity module is absent).

- [ ] **Step 2: Implement get_actor_capacity**

```java
@Tool(description = "Get an actor's current capacity pressure across all signal sources")
public String getActorCapacity(@ToolArg(description = "actor ID") String actorId) {
    if (!capacityView.isResolvable()) return "Capacity view not available";
    var capacity = capacityView.get().getCapacity(actorId);
    return mapper.writeValueAsString(Map.of(
            "actorId", capacity.actorId(),
            "aggregatePressure", capacity.aggregatePressure(),
            "pressureBySignalType", capacity.pressureBySignalType(),
            "observedAt", capacity.observedAt().toString()));
}
```

- [ ] **Step 3: Implement list_overloaded_actors**

```java
@Tool(description = "List actors whose aggregate capacity pressure exceeds the threshold")
public String listOverloadedActors(
        @ToolArg(description = "pressure threshold (0.0-1.0, default 0.7)") Double threshold) {
    if (!capacityView.isResolvable()) return "Capacity view not available";
    double t = threshold != null ? threshold : 0.7;
    var overloaded = capacityView.get().getOverloaded(t);
    return renderList("overloaded_actors", overloaded.stream()
            .map(c -> Map.of("actorId", c.actorId(),
                    "aggregatePressure", c.aggregatePressure(),
                    "pressureBySignalType", (Object) c.pressureBySignalType()))
            .toList());
}
```

- [ ] **Step 4: Implement get_redistribution_history**

```java
@Tool(description = "Get redistribution history — HANDOFF messages from system:redistribution")
public String getRedistributionHistory(
        @ToolArg(description = "actor ID to filter by (optional)") String actorId,
        @ToolArg(description = "channel name or UUID (optional)") String channel,
        @ToolArg(description = "max entries to return (default 20)") Integer limit) {
    int maxEntries = limit != null ? Math.min(limit, 100) : 20;
    UUID channelId = null;
    if (channel != null) {
        channelId = resolveChannel(channel).id();
    }
    String tenancyId = currentPrincipal.tenancyId();
    var entries = messageRepo.listEntries(
            channelId, "HANDOFF", "system:redistribution",
            null, null, null, maxEntries, null, tenancyId);
    var filtered = entries.stream()
            .filter(e -> actorId == null || actorId.equals(e.routingOriginalTarget))
            .map(e -> toLedgerEntryMap(e))
            .toList();
    return renderList("redistribution_history", filtered);
}
```

Uses `MessageLedgerEntryRepository.listEntries()` with sender=`system:redistribution` and type=`HANDOFF`. Post-filters by `routingOriginalTarget` (actorId). The `routingOriginalTarget` column (V2003 migration) stores the original target before routing resolved a delegate.

- [ ] **Step 5: Implement set/get_channel_redistribution_threshold**

```java
@Tool(description = "Set per-channel redistribution capacity threshold")
public String setChannelRedistributionThreshold(
        @ToolArg(description = "channel name or UUID") String channel,
        @ToolArg(description = "threshold (0.0-1.0) or null to clear") Double threshold) {
    var ch = resolveChannel(channel);
    channelService.setRedistributionCapacityThreshold(ch.id(), threshold);
    return "Redistribution threshold " + (threshold != null ? "set to " + threshold : "cleared")
           + " for channel " + ch.name();
}

@Tool(description = "Get per-channel redistribution capacity threshold with effective fallback")
public String getChannelRedistributionThreshold(
        @ToolArg(description = "channel name or UUID") String channel) {
    var ch = resolveChannel(channel);
    Double configured = ch.redistributionCapacityThreshold();
    // Read global fallback from config
    return mapper.writeValueAsString(Map.of(
            "channel", ch.name(),
            "configured", configured != null ? configured : "null",
            "effective", configured != null ? configured : globalRedistributeThreshold));
}
```

- [ ] **Step 6: Verify no @Tool name collisions**

Run `ToolOverloadDiscoverabilityTest` to ensure no public non-@Tool method shares a name with a @Tool method:

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ToolOverloadDiscoverabilityTest -pl runtime`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java
git commit -m "feat(#434): capacity MCP tools — get_actor_capacity, list_overloaded_actors, get_redistribution_history, set/get threshold

Adds 5 @Tool methods for capacity observability. get_actor_capacity and
list_overloaded_actors read platform ActorCapacityView (temporary placement).
get_redistribution_history queries ledger for system:redistribution HANDOFFs.

Refs #434"
```

### Task 6: Integration tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/capacity/CapacityRedistributionIT.java`

**Interfaces:**
- Consumes: all capacity components (ContextPressureCapacitySource, CommitmentCountCapacitySource, QhorusRedistributionExecutor, RedistributionDelegate, Channel thresholds, RoutingBridge)
- Produces: 3 integration test scenarios verifying end-to-end redistribution

- [ ] **Step 1: Create test class with @TestProfile**

Create `CapacityRedistributionIT.java` with a profile enabling capacity and setting up required config:

```java
package io.casehub.qhorus.runtime.capacity;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

@QuarkusTest
@TestProfile(CapacityRedistributionIT.Profile.class)
class CapacityRedistributionIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                "casehub.qhorus.capacity.max-obligations", "20",
                "casehub.capacity.redistribution.compress-threshold", "0.7",
                "casehub.capacity.redistribution.redistribute-threshold", "0.85",
                "casehub.capacity.redistribution.immediate-threshold", "0.95",
                "casehub.qhorus.delivery.enabled", "false",
                "quarkus.datasource.qhorus.db-kind", "h2",
                "quarkus.datasource.qhorus.jdbc.url", "jdbc:h2:mem:qhorus-capacity-test;DB_CLOSE_DELAY=-1",
                "quarkus.datasource.qhorus.username", "sa",
                "quarkus.datasource.qhorus.password", "",
                "quarkus.datasource.qhorus.reactive", "false",
                "quarkus.hibernate-orm.qhorus.database.generation", "drop-and-create"
            );
        }
    }
    // ... tests follow
}
```

- [ ] **Step 2: Implement Scenario 1 — channel-threshold filtering**

Test that when agent-1 is at 0.90 pressure, obligations in channel-A (threshold 0.80) are redistributed but obligations in channel-B (threshold 0.95) are not.

Uses `QuarkusTransaction.requiringNew()` for setup. Fires `CapacityPressureEvent` directly via CDI `Event.fireAsync()`. Asserts HANDOFF message exists for channel-A, no HANDOFF for channel-B. Verifies commitment states and ledger entries.

- [ ] **Step 3: Implement Scenario 2 — commitment-count-driven redistribution**

Test that an agent with 20 open commitments (maxObligations=20, pressure=1.0) and low context pressure (0.1) triggers redistribution from commitment pressure alone.

- [ ] **Step 4: Implement Scenario 3 — compress-fallback when all filtered**

Test that when all obligations are in channels with thresholds above the aggregate pressure, no HANDOFF is dispatched and compress fires instead.

- [ ] **Step 5: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapacityRedistributionIT -pl runtime`
Expected: ALL 3 PASS

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/test/java/io/casehub/qhorus/runtime/capacity/CapacityRedistributionIT.java
git commit -m "test(#434): integration tests — channel-threshold filtering, commitment-count redistribution, compress-fallback

Three end-to-end scenarios verify the complete capacity redistribution
pipeline with real MessageService dispatch and RoutingBridge resolution.

Refs #434"
```

---

## References

- `specs/issue-434-capacity-spi-adoption/2026-09-08-capacity-spi-adoption-design.md` — design spec
- `runtime/src/main/java/io/casehub/qhorus/runtime/capacity/` — existing capacity code from #428
- `platform/platform-api/src/main/java/io/casehub/platform/api/capacity/` — platform SPI types
- `platform/platform/src/main/java/io/casehub/platform/capacity/DefaultRedistributionPolicy.java` — platform policy to enrich
- `api/src/main/java/io/casehub/qhorus/api/store/CrossTenantCommitmentStore.java` — store interface for new query
- `docs/specs/2026-05-14-defaultbean-spi-noops-design.md` — @DefaultBean pattern
- casehubio/qhorus#434 — focal issue
- casehubio/qhorus#428 — initial capacity implementation
- casehubio/platform#268 — platform capacity SPI

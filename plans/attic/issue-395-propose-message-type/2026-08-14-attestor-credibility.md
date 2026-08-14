# Attestor Credibility Tracking — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #371 — Attestor credibility tracking for trust model
**Issue group:** #371

**Goal:** Add attestor credibility as a third weighting factor in trust computation so that low-quality attestors are down-weighted automatically.

**Architecture:** Pluggable SPI (`AttestorCredibilityPolicy`) in casehub-ledger api, consumed by `TrustScoreCalculator`. Default `AgreementCredibilityPolicy` in casehub-qhorus computes credibility from peer-vs-policy agreement rate via Bayesian Beta. Config-gated `CollusionAwareCredibilityPolicy` adds mutual-endorsement pair detection.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger 0.2-SNAPSHOT, casehub-qhorus 0.2-SNAPSHOT

## Global Constraints

- casehub-ledger changes must be SNAPSHOT-installed before qhorus can compile against them
- `ActorScore` record change is source-breaking — all 7→8 component constructor sites must update atomically
- `TrustScoreComputer` stays pure Java (no CDI) — credibility weights are pre-resolved by the CDI-aware caller
- Cross-tenant scope for credibility (no tenancyId on SPI methods)
- `credibilityRetention` uses `NaN` sentinel when all attestors have INSUFFICIENT_DATA

## Execution Order

Tasks 1–3 are in casehub-ledger. Task 4 installs the SNAPSHOT. Tasks 5–7 are in casehub-qhorus. Sequential execution required (each task depends on the previous).

---

### Task 1: SPI interface and CredibilityAssessment record (casehub-ledger api)

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/spi/AttestorCredibilityPolicy.java`
- Create: `api/src/main/java/io/casehub/ledger/api/model/CredibilityFlag.java`
- Test: `api/src/test/java/io/casehub/ledger/api/spi/AttestorCredibilityPolicyTest.java`

**Interfaces:**
- Produces: `AttestorCredibilityPolicy.assess(String attestorId)` → `CredibilityAssessment`
- Produces: `AttestorCredibilityPolicy.assessBatch(Set<String> attestorIds)` → `Map<String, CredibilityAssessment>`
- Produces: `CredibilityAssessment(double weight, String reason, Set<CredibilityFlag> flags)`
- Produces: `CredibilityAssessment.NEUTRAL` — static constant (1.0, null, empty)
- Produces: `CredibilityFlag` enum — `SUPPRESSED, COLLUSION_SUSPECT, INSUFFICIENT_DATA, LOW_AGREEMENT`

- [ ] **Step 1: Write the CredibilityFlag enum**

```java
package io.casehub.ledger.api.model;

public enum CredibilityFlag {
    SUPPRESSED,
    COLLUSION_SUSPECT,
    INSUFFICIENT_DATA,
    LOW_AGREEMENT
}
```

- [ ] **Step 2: Write the failing test for CredibilityAssessment**

```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.model.CredibilityFlag;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class AttestorCredibilityPolicyTest {

    @Test
    void neutral_hasFullWeight_noFlags() {
        var neutral = AttestorCredibilityPolicy.CredibilityAssessment.NEUTRAL;
        assertThat(neutral.weight()).isEqualTo(1.0);
        assertThat(neutral.reason()).isNull();
        assertThat(neutral.flags()).isEmpty();
    }

    @Test
    void assessBatch_defaultDelegatesToAssess() {
        AttestorCredibilityPolicy policy = attestorId ->
                new AttestorCredibilityPolicy.CredibilityAssessment(
                        0.8, "test", Set.of(CredibilityFlag.LOW_AGREEMENT));

        var result = policy.assessBatch(Set.of("agent-a", "agent-b"));

        assertThat(result).hasSize(2);
        assertThat(result.get("agent-a").weight()).isEqualTo(0.8);
        assertThat(result.get("agent-b").weight()).isEqualTo(0.8);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AttestorCredibilityPolicyTest -pl api -f /Users/mdproctor/claude/casehub/ledger/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 4: Write the SPI interface**

```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.model.CredibilityFlag;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;

public interface AttestorCredibilityPolicy {

    CredibilityAssessment assess(String attestorId);

    default Map<String, CredibilityAssessment> assessBatch(Set<String> attestorIds) {
        Map<String, CredibilityAssessment> result = new LinkedHashMap<>();
        for (String id : attestorIds) {
            result.put(id, assess(id));
        }
        return result;
    }

    record CredibilityAssessment(
            double weight,
            String reason,
            Set<CredibilityFlag> flags) {

        public static final CredibilityAssessment NEUTRAL =
                new CredibilityAssessment(1.0, null, Set.of());
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AttestorCredibilityPolicyTest -pl api -f /Users/mdproctor/claude/casehub/ledger/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add api/src/main/java/io/casehub/ledger/api/spi/AttestorCredibilityPolicy.java api/src/main/java/io/casehub/ledger/api/model/CredibilityFlag.java api/src/test/java/io/casehub/ledger/api/spi/AttestorCredibilityPolicyTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#371): AttestorCredibilityPolicy SPI + CredibilityAssessment record + CredibilityFlag enum

Refs casehubio/qhorus#371"
```

---

### Task 2: ActorScore credibilityRetention + TrustScoreComputer credibility overloads (casehub-ledger runtime)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java` — ActorScore record (7→8 components), new `compute()` overload, new `computeDimensionScore()` overload
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreCalculator.java` — pass empty map (forward-compatible, no SPI injection yet)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/ComputedTrustScoreSource.java` — update EMPTY_SENTINEL constructor
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java` — update ActorScore usage
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/FrequencyWeightedGlobalStrategy.java` — update ActorScore constructor
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java` — new credibility tests + update existing constructors
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCalculatorTest.java` — update ActorScore constructors

**Interfaces:**
- Consumes: `CredibilityAssessment` from Task 1
- Produces: `ActorScore(trustScore, alpha, beta, decisionCount, overturnedCount, attestationPositive, attestationNegative, credibilityRetention)` — 8-component record
- Produces: `TrustScoreComputer.compute(decisions, attestationsByEntryId, now, credibilityByAttestorId)` — 4-arg overload
- Produces: `TrustScoreComputer.computeDimensionScore(dimensionAttestations, now, credibilityByAttestorId)` — 3-arg overload

- [ ] **Step 1: Write failing test for credibility-weighted compute**

Add to `TrustScoreComputerTest.java`:

```java
@Test
void compute_appliesCredibilityWeight() {
    var computer = new TrustScoreComputer(90);
    var entry = testEntry();
    var attestation = soundAttestation(entry.id, 1.0);
    attestation.attestorId = "reviewer-a";

    var credibility = Map.of("reviewer-a",
            new AttestorCredibilityPolicy.CredibilityAssessment(
                    0.5, "low agreement", Set.of(CredibilityFlag.LOW_AGREEMENT)));

    var result = computer.compute(
            List.of(entry),
            Map.of(entry.id, List.of(attestation)),
            Instant.now(),
            credibility);

    // weight = decayWeight(~1.0) × confidence(1.0) × credibility(0.5)
    // alpha should be ~1.5 (prior 1.0 + 0.5), not ~2.0 (prior 1.0 + 1.0)
    assertThat(result.alpha()).isLessThan(1.8);
    assertThat(result.credibilityRetention()).isLessThan(1.0);
}

@Test
void compute_withoutCredibilityMap_returnsRetentionOne() {
    var computer = new TrustScoreComputer(90);
    var entry = testEntry();
    var attestation = soundAttestation(entry.id, 1.0);

    var result = computer.compute(
            List.of(entry),
            Map.of(entry.id, List.of(attestation)),
            Instant.now());

    assertThat(result.credibilityRetention()).isEqualTo(1.0);
}

@Test
void compute_allInsufficientData_returnsNaN() {
    var computer = new TrustScoreComputer(90);
    var entry = testEntry();
    var attestation = soundAttestation(entry.id, 1.0);
    attestation.attestorId = "reviewer-a";

    var credibility = Map.of("reviewer-a",
            new AttestorCredibilityPolicy.CredibilityAssessment(
                    1.0, null, Set.of(CredibilityFlag.INSUFFICIENT_DATA)));

    var result = computer.compute(
            List.of(entry),
            Map.of(entry.id, List.of(attestation)),
            Instant.now(),
            credibility);

    assertThat(result.credibilityRetention()).isNaN();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TrustScoreComputerTest -pl runtime -f /Users/mdproctor/claude/casehub/ledger/pom.xml`
Expected: FAIL — no 4-arg compute, no credibilityRetention field

- [ ] **Step 3: Add credibilityRetention to ActorScore record**

Use `ide_replace_member` on the `ActorScore` record in `TrustScoreComputer.java` to add the 8th component. Update the Javadoc.

Record becomes:
```java
public record ActorScore(
        double trustScore,
        double alpha,
        double beta,
        int decisionCount,
        int overturnedCount,
        int attestationPositive,
        int attestationNegative,
        double credibilityRetention) {
}
```

- [ ] **Step 4: Fix all ActorScore construction sites in ledger**

Update every `new ActorScore(...)` call to include the 8th argument:
- `TrustScoreComputer.compute()` line 94: add `1.0` (empty decisions → no credibility data)
- `TrustScoreComputer.compute()` line 134: add computed retention value
- `ComputedTrustScoreSource.EMPTY_SENTINEL` line 49: add `1.0`
- `FrequencyWeightedGlobalStrategy.derive()` line 95: add `1.0` (derived score, not direct computation)
- `PerActorTrustComputer.buildScore()`: propagate from `ActorScore.credibilityRetention()`

Use `ide_search_text` with `"new ActorScore("` to find all sites. Use `ide_replace_text_in_file` for each fix.

- [ ] **Step 5: Implement the 4-arg compute() overload**

Add after the existing `compute()` method. The existing 3-arg method delegates to the new one with `Map.of()`:

```java
public ActorScore compute(
        final List<LedgerEntry> decisions,
        final Map<UUID, List<LedgerAttestation>> attestationsByEntryId,
        final Instant now) {
    return compute(decisions, attestationsByEntryId, now, Map.of());
}

public ActorScore compute(
        final List<LedgerEntry> decisions,
        final Map<UUID, List<LedgerAttestation>> attestationsByEntryId,
        final Instant now,
        final Map<String, AttestorCredibilityPolicy.CredibilityAssessment> credibilityByAttestorId) {

    if (decisions.isEmpty()) {
        return new ActorScore(0.5, 1.0, 1.0, 0, 0, 0, 0, 1.0);
    }

    double alpha = 1.0;
    double beta = 1.0;
    int overturnedCount = 0;
    int totalPositive = 0;
    int totalNegative = 0;
    double totalRawWeight = 0.0;
    double totalEffectiveWeight = 0.0;
    int assessedAttestorCount = 0;

    for (final LedgerEntry entry : decisions) {
        final List<LedgerAttestation> attestations =
                attestationsByEntryId.getOrDefault(entry.id, List.of());
        boolean hasNegative = false;

        for (final LedgerAttestation attestation : attestations) {
            final Instant attestationTime =
                    attestation.occurredAt != null ? attestation.occurredAt : now;
            final long ageInDays = Math.max(0,
                    java.time.Duration.between(attestationTime, now).toDays());
            final double decayWeight =
                    decayFunction.weight(ageInDays, attestation.verdict);
            final double rawWeight =
                    decayWeight * Math.max(0.0, Math.min(1.0, attestation.confidence));

            var assessment = credibilityByAttestorId.getOrDefault(
                    attestation.attestorId,
                    AttestorCredibilityPolicy.CredibilityAssessment.NEUTRAL);
            final double credibilityWeight =
                    Math.max(0.0, Math.min(1.0, assessment.weight()));
            final double effectiveWeight = rawWeight * credibilityWeight;

            boolean hasInsufficientData = assessment.flags() != null
                    && assessment.flags().contains(
                            io.casehub.ledger.api.model.CredibilityFlag.INSUFFICIENT_DATA);
            if (!hasInsufficientData) {
                totalRawWeight += rawWeight;
                totalEffectiveWeight += effectiveWeight;
                assessedAttestorCount++;
            }

            if (attestation.verdict == AttestationVerdict.SOUND
                    || attestation.verdict == AttestationVerdict.ENDORSED) {
                alpha += effectiveWeight;
                totalPositive++;
            } else if (attestation.verdict == AttestationVerdict.FLAGGED
                    || attestation.verdict == AttestationVerdict.CHALLENGED) {
                beta += effectiveWeight;
                totalNegative++;
                if (effectiveWeight > 0.0) { hasNegative = true; }
            }
        }
        if (hasNegative) { overturnedCount++; }
    }

    final double trustScore = Math.max(0.0, Math.min(1.0, alpha / (alpha + beta)));
    final double retention = assessedAttestorCount == 0
            ? Double.NaN
            : (totalRawWeight > 0.0 ? totalEffectiveWeight / totalRawWeight : 1.0);

    return new ActorScore(trustScore, alpha, beta,
            decisions.size(), overturnedCount,
            totalPositive, totalNegative, retention);
}
```

- [ ] **Step 6: Add computeDimensionScore credibility overload**

Same pattern — existing 2-arg delegates to new 3-arg with `Map.of()`:

```java
public OptionalDouble computeDimensionScore(
        final List<LedgerAttestation> dimensionAttestations,
        final Instant now) {
    return computeDimensionScore(dimensionAttestations, now, Map.of());
}

public OptionalDouble computeDimensionScore(
        final List<LedgerAttestation> dimensionAttestations,
        final Instant now,
        final Map<String, AttestorCredibilityPolicy.CredibilityAssessment> credibilityByAttestorId) {
    // Same as existing but multiply weight by credibilityWeight
    // ...
}
```

- [ ] **Step 7: Run all ledger tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/ledger/pom.xml`
Expected: ALL PASS (including updated existing tests with 8-arg constructors)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add -A
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#371): credibility-weighted trust computation + ActorScore.credibilityRetention

TrustScoreComputer gains 4-arg compute() and 3-arg computeDimensionScore()
overloads. Existing methods delegate with empty credibility map.
ActorScore record gains credibilityRetention (8th component — source-breaking).
NaN sentinel when all attestors have INSUFFICIENT_DATA.

Refs casehubio/qhorus#371"
```

---

### Task 3: TrustScoreCalculator SPI injection + new LedgerEntryRepository queries (casehub-ledger)

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java` — add `findPeerAttestationsByAttestorIds()` and `findPeerAttestationPairCounts()`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreCalculator.java` — inject SPI, call `assessBatch()`, pass map to all four passes
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/JpaLedgerEntryRepository.java` — implement new queries
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/NoOpAttestorCredibilityPolicy.java` — `@DefaultBean` returning NEUTRAL
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCalculatorTest.java` — verify credibility map passed through

**Interfaces:**
- Consumes: `AttestorCredibilityPolicy` from Task 1, 4-arg `compute()` from Task 2
- Produces: `LedgerEntryRepository.findPeerAttestationsByAttestorIds(Set<String>, String)` → `List<LedgerAttestation>`
- Produces: `LedgerEntryRepository.findPeerAttestationPairCounts(Set<String>, String)` → `Map<String, Map<String, Long>>`
- Produces: `NoOpAttestorCredibilityPolicy` — `@DefaultBean` returning NEUTRAL for all
- Produces: `TrustScoreCalculator.computeAll()` now passes credibility map to `TrustScoreComputer`

- [ ] **Step 1: Write failing test for TrustScoreCalculator credibility integration**

Add to `TrustScoreCalculatorTest.java`:

```java
@Test
void computeAll_passesCredibilityMapToComputer() {
    // Set up a policy that returns 0.5 weight for "reviewer-a"
    AttestorCredibilityPolicy policy = id ->
            "reviewer-a".equals(id)
                    ? new AttestorCredibilityPolicy.CredibilityAssessment(
                            0.5, "test", Set.of(CredibilityFlag.LOW_AGREEMENT))
                    : AttestorCredibilityPolicy.CredibilityAssessment.NEUTRAL;

    var calculator = new TrustScoreCalculator(decayFunction, globalScoreStrategy, policy);
    // ... set up entries and attestations with attestorId = "reviewer-a"

    var result = calculator.computeAll(decisions, attestationsByEntry, Instant.now());

    // Global score should reflect credibility weighting
    assertThat(result.globalScore().credibilityRetention()).isLessThan(1.0);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TrustScoreCalculatorTest -pl runtime -f /Users/mdproctor/claude/casehub/ledger/pom.xml`
Expected: FAIL — no 3-arg constructor for TrustScoreCalculator

- [ ] **Step 3: Add NoOpAttestorCredibilityPolicy @DefaultBean**

```java
package io.casehub.ledger.runtime.service;

import io.casehub.ledger.api.spi.AttestorCredibilityPolicy;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpAttestorCredibilityPolicy implements AttestorCredibilityPolicy {
    @Override
    public CredibilityAssessment assess(String attestorId) {
        return CredibilityAssessment.NEUTRAL;
    }
}
```

- [ ] **Step 4: Modify TrustScoreCalculator to inject SPI and pass credibility map**

Add `AttestorCredibilityPolicy` to constructor injection. In `computeAll()`:
1. Collect all distinct `attestorId` values from `rawAttestations`
2. Call `policy.assessBatch(attestorIds)`
3. Pass the map to all `computer.compute()` and `computer.computeDimensionScore()` calls

- [ ] **Step 5: Add new query methods to LedgerEntryRepository**

Add to the interface:
```java
List<LedgerAttestation> findPeerAttestationsByAttestorIds(Set<String> attestorIds, String tenancyId);

Map<String, Map<String, Long>> findPeerAttestationPairCounts(Set<String> attestorIds, String tenancyId);
```

Add default implementations returning empty results (for backward compat with existing impls).

- [ ] **Step 6: Implement queries in JpaLedgerEntryRepository**

JPQL for `findPeerAttestationsByAttestorIds`:
```sql
SELECT a FROM LedgerAttestation a WHERE a.attestorId IN :attestorIds
  AND a.verdict IN ('ENDORSED', 'CHALLENGED')
```

`findPeerAttestationPairCounts` is a more complex aggregation — group by (attestorId, entryActorId via join) and count.

- [ ] **Step 7: Run all ledger tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/ledger/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add -A
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#371): TrustScoreCalculator credibility injection + repository queries

TrustScoreCalculator injects AttestorCredibilityPolicy, calls assessBatch()
and passes credibility map to all four computation passes.
NoOpAttestorCredibilityPolicy @DefaultBean returns NEUTRAL.
New LedgerEntryRepository queries for peer attestation lookup.

Refs casehubio/qhorus#371"
```

---

### Task 4: Install ledger SNAPSHOT (cross-repo gate)

**Files:** None — build step only.

- [ ] **Step 1: Install ledger SNAPSHOT to local Maven repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -f /Users/mdproctor/claude/casehub/ledger/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 2: Verify qhorus compiles against new SNAPSHOT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: BUILD SUCCESS (ActorScore 8-arg constructor sites in qhorus may need updating — fix any compilation errors now)

- [ ] **Step 3: Fix any qhorus compilation errors from ActorScore change**

Search qhorus for `new ActorScore(` or `ActorScore(` pattern-matching and update all sites to include the 8th component.

- [ ] **Step 4: Run qhorus tests to verify nothing broke**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -Dno-format`
Expected: ALL PASS

- [ ] **Step 5: Commit any qhorus fixes**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "chore(#371): update ActorScore construction sites for 8-component record

Refs #371"
```

---

### Task 5: AgreementCredibilityPolicy — default implementation (casehub-qhorus)

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/AgreementCredibilityPolicy.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` — add Credibility sub-interface to Attestation
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/AgreementCredibilityPolicyTest.java`

**Interfaces:**
- Consumes: `AttestorCredibilityPolicy` from Task 1, `LedgerEntryRepository.findPeerAttestationsByAttestorIds()` from Task 3
- Produces: `AgreementCredibilityPolicy` — `@DefaultBean @ApplicationScoped` implementing `AttestorCredibilityPolicy`

- [ ] **Step 1: Add config properties to QhorusConfig.Attestation**

Add to the `Attestation` interface:
```java
@WithDefault("5")
int credibilityMinDataPoints();

@WithDefault("0.3")
double credibilityLowAgreementThreshold();
```

- [ ] **Step 2: Write failing test — agreement increments α**

```java
@Test
void assess_endorsedMatchingSound_incrementsAlpha() {
    // Set up: attestor "reviewer-a" endorsed entry X, policy attested SOUND on X
    // Expect: agreement → α increases → weight > 0.5
}
```

- [ ] **Step 3: Write failing test — disagreement increments β**

```java
@Test
void assess_endorsedButPolicyFlagged_incrementsBeta() {
    // Set up: attestor "reviewer-a" endorsed entry X, policy attested FLAGGED on X
    // Expect: disagreement → β increases → weight < 0.5
}
```

- [ ] **Step 4: Write failing test — insufficient data flag**

```java
@Test
void assess_fewerThanMinDataPoints_setsInsufficientDataFlag() {
    // Set up: attestor with 2 overlapping attestations, min-data-points=5
    // Expect: INSUFFICIENT_DATA flag, weight = 1.0 (NEUTRAL weight, not penalised)
}
```

- [ ] **Step 5: Write failing test — assessBatch efficiency**

```java
@Test
void assessBatch_queriesOnce_notPerAttestor() {
    // Set up: 3 attestors
    // Expect: findPeerAttestationsByAttestorIds called once with Set.of(3 ids)
}
```

- [ ] **Step 6: Implement AgreementCredibilityPolicy**

```java
@DefaultBean
@ApplicationScoped
public class AgreementCredibilityPolicy implements AttestorCredibilityPolicy {

    @Inject LedgerEntryRepository ledger;
    @Inject QhorusConfig config;

    @Override
    public CredibilityAssessment assess(String attestorId) {
        return assessBatch(Set.of(attestorId)).getOrDefault(attestorId, CredibilityAssessment.NEUTRAL);
    }

    @Override
    public Map<String, CredibilityAssessment> assessBatch(Set<String> attestorIds) {
        // 1. Query all peer attestations for these attestors (single query)
        // 2. For each peer attestation, find the policy attestation on the same entry
        // 3. Compute agreement: peer ENDORSED + policy SOUND = agreement
        //                       peer CHALLENGED + policy FLAGGED = agreement
        //                       else = disagreement
        // 4. Beta(1,1) prior per attestor, accumulate α/β
        // 5. Return CredibilityAssessment per attestor with flags
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=AgreementCredibilityPolicyTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#371): AgreementCredibilityPolicy — default attestor credibility implementation

Agreement-rate Beta model comparing peer verdicts against policy outcomes.
@DefaultBean displaces NoOpAttestorCredibilityPolicy from ledger.
Config: credibility.min-data-points, credibility.low-agreement-threshold.

Refs #371"
```

---

### Task 6: CollusionAwareCredibilityPolicy — config-gated implementation (casehub-qhorus)

**Files:**
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CollusionAwareCredibilityPolicy.java`
- Create: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/CollusionAwareCredibilityProducer.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/config/QhorusConfig.java` — add collusion config
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/CollusionAwareCredibilityPolicyTest.java`

**Interfaces:**
- Consumes: `AgreementCredibilityPolicy` from Task 5 (composition), `LedgerEntryRepository.findPeerAttestationPairCounts()` from Task 3
- Produces: `CollusionAwareCredibilityPolicy` — delegates to AgreementCredibilityPolicy, adds COLLUSION_SUSPECT flag
- Produces: `CollusionAwareCredibilityProducer` — `@Produces` factory selecting A or B based on runtime config

- [ ] **Step 1: Add config properties**

Add to `QhorusConfig.Attestation`:
```java
@WithDefault("false")
boolean collusionDetectionEnabled();

@WithDefault("0.8")
double collusionThreshold();
```

- [ ] **Step 2: Write failing test — mutual endorsement detected**

```java
@Test
void assess_highMutualEndorsement_setsCollusionSuspectFlag() {
    // Set up: agent-a endorsed 5 of agent-b's entries, agent-b endorsed 5 of agent-a's entries
    // Total: 10 mutual, 0 non-mutual → ratio = 1.0, threshold = 0.8
    // Expect: COLLUSION_SUSPECT flag
}
```

- [ ] **Step 3: Write failing test — below threshold, no flag**

```java
@Test
void assess_lowMutualEndorsement_noCollusionFlag() {
    // Set up: agent-a endorsed 1 of agent-b's, agent-b endorsed 1 of agent-a's
    // But each has 10 total endorsements → ratio = 0.2, threshold = 0.8
    // Expect: no COLLUSION_SUSPECT flag
}
```

- [ ] **Step 4: Write failing test — weight comes from base, not collusion**

```java
@Test
void assess_collusionFlagDoesNotChangeWeight() {
    // Set up: high mutual endorsement (flag should fire)
    // But agreement rate is good (weight should be high)
    // Expect: COLLUSION_SUSPECT flag AND high weight (flag signals, doesn't penalise)
}
```

- [ ] **Step 5: Implement CollusionAwareCredibilityPolicy**

Uses composition — receives `AgreementCredibilityPolicy` and `QhorusConfig`:

```java
public class CollusionAwareCredibilityPolicy implements AttestorCredibilityPolicy {

    private final AgreementCredibilityPolicy base;
    private final QhorusConfig config;
    private final LedgerEntryRepository ledger;

    // constructor injection

    @Override
    public Map<String, CredibilityAssessment> assessBatch(Set<String> attestorIds) {
        Map<String, CredibilityAssessment> baseResults = base.assessBatch(attestorIds);
        Map<String, Map<String, Long>> pairCounts =
                ledger.findPeerAttestationPairCounts(attestorIds, null);

        // For each attestor, check mutual endorsement ratio
        // If ratio > threshold, add COLLUSION_SUSPECT to flags
        // Weight unchanged — comes from base

        return enrichedResults;
    }
}
```

- [ ] **Step 6: Implement CollusionAwareCredibilityProducer**

```java
@ApplicationScoped
public class CollusionAwareCredibilityProducer {

    @Inject QhorusConfig config;
    @Inject AgreementCredibilityPolicy base;
    @Inject LedgerEntryRepository ledger;

    @Produces
    @ApplicationScoped
    AttestorCredibilityPolicy policy() {
        if (config.attestation().collusionDetectionEnabled()) {
            return new CollusionAwareCredibilityPolicy(base, config, ledger);
        }
        return base;
    }
}
```

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CollusionAwareCredibilityPolicyTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#371): CollusionAwareCredibilityPolicy — config-gated collusion detection

Composition over AgreementCredibilityPolicy. Mutual-endorsement pair
detection sets COLLUSION_SUSPECT flag without changing weight.
Runtime config gate: casehub.qhorus.attestation.collusion-detection-enabled.

Refs #371"
```

---

### Task 7: Integration test + CLAUDE.md + QhorusLedgerEntryRepository queries (casehub-qhorus)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/ledger/QhorusLedgerEntryRepository.java` — implement new query methods
- Create: `runtime/src/test/java/io/casehub/qhorus/runtime/ledger/AttestorCredibilityIntegrationTest.java`
- Modify: `CLAUDE.md` — document new SPI, config properties, testing conventions

**Interfaces:**
- Consumes: All prior tasks
- Produces: End-to-end verification that credibility affects trust scores

- [ ] **Step 1: Implement new queries in QhorusLedgerEntryRepository**

Override `findPeerAttestationsByAttestorIds()` and `findPeerAttestationPairCounts()` with JPQL implementations using the qhorus named persistence unit.

- [ ] **Step 2: Write integration test**

```java
@QuarkusTest
class AttestorCredibilityIntegrationTest {

    @Test
    void credibilityWeighting_reducesUntrustworthyAttestorImpact() {
        // 1. Create channel, register two instances (subject, reviewer)
        // 2. Dispatch COMMAND → DONE (creates policy attestation SOUND)
        // 3. Write peer attestation CHALLENGED on the same entry (reviewer disagrees with policy)
        // 4. Repeat for 5+ entries so reviewer builds a disagreement track record
        // 5. Compute trust score for the subject
        // 6. Verify credibilityRetention < 1.0 (reviewer's attestations were down-weighted)
    }
}
```

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -Dno-format`
Expected: BUILD SUCCESS, ALL TESTS PASS

- [ ] **Step 4: Update CLAUDE.md**

Add documentation for:
- `AttestorCredibilityPolicy` SPI (in api/spi/ taxonomy)
- `AgreementCredibilityPolicy` and `CollusionAwareCredibilityPolicy` (in runtime/ledger/)
- Config properties
- Testing conventions (CDI-free for policy tests, @QuarkusTest for integration)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#371): attestor credibility integration test + repository queries + docs

End-to-end test verifying credibility weighting reduces untrusworthy
attestor impact on trust scores. QhorusLedgerEntryRepository implements
new peer attestation queries. CLAUDE.md updated.

Closes #371"
```

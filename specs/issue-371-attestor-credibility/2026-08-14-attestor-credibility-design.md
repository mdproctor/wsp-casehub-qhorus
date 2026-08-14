# Attestor Credibility Tracking — Design Spec

**Issue:** casehubio/qhorus#371
**Parent:** #351 (Verification and Trust)
**Date:** 2026-08-14

---

## Problem

The Bayesian Beta trust model (`TrustScoreComputer` in casehub-ledger) scores actors by accumulating attestation evidence on their ledger entries. Each attestation contributes `decayWeight × confidence` to α (SOUND/ENDORSED) or β (FLAGGED/CHALLENGED). The score converges as evidence accumulates.

The model treats all attestations equally regardless of who wrote them. An attestor that consistently ENDORSEs entries that later receive FLAGGED policy attestations suffers no consequence. Two agents can cross-ENDORSE each other's poor work with no penalty beyond the self-attestation guard.

This is a credibility gap: the model scores subjects (actors whose work is attested) but does not score attestors (actors who write attestations).

## Design

### SPI interface — `AttestorCredibilityPolicy`

A pluggable SPI following the established pattern (`CommitmentAttestationPolicy`, `ObligorTrustPolicy`). Consumers override via `@Alternative @Priority` or config-gated bean selection.

**Location:** `casehub-ledger` api module. The SPI is consumed by `TrustScoreCalculator` (the CDI-aware orchestration layer in casehub-ledger runtime) — not by `TrustScoreComputer` directly. The interface lives in the api module so that consumer implementations (in casehub-qhorus or downstream) depend on the api, not the runtime.

Note: `CommitmentAttestationPolicy` and `ObligorTrustPolicy` live in casehub-qhorus api (they are qhorus SPIs). `AttestorCredibilityPolicy` lives in casehub-ledger api (it is a ledger SPI). Different module, same pattern.

```java
public interface AttestorCredibilityPolicy {

    CredibilityAssessment assess(String attestorId);

    Map<String, CredibilityAssessment> assessBatch(Set<String> attestorIds);

    record CredibilityAssessment(
            double weight,
            String reason,
            Set<CredibilityFlag> flags) {

        public static final CredibilityAssessment NEUTRAL =
                new CredibilityAssessment(1.0, null, Set.of());
    }

    enum CredibilityFlag {
        SUPPRESSED,
        COLLUSION_SUSPECT,
        INSUFFICIENT_DATA,
        LOW_AGREEMENT
    }
}
```

- `assess(String attestorId)`: single-attestor lookup. Default implementation delegates to `assessBatch(Set.of(attestorId))`.
- `assessBatch(Set<String> attestorIds)`: batch lookup — avoids N+1 queries when `TrustScoreCalculator` processes all attestations for an actor. The default implementation in casehub-ledger api provides a loop-based fallback; concrete implementations override with a single query.
- `weight` (0.0–1.0): multiplier applied to the attestation's contribution in `TrustScoreComputer`
- `reason` (nullable): human-readable audit trail for why this weight was assigned
- `flags`: machine-readable signals for downstream consumers (watchdogs, dashboards)
- `NEUTRAL` constant: weight 1.0, no reason, no flags — returned when no credibility data exists. This is distinct from the Beta prior of 0.5 (maximum uncertainty) — `NEUTRAL` means "no credibility policy is active or no data exists for this attestor, treat at full weight." The Beta prior 0.5 is internal to the default implementation's computation; `NEUTRAL` is the SPI's "I have nothing to say" signal.

**Tenancy:** Credibility is cross-tenant, consistent with trust scores. `assess()` and `assessBatch()` do not take a tenancyId parameter — the implementation queries across tenants, same as `TrustScoreComputer`.

### Integration with TrustScoreCalculator and TrustScoreComputer

The trust computation pipeline has two layers:

1. **`TrustScoreCalculator`** (`@ApplicationScoped`, CDI-aware) — orchestrates the four-pass algorithm (capability → dimension → capability×dimension → global). Creates `TrustScoreComputer` internally. Both `PerActorTrustComputer` (write path) and `ComputedTrustScoreSource` (read path) delegate here.

2. **`TrustScoreComputer`** (pure Java, no CDI) — the mathematical core. Computes Beta scores from attestation data.

**Credibility flows through `TrustScoreCalculator`:**

`TrustScoreCalculator` injects `AttestorCredibilityPolicy`. Before calling `TrustScoreComputer.compute()`, it:
1. Collects all distinct `attestorId` values from the attestation set
2. Calls `policy.assessBatch(attestorIds)` — one call, not N
3. Passes the resulting `Map<String, CredibilityAssessment>` to `TrustScoreComputer.compute()`

`TrustScoreComputer` gains a new `compute()` overload:

```java
public ActorScore compute(
        List<LedgerEntry> decisions,
        Map<UUID, List<LedgerAttestation>> attestationsByEntryId,
        Instant now,
        Map<String, CredibilityAssessment> credibilityByAttestorId) {
    // weight = decayWeight × confidence × credibilityWeight
}
```

The existing 3-arg `compute()` delegates to the new overload with `Map.of()`. `TrustScoreComputer` remains pure Java — it receives pre-resolved weights, never calls the SPI.

**`computeDimensionScore()` also gains credibility weighting** — a parallel overload accepting `Map<String, CredibilityAssessment>`. Without this, capability and dimension scores would be computed differently from the global score, creating inconsistent trust signals. `TrustScoreCalculator.computeAll()` passes the same credibility map to both methods.

**Source-breaking change:** `ActorScore` is a Java record with 7 components. Adding `credibilityRetention` (double, 0.0–1.0) as an 8th component breaks all call sites using the canonical constructor or positional destructuring. All callers constructing `ActorScore` — `TrustScoreComputer.compute()`, unit tests, `TrustScoreCalculator` — must be updated atomically with the ledger SNAPSHOT. The field is `1.0` when no credibility map is provided (behaviorally backward compatible, but structurally breaking).

`credibilityRetention` = `totalEffectiveWeight / totalRawWeight`, computed only from attestors whose `CredibilityAssessment` does NOT carry the `INSUFFICIENT_DATA` flag. Attestors with insufficient data are excluded from both numerator and denominator — they contribute at full weight (NEUTRAL) but don't count toward the retention metric. This avoids the misleading case where `credibilityRetention = 1.0` when the credibility system simply has no data.

If ALL attestors have `INSUFFICIENT_DATA` (no attestor has enough history to assess), `credibilityRetention` is reported as `NaN` — a sentinel meaning "credibility system has no data for any attestor; the score is unweighted." Callers must handle `NaN` explicitly (display as "N/A" or "insufficient data," not as 1.0). `Double.isNaN()` is the check.

A value of 0.3 means 70% of attestation weight from credibility-assessed attestors was down-weighted — the score rests on thin evidence. A value of 1.0 means all assessed attestors have full credibility.

### Default implementation — `AgreementCredibilityPolicy`

**Location:** `casehub-qhorus` runtime module (`@DefaultBean @ApplicationScoped`).

Computes attestor credibility by comparing peer verdicts against policy outcomes on the same entry:

1. For each entry where attestor A wrote a peer attestation (ENDORSED/CHALLENGED):
   - Find the policy attestation (SOUND/FLAGGED) on the same entry
   - If both exist: agreement (peer ENDORSED + policy SOUND, or peer CHALLENGED + policy FLAGGED) increments α; disagreement increments β
   - If no policy attestation exists on that entry: skip (no ground truth)
2. Score = α/(α+β) with Beta(1,1) prior — no data = 0.5 (neutral)
3. Return `CredibilityAssessment(score, reason, flags)`:
   - `flags` includes `INSUFFICIENT_DATA` when fewer than N entries have both peer and policy attestations (configurable threshold)
   - `flags` includes `LOW_AGREEMENT` when score < configurable threshold

**Scope:** Global per attestor — one Beta distribution per attestor, not per capability tag. Capability-scoped credibility is left to SPI overrides where consumers partition by `capabilityTag`.

**Data access — new query required:** The existing `LedgerEntryRepository` has `findAttestationsByAttestorIdAndCapabilityTag(String, String, String)` but no method to find all attestations by a given attestor across all capabilities, or to find policy attestations on entries that a given attestor also attested. The `assessBatch()` implementation needs:

1. New query: `findPeerAttestationsByAttestorIds(Set<String> attestorIds, String tenancyId)` — returns all ENDORSED/CHALLENGED attestations written by the given attestors. Added to `LedgerEntryRepository` in casehub-ledger api.
2. Existing query: `findAttestationsByEntryId(UUID, String)` — used to find the policy attestation on the same entry. Called per entry (acceptable because the batch is already scoped by the attestor's peer attestation count, not the total ledger size).

Alternative: a single join query that returns (peerAttestation, policyAttestation) pairs per entry — more efficient but requires a new repository method with a complex return type. Start with the two-query approach; optimize if profiling shows it's a bottleneck.

**Timing attack resistance:** An attestor could attempt to game agreement rate by timing peer attestations to arrive before or after policy attestations. This is mitigated by the fact that policy attestations are written automatically by `StoredCommitmentAttestationPolicy` at terminal-message time — they are not controllable by the attestor. The attestor can choose *when* to review (before or after the terminal message), but cannot control the policy outcome. Early endorsement of an entry that later gets FLAGGED still counts as disagreement.

### Collusion-aware implementation — `CollusionAwareCredibilityPolicy`

**Location:** `casehub-qhorus` runtime module. Uses composition (delegates to `AgreementCredibilityPolicy`), not inheritance — avoids tight coupling to the base class internals.

Activated by config gate `casehub.qhorus.attestation.collusion-detection-enabled` (default `false`). Uses `@Produces` with runtime config check, not `@IfBuildProperty` — collusion detection should be toggleable at runtime without rebuilding.

```java
@ApplicationScoped
public class CollusionAwareCredibilityProducer {

    @Inject QhorusConfig config;
    @Inject AgreementCredibilityPolicy base;

    @Produces
    @ApplicationScoped
    AttestorCredibilityPolicy policy() {
        if (config.attestation().collusionDetectionEnabled()) {
            return new CollusionAwareCredibilityPolicy(base, config);
        }
        return base;
    }
}
```

Adds mutual-endorsement pair detection on top of the agreement-rate base:

1. Delegate to the base `AgreementCredibilityPolicy` for the agreement-rate computation
2. For each attestor in the batch, scan for mutual-endorsement anomalies:
   - Find all entries where attestor A endorsed attestor B's work
   - Find all entries where attestor B endorsed attestor A's work
   - Compute mutual-endorsement ratio: `mutual / (totalA + totalB - mutual)`
   - If ratio exceeds threshold (`casehub.qhorus.attestation.collusion-threshold`, default 0.8): add `COLLUSION_SUSPECT` flag
3. Collusion flag does NOT automatically reduce weight — it signals. The weight comes from the agreement-rate base. A colluding pair that also disagrees with policy outcomes already has low credibility from the base. The flag adds visibility for operators.

**Data access for collusion detection:** Requires a query to find all peer attestations grouped by (attestorId, entryActorId) pairs. New query: `findPeerAttestationPairCounts(Set<String> attestorIds, String tenancyId)` — returns counts of mutual endorsements between attestor pairs. Added to `LedgerEntryRepository`.

**Limitation — pair detection only:** This detects 2-actor collusion rings (A↔B). Larger rings (A→B→C→A) where no single pair has anomalous mutual endorsement are not detected. This is an acknowledged limitation — the existing `CIRCULAR_DELEGATION` watchdog condition handles chain cycles in the delegation domain; a similar network-analysis approach for attestation rings is deferred to a future issue.

### Configuration

```properties
# Credibility — agreement rate
casehub.qhorus.attestation.credibility.min-data-points=5
casehub.qhorus.attestation.credibility.low-agreement-threshold=0.3

# Collusion detection (opt-in, runtime-toggleable)
casehub.qhorus.attestation.collusion-detection-enabled=false
casehub.qhorus.attestation.collusion-threshold=0.8
```

### What changes — by repo

**casehub-ledger (api module):**
- New: `AttestorCredibilityPolicy` interface + `CredibilityAssessment` record + `CredibilityFlag` enum
- Modified: `LedgerEntryRepository` — new `findPeerAttestationsByAttestorIds()` and `findPeerAttestationPairCounts()` query methods

**casehub-ledger (runtime module):**
- Modified: `TrustScoreComputer` — new 4-arg `compute()` overload + new 2-arg `computeDimensionScore()` overload; existing methods delegate with empty map
- Modified: `ActorScore` record — gains `credibilityRetention` field (8th component, source-breaking)
- Modified: `TrustScoreCalculator` — injects `AttestorCredibilityPolicy`, calls `assessBatch()` before `compute()`, passes credibility map to all four passes (capability, dimension, capability×dimension, global)
- Modified: `JpaLedgerEntryRepository` — implements new query methods

**casehub-qhorus (runtime module):**
- New: `AgreementCredibilityPolicy` — `@DefaultBean` implementation
- New: `CollusionAwareCredibilityPolicy` — composition-based, runtime config-gated
- New: `CollusionAwareCredibilityProducer` — `@Produces` factory
- New: config properties in `QhorusConfig.Attestation`
- Modified: `QhorusLedgerEntryRepository` — implements new query methods (if qhorus has its own repo subclass)

### What does NOT change

- **Attestation write path:** `LedgerWriteService`, `PeerAttestationWriter`, `StoredCommitmentAttestationPolicy` are unchanged. Raw attestation data (confidence, verdict) is preserved as-is in the ledger.
- **Peer attestation flow:** `PeerReviewAutoTrigger`, `PeerReviewResponseHandler`, `ReviewerResolver` are unchanged.
- **ObligorTrustPolicy:** Consumes trust scores from the gate service. Credibility is already factored into the score it receives.

### Inherited integration — no changes needed

**`IncrementalTrustUpdateObserver`** — CDI observer that triggers per-actor recomputation when an attestation is persisted. Delegates to `PerActorTrustComputer`, which delegates to `TrustScoreCalculator.computeAll()`. Credibility is resolved inside `TrustScoreCalculator` — the incremental path inherits it automatically. No changes needed.

**`GlobalScoreStrategy`** — SPI that selects which attestations feed the global Beta model. `selectAttestations()` filters raw attestations; the filtered result flows through `TrustScoreComputer.compute()` inside `TrustScoreCalculator`, where credibility weighting is applied. `derive()` can optionally override the global score from capability scores — capability scores already include credibility weighting (computed in the capability pass). No changes needed.

**`PerActorTrustComputer`** and **`ComputedTrustScoreSource`** — both delegate to `TrustScoreCalculator.computeAll()`. Credibility inherited automatically. No changes needed.

### EigenTrust — explicitly out of scope

`EigenTrustComputer` operates on raw `LedgerAttestation` data — it builds a trust matrix from positive/negative attestation counts between actors. It does NOT consume `TrustScoreComputer` output or computed trust scores. Credibility weighting applied inside `TrustScoreComputer` is invisible to EigenTrust.

Integrating credibility into EigenTrust would require either: (a) pre-filtering attestations by credibility weight before passing them to `EigenTrustComputer`, or (b) modifying the trust matrix construction to weight by credibility. Both are non-trivial and deferred. The EigenTrust scores will reflect raw attestation counts regardless of attestor credibility — this is a known limitation of the current scope.

### Testing

**casehub-ledger:**
- `TrustScoreComputer` unit tests (CDI-free): verify `decayWeight × confidence × credibilityWeight` for the new overload; verify existing 3-arg overload unchanged; verify `credibilityRetention` computation; verify `computeDimensionScore()` credibility overload
- `TrustScoreCalculator` unit tests: verify `assessBatch()` called with correct attestor IDs; verify credibility map passed to all four passes
- `ActorScore` construction: verify all callers updated for 8-arg constructor
- `LedgerEntryRepository` new query methods: contract tests for `findPeerAttestationsByAttestorIds` and `findPeerAttestationPairCounts`

**casehub-qhorus:**
- `AgreementCredibilityPolicy` CDI-free unit tests: mock `LedgerEntryRepository`, verify Beta computation from agreement data, verify INSUFFICIENT_DATA and LOW_AGREEMENT flags, verify `assessBatch()` efficiency (single query, not N)
- `CollusionAwareCredibilityPolicy` CDI-free unit tests: verify mutual-endorsement detection, verify COLLUSION_SUSPECT flag, verify threshold config, verify delegation to base
- `CollusionAwareCredibilityProducer` test: verify config gate selects correct implementation
- Integration test: dispatch COMMAND → DONE with peer attestation, verify credibility-weighted trust score differs from unweighted

### Scope boundaries

- **Not in scope:** Capability-scoped credibility in the default implementation (D7 — SPI override concern)
- **Not in scope:** EigenTrust credibility integration (requires separate design — see section above)
- **Not in scope:** Automatic remediation of colluding pairs (detection only — operators decide)
- **Not in scope:** Credibility decay over time (credibility is recomputed from the full attestation history each time; recency decay on the underlying attestations already handles temporal weighting)
- **Not in scope:** UI/dashboard for credibility visualization (the data is available via the SPI; presentation is a consumer concern)
- **Not in scope:** Collusion rings larger than 2 actors (pair detection only — network analysis deferred)

## Acceptance criteria

- Attestor credibility signal exists as a pluggable SPI (`AttestorCredibilityPolicy`) in casehub-ledger api with both single and batch assessment methods
- Default implementation (`AgreementCredibilityPolicy`) computes credibility from agreement rate with policy outcomes via Bayesian Beta model
- `TrustScoreCalculator` resolves credibility via batch SPI call and passes weights to `TrustScoreComputer`
- `TrustScoreComputer` applies credibility as a third weighting factor in both `compute()` and `computeDimensionScore()`
- `ActorScore.credibilityRetention` tells callers how much credibility weighting affected the score
- Collusion-aware implementation ships (runtime config-gated, off by default) with mutual-endorsement pair detection via composition
- New `LedgerEntryRepository` query methods for peer attestation lookup
- Raw attestation data (confidence, verdict) preserved unchanged in the ledger

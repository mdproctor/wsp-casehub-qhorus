## D1: Attestor credibility as pluggable SPI

**Choice:** Pluggable SPI with @DefaultBean default implementation, consistent with CommitmentAttestationPolicy, ObligorTrustPolicy, InboundNormaliser pattern
**Alternatives:**
- Fixed Bayesian Beta only — simpler but not extensible
- Fixed agreement rate only — simpler but no uncertainty awareness
**Rationale:** Consumers may want domain-specific credibility logic (e.g. capability-scoped credibility). SPI lets qhorus ship a sensible default while Claudony or other consumers can override.
**Trade-offs:** One more SPI to maintain; consumers must understand the contract to override safely
**Exploration:** quick
**Status:** captured

## D2: Credibility weight applied at trust computation time (ledger), not attestation write time (qhorus)

**Choice:** TrustScoreComputer applies credibility as a third factor (decayWeight × confidence × credibility) at computation time
**Alternatives:**
- Apply at attestation write time — multiply confidence before persisting. Zero ledger changes but bakes credibility into the immutable record; rehabilitated attestors' old attestations stay penalised forever
**Rationale:** Preserves raw attestation confidence in the ledger (auditable, immutable). Credibility is applied dynamically — a rehabilitated attestor's old attestations automatically get re-weighted on next computation. Separates "what the policy assessed" from "how much we trust the assessor."
**Trade-offs:** Requires casehub-ledger changes (TrustScoreComputer gains a credibility lookup). Computation is slightly more expensive (one lookup per attestation).
**Exploration:** quick
**Status:** captured

## D3: SPI returns a rich result record, not a bare double

**Choice:** Return a record with weight (double, 0-1), reason (String, nullable for audit trail), and flags (e.g. SUPPRESSED, COLLUSION_SUSPECT) rather than a plain double
**Alternatives:**
- Bare double — minimal contract, trivial integration, but no audit trail and no way to distinguish "low credibility" from "actively suppressed"
**Rationale:** Audit trail matters for trust — when an attestation is down-weighted, operators need to know why. Flags enable downstream consumers (watchdogs, dashboards) to react to specific conditions (collusion detection) without parsing the weight value.
**Trade-offs:** Richer contract means more surface area for SPI implementors. TrustScoreComputer must handle the record, not just a number.
**Exploration:** quick
**Status:** captured

## D4: Default implementation uses agreement rate with policy outcomes via Beta model

**Choice:** Default compares each attestor's peer verdicts (ENDORSED/CHALLENGED) against the eventual policy attestation (SOUND/FLAGGED) on the same entry. Agreement increments α, disagreement increments β. Beta(1,1) prior — no data = 0.5 (neutral), not penalised.
**Alternatives:**
- Blend in EigenTrust-derived standing — muddies what credibility means; EigenTrust operates at network level, not attestor level
- Include attestation volume as a signal — already implicit in the Beta distribution (more data → tighter convergence)
- Raw agreement rate without Beta — no uncertainty awareness; can't distinguish "no data" from "mediocre"
**Rationale:** Policy outcomes are the only ground truth. Peer verdicts are opinions; agreement with outcomes is the measurable signal. Sparse data (few overlapping entries) is handled by the prior. Clean single-signal default; SPI exists for consumers who want to blend additional signals.
**Trade-offs:** Only entries with both a peer attestation AND a policy attestation contribute signal. Attestors reviewing entries that never get policy outcomes have no credibility data (stay at 0.5).
**Exploration:** quick
**Status:** captured

## D5: Ship both implicit mitigation (A) and explicit detection (B); default to A

**Choice:** Implement two concrete SPI implementations. (A) `AgreementCredibilityPolicy` — @DefaultBean, agreement-rate Beta model only, no flags. (B) `CollusionAwareCredibilityPolicy` — extends A with mutual-endorsement pair detection, sets COLLUSION_SUSPECT flag on anomalous pairs. Both ship in qhorus. A is the default; B activates via config gate `casehub.qhorus.attestation.collusion-detection-enabled` (default false).
**Alternatives:**
- Ship A only, defer B — risks forgetting the detection implementation; loses the design context
- Ship B as default — premature without real-world data on false positive rates; risks alert fatigue
- Activate B via @Alternative @Priority — too opaque for operators; config gate is explicit and discoverable
**Rationale:** Both implementations are designed now while the context is fresh. A is safe as default (implicit mitigation via credibility decay). B is gated behind config because it hasn't been validated against real-world data — config gate makes its experimental status explicit and lets operators enable it without CDI bean overrides. B extends A (not a parallel fork), so maintenance cost is proportionate.
**Trade-offs:** Two implementations to maintain. B's mutual-endorsement heuristic needs a threshold — shipped with a conservative default, tunable via config.
**Review flag (Light):** B risks rotting without real data. Config gate (not @Alternative) makes experimental status explicit.
**Depends on:** D1 (SPI pattern), D3 (rich result with flags), D4 (agreement-rate signal)
**Exploration:** quick
**Status:** captured

## D6: SPI interface in casehub-ledger api, implementations in casehub-qhorus runtime

**Choice:** Interface lives in casehub-ledger api module (alongside TrustScoreComputer which consumes it). Concrete implementations (AgreementCredibilityPolicy, CollusionAwareCredibilityPolicy) live in casehub-qhorus runtime (where peer and policy attestation data is accessible).
**Alternatives:**
- Interface in casehub-qhorus api — TrustScoreComputer would need a qhorus dependency, inverting the dependency direction
- Interface and impls both in ledger — impls need qhorus data (peer attestations), creating a circular dependency
**Rationale:** Same split as LedgerEntryRepository (interface in ledger, JPA impl in qhorus). TrustScoreComputer references the interface without knowing about qhorus. Qhorus provides the CDI bean at runtime.
**Trade-offs:** Interface is in a different repo than its implementations — cross-repo API evolution requires coordination.
**Depends on:** D2 (computation-time application in ledger)
**Exploration:** quick
**Status:** captured

## D7: Global credibility per attestor in default; capability-scoped is an SPI override concern

**Choice:** Default implementation tracks one Beta distribution per attestor (global). Capability-scoped credibility (separate distribution per attestor × capabilityTag) is left to SPI overrides.
**Alternatives:**
- Capability-scoped in default — more precise but sparser data per bucket; many capability tags with few attestations each produce noisy scores
- Hybrid (global with capability bonus/penalty) — added complexity without clear benefit until real-world data exists
**Rationale:** Global avoids the sparse-data problem. The capabilityTag is already on every attestation — a consumer override can partition by it. Keeps the default simple and convergence fast.
**Trade-offs:** An attestor credible in one domain but not another gets a blended score. Acceptable for the default; consumers who care override the SPI.
**Exploration:** quick
**Status:** captured

## D8: Add credibilityRetention to ActorScore; TrustScoreComputer receives pre-resolved weights

**Choice:** Add one field — `credibilityRetention` (double, 0.0–1.0) — to ActorScore. TrustScoreComputer receives pre-resolved credibility weights as `Map<String, CredibilityAssessment>` from the caller, stays CDI-free and pure. 1.0 = no effect (or no credibility data — backward compatible). Lower = more attestations down-weighted.
**Alternatives:**
- Keep ActorScore unchanged, separate companion record — forces callers to correlate two objects; existing fields (overturnedCount) already serve as evidence-quality signals
- Add flaggedAttestorCount alongside retention — couples ActorScore to implementation B's flag vocabulary; flags available from SPI directly
- Pass SPI interface to TrustScoreComputer — breaks its pure-Java, CDI-free design; better to pre-resolve in the CDI-aware caller
**Rationale:** Operators need to know "can I trust this number?" — credibilityRetention answers that with one ratio. Free to compute during the existing attestation iteration. Keeps TrustScoreComputer pure: caller resolves credibility, passes weights as a map. No SPI dependency in the computation.
**Trade-offs:** ActorScore is a Java record — adding an 8th component is a **source-breaking change**. Every call site using the canonical constructor or positional destructuring breaks. All callers constructing ActorScore (TrustScoreComputer.compute(), unit tests, EigenTrust pipeline) must be updated. This is a cross-repo migration concern (ActorScore is in casehub-ledger, consumed by casehub-qhorus and potentially casehub-engine). The compute() method's behavior is backward compatible (retention = 1.0 when no credibility map provided), but the record shape is not — callers must be updated atomically with the ledger SNAPSHOT.
**Depends on:** D2 (computation-time)
**Exploration:** deep-analysis
**Status:** captured
**Review flag (Light):** D6 removed from dependency chain — field addition depends on D2 only. Source-breaking record change called out explicitly.

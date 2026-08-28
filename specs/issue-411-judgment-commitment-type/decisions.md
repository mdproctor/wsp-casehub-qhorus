## D1: Compose with existing types — no new message types

**Choice:** Use COMMAND + DONE + EVENTs + attestation for governed yields instead of adding JUDGMENT_REQUEST/JUDGMENT_RESPONSE/JUDGMENT_ACCEPTANCE types.
**Alternatives:**
- New message types (JUDGMENT_REQUEST, JUDGMENT_RESPONSE, JUDGMENT_ACCEPTANCE) — violates ADR-0005 stopping criterion; all three occupy existing Searle-category cells
- PROPOSE for judgment requests — PROPOSE has the same commitment direction as COMMAND (requester=sender, obligor=receiver) and RESPONSE is non-fulfilling (desirable for verified acceptance). However, PROPOSE is a commissive ("I offer to do X if you agree") while a judgment request is a directive ("I need you to do X"). Using PROPOSE would misclassify the speech act, undermining LLM classification accuracy — the core design goal of ADR-0005.
**Rationale:** ADR-0005 stopping criterion: new types justified only when occupying a unique cell in the Searle-category × deontic-effect matrix. None of the proposed types occupy empty cells. Between existing types, COMMAND (directive) is the correct Searle classification for judgment requests — the engine is directing an agent to act, not making a conditional offer. The dual-lifecycle this creates (commitment lifecycle tracks obligation discharge, judgment EVENTs track quality verification) is correct by design: "did you do the work?" and "was the work good?" are orthogonal assessments that should be tracked separately. A reviewer who sends DONE has fulfilled their obligation to provide a review. Whether the review meets quality standards is a separate quality assessment handled by the attestation layer, not the commitment lifecycle.
**Trade-offs:** No type-level discrimination between regular COMMANDs and judgment COMMANDs — judgment semantics are in telemetry metadata, not the type system. Obligation fulfillment (DONE → FULFILLED) and quality assessment (VERIFIED → ACCEPTED/REJECTED) are tracked in separate systems. This is intentional: a contractor fulfilling their obligation by delivering work is distinct from an inspector accepting the work.
**Sources:** ADR-0005 (stopping criterion matrix), PROPOSE spec (#395 — commitment direction clarified: same as COMMAND, not inverted), #413 spec (judgment compliance evidence — already landed with V2004 migration)
**Review response:** R1-01 correctly identified the PROPOSE characterization error (PROPOSE direction IS same as COMMAND). R1-02's dual-lifecycle concern is addressed: the divergence is a deliberate design separation of obligation fulfillment from quality assessment. R1-03's classification concern is addressed: judgment discrimination is in metadata/EVENTs, not content parsing — the type system correctly classifies the speech act.
**Exploration:** quick
**Status:** revised

## D2: Per-capability trust differentiation is an eidos concern

**Choice:** Trust differentiation per judgment type (trusted for code_review but not legal_review) belongs in eidos's AgentSelector, not qhorus.
**Alternatives:**
- Per-capability trust thresholds on Channel — qhorus owns routing bridge, could own threshold granularity
- Layered (qhorus channel floor + eidos capability refinement) — correct but premature wiring
**Rationale:** Trust scores are per-actor in casehub-ledger. Capability differentiation requires the capability model, which eidos owns. Qhorus passes the capability tag through RoutingBridge; eidos selects based on it. File eidos issue for AgentSelector per-capability trust refinement.
**Trade-offs:** Qhorus cannot enforce judgment-type-specific trust thresholds without eidos on the classpath. Standalone qhorus installations get uniform trust thresholds only.
**Sources:** RoutingBridge.java (runtime/message/), AgentRegistry/AgentSelector (casehub-eidos-api), reputation routing spec (#401)
**Exploration:** quick
**Status:** captured

## D3: Branch scope — all three issues, reframed

**Choice:** All three issues on this branch: #411 → judgment composition pattern + attestation feedback; #412 → close as covered by #401 + file eidos issue; #414 → formal verification properties + offline tool in compliance-report module.
**Alternatives:**
- #411 + #414 only — closes #412 separately
- #414 only — closes #411 and #412 as already-covered
**Rationale:** All three issues share the governed yield theme. #411 and #412 are smaller than originally scoped (composition not new types, routing already exists). Combining gives a coherent branch with clear deliverables.
**Trade-offs:** Branch touches compliance-report module AND runtime attestation — two areas. But the changes are additive, not structural.
**Sources:** Epic #410 body, #401 spec (reputation routing), #413 spec (judgment compliance evidence)
**Exploration:** quick
**Status:** captured

## D4: Deferred attestation via SPI + verification observer

**Choice:** Two-part attestation for judgment commitments: (1) a `JudgmentCommitmentAttestationPolicy` (SPI override) that DEFERS attestation on DONE — writes no attestation at DONE time, returning a "deferred" outcome; (2) a `JudgmentVerificationObserver` (MessageObserver, LOCAL scope) that writes the sole attestation when the VERIFIED event lands. This ensures exactly one attestation per commitment (matching the existing trust model), timed to when quality is actually known.
**Alternatives:**
- Supplementary attestation (original approach) — writes TWO attestations per judgment commitment (DONE + VERIFIED). Causes double-counting in Beta trust model, contradictory signals on rejection (SOUND + FLAGGED on same entry). Rejected per R1-04.
- Scheduled sweep — batched, resilient to missed events, but delayed trust feedback
- Part of LedgerWriteService — couples ledger writing to attestation logic
**Rationale:** The `CommitmentAttestationPolicy` SPI exists for exactly this: consumers override default attestation behavior. The judgment flow defers attestation because quality isn't known at DONE time. When the VERIFIED event arrives, the observer writes the attestation with verification-informed verdict and confidence: ACCEPTED → SOUND with `evidenceQuality` as confidence; REJECTED → FLAGGED/0.3; PARTIAL → FLAGGED/0.5. One attestation per commitment, no double-counting.
**Depends on:** D1 (COMMAND composition — commitment correlationId links DONE to VERIFIED events)
**Recovery path:** If the observer misses a VERIFIED event (crash, restart), the attestation is never written. The offline verification tool (D8, evidence completeness property) detects this and writes the missing attestation as remediation — not just detection. The remediation query: find DONE ledger entries with matching VERIFIED events but no attestation.
**Trade-offs:** Attestation is delayed from DONE time to VERIFIED event time. Trust scores for judgment callers update slower than for regular COMMAND callers. The deferred policy must be activated per-channel or per-deployment — judgment channels opt in, regular channels keep the default immediate attestation.
**Sources:** CommitmentAttestationPolicy SPI (api/spi/), MessageObserver SPI (api/gateway/), JudgmentEventKinds (#413 contract), StoredCommitmentAttestationPolicy (runtime/audit/)
**Review response:** R1-04 correctly identified double-counting in the supplementary attestation approach. R1-05's recovery concern addressed with remediation in the offline verification tool.
**Exploration:** quick
**Status:** revised

## D5: Java predicates for formal verification

**Choice:** Document properties in CTL/LTL notation. Implement each as a Java predicate class that queries the ledger and returns violations.
**Alternatives:**
- Property DSL (enum-based language compiling to ledger queries) — adds parser/compiler for marginal benefit
- Full CTL model checker library — massive dependency for ~5 properties
**Rationale:** Simple, testable, no dependencies. Each property is a class with a `check(tenancyId, from, to)` method that returns `List<PropertyViolation>`. Ledger queries already exist or are straightforward to add. Properties documented in formal CTL/LTL notation in the spec for design intent.
**Semantic gap (R1-06):** CTL/LTL notation describes system properties over ALL paths and ALL futures (model checking). Java predicates are trace checkers — they verify "no violations observed in this time window," not "the system satisfies this temporal property." The spec must be explicit: CTL/LTL documents INTENT; predicates verify observed HISTORY. Consumers must not over-rely on a passing check as a formal proof.
**Trade-offs:** Not composable or declarative — adding a property means writing a new Java class. Acceptable for 5 properties; reconsider at 20+.
**Sources:** ADR-0005 (temporal semantics section), CommitmentService state machine, MessageLedgerEntryRepository query methods
**Exploration:** quick
**Status:** captured

## D6: Offline verification first, runtime deferred

**Choice:** Scheduled job or MCP tool that sweeps the ledger and reports violations. Runtime monitoring deferred.
**Alternatives:**
- Both offline + runtime from the start — runtime checks advisory at dispatch time
- Runtime only — catches new violations immediately but misses historical
**Rationale:** Watchdog conditions (CIRCULAR_DELEGATION, OBLIGATION_FAN_OUT, CONVERSATION_STALL) already cover critical real-time cases. Property verification is an audit concern — it answers "did we maintain our invariants over this time period?" not "should this dispatch proceed?" Adding dispatch-time property checking risks latency and duplicates existing watchdog enforcement.
**Trade-offs:** Violations detected after the fact, not prevented. Acceptable for audit/compliance; runtime monitoring can be added later if needed.
**Sources:** WatchdogEvaluationService (existing real-time enforcement), ComplianceReportScheduler (existing scheduled generation)
**Exploration:** quick
**Status:** captured

## D7: Verification tool in compliance-report module

**Choice:** Add PropertyVerificationReport as a new report type in the existing compliance-report module.
**Alternatives:**
- New casehub-qhorus-verification module — cleaner separation but adds a Maven module
- Runtime module — always available but verification is audit, not dispatch
**Rationale:** The compliance-report module already queries MessageLedgerEntry, has REST+GraphQL+MCP, scheduled generation, and report renderers. Property verification reports are compliance evidence. Natural home — same infrastructure, same scheduled generation mechanism, same content negotiation.
**Trade-offs:** compliance-report module grows. But it's already the audit/reporting module — this is on-mission.
**Sources:** compliance-report module structure, ComplianceReportScheduler, existing report types (Attribution, Obligation, TrustHistory, Violation, Provenance, JudgmentAttribution, JudgmentFulfillment)
**Exploration:** quick
**Status:** captured

## D8: All five formal verification properties

**Choice:** Implement all five properties: Liveness, Safety, Fairness, Deadlock freedom, Evidence completeness.
**Alternatives:**
- Core four (skip Fairness) — avoids routing distribution analysis
- Just Liveness + Safety — minimal viable set
**Rationale:** Fairness analysis is feasible from qhorus's own ledger data — routing metadata columns (V2003: routing_selected_agent, routing_candidate_count) capture distribution. No eidos dependency needed for post-hoc fairness verification. Per-capability trust differentiation (routing SELECTION quality) is a separate eidos concern — file issue there.
**Fairness limitation (R1-07):** V2003 captures the selected agent and candidate COUNT per selection, but not the full candidate list. True fairness (comparing selections against the eligible pool) requires knowing WHICH agents were candidates, not just how many. The fairness predicate computes a selection frequency distribution (Gini coefficient over `routing_selected_agent` counts), normalized by `routing_candidate_count` as a pool-size proxy. This is approximate — flagging concentration in the winner column, not provably unfair distribution across the candidate field. The spec must document this limitation.
**Trade-offs:** Five properties means five predicate classes + tests. Each is small. Volume is manageable.
**Sources:** V2003 routing metadata migration, MessageLedgerEntryRepository, WatchdogConditionType.CIRCULAR_DELEGATION (#368)
**Exploration:** quick
**Cross-repo:** File eidos issue for per-capability trust differentiation in AgentSelector
**Status:** captured

## D9: Cross-repo dependency management

**Choice:** This branch's deliverables work independently of engine#998 (judgment EVENTs). The formal verification tool verifies properties over the existing commitment lifecycle — judgment EVENTs enrich the data but aren't required. The attestation enrichment observer is dormant until engine#998 dispatches VERIFIED events. #412 closure notes that the qhorus-side routing is covered by #401; engine-side integration (JudgmentScheduler → COMMAND dispatch) is tracked on engine#996.
**Alternatives:**
- Block on engine#998 — wait until judgment EVENTs exist before building the verification tool
- Implement engine-side integration in this branch — scope creep, wrong repo
**Rationale:** The verification tool operates on the existing commitment lifecycle (liveness, safety, deadlock freedom) and ledger routing data (fairness). These work today with zero judgment EVENTs. Evidence completeness (the fifth property) checks for judgment attestations — it returns "no judgment data" when engine#998 hasn't landed, not "violation." Attestation enrichment activates automatically when VERIFIED events start arriving.
**Trade-offs:** Evidence completeness property and attestation enrichment are dormant until engine#998. This is acceptable — they're ready, just waiting for data.
**Sources:** Engine#998 issue body, engine#996 issue body, #413 telemetry contract
**Review response:** R1-09 (premature #412 closure) addressed by noting engine-side work remains on engine#996. R1-14 (engine#998 dependency) addressed by specifying graceful degradation.
**Exploration:** quick
**Status:** captured

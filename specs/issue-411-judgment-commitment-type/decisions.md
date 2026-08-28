## D1: Compose with existing types — no new message types

**Choice:** Use COMMAND + DONE + EVENTs + attestation for governed yields instead of adding JUDGMENT_REQUEST/JUDGMENT_RESPONSE/JUDGMENT_ACCEPTANCE types.
**Alternatives:**
- New message types (JUDGMENT_REQUEST, JUDGMENT_RESPONSE, JUDGMENT_ACCEPTANCE) — violates ADR-0005 stopping criterion; all three occupy existing Searle-category cells
- PROPOSE-based pattern (inverted commitment direction) — wrong semantics; PROPOSE is sender-binding, judgment is receiver-obligation
**Rationale:** ADR-0005 stopping criterion: new types justified only when occupying a unique cell in the Searle-category × deontic-effect matrix. JUDGMENT_REQUEST = Directive (action) = COMMAND's cell. JUDGMENT_RESPONSE = Assertive = RESPONSE's cell. JUDGMENT_ACCEPTANCE = Declaration = DONE's cell. The judgment lifecycle composes cleanly with existing types: COMMAND creates the obligation, DONE fulfills it, EVENTs (#413 already landed) track judgment-specific metadata (YIELDED/RESPONDED/VERIFIED/ESCALATED), attestation validates quality.
**Trade-offs:** No type-level discrimination between regular COMMANDs and judgment COMMANDs — judgment semantics are in telemetry metadata, not the type system. LLMs don't get a dedicated type to classify; they use COMMAND with judgment-specific content.
**Sources:** ADR-0005 (stopping criterion matrix), PROPOSE spec (#395, taxonomy extension precedent), #413 spec (judgment compliance evidence — already landed with V2004 migration)
**Exploration:** quick
**Status:** captured

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

## D4: Attestation enrichment via MessageObserver

**Choice:** JudgmentVerificationObserver (MessageObserver, LOCAL scope) writes supplementary attestations when VERIFIED events land.
**Alternatives:**
- Scheduled sweep — batched, resilient to missed events, but delayed trust feedback
- Part of LedgerWriteService — couples ledger writing to attestation logic
**Rationale:** Reactive, immediate trust feedback. Uses existing dispatch infrastructure (MessageObserver pattern). Observer picks up events with tool_name matching JudgmentEventKinds.VERIFIED, finds the original commitment's ledger entry via correlationId, writes supplementary attestation with "system:judgment-verifier" as attestorId. Original attestation (SOUND/0.7 from DONE) stays — supplementary attestation adds verification-informed confidence. Trust computation aggregates both.
**Trade-offs:** If the observer misses an event (crash, restart), the supplementary attestation is never written. Mitigated by the offline verification tool (property: evidence completeness) which detects missing attestations.
**Sources:** MessageObserver SPI (api/gateway/), StoredCommitmentAttestationPolicy (runtime/audit/), JudgmentEventKinds (#413 contract)
**Exploration:** quick
**Status:** captured

## D5: Java predicates for formal verification

**Choice:** Document properties in CTL/LTL notation. Implement each as a Java predicate class that queries the ledger and returns violations.
**Alternatives:**
- Property DSL (enum-based language compiling to ledger queries) — adds parser/compiler for marginal benefit
- Full CTL model checker library — massive dependency for ~5 properties
**Rationale:** Simple, testable, no dependencies. Each property is a class with a `check(tenancyId, from, to)` method that returns `List<PropertyViolation>`. Ledger queries already exist or are straightforward to add. Properties documented in formal CTL/LTL notation in the spec for academic rigor.
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
**Trade-offs:** Five properties means five predicate classes + tests. Each is small. Volume is manageable.
**Sources:** V2003 routing metadata migration, MessageLedgerEntryRepository, WatchdogConditionType.CIRCULAR_DELEGATION (#368)
**Exploration:** quick
**Cross-repo:** File eidos issue for per-capability trust differentiation in AgentSelector
**Status:** captured

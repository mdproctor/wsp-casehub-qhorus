# Governed Yield Governance — Design Spec

**Issues:** casehubio/qhorus#411 (primary), #412, #414
**Parent:** casehubio/qhorus#410 (governed yield governance epic)
**Cross-repo:** casehubio/engine#994, #996, #998; casehubio/eidos (per-capability trust — to be filed)
**Date:** 2026-08-28
**Status:** Draft
**Decisions:** [decisions.md](decisions.md)

---

## Summary

Governed yields — the engine yielding judgment requests to callers, callers responding with evidence, the engine verifying — compose entirely with qhorus's existing 10-type speech-act taxonomy. No new message types are added (D1). The ADR-0005 stopping criterion holds: PROPOSE was the last justified type addition.

This spec delivers three capabilities:
1. **Judgment attestation feedback** — deferred attestation policy + verification observer that feeds judgment outcomes into trust scores (#411)
2. **Formal verification properties** — five CTL/LTL-documented, Java-predicate-implemented properties verified offline against the ledger (#414)
3. **Issue closures** — #412 closed as covered by RoutingBridge (#401); eidos issue filed for per-capability trust differentiation (#412)

---

## Composition Pattern

The governed yield lifecycle maps to existing qhorus primitives:

```
Engine                         Reviewer (caller)              Qhorus infrastructure
  │                               │                               │
  ├─── COMMAND ──────────────────►│                               │
  │    target: "role:reviewer"    │                    RoutingBridge resolves to best agent
  │    content: "Review this"     │                    CommitmentService.open() → OPEN
  │                               │                               │
  │                               ├─── DONE ─────────────────────►│
  │                               │    content: "Here's my        │
  │                               │    review with evidence"      CommitmentService.fulfill() → FULFILLED
  │                               │                               StoredCommitmentAttestationPolicy → DEFERRED
  │                               │                               │
  ├─── EVENT (judgment_verified) ─┼──────────────────────────────►│
  │    verification_outcome:      │                    JudgmentVerificationObserver
  │    "ACCEPTED"                 │                    writes attestation: SOUND/0.85
  │    evidence_quality: 0.85     │                    Trust score updated
```

### Why COMMAND, not PROPOSE

Judgment requests are directives — the engine is telling an agent to act, not making a conditional offer. COMMAND is the correct Searle classification (D1). PROPOSE was considered (same commitment direction as COMMAND, non-fulfilling RESPONSE) but rejected because it's a commissive — using it would misclassify the speech act, undermining LLM classification accuracy.

### Dual-lifecycle by design

The commitment lifecycle (COMMAND → DONE → FULFILLED) and the judgment lifecycle (YIELDED → RESPONDED → VERIFIED) are deliberately separate. They track orthogonal assessments:

- **Obligation fulfillment** — "Did the reviewer provide a review?" → Commitment: FULFILLED
- **Quality assessment** — "Was the review adequate?" → Attestation: SOUND or FLAGGED

A reviewer who delivers a poor-quality review has still fulfilled their obligation. The quality assessment affects their trust score, not their obligation state. If the review is rejected, the engine issues a NEW COMMAND for remediation — it does not reopen the fulfilled commitment.

---

## Part 1: Judgment Attestation Feedback (#411)

### JudgmentCommitmentAttestationPolicy

Extends `StoredCommitmentAttestationPolicy` to defer attestation for judgment commitments. When a COMMAND's DONE resolution is for a judgment (determined by checking whether a matching YIELDED event exists for the same `correlationId`), the policy returns `Optional.empty()` — the existing SPI deferral mechanism. `LedgerWriteService.writeAttestation()` already skips attestation when the policy returns empty.

FAILURE terminal type is NOT deferred — expired judgments (deadline-triggered FAILURE) must get standard FLAGGED attestation to penalize agents who let judgments expire.

```java
@ApplicationScoped
public class JudgmentCommitmentAttestationPolicy extends StoredCommitmentAttestationPolicy {

    @Inject MessageLedgerEntryRepository messageRepo;
    @Inject CurrentPrincipal currentPrincipal;

    @Override
    public Optional<AttestationOutcome> attestationFor(MessageType terminalType,
            String resolvedActorId, CommitmentContext ctx) {
        if (terminalType == MessageType.DONE && isJudgmentCommitment(ctx)) {
            return Optional.empty(); // deferred — JudgmentVerificationObserver writes attestation later
        }
        // non-judgment commitments + FAILURE/DECLINE: delegate to parent (standard attestation)
        return super.attestationFor(terminalType, resolvedActorId, ctx);
    }

    private boolean isJudgmentCommitment(CommitmentContext ctx) {
        if (ctx.correlationId() == null) return false;
        String tenancyId = currentPrincipal.tenancyId();
        return messageRepo.hasJudgmentEvent(ctx.correlationId(), tenancyId);
    }
}
```

**CDI resolution:** Extends `StoredCommitmentAttestationPolicy` (the `@DefaultBean`). CDI selects the subclass over the default — the `super` call for non-judgment commitments preserves the parent's configurable confidence values. No `@Alternative` or priority needed.

**Activation:** This policy lives in the compliance-report module. Activates by classpath presence. Without the module, the parent `StoredCommitmentAttestationPolicy` applies — judgment COMMANDs get standard DONE attestation.

**Escalated judgments:** ESCALATED is not a commitment terminal type — it's a judgment EVENT. The commitment stays OPEN after escalation (the engine will open a new COMMAND to the escalated-to agent). No attestation involved.

**Expired judgments:** Deadline-triggered FAILURE is handled by the parent class (FLAGGED/0.6). The judgment policy does NOT defer FAILURE — agents who let judgments expire receive the standard trust penalty.

### JudgmentVerificationObserver

A `MessageObserver` (LOCAL scope) that writes the sole attestation when a VERIFIED event lands. Since `MessageReceivedEvent` does not carry telemetry columns (and EVENT `content` is enforced null), the observer queries `MessageLedgerEntryRepository` for the VERIFIED entry's extracted columns.

```java
@ApplicationScoped
public class JudgmentVerificationObserver implements MessageObserver {

    @Inject LedgerEntryRepository ledger;
    @Inject MessageLedgerEntryRepository messageRepo;
    @Inject CurrentPrincipal currentPrincipal;
    @Inject QhorusConfig config;

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (event.messageType() != MessageType.EVENT) return;
        if (event.messageId() == null) return;

        String tenancyId = event.tenancyId();

        // Query the persisted ledger entry for telemetry columns
        var verifiedEntry = messageRepo.findByMessageId(event.messageId());
        if (verifiedEntry.isEmpty()) return;
        var entry = verifiedEntry.get();
        if (!JudgmentEventKinds.VERIFIED.equals(entry.toolName)) return;

        // Find the originating COMMAND entry (attestation target)
        var commandEntry = messageRepo.findLatestByCorrelationId(
                entry.channelId, event.correlationId(), tenancyId);
        if (commandEntry.isEmpty()) return;

        var target = commandEntry.get();
        var verdict = mapVerdict(entry.verificationOutcome);
        var confidence = mapConfidence(entry.verificationOutcome, entry.evidenceQuality);

        final LedgerAttestation attestation = new LedgerAttestation();
        attestation.ledgerEntryId = target.id;
        attestation.subjectId = target.subjectId;
        attestation.attestorId = "system:judgment-verifier";
        attestation.attestorType = ActorType.SYSTEM;
        attestation.verdict = verdict;
        attestation.confidence = confidence;
        attestation.capabilityTag = CapabilityTag.GLOBAL;
        ledger.saveAttestation(attestation, tenancyId);
    }

    private AttestationVerdict mapVerdict(String verificationOutcome) {
        if ("ACCEPTED".equals(verificationOutcome)) return AttestationVerdict.SOUND;
        return AttestationVerdict.FLAGGED;
    }

    private double mapConfidence(String verificationOutcome, Double evidenceQuality) {
        return switch (verificationOutcome != null ? verificationOutcome : "") {
            case "ACCEPTED" -> Math.max(
                    config.attestation().judgmentAcceptedConfidence(),
                    evidenceQuality != null ? evidenceQuality : 0.7);
            case "REJECTED" -> config.attestation().judgmentRejectedConfidence();
            case "PARTIAL" -> config.attestation().judgmentPartialConfidence();
            default -> config.attestation().judgmentRejectedConfidence();
        };
    }
}
```

**Configuration (follows existing `QhorusConfig.Attestation` pattern):**
- `casehub.qhorus.attestation.judgment-accepted-confidence` (default 0.7)
- `casehub.qhorus.attestation.judgment-rejected-confidence` (default 0.3)
- `casehub.qhorus.attestation.judgment-partial-confidence` (default 0.5)

**Confidence mapping:**
- ACCEPTED: `max(judgmentAcceptedConfidence, evidenceQuality)` — at least the configured floor, up to 1.0 for perfect evidence
- REJECTED: `judgmentRejectedConfidence` (default 0.3) — low, worse than standard DECLINE (0.4)
- PARTIAL: `judgmentPartialConfidence` (default 0.5) — intermediate, worse than standard FAILURE (0.6)

**Recovery path:** The offline verification tool (Part 2, evidence completeness property) detects missing attestations. Remediation is a separate `remediate()` method (not inside `check()`) that writes missing attestations using the same mapping.

### New repository method

`MessageLedgerEntryRepository.hasJudgmentEvent(String correlationId, String tenancyId)`:

```java
public boolean hasJudgmentEvent(String correlationId, String tenancyId) {
    return getEntityManager().createQuery(
            "SELECT COUNT(e) FROM MessageLedgerEntry e " +
            "WHERE e.correlationId = :corrId " +
            "AND e.tenancyId = :tenancyId " +
            "AND e.toolName = :yieldedKind", Long.class)
        .setParameter("corrId", correlationId)
        .setParameter("tenancyId", tenancyId != null ? tenancyId : CurrentPrincipal.DEFAULT_TENANT_ID)
        .setParameter("yieldedKind", JudgmentEventKinds.YIELDED)
        .getSingleResult() > 0;
}
```

### No Flyway migration

No schema changes. The observer writes standard attestations to the existing `ledger_attestation` table. The `Optional.empty()` deferral mechanism is pure Java — no DB representation.

---

## Part 2: Formal Verification Properties (#414)

### Property interface

```java
public interface VerificationProperty {
    String name();
    String ctlFormula();
    String description();
    CheckResult check(String tenancyId, Instant from, Instant to);
}

public record CheckResult(
    List<PropertyViolation> violations,
    int remediationsAvailable
) {}

public record PropertyViolation(
    String propertyName,
    String description,
    String evidence,
    Instant occurredAt,
    String severity    // HIGH, MEDIUM, LOW
) {}
```

Properties that support remediation (e.g., evidence completeness) also implement `RemediatingProperty`:

```java
public interface RemediatingProperty extends VerificationProperty {
    int remediate(String tenancyId, Instant from, Instant to);
}
```

`remediate()` is a separate method, never called inside `check()`. The `PropertyVerificationService` calls `check()` first, then optionally calls `remediate()` on properties that implement it. This keeps `check()` idempotent and side-effect-free.

**Semantic gap:** CTL/LTL notation documents INTENT — system properties over all paths and all futures. Java predicates are TRACE CHECKERS — they verify "no violations observed in this time window." A passing check means the observed history satisfies the property, not that the system provably maintains it. The spec is explicit about this distinction (D5).

### Five properties

#### 1. Liveness

**CTL:** `AG(OPEN → AF(FULFILLED ∨ DECLINED ∨ FAILED ∨ DELEGATED ∨ EXPIRED))`

**Predicate:** Query commitments in OPEN state with `createdAt < (now - threshold)`. Default threshold from `casehub.qhorus.verification.liveness-threshold` (PT24H). Returns violations for commitments that have been OPEN longer than the threshold without resolution.

```java
public class LivenessProperty implements VerificationProperty {
    // Queries: CommitmentReader.findOpenOlderThan(Instant cutoff, String tenancyId)
    // New method on CommitmentReader (read interface):
    //   List<Commitment> findOpenOlderThan(Instant cutoff, String tenancyId)
    // Implementations: JpaCommitmentStore, InMemoryCommitmentStore, CrossTenantCommitmentStore
}
```

#### 2. Safety (attestation completeness)

**CTL:** `AG(FULFILLED → attestation_exists)`

**Predicate:** Query ledger entries for DONE messages that have no attestation. For judgment commitments (those with matching YIELDED events), also check that the attestation was written by `system:judgment-verifier` (not deferred forever).

```java
public class SafetyProperty implements VerificationProperty {
    // Queries: findDoneEntriesWithoutAttestation(Instant from, Instant to, String tenancyId)
    // New repository method — LEFT JOIN ledger_attestation, WHERE attestation IS NULL
}
```

#### 3. Deadlock Freedom

**CTL:** `¬EF(∃ correlationId: delegation_chain_contains_cycle)`

**Predicate:** Reuse the existing `CIRCULAR_DELEGATION` watchdog logic. Query all HANDOFF chains within the time window and check for repeated obligors.

```java
public class DeadlockFreedomProperty implements VerificationProperty {
    // Queries: CommitmentStore.findAllByCorrelationId(correlationId)
    // Existing method — same logic as WatchdogEvaluationService.evaluateCircularDelegation()
    // Property sweeps ALL correlationIds in the time window, not just watched ones
}
```

#### 4. Fairness (routing distribution)

**CTL:** `AG(routing_entropy > threshold)` (informally — not strictly CTL-expressible)

**Predicate:** Query routing metadata from `MessageLedgerEntry` within the time window. Compute selection frequency per agent. Flag concentration: if any single agent receives >N% of selections when `routing_candidate_count > 1`.

**Limitation (D8/R1-07):** V2003 captures the selected agent and candidate COUNT, not the full candidate list. The fairness metric is a Gini coefficient over selection frequency, using candidate count as a pool-size proxy. This detects concentration in the winner column but cannot prove unfair distribution across the candidate field.

```java
public class FairnessProperty implements VerificationProperty {
    // Queries: MessageLedgerEntryRepository.findRoutingEntries(Instant from, Instant to, String tenancyId)
    // New method — WHERE routing_selected_agent IS NOT NULL
    // Config: casehub.qhorus.verification.fairness-threshold (default 0.7 Gini)
}
```

#### 5. Evidence Completeness (judgment-specific)

**CTL:** `AG(YIELDED → AF(VERIFIED ∨ ESCALATED))`

**Predicate:** Query YIELDED judgment events without a corresponding VERIFIED or ESCALATED event for the same `judgment_id`. Also check for deferred attestations that were never written (DONE entry with no attestation and a matching VERIFIED event).

**Remediation (separate method):** Implements `RemediatingProperty`. The `remediate()` method writes missing attestations using the same mapping as `JudgmentVerificationObserver`. Called by `PropertyVerificationService` after `check()`, not inside it.

```java
public class EvidenceCompletenessProperty implements RemediatingProperty {
    // check(): Queries:
    //   MessageLedgerEntryRepository.findPendingJudgments(tenancyId) (existing from #413)
    //   + findDoneEntriesWithDeferredAttestation(tenancyId) (new — DONE + VERIFIED exists + no attestation)
    // remediate(): writes missing attestations via LedgerEntryRepository.saveAttestation()
    //   Uses same confidence mapping as JudgmentVerificationObserver
}
```

### PropertyVerificationReport

New report type in `ReportType` enum:

```java
PROPERTY_VERIFICATION
```

Report record:

```java
public record PropertyVerificationReport(
    Instant from,
    Instant to,
    List<PropertyResult> results,
    int totalProperties,
    int passed,
    int violated,
    int remediationsApplied,
    String merkleRoot,
    Instant generatedAt,
    int schemaVersion
) {}

public record PropertyResult(
    String propertyName,
    String ctlFormula,
    boolean passed,
    int violationCount,
    int remediationsApplied,
    List<PropertyViolation> violations
) {}
```

### PropertyVerificationService

```java
@ApplicationScoped
public class PropertyVerificationService {

    @Inject Instance<VerificationProperty> properties;

    public PropertyVerificationReport verify(String tenancyId, Instant from, Instant to) {
        List<PropertyResult> results = new ArrayList<>();
        for (var handle : properties.handles()) {
            try {
                var property = handle.get();
                var violations = property.check(tenancyId, from, to);
                results.add(new PropertyResult(
                        property.name(), property.ctlFormula(),
                        violations.isEmpty(), violations.size(), 0, violations));
            } finally {
                handle.close();
            }
        }
        // ... build report with merkle root, counts
    }
}
```

### REST + GraphQL + MCP

**REST:** `GET /api/compliance/property-verification?from=...&to=...`
Content negotiation via `Accept` header (JSON/CSV/HTML).

**GraphQL:** `compliancePropertyVerification(from: String!, to: String!): PropertyVerificationReportType`

**Scheduled generation:** `PROPERTY_VERIFICATION` is schedulable. Uses `lastRunAt → now` as the `[from, to]` window.

---

## Part 3: Issue Closures (#412)

### #412 — Close as covered

RoutingBridge (#401) already handles judgment caller routing:
- COMMAND with `target: "role:reviewer"` → RoutingBridge resolves to best agent
- Per-channel `routingTrustThreshold` gates minimum quality
- Routing decision recorded in ledger (V2003 metadata)
- `get_routing_candidates` MCP tool shows diagnostic info

The engine-side integration (JudgmentScheduler dispatching COMMANDs with `role:` targets) is tracked on engine#996. Close #412 with a note referencing #401 and engine#996.

### Cross-repo issues to file

1. **Eidos:** Per-capability trust differentiation in AgentSelector — an agent trusted for `code_review` may not be trusted for `legal_review`. Currently trust scores are per-actor, not per-capability.
2. **Engine#996:** JudgmentScheduler → COMMAND dispatch integration with RoutingBridge (engine-side wiring).

---

## Files Changed / Created

### New files (compliance-report module)

| File | What |
|---|---|
| `compliance-report/.../verification/VerificationProperty.java` | Property interface |
| `compliance-report/.../verification/PropertyViolation.java` | Violation record |
| `compliance-report/.../verification/LivenessProperty.java` | Commitment liveness check |
| `compliance-report/.../verification/SafetyProperty.java` | Attestation completeness check |
| `compliance-report/.../verification/DeadlockFreedomProperty.java` | Circular delegation check |
| `compliance-report/.../verification/FairnessProperty.java` | Routing distribution check |
| `compliance-report/.../verification/EvidenceCompletenessProperty.java` | Judgment evidence check + remediation |
| `compliance-report/.../verification/PropertyVerificationService.java` | Orchestrates all properties |
| `compliance-report/.../model/PropertyVerificationReport.java` | Report record |
| `compliance-report/.../model/PropertyResult.java` | Per-property result record |
| `compliance-report/.../graphql/dto/PropertyVerificationReportType.java` | GraphQL DTO |
| `compliance-report/.../graphql/dto/PropertyResultType.java` | GraphQL DTO |
| `compliance-report/.../graphql/dto/PropertyViolationType.java` | GraphQL DTO |

### New files (compliance-report module — attestation)

| File | What |
|---|---|
| `compliance-report/.../attestation/JudgmentCommitmentAttestationPolicy.java` | SPI override: defers attestation for judgment commitments. Lives in compliance-report because it queries `MessageLedgerEntryRepository` for judgment events. Activates by classpath presence. |
| `compliance-report/.../attestation/JudgmentVerificationObserver.java` | MessageObserver: writes attestation on VERIFIED events. Co-located with the policy it complements. |

### Modified files

| File | Change |
|---|---|
| `runtime/.../ledger/MessageLedgerEntryRepository.java` | `hasJudgmentEvent()`, `findRoutingEntries()`, `findDoneEntriesWithoutAttestation()`, `findDoneEntriesWithDeferredAttestation()` |
| `runtime/.../config/QhorusConfig.java` | Add `Attestation.judgmentAcceptedConfidence`, `judgmentRejectedConfidence`, `judgmentPartialConfidence` |
| `api/.../store/CommitmentStore.java` (or `CommitmentReader`) | Add `findOpenOlderThan(Instant, String)` |
| `runtime/.../message/Commitment*Store.java` (JPA + InMemory + CrossTenant) | Implement `findOpenOlderThan` |
| `compliance-report/.../model/ReportType.java` | Add `PROPERTY_VERIFICATION` |
| `compliance-report/.../schedule/ComplianceReportScheduler.java` | Add `PROPERTY_VERIFICATION` case |
| `compliance-report/.../api/ComplianceReportResource.java` | Property verification endpoint |
| `compliance-report/.../graphql/ComplianceQueryResolver.java` | Property verification query |
| `compliance-report/.../format/CsvReportRenderer.java` | Property verification CSV |
| `compliance-report/.../format/HtmlReportRenderer.java` | Property verification HTML |

### No Flyway migration (entire spec)

No schema changes anywhere. All data is in existing tables (`ledger_attestation`, `message_ledger_entry`, `commitment`). Verification properties query existing columns. Attestation enrichment writes to the existing `ledger_attestation` table.

---

## Testing Strategy

| Component | Test type | Notes |
|-----------|----------|-------|
| `JudgmentCommitmentAttestationPolicy` | CDI-free unit tests | Mock `MessageLedgerEntryRepository`. Verify: defers for judgment commitments, delegates for non-judgment. |
| `JudgmentVerificationObserver` | CDI-free unit tests | Mock `LedgerEntryRepository`, `MessageLedgerEntryRepository`. Verify: ACCEPTED→SOUND, REJECTED→FLAGGED, non-VERIFIED events ignored. |
| `LivenessProperty` | CDI-free unit tests | Mock `CommitmentStore`. Verify: OPEN commitments older than threshold are violations. |
| `SafetyProperty` | CDI-free unit tests | Mock repository. Verify: DONE entries without attestation are violations. |
| `DeadlockFreedomProperty` | CDI-free unit tests | Mock `CommitmentStore`. Verify: circular chains detected. |
| `FairnessProperty` | CDI-free unit tests | Mock repository. Verify: concentrated routing flagged, uniform routing passes. |
| `EvidenceCompletenessProperty` | CDI-free unit tests | Mock repository + `LedgerEntryRepository`. Verify: pending judgments detected, missing attestations remediated. |
| `PropertyVerificationService` | CDI-free unit tests | Mock `Instance<VerificationProperty>`. Verify: aggregation, counts, report assembly. |
| Integration: attestation policy + observer | `@QuarkusTest` with `QuarkusTransaction.requiringNew()` | Dispatch COMMAND, DONE (verify no attestation via Optional.empty()), then VERIFIED EVENT (verify attestation written by observer). Observer fires after commit — must use requiringNew, not @TestTransaction. |
| Integration: property verification report | `@QuarkusTest` | Generate report via REST/GraphQL. Verify JSON structure and content negotiation. |
| `EvidenceCompletenessProperty.remediate()` | CDI-free unit tests | Mock repository. Verify: missing attestations written with correct verdict/confidence. Verify: `check()` is idempotent (no side effects). |

---

## Cross-repo Dependencies

| Dependency | Status | Impact if delayed |
|---|---|---|
| Engine#998 (judgment EVENTs) | Not landed | Attestation enrichment dormant. Formal verification works on existing commitment data. Evidence completeness returns "no judgment data." |
| Engine#996 (JudgmentScheduler) | Not landed | No judgment COMMANDs dispatched. RoutingBridge ready but unused for judgments. |
| Eidos per-capability trust | Issue to file | Standalone deployments get uniform trust thresholds. Not a functional blocker. |

---

## References

- ADR-0005 (`docs/adr/0005-message-type-taxonomy-theoretical-foundation.md`) — stopping criterion, Searle matrix, completeness argument
- PROPOSE spec (`docs/specs/issue-395-propose-message-type/2026-08-14-propose-message-type-design.md`) — commitment direction clarification, taxonomy extension precedent
- Reputation routing spec (`docs/specs/issue-401-reputation-routing/2026-08-25-reputation-routing-design.md`) — RoutingBridge architecture, #412 coverage
- Judgment compliance evidence spec (`docs/specs/issue-413-judgment-compliance-evidence/2026-08-27-judgment-compliance-evidence-design.md`) — telemetry contract, V2004 migration, JudgmentEventKinds
- `CommitmentAttestationPolicy` SPI (`api/src/main/java/io/casehub/qhorus/api/spi/CommitmentAttestationPolicy.java`)
- `StoredCommitmentAttestationPolicy` (`runtime/src/main/java/io/casehub/qhorus/runtime/audit/StoredCommitmentAttestationPolicy.java`)
- `MessageObserver` SPI (`api/src/main/java/io/casehub/qhorus/api/gateway/MessageObserver.java`)
- `MessageService.dispatch()` (`runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`)
- `CommitmentService` (`runtime/src/main/java/io/casehub/qhorus/runtime/message/CommitmentService.java`)
- `RoutingBridge` (`runtime/src/main/java/io/casehub/qhorus/runtime/message/RoutingBridge.java`)
- `WatchdogEvaluationService` — CIRCULAR_DELEGATION logic
- `ComplianceReportScheduler` — existing scheduled generation mechanism
- `ReportType` enum — existing 7 report types
- Epic #410 body — governed yield governance vision
- decisions.md (D1-D9) — all captured design decisions with review responses

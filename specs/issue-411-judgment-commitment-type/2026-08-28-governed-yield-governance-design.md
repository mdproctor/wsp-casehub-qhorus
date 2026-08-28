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

A `CommitmentAttestationPolicy` SPI override that defers attestation for judgment commitments. When a COMMAND's DONE resolution is for a judgment (determined by checking whether a matching YIELDED event exists for the same `correlationId`), the policy returns a deferred outcome — no attestation is written at DONE time.

```java
@ApplicationScoped
public class JudgmentCommitmentAttestationPolicy implements CommitmentAttestationPolicy {

    @Inject MessageLedgerEntryRepository messageRepo;
    @Inject @Any Instance<CommitmentAttestationPolicy> delegates;

    @Override
    public AttestationOutcome attestationFor(MessageType terminalType,
            String content, CommitmentContext ctx) {
        if (terminalType == MessageType.DONE && isJudgmentCommitment(ctx)) {
            return AttestationOutcome.DEFERRED;
        }
        // delegate to StoredCommitmentAttestationPolicy for non-judgment commitments
        return resolveDefault().attestationFor(terminalType, content, ctx);
    }

    private boolean isJudgmentCommitment(CommitmentContext ctx) {
        // Check if a YIELDED event exists for this correlationId
        return ctx.correlationId() != null
                && messageRepo.hasJudgmentEvent(ctx.correlationId(), tenancyId);
    }
}
```

**Activation:** This policy activates by classpath presence when the compliance-report module is included. Without the module, the default `StoredCommitmentAttestationPolicy` applies — judgment COMMANDs get standard DONE attestation.

**AttestationOutcome.DEFERRED:** New value in the `AttestationOutcome` enum. `LedgerWriteService.writeAttestation()` skips attestation when outcome is DEFERRED. No other code path reads this value — it's a sentinel for "don't write now, something else will."

### JudgmentVerificationObserver

A `MessageObserver` (LOCAL scope) that writes the sole attestation when a VERIFIED event lands.

```java
@ApplicationScoped
public class JudgmentVerificationObserver implements MessageObserver {

    @Inject LedgerEntryRepository ledger;
    @Inject MessageLedgerEntryRepository messageRepo;

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (!JudgmentEventKinds.VERIFIED.equals(event.toolName())) return;

        // Find the DONE entry for this judgment's correlationId
        var yieldedEntry = messageRepo.findEarliestWithSubjectByCorrelationId(
                event.correlationId(), event.tenancyId());
        if (yieldedEntry.isEmpty()) return;

        // Find the terminal DONE entry
        var doneEntry = messageRepo.findLatestByCorrelationId(
                event.correlationId(), event.tenancyId());
        if (doneEntry.isEmpty()) return;

        var entry = doneEntry.get();
        var verdict = mapVerdict(event);
        var confidence = mapConfidence(event);

        var attestation = LedgerAttestation.builder()
                .entryId(entry.id)
                .attestorId("system:judgment-verifier")
                .verdict(verdict)
                .confidence(confidence)
                .evidence("Judgment verification: " + event.verificationOutcome())
                .build();
        ledger.saveAttestation(attestation);
    }

    private LedgerVerdict mapVerdict(MessageReceivedEvent event) {
        return switch (extractVerificationOutcome(event)) {
            case "ACCEPTED" -> LedgerVerdict.SOUND;
            case "REJECTED" -> LedgerVerdict.FLAGGED;
            case "PARTIAL" -> LedgerVerdict.FLAGGED;
            default -> LedgerVerdict.FLAGGED;
        };
    }

    private double mapConfidence(MessageReceivedEvent event) {
        return switch (extractVerificationOutcome(event)) {
            case "ACCEPTED" -> Math.max(0.7, extractEvidenceQuality(event));
            case "REJECTED" -> 0.3;
            case "PARTIAL" -> 0.5;
            default -> 0.4;
        };
    }
}
```

**Confidence mapping:**
- ACCEPTED: `max(0.7, evidenceQuality)` — at least as confident as a standard DONE, up to 1.0 for perfect evidence
- REJECTED: 0.3 — low confidence, worse than standard DECLINE (0.4)
- PARTIAL: 0.5 — intermediate, worse than standard FAILURE (0.6)

**Recovery path:** The offline verification tool (Part 2, evidence completeness property) detects missing attestations and writes them as remediation. Query: find DONE ledger entries with matching VERIFIED events but no attestation on the DONE entry.

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

No schema changes. `AttestationOutcome.DEFERRED` is a Java enum addition — not stored in DB. The observer writes standard attestations to the existing `ledger_attestation` table.

---

## Part 2: Formal Verification Properties (#414)

### Property interface

```java
public interface VerificationProperty {
    String name();
    String ctlFormula();
    String description();
    List<PropertyViolation> check(String tenancyId, Instant from, Instant to);
}

public record PropertyViolation(
    String propertyName,
    String description,
    String evidence,
    Instant occurredAt,
    String severity    // HIGH, MEDIUM, LOW
) {}
```

**Semantic gap:** CTL/LTL notation documents INTENT — system properties over all paths and all futures. Java predicates are TRACE CHECKERS — they verify "no violations observed in this time window." A passing check means the observed history satisfies the property, not that the system provably maintains it. The spec is explicit about this distinction (D5).

### Five properties

#### 1. Liveness

**CTL:** `AG(OPEN → AF(FULFILLED ∨ DECLINED ∨ FAILED ∨ DELEGATED ∨ EXPIRED))`

**Predicate:** Query commitments in OPEN state with `createdAt < (now - threshold)`. Default threshold from `casehub.qhorus.verification.liveness-threshold` (PT24H). Returns violations for commitments that have been OPEN longer than the threshold without resolution.

```java
public class LivenessProperty implements VerificationProperty {
    // Queries: CommitmentStore.findOpenOlderThan(Instant cutoff, String tenancyId)
    // New method — returns OPEN commitments with createdAt < cutoff
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

**Remediation:** When violations of the missing-attestation kind are found, the property checker writes the missing attestation using the same mapping as `JudgmentVerificationObserver`. This is the recovery path specified in D4.

```java
public class EvidenceCompletenessProperty implements VerificationProperty {
    // Queries: MessageLedgerEntryRepository.findPendingJudgments(tenancyId) (existing from #413)
    // + findDoneEntriesWithDeferredAttestation(tenancyId) (new — DONE + VERIFIED exists + no attestation)
    // Remediation: writes missing attestations via LedgerEntryRepository.saveAttestation()
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
| `api/.../spi/CommitmentAttestationPolicy.java` | Add `DEFERRED` to `AttestationOutcome` enum |
| `runtime/.../ledger/LedgerWriteService.java` | Skip attestation write when outcome is DEFERRED |
| `runtime/.../ledger/MessageLedgerEntryRepository.java` | `hasJudgmentEvent()`, `findRoutingEntries()`, `findDoneEntriesWithoutAttestation()` |
| `compliance-report/.../model/ReportType.java` | Add `PROPERTY_VERIFICATION` |
| `compliance-report/.../schedule/ComplianceReportScheduler.java` | Add `PROPERTY_VERIFICATION` case |
| `compliance-report/.../api/ComplianceReportResource.java` | Property verification endpoint |
| `compliance-report/.../graphql/ComplianceQueryResolver.java` | Property verification query |
| `compliance-report/.../format/CsvReportRenderer.java` | Property verification CSV |
| `compliance-report/.../format/HtmlReportRenderer.java` | Property verification HTML |

### No Flyway migration

No schema changes. All data is in existing tables (`ledger_attestation`, `message_ledger_entry`, `commitment`).

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
| Integration: attestation policy + observer | `@QuarkusTest @TestTransaction` | Dispatch COMMAND, DONE, then VERIFIED EVENT. Verify: no attestation after DONE, attestation written after VERIFIED. |
| Integration: property verification report | `@QuarkusTest` | Generate report via REST/GraphQL. Verify JSON structure and content negotiation. |
| `AttestationOutcome.DEFERRED` | CDI-free unit tests | Verify `LedgerWriteService` skips attestation write. |

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

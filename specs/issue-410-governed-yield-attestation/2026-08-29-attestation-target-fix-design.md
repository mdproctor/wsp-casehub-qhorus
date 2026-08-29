# Attestation Target Fix — Design Spec

**Issue:** casehubio/qhorus#410
**Parent spec:** `docs/specs/issue-411-judgment-commitment-type/2026-08-28-governed-yield-governance-design.md`
**Date:** 2026-08-29
**Status:** Draft
**Decisions:** [decisions.md](decisions.md)

---

## Summary

The `JudgmentVerificationObserver` writes a single attestation on the COMMAND entry when a VERIFIED event lands. The normal `LedgerWriteService.writeAttestation()` flow writes TWO attestations — one on the COMMAND entry (requester trust) and one on the terminal DONE entry (obligor/reviewer trust). The observer's single-attestation pattern means reviewer trust is never updated by judgment outcomes, SafetyProperty reports false positives for every judgment DONE, and EvidenceCompletenessProperty's remediation path disagrees with the observer on which entry to target.

This spec fixes all three gaps by aligning the observer and remediation with the established dual-attestation pattern.

---

## Root Cause

`TrustScoreCalculator.computeAll()` (casehub-ledger) receives entries for a specific actor and attestations grouped by entry ID. Trust for actor X is computed from attestations on entries WHERE `actorId = X`:

- Attestation on COMMAND entry (actorId = engine/requester) → feeds engine's trust
- Attestation on DONE entry (actorId = reviewer/obligor) → feeds reviewer's trust

The observer (line 48) writes `attestation.ledgerEntryId = target.id` where `target` is the COMMAND entry from `findLatestByCorrelationId`. Only the engine's trust is affected. The reviewer — whose quality the judgment is assessing — never gets a trust update.

---

## Changes

### 1. New query: `findTerminalEntryByCorrelationId` (D2)

Add to `MessageLedgerEntryRepository`:

```java
public Optional<MessageLedgerEntry> findTerminalEntryByCorrelationId(
        final UUID channelId, final String correlationId, final String tenancyId) {
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e " +
                    "WHERE e.subjectId = :sid AND e.correlationId = :corr " +
                    "AND e.tenancyId = :tid " +
                    "AND e.messageType IN ('DONE', 'FAILURE', 'DECLINE') " +
                    "ORDER BY e.sequenceNumber DESC",
            MessageLedgerEntry.class)
            .setParameter("sid", channelId)
            .setParameter("corr", correlationId)
            .setParameter("tid", tenancyId(tenancyId))
            .setMaxResults(1)
            .getResultStream()
            .findFirst();
}
```

Mirrors `findLatestByCorrelationId` (line 325) which filters to COMMAND/HANDOFF. Symmetric: one finds the initiating entry, the other finds the terminal entry.

### 2. `JudgmentVerificationObserver` — dual attestation (D1)

Replace the single attestation write with the dual-attestation pattern from `LedgerWriteService.writeAttestation()` (lines 302-330).

**Write order:** COMMAND entry first, DONE entry second. If the observer crashes between the two, the DONE entry remains unattested — `findDoneEntriesWithDeferredAttestation` detects the gap and remediation recovers (D4).

```java
@Override
public void onMessage(MessageReceivedEvent event) {
    if (event.messageType() != MessageType.EVENT) return;
    if (event.messageId() == null) return;

    var verifiedEntry = messageRepo.findByMessageId(event.messageId());
    if (verifiedEntry.isEmpty()) return;
    MessageLedgerEntry entry = verifiedEntry.get();
    if (!JudgmentEventKinds.VERIFIED.equals(entry.toolName)) return;

    String tenancyId = event.tenancyId();

    // Find the COMMAND entry (requester attestation target)
    var commandEntry = messageRepo.findLatestByCorrelationId(
            entry.channelId, event.correlationId(), tenancyId);
    if (commandEntry.isEmpty()) return;
    MessageLedgerEntry command = commandEntry.get();

    AttestationVerdict verdict = mapVerdict(entry.verificationOutcome);
    double confidence = mapConfidence(entry.verificationOutcome, entry.evidenceQuality);

    // Attestation 1: COMMAND entry (requester/engine trust)
    writeAttestation(command.id, command.subjectId, verdict, confidence, tenancyId);

    // Attestation 2: DONE entry (obligor/reviewer trust) — only when requester ≠ obligor
    var terminalEntry = messageRepo.findTerminalEntryByCorrelationId(
            entry.channelId, event.correlationId(), tenancyId);
    terminalEntry.ifPresent(done -> {
        if (!command.actorId.equals(done.actorId)) {
            writeAttestation(done.id, done.subjectId, verdict, confidence, tenancyId);
        }
    });
}

private void writeAttestation(UUID entryId, UUID subjectId,
        AttestationVerdict verdict, double confidence, String tenancyId) {
    LedgerAttestation attestation = new LedgerAttestation();
    attestation.ledgerEntryId = entryId;
    attestation.subjectId = subjectId;
    attestation.attestorId = "system:judgment-verifier";
    attestation.attestorType = ActorType.SYSTEM;
    attestation.verdict = verdict;
    attestation.confidence = confidence;
    attestation.capabilityTag = CapabilityTag.GLOBAL;
    try {
        ledger.saveAttestation(attestation, tenancyId);
    } catch (Exception e) {
        LOG.warnf(e, "Failed to write judgment attestation for entry %s", entryId);
    }
}
```

### 3. `SafetyProperty` — exclude judgment DONEs (D3)

Change `findDoneEntriesWithoutAttestation` to exclude DONE entries that have a YIELDED judgment event for the same correlationId. These are judgment commitments — their attestation is deferred by design.

```java
public List<MessageLedgerEntry> findDoneEntriesWithoutAttestation(
        Instant from, Instant to, String tenancyId) {
    return em.createQuery(
            "SELECT e FROM MessageLedgerEntry e " +
            "WHERE e.tenancyId = :tenancyId " +
            "AND e.messageType = 'DONE' " +
            "AND e.occurredAt >= :from AND e.occurredAt <= :to " +
            "AND NOT EXISTS (SELECT 1 FROM " +
            "io.casehub.ledger.runtime.model.LedgerAttestation a " +
            "WHERE a.ledgerEntryId = e.id) " +
            "AND NOT EXISTS (SELECT 1 FROM MessageLedgerEntry v " +
            "WHERE v.correlationId = e.correlationId " +
            "AND v.tenancyId = :tenancyId " +
            "AND v.toolName = :yieldedKind)",
            MessageLedgerEntry.class)
        .setParameter("tenancyId", tenancyId(tenancyId))
        .setParameter("from", from)
        .setParameter("to", to)
        .setParameter("yieldedKind", JudgmentEventKinds.YIELDED)
        .getResultList();
}
```

**Ownership boundary:** SafetyProperty = non-judgment attestation completeness. EvidenceCompletenessProperty = judgment attestation completeness. No overlap.

### 4. `EvidenceCompletenessProperty.remediate()` — dual with guard (D4)

Remediation writes DONE attestation always (it's the detected gap) and COMMAND attestation only if not already written by the observer. `saveAttestation` uses `em.persist()` — NOT idempotent. Duplicate calls create duplicate rows that double-count in trust computation.

```java
@Override
public int remediate(String tenancyId, Instant from, Instant to) {
    List<MessageLedgerEntry> deferred =
            messageRepo.findDoneEntriesWithDeferredAttestation(tenancyId);
    int count = 0;
    for (MessageLedgerEntry doneEntry : deferred) {
        var verifiedEntries = messageRepo.findJudgmentEvents(
                null, doneEntry.judgmentId, null, null, tenancyId);
        var verified = verifiedEntries.stream()
                .filter(e -> JudgmentEventKinds.VERIFIED.equals(e.toolName))
                .findFirst();
        if (verified.isEmpty()) continue;

        MessageLedgerEntry v = verified.get();
        AttestationVerdict verdict = mapVerdict(v.verificationOutcome);
        double confidence = mapConfidence(v);

        // Always write DONE attestation (the detected gap)
        writeAttestation(doneEntry.id, doneEntry.subjectId, verdict, confidence, tenancyId);
        count++;

        // Write COMMAND attestation only if not already written by observer
        var commandEntry = messageRepo.findLatestByCorrelationId(
                doneEntry.channelId, doneEntry.correlationId, tenancyId);
        commandEntry.ifPresent(cmd -> {
            boolean alreadyAttested = ledger.findAttestationsByEntryId(cmd.id, tenancyId)
                    .stream()
                    .anyMatch(a -> "system:judgment-verifier".equals(a.attestorId));
            if (!alreadyAttested) {
                writeAttestation(cmd.id, cmd.subjectId, verdict, confidence, tenancyId);
            }
        });
    }
    return count;
}
```

**Recovery semantics by observer failure mode:**

| Observer state | COMMAND attested? | DONE attested? | Gap detected? | Remediation action |
|---|---|---|---|---|
| Succeeded fully | Yes | Yes | No | None |
| Wrote COMMAND, crashed before DONE | Yes | No | Yes | Writes DONE only (COMMAND guard skips) |
| Did not run | No | No | Yes | Writes both |

---

## Files Changed

| File | Change |
|---|---|
| `runtime/.../ledger/MessageLedgerEntryRepository.java` | Add `findTerminalEntryByCorrelationId()` |
| `runtime/.../ledger/MessageLedgerEntryRepository.java` | Modify `findDoneEntriesWithoutAttestation()` — exclude judgment DONEs |
| `compliance-report/.../attestation/JudgmentVerificationObserver.java` | Dual attestation (COMMAND + DONE), extracted `writeAttestation()` helper |
| `compliance-report/.../verification/EvidenceCompletenessProperty.java` | Dual remediation with COMMAND guard |

### No Flyway migration

No schema changes. All attestations use the existing `ledger_attestation` table.

---

## Testing Strategy

| Component | Test type | Notes |
|-----------|----------|-------|
| `findTerminalEntryByCorrelationId` | CDI-free unit test | Mock EntityManager. Verify DONE/FAILURE/DECLINE filter, correlationId match, most-recent-first ordering. |
| `JudgmentVerificationObserver` — dual write | CDI-free unit tests | Mock repositories + ledger. Verify: (1) both COMMAND and DONE entries attested, (2) self-attestation guard (same actorId → DONE attestation skipped), (3) missing terminal entry → COMMAND-only attestation, (4) non-VERIFIED events ignored. |
| `SafetyProperty` — judgment exclusion | CDI-free unit tests | Mock repository. Verify: judgment DONEs without attestation NOT flagged; non-judgment DONEs without attestation still flagged. |
| `EvidenceCompletenessProperty.remediate()` — dual with guard | CDI-free unit tests | Mock repository + ledger. Verify: (1) DONE always written, (2) COMMAND written only when no existing `system:judgment-verifier` attestation, (3) COMMAND skipped when observer already attested. |
| `findDoneEntriesWithoutAttestation` — judgment exclusion | CDI-free unit tests | Mock EntityManager. Verify YIELDED event exclusion clause. |
| Integration: observer + remediation recovery | `@QuarkusTest` with `QuarkusTransaction.requiringNew()` | Dispatch COMMAND, DONE, VERIFIED EVENT. Verify both entries attested. Then: simulate observer failure (skip observer, dispatch VERIFIED), run remediation, verify both entries attested via recovery path. |

---

## Cross-repo Impact

None. All changes are within casehub-qhorus. The trust computation in casehub-ledger is unchanged — it correctly processes attestations on whichever entries they appear. The fix ensures attestations appear on the right entries.

---

## References

- `LedgerWriteService.writeAttestation()` (runtime/ledger/LedgerWriteService.java:284-338) — the dual-attestation reference pattern
- `TrustScoreCalculator.computeAll()` (casehub-ledger) — attestations grouped by entry, entries queried by actorId
- `QhorusLedgerEntryRepository.saveAttestation()` (runtime/ledger/:132) — uses `em.persist()`, not idempotent
- `findLatestByCorrelationId` (runtime/ledger/MessageLedgerEntryRepository.java:325) — symmetric pair for new `findTerminalEntryByCorrelationId`
- `findDoneEntriesWithDeferredAttestation` (runtime/ledger/MessageLedgerEntryRepository.java:551) — gap detection query (unchanged)
- `findDoneEntriesWithoutAttestation` (runtime/ledger/MessageLedgerEntryRepository.java:507) — modified to exclude judgment DONEs
- PP-20260623-77adf0 — CommitmentAttestationPolicy null context handling
- PP-20260608-07daa6 — observer test transaction discipline (requiringNew)
- decisions.md (D1-D4) — all design decisions
- casehubio/eidos#148 — per-capability trust differentiation (filed as part of #410 work)

## D1: Dual attestation — match the normal writeAttestation() pattern

**Choice:** Observer writes two attestations (COMMAND + DONE entries), mirroring LedgerWriteService.writeAttestation() lines 302-330. Self-attestation guard preserved.
**Alternatives:**
- Single attestation on DONE only — simpler but diverges from established pattern; engine trust never reflects judgment quality outcomes
**Rationale:** The dual attestation is the established pattern for every other commitment type. Deviating means judgment commitments become a special case in the trust model. The engine asking productive questions that lead to good outcomes is a meaningful signal worth recording.
**Trade-offs:** New query method needed to find the DONE entry; observer logic slightly more complex.
**Sources:** LedgerWriteService.writeAttestation() (runtime/ledger/LedgerWriteService.java:284-338), TrustScoreCalculator.computeAll() (casehub-ledger — attestations grouped by entry, entries queried by actorId)
**Exploration:** quick
**Status:** captured

## D2: New query findTerminalEntryByCorrelationId

**Choice:** Add `findTerminalEntryByCorrelationId(UUID channelId, String correlationId, String tenancyId)` to `MessageLedgerEntryRepository`. Filters to `messageType IN ('DONE', 'FAILURE', 'DECLINE')`, returns the most recent terminal entry. Mirrors the existing `findLatestByCorrelationId` (which filters to COMMAND/HANDOFF).
**Alternatives:**
- Reverse lookup via `causedByEntryId` chain — fragile; depends on `inReplyTo` being set on the terminal message
- Inline JPQL in the observer — works but not testable independently, inconsistent with the dedicated-query pattern
**Rationale:** Symmetric design: `findLatestByCorrelationId` finds the initiating entry, `findTerminalEntryByCorrelationId` finds the terminal entry. Both filter by type and return the most recent match. Clean, testable, consistent.
**Depends on:** D1 (dual attestation requires finding both entries)
**Trade-offs:** One more method on the repository. Trivial cost.
**Sources:** MessageLedgerEntryRepository.findLatestByCorrelationId (runtime/ledger/MessageLedgerEntryRepository.java:325-340)
**Exploration:** quick
**Status:** captured

## D3: SafetyProperty excludes judgment DONEs via query

**Choice:** Add `NOT EXISTS (SELECT 1 FROM MessageLedgerEntry v WHERE v.correlationId = e.correlationId AND v.tenancyId = :tenancyId AND v.toolName = :yieldedKind)` to `findDoneEntriesWithoutAttestation`. SafetyProperty covers non-judgment commitments; EvidenceCompletenessProperty covers judgment commitments.
**Alternatives:**
- Post-filter in property logic — loads unnecessary rows from DB, logic split between query and Java
- Check COMMAND attestation for judgment DONEs — conflates SafetyProperty with judgment-specific semantics
**Rationale:** Push the exclusion into the query. Clear ownership: SafetyProperty = non-judgment attestation completeness, EvidenceCompletenessProperty = judgment attestation completeness. No overlap, no false positives.
**Depends on:** D1 (once observer writes dual attestations, judgment DONEs WILL have attestations — but the deferred timing means SafetyProperty may still see them during the window between DONE and VERIFIED)
**Trade-offs:** SafetyProperty no longer catches judgment DONEs with genuinely missing attestations. Acceptable because EvidenceCompletenessProperty covers that exact case.
**Sources:** SafetyProperty.java (compliance-report/verification/), MessageLedgerEntryRepository.findDoneEntriesWithoutAttestation (line 507)
**Exploration:** quick
**Status:** captured

## D4: EvidenceCompletenessProperty — dual remediation, DONE-based gap detection

**Choice:** Remediation writes dual attestations (COMMAND + DONE), matching D1. Gap detection query (`findDoneEntriesWithDeferredAttestation`) unchanged — checks for missing attestation on the DONE entry. After D1 fix, observer writes DONE last; if both writes succeed, no gap detected. If observer crashes mid-write (COMMAND written, DONE missed), gap detected and remediation writes both (COMMAND write is idempotent via `saveAttestation`).
**Alternatives:**
- Check COMMAND entry for gap detection — inverts the detection; partial observer success (COMMAND written, DONE missed) becomes invisible
**Rationale:** DONE-as-last-write gives clean recovery semantics. The observer writes COMMAND first, DONE second. If it crashes between the two, the gap detection catches the missing DONE and remediation writes both. If the observer succeeds fully, no gap. No false positives, no false negatives.
**Depends on:** D1 (observer write order: COMMAND first, DONE second), D2 (findTerminalEntryByCorrelationId for remediation to find DONE entry)
**Trade-offs:** If `saveAttestation` is not idempotent (throws on duplicate), remediation needs a guard. Verify `saveAttestation` behavior.
**Sources:** EvidenceCompletenessProperty.java (compliance-report/verification/), MessageLedgerEntryRepository.findDoneEntriesWithDeferredAttestation (line 551)
**Exploration:** quick
**Status:** captured

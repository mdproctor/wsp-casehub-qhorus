## D1: Dual attestation — match the normal writeAttestation() pattern

**Choice:** Observer writes two attestations (COMMAND + DONE entries), mirroring LedgerWriteService.writeAttestation() lines 302-330. Self-attestation guard preserved.
**Alternatives:**
- Single attestation on DONE only — simpler but diverges from established pattern; engine trust never reflects judgment quality outcomes
**Rationale:** The dual attestation is the established pattern for every other commitment type. Deviating means judgment commitments become a special case in the trust model. The engine asking productive questions that lead to good outcomes is a meaningful signal worth recording.
**Trade-offs:** New query method needed to find the DONE entry; observer logic slightly more complex.
**Sources:** LedgerWriteService.writeAttestation() (runtime/ledger/LedgerWriteService.java:284-338), TrustScoreCalculator.computeAll() (casehub-ledger — attestations grouped by entry, entries queried by actorId)
**Exploration:** quick
**Status:** captured

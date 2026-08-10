# Obligor-Targeted Trust Attribution

**Issue:** casehubio/qhorus#373
**Parent:** #351 (Verification and Trust)
**Date:** 2026-08-10

## Problem

The trust model (`ComputedTrustScoreSource` in casehub-ledger) computes trust for an actor from ledger entries where `entry.actorId = actorId` and attestations on those entries. Policy attestations (SOUND/FLAGGED) are written on the COMMAND entry when a terminal message (DONE/FAILURE/DECLINE) arrives. The COMMAND entry's `actorId` is the requester — the agent who sent the COMMAND.

This means attestation outcomes affect the requester's trust score, not the obligor's. An agent that consistently produces low-quality DONE messages has no direct trust consequence from attestation outcomes.

## Design

### Dual attestation

When a terminal message (DONE, FAILURE, DECLINE, RESPONSE) arrives, `LedgerWriteService.record()` currently writes one attestation on the causally-linked COMMAND entry. This design adds a **second attestation on the terminal entry itself**.

The terminal entry's `actorId` is already the obligor (the agent who sent DONE/FAILURE/DECLINE). Existing trust queries (`findAttestationsByActorId`) work unmodified — when computing trust for the obligor, the query finds attestations stamped on entries where `entry.actorId = obligor`, which now includes terminal entries.

### What changes

**One file:** `LedgerWriteService.writeAttestation()` in casehub-qhorus runtime.

After the existing attestation write on the COMMAND entry (`priorMsg.id`), write a second attestation on the terminal entry (`entry.id` — the entry just saved by `record()`). The terminal entry's `id` is passed as a new parameter to `writeAttestation()`.

The second attestation uses the same verdict and confidence from `CommitmentAttestationPolicy`. Both attestations share the same `capabilityTag` (extracted from the COMMAND content).

### What does NOT change

- **casehub-ledger:** Zero changes. No schema, no model, no queries, no trust computation pipeline.
- **Existing COMMAND attestations:** Unchanged. Requester trust continues to be computed exactly as before.
- **`CommitmentAttestationPolicy` SPI:** Same interface, same call. The policy is invoked once; its result feeds both attestations.
- **Peer attestation (`PeerAttestationWriter`):** Unaffected. Peer attestations target specific entries chosen by the reviewer, independent of this automatic policy attestation.

### Attestation write flow (after change)

```
Terminal message arrives (e.g., DONE from obligor B on COMMAND from requester A)
  │
  ├── record() saves MessageLedgerEntry (actorId = B, the terminal sender)
  │     └── entry.id assigned by ledger.save()
  │
  └── writeAttestation(subjectId, causedByEntryId, terminalEntryId, ...)
        │
        ├── [existing] attestation on COMMAND entry (priorMsg.id)
        │     └── feeds A's trust score (requester)
        │
        └── [new] attestation on terminal entry (terminalEntryId)
              └── feeds B's trust score (obligor)
```

### Self-attestation guard

When the COMMAND sender and terminal sender are the same actor (agent sends a COMMAND to itself), the second attestation would duplicate the first. Guard: skip the terminal-entry attestation when `resolvedActorId == priorMsg.actorId` (same actor on both entries).

### RESPONSE type handling

RESPONSE messages resolve COMMAND obligations with verdict FLAGGED/0.3 (wrong vocabulary — a RESPONSE to a COMMAND is atypical). The obligor attestation applies here too: the RESPONSE sender gets the same verdict. This is correct — sending RESPONSE to a COMMAND is equally atypical regardless of which actor's trust is affected.

### Testing

- CDI-free unit test: construct `LedgerWriteService` with mocks, verify two `saveAttestation()` calls for a terminal message — one with `ledgerEntryId = priorMsg.id`, one with `ledgerEntryId = terminalEntryId`.
- Self-attestation guard test: same sender on COMMAND and terminal → only one attestation.
- Integration test via `QhorusMcpTools.sendMessage()`: dispatch COMMAND then DONE, query attestations on both entries, verify both present with matching verdict/confidence.

### Scope boundaries

- **Not in scope:** Differentiating verdicts per actor (requester vs obligor). Both get the same verdict from the single policy call. Future work could specialize.
- **Not in scope:** DECLINE penalization concerns (R1-03 from decision review). Whether DECLINE should produce FLAGGED for the obligor is a policy question for #371 (attestor credibility tracking), not an attribution question.
- **Not in scope:** Delegation chain intermediate actors. Only the terminal sender (final delegate) receives the obligor attestation. Intermediate delegators' trust signal comes from their own HANDOFF/DONE interactions.

## Acceptance criteria

- Trust computation attributes attestation outcomes to the obligor (terminal message sender)
- Existing requester-focused trust attribution unchanged
- Zero changes to casehub-ledger
- Self-attestation guard prevents duplicate attestations

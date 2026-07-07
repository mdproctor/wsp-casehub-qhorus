# Design: Attestation Error Isolation in LedgerWriteService

**Issue:** casehubio/qhorus#324
**Date:** 2026-07-07
**Branch:** issue-324-attestation-try-catch

---

## Problem

`LedgerWriteService.record()` has four error-handling defects that can cause the
main ledger entry to be lost when attestation fails. The ledger entry is the
critical audit record; attestation is an optional trust signal. The class Javadoc
already states: "Attestation write failures are caught and logged — attestation is
a trust-scoring signal, not a correctness requirement." The implementation does not
honour this contract.

### Root 1 — Wrong execution order

The attestation block (lines 196-207) runs BEFORE `ledger.save(entry, tenancyId)`
(line 209). Any exception in the attestation block means `ledger.save()` is never
reached. The entry is lost — not via transaction rollback, but because execution
never gets there.

The reactive version (`ReactiveLedgerWriteService`) saves the entry first (line 154),
then does attestation (line 160). The blocking version has the order backwards.

### Root 2 — `attestationPolicy.attestationFor()` outside try/catch

In `writeAttestation()` line 216, the SPI call is outside the try/catch (lines 217-233).
`StoredCommitmentAttestationPolicy` is a pure switch — can't throw. But
`TrustGatedAttestationPolicy` (casehub-ledger) calls `source.capabilityScore()`,
`source.decisionCount()`, `policyProvider.forCapability()` — all database/cache
lookups that can fail with RuntimeException.

### Root 3 — `ledger.findEntryById()` unprotected

Line 197 in `record()` — the lookup for the prior command entry is a JPA query that
can throw `PersistenceException`. No try/catch at any level. Propagates up and
prevents entry save.

### Root 4 — Structural inconsistency with reactive

In the reactive version, `writeAttestation()` is self-contained: it owns the
`findEntryById()` call, the policy call, and the attestation save, all under
`.onFailure().recoverWithUni()`.

In the blocking version:
- `findEntryById()` happens in `record()`, outside `writeAttestation()`
- The prior entry is passed as `MessageLedgerEntry commandEntry` parameter
- Error handling is split across two scopes (outer attestation block + inner try/catch)
- Method signature diverges from reactive (`MessageLedgerEntry` vs `UUID causedByEntryId`)

### Known limitation — PersistenceException marks transaction for rollback

If `em.persist(attestation)` inside `saveAttestation()` throws `PersistenceException`,
JPA spec §3.3.7.1 requires the provider to mark the transaction for rollback.
`QhorusLedgerEntryRepository.saveAttestation()` has no `@Transactional` annotation — it
runs directly in the caller's `REQUIRES_NEW` transaction from `record()`. A
PersistenceException from the attestation persist marks that transaction for rollback —
the entry save is rolled back even though the exception is caught in `writeAttestation()`.

This violates the Javadoc contract: "Attestation write failures are caught and logged —
attestation is a trust-scoring signal, not a correctness requirement." The reactive version
does not have this problem because `onFailure().recoverWithUni()` recovers the Uni chain
without JTA rollback semantics — a blocking/reactive parity gap.

In practice the risk is low: UUID auto-generated, FK validated by prior SELECT, entry in
L1 cache. The realistic failure scenario is a database connectivity failure between the
SELECT and persist within the same transaction. Nevertheless, the contract violation is
real and is fixed in this spec (see Change 5).

---

## Design

### Change 1 — Reorder in `record()`

Move `ledger.save(entry, tenancyId)` before the attestation block. Replace the inline
attestation block with a single call to the restructured `writeAttestation()`.

Before:
```
build entry → attestation block → ledger.save() → return
```

After:
```
build entry → ledger.save() → writeAttestation() → return
```

### Change 2 — Restructure `writeAttestation()`

Make it self-contained — include `findEntryById()`, change signature to match reactive:

**Old signature:**
```java
void writeAttestation(UUID subjectId, MessageLedgerEntry commandEntry,
    MessageType terminalType, String resolvedActorId, String tenancyId,
    CommitmentContext context)
```

**New signature:**
```java
void writeAttestation(UUID subjectId, UUID causedByEntryId,
    MessageType terminalType, String resolvedActorId, String tenancyId,
    UUID commitmentId)
```

Changes:
- `MessageLedgerEntry commandEntry` → `UUID causedByEntryId` (lookup moves inside)
- `CommitmentContext context` → `UUID commitmentId` (context built inside)
- Single try/catch covers `findEntryById()`, `attestationFor()`, and `saveAttestation()`
- Catch block includes the exception for diagnostic logging (`LOG.warnf(e, ...)`)
- Matches reactive `writeAttestation()` signature

**New method body:**
```java
private void writeAttestation(final UUID subjectId, final UUID causedByEntryId,
        final MessageType terminalType, final String resolvedActorId, final String tenancyId,
        final UUID commitmentId) {
    try {
        ledger.findEntryById(causedByEntryId, tenancyId).ifPresent(prior -> {
            if (!(prior instanceof MessageLedgerEntry priorMsg)) return;
            if (!"COMMAND".equals(priorMsg.messageType)
                    && !"HANDOFF".equals(priorMsg.messageType)) return;
            final String capabilityTag = extractCapabilityTag(priorMsg.content);
            final CommitmentContext ctx = new CommitmentContext(
                    priorMsg.correlationId, priorMsg.channelId, null, commitmentId, capabilityTag);
            attestationPolicy.attestationFor(terminalType, resolvedActorId, ctx)
                    .ifPresent(outcome -> {
                final LedgerAttestation attestation = new LedgerAttestation();
                attestation.ledgerEntryId = priorMsg.id;
                attestation.subjectId = subjectId;
                attestation.attestorId = outcome.attestorId();
                attestation.attestorType = outcome.attestorType();
                attestation.verdict = outcome.verdict();
                attestation.confidence = outcome.confidence();
                attestation.capabilityTag = ctx.capabilityTag();
                ledger.saveAttestation(attestation, tenancyId);
                LOG.debugf("LedgerAttestation %s written for COMMAND entry %s"
                                + " (correlationId='%s', capability='%s')",
                        attestation.verdict, priorMsg.id,
                        priorMsg.correlationId, attestation.capabilityTag);
            });
        });
    } catch (final Exception e) {
        LOG.warnf(e, "Could not write attestation for entry %s"
                        + " — trust signal lost but pipeline unaffected",
                causedByEntryId);
    }
}
```

The try/catch wraps the entire body including `findEntryById()` (Root 3),
`attestationFor()` (Root 2), and `saveAttestation()`. The catch block includes the
exception `e` for diagnostic logging — stack trace, cause chain, and message are all
preserved for operator visibility.

### Change 3 — No changes to ReactiveLedgerWriteService

The reactive version already has the correct order and `.onFailure().recoverWithUni()`
covers everything. No changes needed.

**Observation:** The reactive version's `recoverWithUni` handler has the same exception-
swallowing gap — the caught `Throwable` parameter is not included in the log message.
Not in scope for this change but noted for a separate fix.

### Change 4 — Javadoc update

Update the class Javadoc to reflect that the entry is saved before attestation, and
document the PersistenceException edge case as a known limitation.

### Change 5 — Transaction isolation for attestation writes

Add `@Transactional(TxType.REQUIRES_NEW)` to `QhorusLedgerEntryRepository.saveAttestation()`.
This isolates the attestation persist in its own transaction — if `em.persist(attestation)`
throws `PersistenceException`, only the attestation transaction rolls back. The outer
`REQUIRES_NEW` transaction from `record()` is unaffected, and the entry save is preserved.

The referenced command entry (looked up by `attestation.ledgerEntryId`) is from a prior
committed transaction, so it remains visible to the new `REQUIRES_NEW` transaction.

**Follow-up issue:** `JpaLedgerEntryRepository.saveAttestation()` in casehub-ledger has
the same gap (`@Transactional(REQUIRED)` instead of `REQUIRES_NEW`). This is a cross-module
change affecting all non-qhorus callers — filed as a separate follow-up issue.

---

## Affected files

| File | Change |
|------|--------|
| `runtime/src/.../ledger/LedgerWriteService.java` | Reorder + restructure writeAttestation() |
| `runtime/src/.../ledger/QhorusLedgerEntryRepository.java` | Add `@Transactional(REQUIRES_NEW)` on saveAttestation() |
| `runtime/src/test/.../ledger/LedgerWriteServiceTest.java` | Add error isolation and ordering tests |

---

## Test plan

1. **New test — attestationFor() throws RuntimeException**: Set policy to throw.
   Assert entry is still saved, attestation list is empty, no exception propagates.
2. **New test — findEntryById() throws in attestation path**: Stub throws on the
   prior entry lookup. Assert entry is still saved.
3. **New test — saveAttestation() throws RuntimeException**: Configure stub to throw
   on `saveAttestation()`. Assert entry is still saved, no exception propagates. This
   path has zero test coverage today.
4. **New test — entry saved before writeAttestation() executes**: Configure the
   attestation policy to inspect `sharedEntries` inside `attestationFor()` and assert
   the entry is already present. Makes the ordering guarantee from Change 1 explicit
   rather than inferential.
5. **Existing attestation tests**: Must continue passing unchanged — the restructure
   preserves all behaviour.
6. **Full build**: `mvn clean install` — verify no regressions across all modules.

**Note:** The `REQUIRES_NEW` transaction isolation (Change 5) is not exercised in unit tests
with `StubLedgerEntryRepository` (no real JTA). The try/catch tests (items 1–3) verify error
isolation at the application level; transaction-level isolation requires an integration test
with a real JTA container.

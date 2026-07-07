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
JPA spec §3.3.7.1 requires the provider to mark the transaction for rollback. Since
attestation shares the `REQUIRES_NEW` transaction with the entry save, the entry would
also be rolled back even if the exception is caught.

In practice this is near-zero risk: UUID auto-generated via `@PrePersist`, FK validated
by prior SELECT, entry in L1 cache. Filed as a follow-up for `REQUIRES_NEW` attestation
isolation if warranted.

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
- Single try/catch covers `findEntryById()`, `attestationFor()`, `saveAttestation()`,
  and CDI event fire
- Matches reactive `writeAttestation()` signature

### Change 3 — No changes to ReactiveLedgerWriteService

The reactive version already has the correct order and `.onFailure().recoverWithUni()`
covers everything. No changes needed.

### Change 4 — Javadoc update

Update the class Javadoc to reflect that the entry is saved before attestation, and
document the PersistenceException edge case as a known limitation.

### Change 5 — Follow-up issue

File a follow-up issue for `REQUIRES_NEW` transaction isolation of attestation writes.

---

## Affected files

| File | Change |
|------|--------|
| `runtime/src/.../ledger/LedgerWriteService.java` | Reorder + restructure writeAttestation() |
| `runtime/src/test/.../ledger/LedgerWriteServiceTest.java` | Add test for attestationFor() throwing |

---

## Test plan

1. **New test — attestationFor() throws RuntimeException**: Set policy to throw.
   Assert entry is still saved, attestation list is empty, no exception propagates.
2. **New test — findEntryById() throws in attestation path**: Stub throws on the
   prior entry lookup. Assert entry is still saved.
3. **Existing attestation tests**: Must continue passing unchanged — the restructure
   preserves all behaviour.
4. **Full build**: `mvn clean install` — verify no regressions across all modules.

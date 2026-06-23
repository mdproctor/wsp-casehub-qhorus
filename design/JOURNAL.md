# Design Journal — issue-303-extract-evidential-checker

## 2026-06-23 — Extract EvidentialChecker + fix RESPONSE attestation gap

**New package: `io.casehub.qhorus.runtime.audit`**

Extracted `EvidentialChecker`, `BenchmarkContext`, and `BenchmarkViolation` from
`examples/agent-communication/benchmark/` into the runtime module as a publishable CDI bean.
`EvidentialChecker` is `@DefaultBean @ApplicationScoped`; two entry points:
- `check(String messageType, String content, BenchmarkContext)` — benchmark path (V1–V4)
- `checkObligation(String terminalType, CommitmentContext)` — attestation path (vocabulary only)

casehub-devtown and other consumers can now declare `io.casehub:casehub-qhorus` as a
dependency and inject `EvidentialChecker` directly.

**New SPI: `CommitmentContext` (api/spi)**

Record: `correlationId, channelId (UUID), channelName (nullable), commitmentId (UUID, nullable)`.
Passed as a third argument to `CommitmentAttestationPolicy.attestationFor()` so evidential policy
implementations can query the ledger before deciding verdict. `CommitmentAttestationPolicy` is no
longer `@FunctionalInterface`; a 2-arg default delegates to 3-arg with `null` context.

**RESPONSE-on-COMMAND attestation gap (Closes #305)**

RESPONSE sent with a COMMAND's corrId fulfills the commitment (FULFILLED) but previously produced
no attestation signal. Fix: RESPONSE added to `ATTESTATION_TYPES` in `LedgerWriteService` and
`ReactiveLedgerWriteService`. The existing `"COMMAND".equals(priorMsg.messageType)` guard ensures
RESPONSE-on-QUERY still produces no attestation. `StoredCommitmentAttestationPolicy` returns
FLAGGED/0.3 for RESPONSE; configurable via `casehub.qhorus.attestation.response-confidence`.

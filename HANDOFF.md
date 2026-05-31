# CaseHub Qhorus — Session Handover
**Date:** 2026-05-31 — S/XS issue sweep + #224 DESIGN.md fix

---

## What Was Done This Session

**Closed qhorus#224.** Fixed stale MessageType vocabulary in `docs/DESIGN.md` (two line edits: MessageType enum table, send_message correlationId note). Also promoted `2026-05-28-watchdog-store-seam-design.md` to `docs/specs/` — was missing but referenced by two platform protocols.

**Cleared 7 GitHub issues** that were shipped last session but never closed (direct push bypasses auto-close): #166, #164, #183, #202, #203, #215, #217.

**Cleared all S/XS open issues in qhorus** (#220 + #221, single branch). #220: replaced hardcoded connector ID strings in `ConnectorKeyStrategy` with `InboundConnectorIds` constants — required cross-repo fix in `casehub-connectors` (casehubio/connectors#11, shipped to upstream). #221: CDI `@ObservesAsync` wiring test — changed `onInboundMessage` to return `CompletionStage<Void>`, added `fireAsync().join()` test. Garden: GE-20260531-5137f7 (mvn install BUILD SUCCESS on wrong git branch — jar silently missing new class).

## Immediate Next Step

**parent#125** — 17 protocol ref path-depth fixes + deep-dive enrichment + 2 new protocols + skill update. Python script ready in issue body. S · Low.

## What's Left

- **casehubio/parent#125** — 17 path-depth fixes + deep-dive enrichment + 2 protocols + skill update · S · Low
- **casehubio/parent#124** — tracking; tick off as sub-issues resolve
- **casehub-ledger#105** — reactive LedgerAttestation persistence · S · Med
- **casehub-ledger#106** — `Uni<Boolean> TrustGateService.meetsThreshold()` · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| parent#125 | Protocol ref path fixes + deep-dive enrichment | S | Low | Python script ready in issue body |
| qhorus#216 | Per-connector InboundNormaliser | S | Med | Unblocked |
| qhorus#214 | Auto-channel creation on first contact | M | High | Needs #215/#217 ✅ |
| qhorus#132 | Delivery guarantees (retry + dead-letter) | L | High | Main feature item |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-31-mdp01-a-string-is-not-a-contract.md` |
| Platform coherence tracking | casehubio/parent#124 |
| Protocol + deep-dive fixes (actionable) | casehubio/parent#125 |
| Garden entry (mvn install on wrong branch) | GE-20260531-5137f7 |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

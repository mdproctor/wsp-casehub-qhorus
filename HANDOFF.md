# HANDOFF — casehub-qhorus

## Last Session

Designed digital signatures (eIDAS qualified seals) for compliance reports (#418). Updated all 5 open issues — every blocker resolved (#401, #399, ledger#202, platform#244, #417 all closed). 8 design decisions through decision review (4 revised: SigningProvider adapter infeasible, silent B-B fallback dangerous) and standard spec review (18 issues, all addressed). Implementation plan written — 4 batches, 7 tasks across platform + qhorus. Completed Batch 1: platform SPI types (DocumentSigningService, DocumentVerificationService, NoOp defaults) committed and SNAPSHOT installed.

## Immediate Next Step

Batch 2: `casehub-platform-signing` module — EU DSS integration. Create module, verify PDFBox 3.x compat, implement KeyStoreManager + DssDocumentSigningService + DssDocumentVerificationService. Plan: `plans/2026-08-31-digital-signatures-eidas.md`.

## Cross-Module

Platform repo has Batch 1 commit on branch `issue-252-yaml-core-modules` — SPI types for #418. Batches 2-3 also touch platform. Needs its own push workflow separate from qhorus work-end.

## References

- `specs/issue-418-digital-signatures-eidas/2026-08-31-digital-signatures-eidas-design.md` — design spec
- `specs/issue-418-digital-signatures-eidas/decisions.md` — D1-D8
- `plans/2026-08-31-digital-signatures-eidas.md` — implementation plan
- `blog/2026-08-31-mdp01-signing-the-evidence.md` — diary entry

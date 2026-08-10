# CaseHub Qhorus — Session Handover

**Date:** 2026-08-10 — #391 + #373 closed.

---

## Immediate Next Step

Pick from remaining open issues: #371 (attestor credibility, M/High), #392 (unnecessary CDI cleanup, S/Low).

## What Was Done

Closed #391: removed all stale reactive stack documentation from CLAUDE.md (14 bullets deleted, 12 edited), deleted 3 reactive protocols, cleaned 6 remaining protocols, removed stale %reactive-pg test profile. Closed #373: obligor trust attribution via dual attestation — second attestation on terminal entry feeds obligor trust through existing query model with zero casehub-ledger changes. Decision review caught a flaw in the original obligorId approach (domain leakage, conflated signals) and surfaced the cleaner dual-attestation alternative.

## References

- Spec: `specs/issue-391-reactive-cleanup-trust-attr/2026-08-10-obligor-trust-attribution-design.md`
- Blog: `blog/2026-08-10-mdp01-trust-attribution-belongs-where-actor-is.md`

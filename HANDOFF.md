# CaseHub Qhorus — Session Handover

**Date:** 2026-08-11 — #392 closed.

---

## Immediate Next Step

Pick from remaining open issues: #371 (attestor credibility, M/High), #394 (GraphQL schema, unsized).

## What Was Done

Closed #392: verified 4 CDI classes flagged by parent#340 audit. 3 of 4 were false positives — StoredMessageTypePolicy (SPI interface), DefaultInboundNormaliser (@DefaultBean displacement), QhorusEntityMapper (real @Inject ObjectMapper). Only AllowedWritersPolicy had genuinely unnecessary @ApplicationScoped — removed it and wired directly in MessageService. Garden entry GE-20260811-bfe973 captures the audit false-positive pattern.

## References

- Blog: `blog/2026-08-11-mdp01-three-out-of-four-were-fine.md`
- Garden: GE-20260811-bfe973 (CDI audit false positives from injection counting)

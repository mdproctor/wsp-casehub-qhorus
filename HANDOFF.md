# CaseHub Qhorus — Session Handover

**Date:** 2026-08-08 — backlog triage + #380 shipped.

---

## Immediate Next Step

Pick from remaining open issues: #371 (attestor credibility, M/High), #373 (obligor trust attribution, M/High), #391 (stale CLAUDE.md reactive cleanup, S/Low).

## What Was Done

Triaged all S/XS issues. Closed #372 (reactive stack removed — issue invalid), #369 and #358 (filed as engine#875/#876). Filed #391 for stale CLAUDE.md reactive cleanup. Implemented #380: per-participant delivery retry with PostResult SPI, selective cursor advancement, retryLaggingParticipants in deliverPending, per-participant health tracking, volume caps. Design review (light, 3 dimensions) caught cursor race and instanceof dispatch — both fixed. Garden entry GE-20260808-c29cdf (ConcurrentHashMap findFirst non-determinism).

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*

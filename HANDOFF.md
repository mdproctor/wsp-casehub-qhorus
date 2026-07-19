# CaseHub Qhorus — Session Handover

*Updated: #355, #368 closed — removed from in-progress. Epic #349 children all closed.*

**Date:** 2026-07-19 — #356 implemented and shipped. Peer attestation — multi-agent verification on ledger entries.

---

## Immediate Next Step

Pick from the backlog below. #357 (protocol enforcement SPI, L/High) is the recommended next pick — depends on #354 (done), the last remaining epic #350 child.

## What Was Done

**Implemented #356 — peer attestation.** Layered architecture from root primitive: `PeerAttestationWriter` validates and writes `LedgerAttestation` with ENDORSED/CHALLENGED verdicts. `ReviewerResolver` 4-layer fallback (explicit → channel config → capability routing → CDI event). `PeerReviewAutoTrigger` observer sends review QUERYs after DONE. `PeerReviewResponseHandler` parses structured responses and auto-writes attestations. 4 new MCP tools: `attest`, `list_attestations`, `request_peer_review`, `set_channel_reviewers`. Channel gains `reviewerInstances` field (V37 migration). Trust integration is free — Bayesian Beta model aggregates automatically. Design-reviewed through 5 adversarial rounds (22 issues, all resolved). Pushed to upstream/main.

**Also landed (slot 2):** #355 (channel context summary) and #368 (circular delegation detection) — both now closed and merged to main.

## Backlog — Prioritised by Effort-to-Impact

### Category B: Medium Effort / High Practical Impact — Core Roadmap

| Priority | # | Title | Scale | Cplx | Why this order |
|---|---|---|---|---|---|
| **1** | #357 | Protocol enforcement SPI | L | High | Depends on #354 (done). 7x accuracy gains evidence. |
| **2** | #349 | Epic: Coordination Resilience | — | — | All children closed (#368 landed). Ready to close. |

### Category D: Cross-Repo Suggestions

| Priority | # | Title | Target | Scale | Cplx |
|---|---|---|---|---|---|
| **3** | #358 | Supervisor + friction interventions | engine | L | High |
| **4** | #359 | Summarisation → Qhorus integration | blocks | M | Med |
| **5** | #361 | CBR routing + coordination memory | blocks/neocortex | M | High |
| **6** | #360 | Common ground projection | blocks | M | High |
| **7** | #364 | Convergence detection | blocks | M | High |

### Epics

| # | Epic | Children |
|---|------|----------|
| #349 | Coordination Resilience | ~~#353~~, ~~#354~~, ~~#362~~, ~~#363~~, ~~#368~~ — **ready to close** |
| #350 | Channel Intelligence | ~~#355~~, #357 |
| #351 | Verification & Trust | ~~#356~~ — **ready to close** |
| #352 | Cross-Repo Suggestions | #358, #359, #360, #361, #364 |

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-19-peer-attestation-design.md` (workspace specs/) |
| Design review workspace | `~/adr/casehub-qhorus/peer-attestation-20260719-133508/` |
| Slot 2 | `/Users/mdproctor/claude/casehub/worktrees/2/` |
| Previous references | *Unchanged — `git show HEAD~1:HANDOFF.md`* |

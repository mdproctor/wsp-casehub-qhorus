---
layout: post
title: "Peer Attestation: When Trust Needs a Second Opinion"
date: 2026-07-19
type: phase-update
entry_type: note
subtype: diary
projects: [Qhorus]
tags: [attestation, trust, verification]
---

The MAST taxonomy puts verification gaps at 21% of multi-agent failures. An agent says DONE and the system writes a SOUND attestation — formal correctness confirmed, vocabulary matched, artefact referenced. But nobody checked whether the work was actually good.

Qhorus already had the infrastructure for this. `AttestationVerdict` has four values: SOUND, FLAGGED, ENDORSED, CHALLENGED. The first two were used by `CommitmentAttestationPolicy` — the system's own judgment on whether a terminal message followed the right protocol. The last two sat there unused, clearly designed for exactly this moment.

The design question was where to draw the line between Qhorus and external intelligence. Qhorus is a communication mesh, not an orchestrator — it shouldn't decide *who* reviews or *how* to evaluate quality. But it should provide the mechanism for recording peer judgments as first-class trust signals.

I landed on a layered architecture built outward from a root primitive. `PeerAttestationWriter` validates and writes a `LedgerAttestation` with ENDORSED or CHALLENGED verdict. That's the atomic operation — everything else composes from it. Above it: three MCP tools (`attest`, `request_peer_review`, `list_attestations`). Above those: two observers that automate the common paths — `PeerReviewAutoTrigger` sends review QUERYs after DONE messages, `PeerReviewResponseHandler` parses structured responses and auto-writes attestations.

The adversarial design review caught several things I'd missed. The self-attestation guard was obvious in retrospect — without it, an agent could ENDORSE its own COMMAND entry and inflate its trust score. The confidence gaming vector was subtler: if agents could set their own confidence values, a CHALLENGED with confidence=1.0 would hit 2.5x harder than the default 0.5. Config-only confidence closes that.

The review also pushed me to eliminate `inReplyTo` tracing from the response handler entirely. The original design had the handler look up the original QUERY via `inReplyTo` to confirm it was a review request. But `MessageObserver` fires in `afterCompletion` — the message is committed but `inReplyTo` lookups hit pre-commit state visibility issues. The fix: content-based identification. The RESPONSE echoes `ledger_entry_id` from the QUERY's payload. No message store queries, no transaction visibility concerns.

Trust integration turned out to be genuinely free. The Bayesian Beta model aggregates all attestations on an entry, weighted by `recencyWeight × confidence`. A COMMAND that receives policy SOUND (0.7) plus peer ENDORSED (0.4) gets two positive observations at different weights — the peer's lower confidence means it nudges the score without dominating it. Mixed signals — SOUND plus CHALLENGED — are exactly what the model is designed to handle.

The one limitation worth naming: attestations target the COMMAND entry, so the *requester's* trust score moves, not the *obligor's*. An agent that consistently produces poor DONE messages won't see its own score degrade from peer attestation — only from policy FLAGGED when its terminal messages fail formal checks. Obligor-targeted trust attribution is tracked separately in [#373](https://github.com/casehubio/qhorus/issues/373).

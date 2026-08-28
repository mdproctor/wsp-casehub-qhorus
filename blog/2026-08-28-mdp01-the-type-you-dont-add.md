---
title: "The Type You Don't Add"
date: 2026-08-28
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
series: issue-411-judgment-commitment-type
tags: [speech-acts, taxonomy, governed-yield, formal-verification, attestation]
---

# The Type You Don't Add

The governed yield epic (#410) asked for three new message types: JUDGMENT_REQUEST, JUDGMENT_RESPONSE, JUDGMENT_ACCEPTANCE. Speech act types for judgment exchanges — the engine yields a judgment to a caller, the caller responds with evidence, the engine verifies.

I looked at the Searle matrix from ADR-0005 and every proposed type fell into an existing cell. JUDGMENT_REQUEST is a directive (action) — that's COMMAND's cell. JUDGMENT_RESPONSE is an assertive — RESPONSE's cell. JUDGMENT_ACCEPTANCE is a declaration — DONE's cell. The stopping criterion we established with PROPOSE says new types need an empty cell. None of these qualify.

But then Claude's decision review pushed back hard on something I'd got wrong. I'd dismissed the PROPOSE alternative with "PROPOSE is sender-binding, judgment is receiver-obligation." Claude pointed out — correctly — that PROPOSE uses the *same* commitment direction as COMMAND. The receiver IS the obligor in both. The dismissal was factually wrong.

The real argument against PROPOSE for judgment requests isn't commitment direction. It's the Searle category. PROPOSE is a commissive — "I offer to do X if you agree." A judgment request is a directive — "I need you to do X." Using PROPOSE would misclassify the speech act, which is exactly what ADR-0005 was designed to prevent. The stopping criterion isn't just about preventing redundant types. It's about using the *right* existing type for the actual illocutionary force.

## The dual-lifecycle insight

This led to a cleaner framing than the original epic envisioned. The commitment lifecycle and the judgment lifecycle are deliberately separate because they track orthogonal assessments:

- **Obligation fulfillment:** "Did the reviewer provide a review?" → Commitment: FULFILLED
- **Quality assessment:** "Was the review adequate?" → Attestation: SOUND or FLAGGED

A contractor who delivers shoddy work has still fulfilled their contractual obligation. The building inspector (attestation) handles quality. If the work fails inspection, the client issues a new contract for remediation — they don't reopen the fulfilled one.

This maps to the implementation directly. `JudgmentCommitmentAttestationPolicy` extends `StoredCommitmentAttestationPolicy` and returns `Optional.empty()` for DONE on judgment commitments — deferring attestation until the engine's VERIFIED event arrives. `JudgmentVerificationObserver` picks up that event and writes the sole attestation with verification-informed confidence. One attestation per commitment, timed to when quality is actually known.

Claude's code review caught a second real bug: the `DeadlockFreedomProperty` was filtering `findOpenOlderThan()` results for DELEGATED state. But that method only returns OPEN and ACKNOWLEDGED commitments — DELEGATED is terminal. The filter was dead code that could never find anything. We fixed it to query HANDOFF ledger entries directly.

## Formal verification as trace checking

The five verification properties (liveness, safety, deadlock freedom, fairness, evidence completeness) are documented in CTL/LTL notation but implemented as Java predicates querying the ledger. There's a real semantic gap here worth naming: CTL says "for all paths and all futures, this property holds." A Java predicate says "in this time window, no violations were observed." The first is a system proof. The second is a historical check. We're explicit about the distinction in the spec because consumers who see `AG(OPEN → AF(FULFILLED))` might assume formal guarantees they're not getting.

The fairness property surfaced an interesting limitation in the routing metadata. V2003 captures which agent was selected and how many candidates there were — but not which agents were in the candidate pool. The Gini coefficient over selection frequency is a proxy: it detects concentration in the winner column but can't prove unfair distribution across the candidate field. Adequate for flagging, not for formal fairness proofs. Honest about its limits.

## What this opens up

The governed yield pattern is ready for the engine side — judgment COMMANDs with `role:reviewer` targets route through the existing RoutingBridge, and the attestation feedback loop activates the moment engine#998 starts dispatching VERIFIED events. No qhorus changes needed when that lands.

The per-capability trust gap (an agent trusted for `code_review` but not `legal_review`) is filed as casehubio/eidos#145. That's where it belongs — the capability model lives in eidos, not qhorus. But until it lands, standalone deployments get uniform trust thresholds. For regulated domains handling clinical or legal judgments, that's a known compliance limitation worth watching.

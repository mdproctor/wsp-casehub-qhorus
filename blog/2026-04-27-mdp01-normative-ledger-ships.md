---
layout: post
title: "The Normative Ledger Ships — and It Turned Out to Be More Than Logging"
date: 2026-04-27
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [ledger, normative-layer, trust, eigentrust]
---

When we started this session the plan was clear: extend the ledger to record all nine message types, not just EVENT. The CommitmentStore had just shipped as the live view of obligation state. The ledger would be the immutable historical record alongside it.

The implementation went smoothly. `MessageLedgerEntry` replaces the old EVENT-only entity with a single JPA subclass covering all nine types — telemetry fields nullable and EVENT-only, normative fields for everything else, `causedByEntryId` linking DONE/FAILURE/DECLINE/HANDOFF entries back to their originating COMMAND. The write path is now unconditional: `LedgerWriteService.record()` fires for every `sendMessage` call with no conditional branching. `list_ledger_entries` replaces `list_events` as the query surface, and all existing tests stay green.

But the more interesting work happened around the edges.

Partway through, I pushed back on my own framing. Every time I described the ledger in terms of "better audit logging" or "richer queries" it felt wrong — too IT, too middleware. The thing we were building wasn't logging. It was infrastructure that makes AI agents accountable. That's a different claim.

We worked out the four-layer theoretical foundation: speech act theory for what messages mean, deontic logic for what agents are obligated to do, defeasible reasoning for how obligations can be overridden or transferred, social commitment semantics for observable commitments as provable contracts. These aren't decorations. They're why the nine-type taxonomy is *complete* — no obligation type is missing — and *closed* — every obligation has exactly one terminal resolution path. The theory guarantees correctness in a way ad-hoc design cannot.

That framing led to `docs/normative-layer.md`. It positions Qhorus as a governance methodology rather than middleware, explains each layer in plain terms, and grounds it in a concrete insurance claim scenario that hits every capability: thirteen steps, four channels, nine agents, all nine message types, the full Commitment lifecycle, two Solvency II regulatory filings, and a surveyor who goes silent and never responds.

The trust models were a late discovery. `quarkus-ledger` already implements two trust scoring mechanisms: Bayesian Beta for per-actor trust from direct attestation history, and EigenTrust for transitive propagation through the peer review network. Both are computed from `LedgerAttestation` records — the same ledger infrastructure as the obligation chain. Trust accretes from observable behaviour, not from configuration. CaseHub has applied this to worker registration, making it ecosystem-wide. A worker introduced by a high-trust provisioner inherits a stronger initial deontic standing through EigenTrust propagation — provenance as a trust chain.

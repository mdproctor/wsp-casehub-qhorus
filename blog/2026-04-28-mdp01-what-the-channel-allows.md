---
layout: post
title: "What the Channel Allows"
date: 2026-04-28
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [normative-layer, trust, transactions, mcp, design-decisions]
---

The last entry was about querying obligations from the ledger. This one is
about closing four more gaps — and one of them surfaced a production bug that
had been hiding behind test infrastructure for days.

## Channels with Opinions

The NormativeChannelLayout — work, observe, oversight — was always the
recommended pattern. It wasn't enforced. The observe channel was supposed to
carry only EVENT messages, but nothing stopped an agent from posting a COMMAND
to it and creating an obligation the CommitmentStore would dutifully track.

The `allowed_types` field changes that. It's a comma-separated list on the
channel entity — `"EVENT"` for observe, `"QUERY,COMMAND"` for oversight,
null for work. `StoredMessageTypePolicy` reads it and enforces it at two
points: at the MCP tool layer before any service call (fail-fast), and inside
`MessageService.send()` as a safety net for non-MCP callers.

The SPI pattern is the same one we used for `MessageTypePolicy` last month.
`@DefaultBean @ApplicationScoped` as the default, replaceable with
`@Alternative @Priority` for custom logic. A lambda compiles as a valid
implementation — `(channel, type) -> {}` to allow everything.

The 3-channel normative layout is now a first-class Qhorus concept. We added
`examples/normative-layout/` to the examples tree — 27 CI tests (no LLM),
deterministic, importable by Claudony and CaseHub as the Layer 1 reference.
The Secure Code Review scenario: two agents, three channels, full commitment
lifecycle. Researcher posts DONE, reviewer picks it up, queries for clarification,
receives RESPONSE, posts DONE. Every obligation discharged. Every EVENT on the
right channel.

## The Bug That Tests Can't See

Issue #123 added automatic `LedgerAttestation` entries when commitments reach
terminal state. DONE → SOUND, FAILURE and DECLINE → FLAGGED. The trust model in
quarkus-ledger computes a Bayesian Beta score from these attestations — or it
should.

We handed the brief to the ledger session and picked it up in quarkus-qhorus the
same day. The ledger Claude implemented the attestation writing inside
`LedgerWriteService.record()`. Tests passed. 899 tests, 0 failures.

Then Claude read the implementation.

The approach: query `CommitmentStore.findByCorrelationId()` inside the
`REQUIRES_NEW` transaction to confirm the commitment is terminal before writing
the attestation. Reasonable. Except `REQUIRES_NEW` *suspends* the outer
transaction. The commitment state update — `CommitmentService.fulfill()` running
in the outer `@Transactional(REQUIRED)` — was uncommitted when the new connection
opened. `findByCorrelationId()` returned OPEN. Always. The attestation was always
skipped.

The InMemory stores used in integration tests ignore transaction boundaries. They
return the latest in-memory state regardless of what's committed. So every test
that exercised this path passed — the InMemory store reflected the FULFILLED state;
the production JPA store would have returned OPEN. The bug only lived in the gap
between testing infrastructure and actual database behaviour.

The fix doesn't touch CommitmentStore at all. We derive the attestation verdict
from `MessageType` directly — DONE means the obligation was discharged, no
database query required. The originating COMMAND's ledger entry is already being
looked up to set `causedByEntryId`; we reuse that result for the attestation. One
query, two purposes, correct transaction semantics.

We also fixed something in quarkus-ledger itself. The `confidence` field on
`LedgerAttestation` was stored but never read — `TrustScoreComputer` used
`recencyWeight` alone, ignoring confidence entirely. DONE with confidence 0.7 and
DONE with confidence 1.0 produced identical trust scores. I wanted that fixed
before wiring the attestations, so the confidence values we assign actually mean
something. The fix: `weight = recencyWeight × clamp(confidence, 0, 1)`. Existing
attestations without explicit confidence default to 1.0 via a schema column default
— backward compatible.

## Persona, Not Session

Issue #124 addresses a subtler trust problem. Trust scores accumulate keyed by
`actorId`. `LedgerWriteService.record()` was writing `actorId = message.sender` —
a Qhorus instance ID like `claudony-worker-abc123-session42`. Session-scoped.
Every new tmux session starts from zero trust, even if the same AI persona ran
hundreds of successful cases in prior sessions. EigenTrust computes over instance
IDs, not personas.

`InstanceActorIdProvider` is the SPI. One method: `String resolve(String instanceId)`.
Default is identity — existing behaviour preserved. Claudony will provide the real
mapping from `SessionRegistry` when it integrates. The resolved actorId flows
into both the ledger entry's `actorId` field and the attestation's `attestorId`,
so both the act and the evaluation point at the persona.

## Decisions Without Users

There's a backlog of MCP consistency issues (#121) — eight design questions that
have been sitting as "for review" for weeks. I made all eight decisions this
session, but only implemented the non-breaking ones.

The framing matters: we have no users. Nothing shipped. Nothing that can break. I
was defaulting to "keep as-is to avoid churn" on several of them — but that's the
wrong prior when you're pre-1.0 and the whole point is to get the API right. So I
reset the lens: what's the most intuitive design if we were starting from scratch?

A few outcomes. The artefact vs. data split — `share_data` vs. `claim_artefact`
for the same underlying entity — goes away next session. Everything becomes
`artefact`. Explicit claim/release becomes auto-managed. The chunked upload API
gets a clean three-step replacement. Observer registration folds into `register`
with a `read_only` flag. `list_pending_waits` and `list_pending_approvals` become
`list_pending_commitments(type_filter?)`. All breaking. All deferred one session.

What shipped now: `delete_channel` (with a `force` guard — the FK has no CASCADE,
so we purge messages first), `get_instance`, and `get_message`. Three new tools,
straightforward, no ceremony. 944 tests, 0 failures.

## Where It Stands

Four issues closed. Five garden entries submitted — the REQUIRES_NEW/CommitmentStore
visibility bug went in first. The breaking MCP surface redesign is the next session's
opening move.

# Decisions — Issue #395: PROPOSE Message Type

## D1: Should PROPOSE be added to the MessageType taxonomy?

**Choice:** Yes — add PROPOSE as the 10th message type
**Alternatives:**
- Keep 9-type taxonomy, handle negotiation via COMMAND + metadata in projections — avoids precedent for type expansion but produces envelope/content separation violations and infrastructure misbehavior (trust gate, type policy, watchdog, ledger queries)
**Rationale:** PROPOSE fills a genuine commissive gap in Searle's taxonomy (STATUS is assertive, not commissive). It has different fulfillment semantics from both COMMAND and QUERY (RESPONSE does not auto-fulfill). The same justification that separates QUERY from COMMAND — different fulfillment semantics — applies to PROPOSE. The FIPA precedent concern is addressed by a principled stopping criterion: new types justified only when occupying a unique cell in the Searle-category × deontic-effect matrix.
**Trade-offs:** Every switch statement, test, consumer, protocol, and projection that handles MessageType must be updated. Future pressure for additional types (APPROVE, VETO, BID) must be evaluated against the stopping criterion. ADR-0005 completeness argument must be revised.
**Exploration:** multi-agent-debate
**Status:** captured

## D2: RESPONSE non-fulfillment mechanism

**Choice:** Check in MessageService.dispatch() — split the `case RESPONSE, DONE ->` arm; for RESPONSE, look up the commitment's `messageType` before calling `fulfill()` and skip if PROPOSE
**Alternatives:**
- Check in CommitmentService.fulfill() — add incoming message type as parameter; fulfill() checks commitment.messageType() and skips PROPOSE+RESPONSE — pushes resolution-type semantics into CommitmentService, which currently only manages state transitions
**Rationale:** The dispatch switch is already where message-type-specific commitment behavior lives. CommitmentService.fulfill() should remain a pure state transition — it fulfills or doesn't. The decision about *when* to fulfill based on the interaction between incoming type and commitment type belongs in the dispatch orchestration layer. fulfill() already does a lookup by correlationId; adding a type-aware guard there would mix policy into the state machine.
**Trade-offs:** Adds a commitment lookup in the RESPONSE path before calling fulfill() — two lookups total (one in dispatch, one inside fulfill()). Could be optimized later if needed, but the dispatch already does DB work at this point so an extra query is negligible.
**Exploration:** quick
**Depends on:** D1 (PROPOSE must exist for the check to reference it)
**Status:** captured

## D3: Builder validation — correlationId enforcement for PROPOSE

**Choice:** Enforce in Builder.build() — PROPOSE case throws IllegalArgumentException if correlationId is null
**Alternatives:**
- Same as COMMAND (no enforcement) — consistent with existing COMMAND behavior but loses the "proposal without tracking is meaningless" invariant
**Rationale:** PROPOSE is inherently commitment-bound — a proposal IS a conditional commitment. Unlike COMMAND, which can be fire-and-forget without response tracking, a proposal without commitment tracking is genuinely meaningless. The asymmetry is justified by the semantic difference. PROPOSE also requires content at build time (a proposal without terms is meaningless) — both constraints are enforced together.
**Trade-offs:** PROPOSE is stricter than COMMAND at build time. This is an intentional asymmetry, not an inconsistency — it reflects the different nature of the speech acts.
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D4: ObligorTrustPolicy — should PROPOSE be trust-gated?

**Choice:** No trust gate for PROPOSE — anyone who can write to the channel can propose
**Alternatives:**
- Shared trust gate with COMMAND — simpler but conflates directive authority with the ability to offer terms
- Separate trust threshold (`min-proposer-trust`) — more granular but more configuration surface for marginal benefit
**Rationale:** PROPOSE is a commissive, not a directive. The trust gate exists to prevent agents from directing action without sufficient authority. Proposals don't direct action — they offer terms the receiver can freely accept or reject. Channel ACLs (`allowedWriters`) already control who can write at all. The receiver's DECLINE is first-class — no proposal can impose obligation without explicit acceptance.
**Trade-offs:** A low-trust agent can spam proposals. Mitigation: rate limiting (already applies to all types except EVENT) and channel ACLs.
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D5: CorrelationIntegrityChecker resolution matching for PROPOSE

**Choice:** RESPONSE on a PROPOSE commitment generates no advisory — it is a valid non-fulfilling interaction (counter-proposal reference, clarification). DONE on PROPOSE generates no advisory (acceptance). Only truly wrong combinations are flagged.
**Alternatives:**
- Flag RESPONSE on PROPOSE as advisory ("RESPONSE does not fulfill PROPOSE — use DONE for acceptance") — informational but noisy for negotiation channels where RESPONSE is the expected conversational flow
**Rationale:** RESPONSE on PROPOSE is a designed interaction — the whole point of PROPOSE's distinct fulfillment semantics is that RESPONSE is valid but non-fulfilling. Flagging it as an advisory would generate noise in every negotiation channel. The checker should only flag genuinely wrong resolutions (e.g. using STATUS to resolve a PROPOSE).
**Trade-offs:** Agents that accidentally send RESPONSE expecting it to accept a proposal will get no warning. Mitigation: the commitment stays OPEN, which surfaces in watchdog and ledger queries — the agent learns fast.
**Exploration:** quick
**Depends on:** D1, D2
**Status:** captured

---
title: "The commissive that wasn't"
date: 2026-08-14
author: Mark Proctor
type: essay
tags: [speech-act-theory, message-taxonomy, negotiation, FIPA, design-decision]
issue: 395
---

When we shipped the 9-type message taxonomy in April, ADR-0005 made a completeness argument: every message type maps to exactly one obligation lifecycle transition, no transition is unrepresented, no type covers two transitions. The taxonomy covers Searle's five illocutionary categories — directives (QUERY, COMMAND), assertives (RESPONSE, DECLINE, STATUS), commissives (STATUS), declarations (HANDOFF, DONE, FAILURE), and expressives (not applicable for machine agents). Formally complete.

Three months later, the [blocks](https://github.com/casehubio/blocks) project started building a NegotiationProjection — a reusable channel projection for proposal/counter-proposal exchange. The natural way to express "I'll do X if you agree" is to send a proposal. The available workaround is to send a COMMAND and carry the proposal semantics in content metadata. We ran a structured debate on whether this is acceptable or whether qhorus needs a tenth type.

The answer is that the taxonomy has a genuine gap, and the gap was hiding in the completeness argument itself.

## STATUS is not a commissive

ADR-0005 maps STATUS to Searle's commissive category. But look at what STATUS actually does: its commitment operation is "No new commitment; extends deadline." Its deontic effect is "Extends: open COMMAND obligation window." A Searlean commissive creates a *new* conditional commitment — "I will deliver the report by Friday" binds the speaker to something they were not previously bound to. "I am still working on the report" does not. It asserts the current state of an existing obligation. That is an assertive, not a commissive.

The taxonomy's completeness proof covers four of Searle's five categories. The fifth — commissives — is occupied by a misclassified assertive. The commissive slot is empty.

## What COMMAND-as-proposal actually breaks

The blocks workaround uses COMMAND with an `entryType` metadata field to distinguish proposals from directives. The argument for this being "architecture working as designed" goes: the commitment envelope (structured, infrastructure-reads) carries obligation semantics, the LLM payload (free text, opaque) carries domain semantics, and the projection layer is exactly where protocol-specific interpretation belongs.

This argument has a problem. If the infrastructure only needs to know "does this message create an obligation on the recipient?", then QUERY and COMMAND should be a single type. They are not. They are separate because they have different *fulfillment semantics* — RESPONSE discharges a QUERY obligation; DONE discharges a COMMAND obligation. The infrastructure already encodes more than lifecycle transitions. It encodes how obligations resolve.

PROPOSE resolves differently from both. RESPONSE on a PROPOSE commitment should *not* auto-fulfill — the expected response to a proposal is often a counter-proposal, not acceptance. Only DONE (explicit acceptance) should fulfill. The same justification that split REQUEST into QUERY and COMMAND applies here.

Beyond fulfillment semantics, the workaround produces concrete infrastructure misbehavior:

- **ObligorTrustPolicy** gates COMMANDs with named targets — it checks whether the sender has sufficient trust to *direct* the target. A proposal does not direct action; it offers terms. An agent permitted to propose but not command cannot be modeled.
- **MessageTypePolicy** cannot discriminate. A channel with `deniedTypes = [COMMAND]` blocks proposals. A channel with `allowedTypes = [COMMAND]` permits both. No granularity.
- **The watchdog** fires OBLIGATION_FAN_OUT on unacknowledged proposals because they look like unacknowledged commands.
- **Ledger queries** degrade. "How many proposals were made?" requires content parsing across every COMMAND entry. With a PROPOSE type it is a single `WHERE message_type = 'PROPOSE'`.

Each of these forces the consumer to build shadow type discrimination in content metadata — violating the foundational principle that the infrastructure operates exclusively on the envelope.

## The FIPA lesson and the stopping criterion

The strongest counter-argument is precedent. FIPA-ACL walked from a principled set of communicative acts to 22 types: `propose`, `accept-proposal`, `reject-proposal`, `cfp`, `inform`, `confirm`, `disconfirm`, `request`, `request-when`, `request-whenever`, `agree`, `refuse`, `cancel`, `subscribe`. ADR-0005 explicitly cites this as a negative example. Adding PROPOSE risks reopening the door that the 9-type design deliberately closed.

The precedent concern is legitimate. But PROPOSE has a principled stopping criterion that separates it from the FIPA tail.

FIPA's bloat came from within-category refinements. `inform`, `confirm`, `disconfirm` are three assertive sub-types. `request`, `request-when`, `request-whenever` are three directive sub-types. The qhorus 9-type design already handles within-category variation through projection-layer semantics — a NegotiationProjection can distinguish counter-proposals from initial proposals without a COUNTER-OFFER type, just as the existing ChannelProtocol SPI distinguishes round-robin from task-completion without dedicated types.

PROPOSE is not a within-category refinement. It is a *between-category* distinction. It occupies a unique cell in the Searle-category × deontic-effect matrix: the only genuine commissive (STATUS is not), creating a unique deontic effect (conditional sender-obligation that RESPONSE does not discharge). No other plausible candidate passes this test:

| Candidate | Why it doesn't qualify |
|---|---|
| ACK | Lifecycle transition on existing obligations — correctly handled by `acknowledgedAt` |
| APPROVE / VETO | HITL-specific refinements — correctly deferred to the human-in-the-loop design |
| COUNTER-OFFER / BID / WITHDRAW | Within-category refinements of the commissive — handled by projection-layer semantics over PROPOSE sequences |

The stopping criterion is falsifiable: a new type is justified only when it occupies a unique cell in the Searle-category × deontic-effect matrix that no existing type covers. PROPOSE is the last type that passes this test. If someone proposes an 11th type, the burden is on them to show which Searle category it fills that is currently empty.

## What changes

The commitment direction claim from the original issue — that PROPOSE "inverts" the obligor — turns out to be overstated. The Commitment entity tracks who must *respond*. For PROPOSE, the receiver must still respond (accept or reject), so `obligor = target` is the same as COMMAND. The conditional execution obligation (the proposer will do X if accepted) is a second-order commitment that the CommitmentStore does not model for either type. The real difference is fulfillment semantics, not obligation direction.

With that correction, the implementation is cleaner than the issue anticipated. The Commitment entity already tracks `messageType` on every commitment record — we discovered this during context gathering. No schema migration is needed for the commitment table. The change is in `MessageService.dispatch()`: RESPONSE on a PROPOSE-originated commitment skips `fulfill()`. Everything else — the obligation lifecycle states, the commitment state machine, the ledger — remains unchanged.

The taxonomy moves from 9 to 10 types. The obligation lifecycle is not expanded — PROPOSE obligations traverse the same seven states. ADR-0005's completeness argument is corrected: coverage over Searle's five categories becomes honest, and the stopping criterion is documented to prevent the FIPA slide. The cost is one enum value and a fulfillment semantic check. The benefit is that the normative layer can govern negotiation — a fundamental mode of multi-agent coordination — without forcing consumers to parse content and maintain shadow type systems outside the infrastructure.

# Advocate FOR PROPOSE Inclusion

## 1. STATUS Is Not a Commissive

ADR-0005 maps STATUS to Searle's commissive category: "committing to a future action." But the ADR's own formal specification betrays this classification. STATUS has commitment operation "No new commitment; extends deadline." Its deontic effect is "Extends: COMMAND obligation window; asserts continued intent."

A Searlean commissive creates a new conditional commitment. "I will deliver the report by Friday" is commissive. "I am still working on the report" is not — it is an assertive about the current state of an existing obligation. STATUS does not bind the speaker to anything new; it reports that a prior binding still holds. The ADR documents a commissive that creates no commitment, which is a contradiction in Searle's framework.

This matters because it means the taxonomy has no genuine commissive type. The commissive slot is empty. PROPOSE fills it: "I will do X if you agree to Y" is a textbook conditional commissive where the speaker binds themselves, contingent on the hearer's acceptance. The taxonomy claims completeness over Searle's five categories but currently covers only four. The fifth — commissives — is occupied by STATUS under a misclassification.

## 2. COMMAND Creates Real Problems, Not Aesthetic Ones

The blocks#104 workaround uses COMMAND for proposals. This is not merely inelegant — it produces incorrect obligation semantics that the infrastructure enforces incorrectly.

**Commitment direction is reversed.** COMMAND creates `C(receiver → sender, execute_and_report)`. The receiver becomes the obligor. But a proposal binds the *sender*: "I will do X if you agree." The correct commitment is `C(sender → receiver, do_X_if_agreed)`. When the NegotiationProjection sends a COMMAND-as-proposal, the infrastructure opens a commitment with the wrong party as obligor. The CommitmentStore tracks the receiver as owing execution. The watchdog fires CONVERSATION_STALL alerts because the "obligor" (actually the party being offered terms) has not responded with DONE. The obligation lifecycle is structurally inverted.

**The enforcement gate misfires.** `MessageService.dispatch()` applies `ObligorTrustPolicy` to COMMANDs with named targets — it checks whether the sender has sufficient trust to *direct* the target. But a proposal does not direct action; it offers terms. An agent with low directive trust but high collaborative standing should be permitted to propose but not command. The trust gate cannot distinguish these because the type system does not.

**MessageTypePolicy cannot discriminate.** A channel configured with `deniedTypes = [COMMAND]` (observation-only channel) blocks proposals along with commands. A channel configured with `allowedTypes = [COMMAND]` permits both. There is no granularity because the type system does not model the distinction. The NegotiationProjection compensates with metadata — an `entryType` field in message content that carries "proposal" semantics. This defeats the foundational design principle of the taxonomy: "The infrastructure operates exclusively on the envelope. It never reads content." Every proposal forces the infrastructure to violate its own contract.

**Ledger queries degrade.** "How many proposals were made vs. accepted in this channel?" requires content parsing across every COMMAND entry, inspecting metadata for entryType. With a PROPOSE type, it is `SELECT COUNT(*) FROM message_ledger_entry WHERE message_type = 'PROPOSE'`. The normative ledger was designed to answer accountability questions from typed envelope data. Content parsing reintroduces the ambiguity the taxonomy was built to eliminate.

## 3. "Message Sequences" Does Not Hold for Negotiation

The redesign spec states: "Negotiation patterns are handled through message sequences using existing types, not through dedicated message types." This argument works when a pattern is a *composition* of existing speech acts. Request-response-confirm is a sequence of a directive and two assertives — each step maps cleanly to an existing type because each step *is* an existing speech act performed in order.

Negotiation is not a composition. A proposal is a distinct illocutionary act with unique felicity conditions. Searle's felicity conditions for commissives require that the speaker intends to bind themselves to a future course of action contingent on some condition. No directive has this condition. No assertive has this condition. The "sequence" argument implicitly claims that a commissive can be decomposed into a sequence of directives and assertives. It cannot — that conflates illocutionary force with perlocutionary effect. The *effect* of a proposal (eliciting agreement) can be achieved through a sequence of commands and responses. But the *meaning* — the speaker's conditional self-binding — is lost. And meaning is what the normative layer is built to capture.

The ADR explicitly justifies DECLINE over "just use RESPONSE with refusal content" on precisely this ground: distinct illocutionary force warrants a distinct type. The same logic applies to PROPOSE.

## 4. FIPA Got This Right

FIPA's 22-type ACL was rightly criticised as over-specified. But `propose` (SC00036) is not one of the types that should have been cut. FIPA defines it as: "The action of submitting a proposal to perform a certain action, given certain preconditions." The FIPA Propose Interaction Protocol (SC00036) models the complete propose/accept/reject cycle as a first-class protocol because it recognised what Searle's taxonomy makes clear: proposing is a commissive, directing is a directive, and no amount of protocol composition turns one into the other.

Qhorus already preserves the distinctions FIPA drew correctly — QUERY vs. COMMAND mirrors FIPA's `query-ref` vs. `request`; DECLINE mirrors `refuse`; HANDOFF mirrors `proxy`. PROPOSE mirrors FIPA's `propose`. The pattern of selective adoption is consistent: keep what FIPA got right, discard the BDI semantics and the fine-grained distinctions within categories (e.g., `inform` vs. `inform-if` vs. `inform-ref`). PROPOSE is a between-category distinction (commissive vs. directive), not a within-category refinement. It belongs.

## 5. Cost to blocks#104 and Beyond

Without PROPOSE, the NegotiationProjection must:

- Parse content metadata to distinguish proposals from commands, violating the envelope/content separation
- Maintain its own commitment-direction logic because `CommitmentStore` tracks the wrong obligor
- Cannot participate in channel protocol enforcement (a `ROUND_ROBIN` protocol that should alternate proposals treats them as commands and enforces turn-taking on the wrong party)
- Cannot use `CorrelationIntegrityChecker` correctly (a counter-proposal is a new "COMMAND" with a new correlationId, but semantically it is a response-in-kind to the prior proposal — the causal chain is invisible to the checker)
- Cannot integrate with the watchdog (`OBLIGATION_FAN_OUT` fires on unacknowledged proposals because they look like unacknowledged commands)

Every future negotiation consumer — not just blocks#104 — inherits these problems. Auction protocols, contract negotiation, resource allocation bargaining, SLA negotiation between services: all require the commissive speech act, all will use the same COMMAND workaround, all will carry the same semantic debt.

## 6. The Minimality Counter-Argument

The strongest objection: the taxonomy was designed to be minimal and complete, and adding types creates precedent for unbounded expansion. This concern is legitimate but misapplied here.

The completeness argument in ADR-0005 proves coverage over the *obligation lifecycle state space* — created, extended, fulfilled, refused, failed, delegated, and not-a-communication-act. This proof is valid but incomplete. It covers obligation *lifecycle transitions* but not obligation *creation modes*. The taxonomy has two creation modes: QUERY (epistemic directive) and COMMAND (action directive). Both place the *receiver* under obligation. There is no creation mode where the *sender* places *themselves* under conditional obligation. This is not an expansion of the lifecycle — PROPOSE obligations would traverse the same states (OPEN, ACKNOWLEDGED, FULFILLED, DECLINED, FAILED, DELEGATED, EXPIRED). It is a new entry point into the same lifecycle, distinguished by who bears the obligation and under what conditions.

The precedent test is whether the candidate type occupies a unique cell in the Searle-category-by-deontic-effect matrix. PROPOSE does: it is the only genuine commissive (STATUS is not), and it creates a unique deontic effect (conditional sender-obligation). No other plausible candidate — ACK, APPROVE, VETO — passes this test as cleanly. ACK is a lifecycle transition on existing obligations (deferred correctly to `acknowledgedAt`). APPROVE and VETO are HITL-specific (deferred correctly to the HITL design). PROPOSE is categorically distinct: it fills the commissive gap.

The taxonomy moves from 9 to 10 types. The obligation lifecycle state machine is unchanged. The enforcement infrastructure gains a new entry point but no new states. The theoretical framework is strengthened, not strained — the Searle mapping becomes honest. The cost is one enum value, one `requiresCorrelationId()` addition, one commitment-creation path with reversed obligor. The benefit is that the normative layer can govern negotiation — a fundamental mode of multi-agent coordination — without forcing consumers to parse content and maintain shadow obligation tracking outside the infrastructure.

Minimality is a virtue when it preserves completeness. When minimality produces a gap that consumers must fill with metadata parsing and inverted commitment tracking, it has become a liability.

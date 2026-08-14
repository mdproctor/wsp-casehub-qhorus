# Mediator Synthesis — D1: Should PROPOSE be added to the MessageType taxonomy?

## Verdict: Include PROPOSE

### Decisive points for inclusion

**1. The commissive gap is real.** STATUS is not a commissive in the Searle sense. A commissive creates a new conditional commitment — "I will do X if Y." STATUS extends an existing obligation window — "I am still working on it." The ADR-0005 completeness argument maps five Searle categories but only genuinely covers four. PROPOSE fills the commissive slot with a textbook conditional self-binding: "I will do X if you agree."

**2. The envelope/content separation violation is architecturally significant.** The NegotiationProjection must parse content metadata (`entryType`) to distinguish proposals from commands. This violates the foundational design principle: "The infrastructure operates exclusively on the envelope. It never reads content." Projections should interpret domain content, not reconstruct speech act classification from payload metadata.

**3. Concrete infrastructure misbehavior.** Without PROPOSE: ObligorTrustPolicy misfires (proposals checked as directives), MessageTypePolicy cannot discriminate (deny COMMAND blocks proposals), watchdog OBLIGATION_FAN_OUT fires on unacknowledged proposals as if they were unacknowledged commands, ledger queries require content parsing.

### What the "against" position got right (incorporated)

**Commitment direction is NOT inverted.** The "for" advocate's claim that PROPOSE reverses the obligor is overstated. The commitment tracks who must *respond* — for PROPOSE, the receiver must still respond (accept/reject), so `obligor = target` is correct, same as COMMAND. The conditional execution obligation (proposer does X if accepted) is a second-order commitment the CommitmentStore doesn't model for either type. The difference between PROPOSE and COMMAND is in *fulfillment semantics* (RESPONSE doesn't auto-fulfill), not in obligation direction.

**The precedent concern is legitimate and requires a principled stopping criterion.** PROPOSE is justified only because it occupies a unique cell in the Searle-category × deontic-effect matrix (commissive + conditional-sender-obligation). Future candidates must pass the same test:
- ACK → lifecycle transition on existing obligations, not a new speech act category. Correctly deferred to `acknowledgedAt`.
- APPROVE/VETO → HITL-specific refinements within existing categories. Correctly deferred to HITL design.
- COUNTER-OFFER/BID/WITHDRAW → within-category refinements of the commissive, not between-category distinctions. Handled by projection-layer semantics over PROPOSE sequences.

### What the "against" position got wrong

**"COMMAND + metadata is the architecture working as designed."** If this were true, QUERY and COMMAND should be a single type — they both create obligations on the receiver to respond. They're separate because they have different *fulfillment semantics* (RESPONSE fulfills QUERY; DONE fulfills COMMAND). PROPOSE has yet another fulfillment semantic (RESPONSE does NOT fulfill). The same justification that separates QUERY from COMMAND separates PROPOSE from both.

**"Obligation lifecycle completeness is sufficient."** The completeness proof covers lifecycle *transitions* but not lifecycle *entry modes*. PROPOSE creates an obligation through the same lifecycle states but with different entry semantics (conditional self-binding vs. directive). This is the same argument that justified splitting REQUEST into QUERY and COMMAND in the original redesign.

## Implications for design

1. The issue's claim about "inverted deontic direction" should be reframed: PROPOSE differs from COMMAND in conditional fulfillment semantics, not in obligation direction
2. ADR-0005 must be updated: amend the completeness argument, correct the STATUS commissive classification, add PROPOSE with its Searle mapping
3. The stopping criterion (unique Searle-category × deontic-effect cell) must be documented to prevent precedent creep
4. All docs referencing the "9-type taxonomy" need revision

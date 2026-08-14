# Advocate AGAINST PROPOSE Inclusion

## 1. The Taxonomy Is Complete Over the Obligation Lifecycle — PROPOSE Adds No New Transition

ADR-0005 maps seven exhaustive obligation lifecycle states to message types: Created (COMMAND/QUERY), Extended (STATUS), Fulfilled (DONE/RESPONSE), Refused (DECLINE), Failed (FAILURE), Delegated (HANDOFF), and Not-a-communication-act (EVENT). The completeness claim is not aesthetic — it is structural. Every obligation that can exist between two agents passes through exactly these states, and every state has exactly one type that transitions into it.

PROPOSE does not introduce a new lifecycle state. A proposal creates an obligation on the recipient to respond — which is what COMMAND already does. The difference is supposedly that COMMAND is directive ("do this") while PROPOSE is commissive ("I will do this if you agree"). But from the obligation lifecycle's perspective, both create the identical structure: agent A sends a message, agent B is now obligated to respond (accept, decline, counter). The commitment record is `C(B, A, received, respond)` in both cases. The lifecycle states available after receipt are identical: B can fulfill (DONE/RESPONSE), refuse (DECLINE), delegate (HANDOFF), fail (FAILURE), or provide progress (STATUS).

If PROPOSE truly represented a new lifecycle transition, you would be able to name an obligation state that exists after PROPOSE but cannot exist after COMMAND. No one has done this, because no such state exists.

## 2. COMMAND + Metadata Is Not a Workaround — It Is the Architecture Working as Designed

The message-type-redesign spec is explicit: the two-part message structure separates the **commitment envelope** (infrastructure) from the **LLM payload** (application). The envelope carries obligation semantics — who must respond, by when, what lifecycle transitions are valid. The payload carries domain semantics — what the proposal contains, what terms are offered, what conditions apply.

When blocks#104 uses COMMAND with an `entryType` metadata field to carry proposal semantics, it is using the architecture exactly as intended. The projection layer is where protocol-specific semantics belong. A `NegotiationProjection` folds over a sequence of COMMANDs and RESPONSEs, interpreting the payload to distinguish proposals from counter-proposals from acceptances. This is the same pattern used everywhere: `REQUEST_RESPONSE` protocol folds QUERYs, `TASK_COMPLETION` folds COMMANDs, `ROUND_ROBIN` folds sender sequences. None of these required a new message type.

Calling this a "workaround" misidentifies the abstraction boundary. The infrastructure tracks obligations. The projection interprets meaning. A negotiation channel where COMMAND means "here is my proposal, you are obligated to respond" is not a semantic mismatch — it is a channel with a well-defined protocol enforced by a projection, which is the entire point of the projection system.

## 3. The Precedent Problem Is Not Hypothetical — It Is the FIPA Lesson

If PROPOSE merits its own type because it is a commissive speech act distinct from a directive, then the same logic demands: APPROVE (accepting a proposal is not the same as RESPONSE to a QUERY), COUNTER-OFFER (a commissive that modifies terms, distinct from PROPOSE which creates them), WITHDRAW (retracting a proposal is not the same as DECLINE, which refuses someone else's), VETO (blocking with authority is not the same as DECLINE, which is peer-level refusal), BID (a commissive in a competitive context, unlike PROPOSE which is bilateral). Each of these maps to a different Searle sub-category. Each has a plausible driving use case. Each is "not quite" captured by the existing types.

This is not speculation. FIPA-ACL walked this exact path and arrived at 22 communicative acts: `propose`, `accept-proposal`, `reject-proposal`, `cfp` (call for proposals), `inform`, `confirm`, `disconfirm`, `request`, `request-when`, `request-whenever`, `agree`, `refuse`, `cancel`, `subscribe`, and more. ADR-0005 explicitly cites FIPA as a negative example. The qhorus design "kept structured performatives, discarded BDI semantics and 22-type bloat." Adding PROPOSE reopens the door that ADR-0005 deliberately closed.

The counter-argument will be: "We can draw the line after PROPOSE." But PROPOSE has no principled stopping criterion that does not also exclude itself. If the criterion is "commissives deserve separate representation," you get APPROVE and COUNTER-OFFER. If the criterion is "types that create obligations differently," PROPOSE creates them identically to COMMAND. If the criterion is "types that negotiation requires," you need at minimum PROPOSE, ACCEPT, and COUNTER — one type is insufficient.

## 4. RESPONSE Non-Fulfillment Is a Lifecycle Change, Not a Type Change

The behavioral claim for PROPOSE is that RESPONSE should not auto-fulfill the commitment. But this is a property of the commitment lifecycle, not of the message type. Today, COMMAND opens a commitment and RESPONSE/DONE fulfills it. If there are cases where a response should not auto-fulfill — and negotiation is one — the correct fix is a commitment policy, not a new message type.

Consider: `CommitmentAttestationPolicy` already controls what happens at commitment resolution. A `NegotiationCommitmentPolicy` that holds commitments open until an explicit acceptance (a DONE with specific payload content) is a policy-layer change that works with any initiating type. It does not require the infrastructure to distinguish "directive obligation" from "commissive obligation" at the enum level — it requires the infrastructure to support configurable fulfillment conditions, which is a strictly more general and more useful capability.

Adding PROPOSE to solve this means the non-fulfillment behavior is permanently welded to a single type. A channel protocol, by contrast, can apply the same behavior to any message sequence. This is more powerful and more composable.

## 5. Searle Categories Are Not Infrastructure Categories

The "semantic mismatch" argument says: COMMAND is a directive (Searle category 4), PROPOSE is a commissive (Searle category 3), therefore they need separate types. This conflates the philosophical taxonomy with the infrastructure taxonomy. ADR-0005 maps Searle's categories as a validation of coverage, not as a 1:1 generation rule. STATUS is assertive. DECLINE is assertive. RESPONSE is assertive. Three types, one Searle category. HANDOFF is declarative. DONE is declarative. FAILURE is declarative. Three types, one Searle category. The mapping is many-to-one from types to lifecycle transitions, not one-to-one from Searle categories to types.

The infrastructure does not need to know whether the sender's illocutionary force is "I direct you to do X" or "I commit to doing X if you agree." It needs to know: does this message create an obligation on the recipient? (Yes, for both.) What are the valid responses? (RESPONSE, DECLINE, STATUS, HANDOFF, DONE, FAILURE — identical for both.) When is the obligation discharged? (That is a policy question, not a type question.)

Singh's social commitment notation makes this clear: `C(recipient, sender, received(PROPOSE), respond)` is structurally identical to `C(recipient, sender, received(COMMAND), respond)`. The debtor, creditor, antecedent, and consequent are the same. The commitment operation is CREATE in both cases. The commitment is the same commitment. Only the payload semantics differ — and payload semantics are application-layer concerns, handled by projections and protocols.

## 6. The Real Cost Is Not Aesthetic — It Is Structural

Adding a tenth type means updating every `switch` statement on `MessageType` (exhaustive matching in Java forces this). Every `MessageTypePolicy` evaluation. Every `CorrelationIntegrityChecker` rule. Every `CommitmentService` state transition. Every `A2ATaskState` mapping. Every `WatchdogConditionType` evaluation. Every `ChannelProtocol` implementation. Every `RenderableProjection` that inspects type. Every `CloudEventMapper` type string. Every consumer application. Every test. The Flyway migration to alter `CHECK` constraints. The MCP tool documentation. The normative-layer doc. The consumer guide. The contributor guide. ADR-0005 itself.

This is not a one-time cost. It is a precedent cost. Every future type addition — and PROPOSE guarantees there will be pressure for more — pays the same tax. The 9-type taxonomy's value is not that nine is a magic number. Its value is that it is stable. Consumers build against it, projections enumerate it, protocols compose over it, and none of them break when a new negotiation pattern emerges. That stability is a feature, and PROPOSE trades it for a convenience that the projection layer already provides.

---

The NegotiationProjection in blocks#104 is not struggling because the type system is incomplete. It is doing exactly what projections are designed to do: interpreting domain-specific message sequences over a complete obligation infrastructure. The right response to "COMMAND feels semantically wrong for proposals" is not to expand the infrastructure vocabulary — it is to name the channel protocol, document the convention, and let the projection carry the meaning. That is what the four-layer architecture was built for.

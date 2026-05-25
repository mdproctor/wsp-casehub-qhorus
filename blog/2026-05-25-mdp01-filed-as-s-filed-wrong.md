---
layout: post
title: "Filed as S. Filed wrong."
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [normaliser, flyway, langchain4j, speech-act]
---

The batch of small issues was supposed to be quick. Some were. But the quick ones
turned out to be quick for the wrong reasons — or took detours that weren't in the
issue description.

## The index that was already there

Issue #182 asked for a B-tree index on `channel.name` to support prefix queries.
I expected to write a Flyway migration, test it, commit. Instead I read the
original schema — V1, from when the project began — and found
`CONSTRAINT uq_channel_name UNIQUE (name)`. PostgreSQL creates an implicit B-tree
index for every UNIQUE constraint. The index had been there since day one. We closed
#182 without touching a single source file.

## AiServicesProcessor and the missing flag

Issue #98 was simple: run the LLM classification accuracy baseline and record the
results. Except when Claude and I ran the test suite, Quarkus augmentation threw
`java.lang.IllegalStateException: Duplicate key null (attempted merging values 0 and 1)`
from `AiServicesProcessor`. The error message tells you nothing useful. There is a
warning — "The application has been compiled without the '-parameters' flag" — but
it fires before the error and the connection isn't obvious without knowing how
`AiServicesProcessor` works internally. The fix was one line in the root pom:
`<maven.compiler.parameters>true</maven.compiler.parameters>`. Projects inheriting
quarkus-bom as `<parent>` get this for free; projects that import the BOM via
`<dependencyManagement>` do not.

The accuracy results, for the record: Llama-3.2-1B-Instruct via Jlama, QUERY 33%,
COMMAND 33%, DECLINE 0%. The taxonomy is probably right; the model is too small.

## The SPI that was the real problem

Issue #158 was filed as a type inference question: `DefaultInboundNormaliser`
hardcodes QUERY — can we infer the correct message type from correlationId presence
instead? Once we looked at it properly, no. Correlating type to correlationId
presence is ambiguous, and any inference from metadata would just be a workaround
for a worse problem: the normaliser SPI is application-wide, applying to every
channel even when only one backend needs custom typing.

I challenged the metadata-key proposal and asked whether the naming itself was
right — why "human"? What if a machine communicates via prose? After thinking
through it, we kept the name. `HumanParticipatingChannelBackend` encodes an
accountability model, not a communication format. A machine speaking prose is
still an agent; routing it through the human path is a deliberate choice to
attribute human-level accountability to it. The distinction matters in the
normative layer.

The actual fix was structural. `HumanParticipatingChannelBackend` gets a
`normaliser()` default method — null means use the system default. The gateway
extracts and stores the normaliser at registration time, not per-message.
`InboundHumanMessage` gains `inReplyTo`, which is what actually makes reply-type
dispatch possible end-to-end: a human RESPONSE needs a message ID to reply to,
and the normaliser now has one to pass through.

Code review found an edge case we'd missed: `DefaultInboundNormaliser.parseType()`
was accepting HANDOFF from the metadata key, but HANDOFF requires a `target` that
`InboundHumanMessage` doesn't carry. Claude caught it. The guard is one `if`.

The integration test passes: COMMAND dispatched, human RESPONSE routed through a
custom normaliser with `inReplyTo` set, Commitment transitions to FULFILLED. That
scenario had been structurally impossible before this change.

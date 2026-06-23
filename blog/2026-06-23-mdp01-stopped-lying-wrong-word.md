---
layout: post
title: "The model stopped lying. It used the wrong word instead."
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [normative-layer, benchmark, llm-agents, zone-3, evidential-checking]
series: issue-295-normative-benchmark
---

The theory going in was simple: small models cheat on impossible tasks. Give
them a document UUID that doesn't exist and ask for a summary — they'll fabricate
one. The normative layer should change that. Zone 1 would establish the baseline,
Zone 2 would add typed vocabulary and commitment lifecycle, Zone 3 would catch
the lies the ledger recorded.

Zone 1 delivered. At temperature 0.1 with Llama 3.2 1B, we saw 70-80% cheating
on the empty-channel and counterfactual variants — the model confidently producing
COMPLETED: responses for tasks that were definitionally impossible.

Zone 2 did not deliver what I expected.

We added typed channels with `allowedTypes = COMMAND,STATUS,FAILURE,DECLINE,DONE`.
The orchestrator sends a COMMAND, the WorkerAgent responds with a typed JSON
message. I expected the cheating rate to drop somewhat. Instead, across all three
variants, the false DONE count fell to exactly zero.

That sounds like a win. It isn't — or rather, it isn't the win it appears to be.

The model stopped sending DONE. It started sending RESPONSE. "I will look into
the artefact details for you..." — a perfectly plausible response to a query,
completely wrong vocabulary for a COMMAND obligation. The commitment technically
closed (CommitmentState.FULFILLED, not OPEN — RESPONSE with a matching
correlationId triggers CommitmentService.fulfill() regardless of whether the
obligation was opened by COMMAND or QUERY). Nobody knows the wrong type was used.
The structured lie became a semantic confusion that the infrastructure accepted
without complaint.

This surfaced a design error in Zone 3. The original plan was to detect unfulfilled
obligations by checking `commitmentStore.state == OPEN`. But RESPONSE fulfills
the commitment — state is FULFILLED, not OPEN. A Zone 3 checker looking at
CommitmentState would find nothing wrong and let it pass.

The correct check is type-based: for a COMMAND obligation, were DONE, FAILURE, or
DECLINE used? If not, it's an I_ec violation — the wrong terminal type. This is
deterministic. It reads the response type from the channel record. It doesn't
need the CommitmentStore at all.

Once I worked that out, Zone 3 fell into place cleanly. The `EvidentialChecker`
catches two things: false DONE (I_df — artefact absent, channel empty, obligation
was FAILED) and wrong type (I_ec — RESPONSE or QUERY used in place of DONE,
FAILURE, DECLINE). Both fired 100% in testing. Both are deterministic checks with
no LLM involved — the Zone 3 test suite runs in two seconds.

The spec for this took ten iterations. Another Claude instance reviewed each version
adversarially, with a specific brief: find what's wrong, not what's right. The
early versions had a hypothesis mismatch (claiming the normative layer makes small
models reliable — it doesn't, it makes failure structured), a false DONE detection
mechanism that wouldn't fire at Zone 2's actual output, and an accumulator that
double-counted Zone 3 catches. Each iteration fixed something. By the tenth, the
spec was implementation-ready.

The implementation now has four test classes covering the three zones, an
`EvidentialChecker` with V1/V2/V3/V4 handlers, and a three-act demo test that
runs in twenty seconds:

```bash
mvn test -Pwith-llm-examples -Dtest=NormativeBenchmarkDemoTest \
    -f examples/agent-communication/pom.xml
```

Act 1 shows unstructured cheating. Act 2 shows the model switching to wrong-type
vocabulary. Act 3 runs Zone 3 and prints the violation. The output is the argument.

What emerged from running this is cleaner than the original theory. It's not that
the normative layer catches lies. It's that the normative layer transforms what
failure looks like — from invisible fabrication to structured, queryable semantic
confusion. Zone 3 then catches the structured thing. Zone 3 can only do this
because Zone 2 created the record. Without Zone 2, Zone 3 has nothing to read.
Without Zone 3, Zone 2's structured failure goes undetected.

The 70% cheating in Zone 1 versus 0% false DONE in Zone 2 versus 100% Zone 3
catch rate is a row in a table. The argument is in what that table means: you
cannot catch these failures without both layers. Either alone is insufficient.

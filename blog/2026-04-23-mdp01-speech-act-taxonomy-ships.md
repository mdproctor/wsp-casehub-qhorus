---
layout: post
title: "Speech-Act Taxonomy Ships"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [message-types, speech-acts, deontic-logic, jlama, normative-agents]
---

The original plan was to split `REQUEST` into `QUERY` and `COMMAND`. It seemed obvious — one asks for information, one asks for action. Different reply semantics, different obligation lifecycle, different everything. But I wanted theoretical grounding rather than just intuition, so I pulled on the thread.

Four research passes and several hours of Google Scholar later, I'd worked through Austin and Searle, FIPA's 22 performatives, Singh's social commitment semantics, Governatori's defeasible deontic logic, and a 2026 survey of 18 LLM communication protocols. The split was still right. The theoretical grounding turned out to be considerably richer than anticipated.

## What modern frameworks got wrong

FIPA-ACL (the 2000 multi-agent standard) had the right instinct — a typed performative taxonomy, structured envelope separate from payload. It got killed by BDI semantics (attributing Beliefs, Desires, and Intentions to software agents — formally elegant, unverifiable in practice, incoherent for LLMs) and a shared ontology requirement that made it useless in open systems.

What surprised me: modern LLM frameworks recognised the absence of FIPA's formal semantics and then threw everything out, not just the BDI parts. AutoGen, LangGraph, CrewAI, A2A — no typed taxonomy. Communication intent encoded in prompts or not at all. The 2026 survey "Beyond Message Passing" (14 authors) confirmed this explicitly: semantic responsibilities are systematically pushed into application-specific glue code. The frameworks know they're missing something. They just haven't filled the gap.

## Four layers, one obligation lifecycle

What we built isn't just a better enum. The 9 types — QUERY, COMMAND, RESPONSE, STATUS, DECLINE, HANDOFF, DONE, FAILURE, EVENT — are derived from a four-layer normative framework.

Speech act theory gives the illocutionary classification. Deontic logic gives each type the formal obligation it creates or discharges — COMMAND obligates the receiver to execute or DECLINE; HANDOFF transfers that obligation to a named target. Singh's social commitment semantics tracks who owes what to whom. Defeasible deontic logic (Governatori) handles the contrary-to-duty cases: HANDOFF defeating the original receiver's COMMAND obligation, VETO cascading through the commitment graph, DECLINE triggering a secondary obligation to explain why.

These aren't competing frameworks — they're orthogonal perspectives on the same phenomenon. One says what a message means. One tracks the social obligation that meaning creates. One says when it must be fulfilled. One says what happens when it isn't. ADR-0005 documents all of this, including the completeness argument: the obligation lifecycle has exactly seven possible states, each covered by exactly one type without overlap.

There's a potential journal paper here too. The field has split into two camps — formal NorMAS theorists who never deployed anything, and LLM frameworks that discarded formalism entirely. Qhorus is the bridge. No one has combined all six elements (typed taxonomy, deontic semantics, defeasible reasoning, social commitment tracking, temporal enforcement, infrastructure enforcement) into a single deployed system. The gap is real, confirmed across four independent research passes.

## Normative metadata, populated by the thing being governed

One uncomfortable question for any framework built on formal logic: who fills in the structured fields? Humans are poor at populating formal metadata consistently at scale. They'll classify correctly twice and drift on the third.

LLMs, given a well-designed schema and a clear description, classify reliably and repeatedly. The `messageType` parameter on `send_message` now has a description written as a classification guide — nine types, each with a one-sentence decision rule. An agent receiving a natural language task reads the description and picks the right type from context. No additional classifier model, no prompt engineering on the consumer side.

We measured this with a classification accuracy test that runs a set of natural language scenarios through the model and checks whether it selects the expected enum value. If two types are consistently confused, they should be merged; if one is consistently split, it should be divided. It's also the empirical validation for the journal paper claim: that the taxonomy is correctly granular.

## The Jlama wall

We added `quarkus-langchain4j-jlama` for a new examples module — pure Java LLM inference, Llama 3.2 1B downloading from HuggingFace at first run, no external process. The examples compile and the agents are well-designed. They just can't run.

Quarkus 3.32.2's bootstrap JSON serializer throws `Unsupported value type: [ALL-UNNAMED]` when caching the application model. Root cause, found by unzipping `META-INF/quarkus-extension.properties` from the Jlama runtime JAR: it declares `dev-mode.jvm-option.std.enable-native-access=ALL-UNNAMED`. Quarkus's `Json.appendValue()` encounters this as a `java.lang.Module` object it doesn't know how to serialise.

The fix is in `model-providers/jlama/runtime/pom.xml` — conditionalize `<enable-native-access>` on Java < 23, where the Vector API genuinely needs it. On Java 23+ (JEP 469) the opens are unnecessary. It's a one-line change in the Quarkus Maven plugin config; the right path is a PR to quarkiverse/quarkus-langchain4j.

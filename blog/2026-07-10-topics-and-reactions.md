---
title: "Topics and Reactions — Zulip's Regret, Matrix's Pattern, and What Qhorus Gets Right"
date: 2026-07-10
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [qhorus, conversation-model, topics, reactions, design, normative]
---

Zulip has had an open issue since 2014 about creating a topic table in the database. Twelve years. Their original design stored topics as strings directly on the message table — simple for reads, painful for renames. Every rename fans out writes to every affected message. Every topic listing requires a GROUP BY across potentially millions of rows. They know it's wrong, they've documented exactly what they'd do differently, and they can't fix it because the migration cost on a production system with years of data is prohibitive.

We're designing qhorus's conversation model from scratch. No existing data, no migration constraint. The question was: learn from Zulip's mistake, or repeat it?

The answer turned out to be neither — and both.

## The Hybrid

A Topic entity gives you efficient listing (query the topic table, not GROUP BY on messages) and a natural home for metadata like resolved state. But a topicId foreign key on Message means every message poll — the hottest query path in the system — needs a JOIN to get the human-readable topic name. That's a real cost on the operation that happens most.

So we did both: a Topic entity for the registry and metadata, plus a denormalized `topic` string on each message for zero-JOIN reads. Rename updates both — the Topic record (one row) and the message strings (bulk, but infrequent). The denormalization is intentional, justified by the read/write asymmetry: topic reads happen on every message poll, renames happen rarely.

This is Zulip's desired architecture, minus the migration they can't do.

## The Ledger Question

The harder design problem was the ledger. Qhorus has an immutable audit ledger — every dispatched message creates a `MessageLedgerEntry` with SHA-256 tamper evidence. If a topic gets renamed, the message table reflects the new name but the ledger entries still record the original. Is that a problem?

No — it's the design.

The pattern is straight from Matrix's event DAG and classical CQRS: the ledger is the immutable write model (what happened at time T), the message table is the mutable read model (what the conversation looks like now). Topic rename is a mutation of the read model, not the event store. The ledger records the original topic at dispatch time — always. A rename emits a system EVENT in the ledger documenting the administrative action. An auditor can reconstruct the full history: "Messages 1–50 were dispatched in topic 'auth-review'. At time T2, the topic was renamed to 'auth-investigation'."

## Where Topics Meet the Normative Layer

Topics are organizational — they don't create obligations, attestations, or commitments. But they complement the normative layer in specific ways that matter for agent communication.

The existing normative channel layout uses `case-*/work`, `case-*/observe`, `case-*/oversight` for role-based structure. Topics add task-based structure within each role. Instead of creating a new channel for every sub-task (which explodes channel count), an orchestrator issues a COMMAND in topic "auth-analysis" and all related messages — STATUS updates, DONE, FAILURE — land in that topic by name. The correlation chain provides threading within the topic; the topic provides scoping across the channel.

Resolving a topic ("this conversation is done") is deliberately independent of obligation state. A topic can be resolved while obligations are still OPEN — a human decided to move on. All obligations can be FULFILLED while the topic stays unresolved — work done, conversation continues. This decoupling is correct: obligations are formal (normative), topic resolution is informal (organizational).

## Reactions: The Non-Normative Signal

Reactions sit deliberately outside the speech-act taxonomy. A thumbs-up on a STATUS message is a lightweight acknowledgment — "seen" — without the formal weight of a RESPONSE, which triggers commitment resolution. The boundary matters: humans observing agent conversations want to signal awareness without creating obligations.

We deferred `moveTopic` and `mergeTopics`. Move changes `message.channelId`, which breaks `Commitment.channelId` references — a normative integrity violation. Merge mixes correlation chains across independent conversations. Rename is safe because it touches an organizational field, not a normative one.

## What Comes Next

The conversation hierarchy is taking shape: Space → Channel → Topic → Thread. Topics are the missing middle layer — the one that makes agent communication navigable. The next pieces are Rich ArtefactRef (typed document references replacing the current `List<UUID>`), Channel Membership (who's in the channel, distinct from ACL), Presence (heartbeat-driven availability), and Space (organizational hierarchy replacing the naming-convention approach).

Further out, there are conversation-level aids that would genuinely help LLMs coordinate: structured content schemas per topic (so agents know the expected message format without negotiating it in natural language), progress signals as first-class metadata (so orchestrators can make intelligent timeout decisions), and making the engine's choreography visible in the conversation layer — not enforcing it (that's the engine's job) but rendering it as navigational context in the chat UI.

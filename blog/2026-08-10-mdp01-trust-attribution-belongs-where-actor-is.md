---
layout: post
title: "Trust attribution: the record belongs where the actor is"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [trust-model, attestation, design]
---

The qhorus trust model had a gap. Attestations — the SOUND/FLAGGED verdicts that feed Bayesian Beta trust scores — were written on the COMMAND entry. The COMMAND entry's `actorId` is the requester, the agent who asked for the work. The obligor, the agent who actually did the work and sent DONE or FAILURE, never had their trust score affected.

An agent could consistently produce terrible DONE messages with no trust consequence. The signal flowed to the wrong actor.

I started with what seemed obvious: add an `obligorId` field to `LedgerAttestation` in casehub-ledger. One column, one query change. Clean.

Claude ran a decision review and caught the flaw. Dual attribution from a single verdict conflates two independent quality signals. When agent A sends a COMMAND and agent B sends DONE, the verdict is SOUND. Under the original design, both A and B get SOUND — but A's "delegation quality" is being measured by B's work product. A requester who delegates to strong agents accumulates positive trust, regardless of their own delegation behaviour. That's noise, not signal.

The review also flagged a more fundamental issue: `obligorId` leaks qhorus-specific commitment semantics into a domain-agnostic ledger model. Other consumers might call the same concept "assignee" or "executor." The field name bakes in one domain's vocabulary.

The fix was cleaner than the original: write a second attestation on the terminal entry itself. The terminal entry (DONE, FAILURE, DECLINE) already has `actorId = obligor`. Existing trust queries — which find attestations on entries where `entry.actorId` matches the actor — work unmodified. Zero changes to casehub-ledger. No new columns, no new queries, no cross-repo coordination.

A self-attestation guard handles the edge case where requester and obligor are the same actor. One `if (!resolvedActorId.equals(priorMsg.actorId))` check. No duplicate attestation.

The broader pattern: when you need to attribute an existing signal to a second actor, don't add a foreign key pointing at them. Write a second record linked to an entry whose `actorId` IS that actor. The query model stays unchanged; each record has exactly one attribution target. The domain-agnostic model stays domain-agnostic.

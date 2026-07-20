---
layout: post
title: "Protocols Are Feedback, Not Firewalls"
date: 2026-07-20
type: phase-update
entry_type: note
subtype: diary
projects: [Qhorus]
tags: [protocol-enforcement, spi, advisory, dispatch-pipeline]
---

# Qhorus — Protocols Are Feedback, Not Firewalls

**Date:** 2026-07-20
**Type:** phase-update

---

## What I was trying to achieve: pluggable message sequence validation

Qhorus channels have semantics — BARRIER forces quorum, COLLECT accumulates, LAST_WRITE enforces single-writer — but no constraints on message *sequences*. Two agents can exchange STATUS messages indefinitely without progress. The commitment lifecycle catches COMMAND→terminal and QUERY→RESPONSE obligations, but everything else is unconstrained.

PwC's research showed 7x accuracy gains through structured orchestration. The pathology watchdog validated the concern empirically — LOOP_DETECTED, CONVERSATION_STALL, ECHO_CHAMBER all fire on real coordination breakdowns. Protocols address the same pathologies proactively, at dispatch time, before they develop.

## The enforcement question

The design decision that shaped everything else: should protocols hard-block or advise?

I worked through this from first principles. The existing hard-enforcement gates in the dispatch pipeline — ACL, rate limiter, obligor trust, MessageTypePolicy for COMMAND/QUERY — each protect invariants where violation causes *structural damage*. Orphan commitments, unauthorized writes, resource exhaustion. Protocol violations are different. A message that violates turn-taking isn't a security breach or resource threat — it's a coordination quality issue.

Hard-rejecting an LLM agent's message creates brittleness. The agent retries, potentially in a loop worse than the one we're preventing. Or it gives up entirely, which may be worse than the violation. Advisory gives the same information — "you spoke out of turn" — without the error-recovery pathology.

The "held until your turn" semantics the issue described for round-robin aren't achievable without a message queue. True holding means buffering and deferred dispatch — a fundamentally different architecture. Rejection is a poor substitute. Advisory is honest feedback.

## What we built

A `ChannelProtocol` SPI in `api/spi/` — one method: `evaluate(ProtocolContext)` returns advisory strings. The `ProtocolContext` carries everything a protocol needs: recent messages (bounded lookback, oldest-first), active commitments, the incoming message type and sender, and the channel's declared participants. Two shared queries per dispatch — one for message history, one for commitments — feed all active protocols.

Four built-in protocols:

**REQUEST_RESPONSE** and **TASK_COMPLETION** are thin wrappers over the commitment store. They surface existing obligation state at dispatch time — "you have 3 unanswered QUERYs" — rather than waiting for the watchdog to fire after the fact. The value is speed of feedback, not new detection.

**ROUND_ROBIN** enforces turn-taking. It requires explicit participants — deriving turn order from message history produces non-deterministic ordering that changes as the lookback window slides. The design review caught this: turn-taking without a defined order is meaningless.

**CONTRIBUTION_REQUIRED** detects when one sender monopolises the channel. No artificial "rounds" — it counts consecutive messages from the same sender and advises when others haven't contributed. Participants can be explicit or derived from the lookback window.

## The design review sharpened the spec

The adversarial review ran four rounds and raised 17 issues. Several were genuine gaps:

The Flyway migration was numbered V38, which was already taken by channel_summary. The reviewer's codebase scan caught the collision before it reached a database.

`ProtocolContext` originally didn't carry commitment data — the REQUEST_RESPONSE and TASK_COMPLETION protocols would have needed CDI injection to query the commitment store. The reviewer proposed making the context self-contained: pre-query commitments once in the dispatch pipeline, pass them through. Protocols become pure functions of their context. No injection, simpler testing, one query instead of N.

The spec said `recentMessages` without specifying ordering. Oldest-first is the right choice — protocol analysis reads naturally as a chronological sequence — but the store queries `ORDER BY id DESC` for efficiency. The reversal happens before populating the context.

The `ProtocolRegistry` follows the `ProjectionRegistry` pattern exactly: CDI discovery at startup, duplicate name validation, unknown names warned at dispatch. Channels declare protocols as a `List<String>` — same pattern as `allowedWriters`, `barrierContributors`.

The watchdog detects and alerts after the fact. Protocols advise at dispatch time before pathologies develop. They're complementary — the watchdog fires when protocols are absent or ignored. The infrastructure enforces; the LLM reasons and adjusts.

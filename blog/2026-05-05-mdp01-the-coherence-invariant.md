---
layout: post
title: "The Coherence Invariant"
date: 2026-05-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [gateway, architecture, a2a, normative]
---

Qhorus channels were built for agent-to-agent communication. Adding external transports — WhatsApp, Slack, a Claudony panel — meant introducing a gateway layer, which sounds straightforward until you ask: what happens when two humans reply on two different platforms to the same channel?

The answer is that you have two conversations. An agent responding to one reply has no way to know it contradicts the other. I'd call this a split-conversation problem. It's not a race condition; it's a coherence failure built into the design. The fix is architectural: a channel may have at most one `HumanParticipatingChannelBackend`. Not a soft limit — a hard constraint enforced at registration time with a dedicated exception. Observers can fan out freely; participants cannot.

We worked out the vocabulary during brainstorming. The existing `ActorType` enum in `casehub-ledger` already had the three categories we needed: `HUMAN`, `AGENT`, `SYSTEM`. So `AgentChannelBackend`, `HumanParticipatingChannelBackend`, and `HumanObserverChannelBackend` map directly onto the same taxonomy used by the ledger's `ActorTypeResolver`. The backend type names, the ledger entries, and the resolver now share one vocabulary. That alignment was a genuine discovery — I went looking for a naming convention and found it was already there.

## The A2A Problem

A2A uses `role: "user"` for the initiating party. That party might be a human, an AI orchestrator, or an automated pipeline — A2A doesn't distinguish. From a pure messaging standpoint this is fine. From a normative governance standpoint it's a problem: trust scores, attestation weights, and post-incident forensics all depend on correctly classifying who acted.

The current `ActorTypeResolver` in `casehub-ledger` handles A2A's `role: "agent"` via catch-all and classifies it as `HUMAN`. That's a silent bug in every A2A agent message written to the ledger today. We created a companion issue in `casehub-ledger` to add explicit rules, but the broader design point is that A2A isn't a transport backend — it's a protocol multiplexer that carries both actor types and needs a bridge, not a simple adapter. The `A2AChannelBackend` is tracked separately for this reason.

## What the Review Caught

We wrote the gateway with a `synchronized` block missing. The check for a second `HumanParticipatingChannelBackend` read the list and then added to it — two steps, not one. Two concurrent registrations could both pass the check before either reached the add. Claude flagged this as a TOCTOU race during the code review. It's the kind of thing that works perfectly in every single-threaded test and fails in production under the exact conditions that matter.

The other issue was subtler. The original design said to route `sendMessage()` through `channelGateway.post()`. The problem: `sendMessage()` is `@Transactional` and already calls `messageService.send()` to get back the persisted entity for the tool response. Routing through `post()` would call `messageService.send()` again inside the backend — double persist, same message, same transaction. The fix was to split: keep `messageService.send()` in the tool method for persistence, add `channelGateway.fanOut()` for the external-only dispatch. `post()` became package-private, test-only, and clearly documented as a trap for anyone who calls it in the future.

The gateway is now wired, reviewed, and pushed. Qhorus channels can route to any backend without changing the normative layer — everything still flows through `MessageService` and `LedgerWriteService` regardless of which transport delivers it.

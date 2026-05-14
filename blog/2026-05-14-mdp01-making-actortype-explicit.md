---
layout: post
title: "Making ActorType Explicit"
date: 2026-05-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [a2a, actortype, ledger, governance]
---

A2A sends bare role strings as actor identifiers. `role: "agent"` arrives in Qhorus as the literal string `"agent"`. The `ActorTypeResolver` in `casehub-ledger` classifies actors by pattern: `"agent:*"` prefix → `AGENT`, versioned persona → `AGENT`, `"system"` or `"system:*"` → `SYSTEM`. Everything else → `HUMAN`.

`"agent"` matches nothing. Every A2A agent message was silently classified as `HUMAN` in the ledger — trust scores, attestation weights, post-incident forensics all corrupted for any A2A interaction. The fix was two `if` statements added before the catch-all: `"agent"` → `AGENT`, `"user"` → `HUMAN`. Trivial to write, significant to notice.

That dealt with the ledger side. Then came the harder part.

## Identity and type are two different problems

Qhorus's A2A integration had a design question at its centre: when a message arrives with `role: "user"`, how do you decide whether the sender is actually a human?

A2A `role: "user"` just means "the initiating party." That could be a human, an AI orchestrator delegating to a specialist, or a scheduled pipeline. Misclassifying it is a governance error — the wrong actor gets the wrong trust profile in the audit trail.

I brought Claude in to work through the resolution chain in `A2AActorResolver`. Six steps in order: an explicit `x-qhorus-actor-type` header, a Qhorus Instance registry lookup on `metadata.agentId`, an A2A Agent Card URL check in metadata, then `ActorTypeResolver.resolve(agentId)` for persona and system patterns, and finally the conservative default of `HUMAN`. For `role: "agent"`: unconditional `AGENT`, no chain runs.

The interesting design call was about where the sender ID for steps 2–5 comes from. A header would work but custom HTTP headers don't survive A2A message relay. Metadata from the message body does. We settled on that split: header for the explicit override at step 1, metadata for identity signals at steps 2–4.

## Making actorType explicit at write time

As we worked through the design, a structural problem became visible. `MessageService.send()` didn't take an `ActorType` parameter. The type was derived later — in `LedgerWriteService` — by running the sender string through `ActorTypeResolver` again via `InstanceActorIdProvider` enrichment. For Claudony sessions, where the sender is an opaque session ID like `"claudony-session-abc123"`, that enrichment chain is what makes the classification correct. Skipping it means misclassifying your own agents as `HUMAN`.

The fix was to make `ActorType` explicit at message creation time. Three overloads of `MessageService.send()` became one canonical signature with `ActorType actorType` as the final required parameter. Every call site declares what it knows. `LedgerWriteService` reads `message.actorType` directly. `QhorusMcpTools` now injects `InstanceActorIdProvider` and resolves the enrichment chain before calling the service. Roughly ninety call sites updated.

It looks like a refactor. It's actually correctness: the type was always being derived somewhere, just not where the context existed to do it right.

Claude caught one gap in code review. The LAST_WRITE channel path in `QhorusMcpTools` has an early-return branch that mutates existing message fields before returning. The `resolvedActorType` was computed after that block, so overwrite-path messages never had their type set. One line to move — but exactly the class of bug that passes every test and surfaces in production on the specific workflow that triggers the overwrite path.

## What A2AChannelBackend is, and what it isn't

The backend couldn't implement `AgentChannelBackend` — that's the interface `ChannelGateway` injects by type, and `QhorusChannelBackend` already satisfies it. Two beans, one injection point: startup failure. It couldn't implement `HumanParticipatingChannelBackend` either — that declares `actorType()` returns `HUMAN` and the at-most-one coherence invariant applies. A2A carries both actor types.

It implements `ChannelBackend` directly. `actorType()` returns `AGENT` as the declared default; per-message routing is handled internally. `ensureRegistered()` uses `ConcurrentHashMap.newKeySet().add()` — one atomic operation, exactly one thread wins the race, exactly one `registerBackend()` call per channel UUID.

`getTask()` now reads from `CommitmentStore` first. The commitment lifecycle tracks obligation state precisely — `FULFILLED`, `ACKNOWLEDGED`, `OPEN` — rather than inferring it from the last message type. The fallback to message-history `deriveState()` remains for channels without a commitment.

One issue surfaced late: `DELEGATED` maps to `"completed"` in the current code, but a delegated task is still in progress with the delegate agent. Claude flagged it in the final review. That's a follow-on.

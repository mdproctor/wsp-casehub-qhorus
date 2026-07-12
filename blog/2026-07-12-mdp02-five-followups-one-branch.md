---
title: "Five Followups, One Branch — Merge, Move, Presence, and the Normative Boundary Question"
date: 2026-07-12
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [qhorus, conversation-model, presence, topics, normative, design]
---

The previous session landed topics, reactions, rich artefact refs, and channel membership. This session picked up the trailing work — five deferred issues batched onto a single branch. The interesting part wasn't the features themselves; it was the design decisions that emerged from thinking about what "move a topic" actually means in a system with normative commitments.

## Merge Is Just Rename Without the Guard

`mergeTopics` turned out to be trivially simple. The existing `rename_topic` already updates all messages and deletes the source topic record. The only difference: rename blocks when the target exists, merge allows it. Mechanically identical. The source topic's resolved state disappears — target's state governs. If someone merged a resolved topic into an unresolved one and wants it re-resolved, that's an explicit admin action, not an automatic side effect.

The general-topic rule was worth getting right: you can merge INTO general (collapsing a side conversation back into the default), but you can't merge FROM general (emptying the default topic is structurally meaningless).

## Move Is Where Normative Boundaries Bite

`moveTopic` crosses channels, and channels are normative boundaries. A COMMAND dispatched in channel A creates an obligation under channel A's enforcement context — its allowed writers, type policies, rate limits. If we move those messages to channel B while the obligation is live, where does the resolution go? Which channel's policies apply to the DONE message?

The answer: don't create the ambiguity. Block the move when open commitments exist. The user resolves or cancels obligations first, then moves. It's an extra step, but it's honest — the alternative is silently orphaning commitments in a channel whose messages no longer exist.

The ledger stays immutable. Moved messages retain their original channelId in the audit trail, same principle as topic rename. The divergence between ledger and message table is by design: historical truth vs current organisation.

One limitation we accepted: moved messages won't be re-delivered to the target channel's AT_LEAST_ONCE backends. Their message IDs were allocated in the source channel's sequence and may be below the target's delivery cursor. This is fine — moveTopic is administrative reorganisation of historical messages, not real-time dispatch.

## Identity Is Not Membership

The design review surfaced an interesting negative decision: `ChannelMembership` should NOT carry a `displayName`. The chat-demo UI needs one for the member panel, and the temptation was to slap a denormalized string on the record.

But display name is a property of identity, not membership. The same member should display consistently across channels — that comes from the identity layer (Instance.description for agents, connector SPI for humans). Putting it on membership creates a second source of truth with no mechanism to keep it current when the identity changes. The connector adapter already handles display name resolution. Adding it to Qhorus would be adding complexity with negative architectural value.

## Presence Without a Scheduler

Presence was the largest piece. The design choice I'm happiest with: lazy degradation. No `@Scheduled` evaluator scanning entries — just compute the effective status at read time. Caffeine cache handles the TTL; `getPresence()` checks `now - lastSeenAt` against the timeout thresholds. If the heartbeat is stale, you're AWAY. If the cache evicted you, you're OFFLINE. The computation is a handful of nanoseconds.

The config invariant is the kind of thing that bites people in production: `awayTimeout` must be less than `offlineTimeout`, otherwise AWAY is unreachable (the cache evicts before the degradation threshold). Validated at startup — fail fast with a clear message.

AWAY and OFFLINE are computed-only. If a client heartbeats with `OFFLINE`, we reject it. You don't declare yourself offline — you stop heartbeating and the system notices. The asymmetry is deliberate: clients report intent (ONLINE, AVAILABLE, BUSY), infrastructure computes state (AWAY, OFFLINE).

## The Boring One

`ArtefactType.DEBATE` — a single enum value added to align with the chat-demo UI. One line of production code, one test asserting it exists. Sometimes the right answer is that simple.

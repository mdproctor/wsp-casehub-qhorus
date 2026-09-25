---
layout: post
title: "Broadcast Without Channels (Sort Of)"
date: 2026-09-25
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [broadcast, channel-semantics, notification-bridge, hive-mind]
---

# Broadcast Without Channels (Sort Of)

The hive mind epic needs agents to fan out coordination hints — "all analyzers, I found something in region X" — without anyone setting up channels or managing enrollment. The first instinct was to route through the platform notification system. It already has subscriptions, targeting, delivery tracking. Just fire a `SubscribableEvent` and let the subscription engine figure out who cares.

That almost worked. But it would have deepened the split between qhorus's channel model and the platform notification system into two parallel routing systems that don't understand each other. An agent sending a broadcast wouldn't get a ledger entry. Watchdogs wouldn't see it. Kafka observers wouldn't see it. The notification system would be a black hole that qhorus can't reason about.

The fix was to flip the relationship. Qhorus owns the semantic model — a new `BROADCAST` channel semantic that restricts to STATUS and EVENT messages only (no commitments, no obligations). The platform notification system becomes a delivery backend, the same way Slack, webhooks, and Kafka are backends. One dispatch, all transports.

Auto-membership turned out to be the more interesting part. When an agent registers with capability "analyzer", `BroadcastMembershipManager` observes the CDI event, diffs the old and new capability sets, and creates `broadcast/analyzer` as a channel. The agent joins automatically. Deregistration removes the membership but keeps the channel — the slug, ledger history, and any channel-level config survive for the next agent that registers the same capability.

The slug convention was a small discovery. The design spec said `broadcast:analyzer` but the channel slug validator only allows `[a-z0-9-]` segments. Colons aren't valid. Slashes are — they're the existing path hierarchy separator. So `broadcast/analyzer` it is. The validator caught it silently (the join was try-caught), which meant "zero interactions with this mock" in the test rather than an obvious failure. Worth remembering: when a channel creation path silently fails, check the slug first.

The `broadcast()` convenience on `MessageDispatcher` is deliberately thin — resolve `broadcast/<tag>` by name, build a `MessageDispatch`, call `dispatch()`. No new dispatch path. The standard enforcement pipeline (type policy, rate limits, ledger, fan-out) handles everything. EVENT content routes to `telemetry` per existing convention.

What this opens up: the engine's dynamic interest registration (engine#1107) can now hook into capability-based auto-membership. An agent that discovers a new domain of interest at runtime registers the capability, and the broadcast channel materialises. Stigmergy signals — the environmental markers that drive swarm coordination — have a delivery primitive that's governed, auditable, and multi-transport.

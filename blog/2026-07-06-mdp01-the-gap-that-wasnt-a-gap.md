---
layout: post
title: "The Gap That Wasn't a Gap"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [architecture, multi-node, postgresql]
---

## The Gap That Wasn't a Gap

Qhorus issue #162 had been open since May — "cross-node ChannelBackend delivery gap in multi-node embedded fleet." The issue listed three resolution paths: shared Qhorus service, CLUSTER relay via Kafka, or replicated channel store. Meanwhile, Claudony had already shipped its own fix: `FleetMessageRelayObserver`, a REST-based tick relay to fleet peers. The issue stayed open because nobody had decided which of the three options was "right."

I started by tracing the actual message flow end to end. `MessageService.dispatch()` persists, writes the ledger, fires `MessageObserverDispatcher`, then calls `ChannelGateway.fanOut()`. Both notification paths — observers and backends — operate on the dispatching node only. The gateway's backend registry is an in-memory `ConcurrentHashMap`. Backends on other nodes are invisible.

The first question was whether the three options in the issue were even the right frame. They weren't. All three assume the deployment topology is a variable. It isn't. Qhorus is a governance mesh — the ledger, commitments, and channel history must be consistent across all participants. Independent databases per node produce two independent governance systems with incoherent audit trails. That's not a deployment limitation; it's a logical contradiction.

Once shared PostgreSQL is established as a prerequisite (not an option), the problem shrinks dramatically. All reads are consistent. All writes are consistent. The only gap is push notification — telling other nodes a message arrived so they can fire their local backends.

PostgreSQL LISTEN/NOTIFY is the natural transport. Zero additional infrastructure — the database is already shared. Near-real-time delivery. `casehub-work` already uses this exact pattern with `PostgresWorkItemEventBroadcaster` for cross-node WorkItem SSE.

We built it as an SPI: `ChannelActivityBroadcaster` in `api/gateway/`, with a no-op default and a `postgres-broadcaster/` module that activates by classpath presence. After a message commits, `pg_notify` fires. Other nodes receive the notification, read the message from the shared DB, and call `post()` on their local backends. One SPI, one module, and the architectural gap closes.

The design review surfaced something I'd missed: the LAST_WRITE overwrite path in `MessageService.dispatch()` returned early before `fanOut()`. Local backends never saw content updates on LAST_WRITE channels — a pre-existing gap that affected single-node deployments too. We fixed both paths while wiring the broadcaster.

The Claudony fleet relay — `FleetMessageRelayObserver` — becomes unnecessary. It did the right thing (tick-only relay, peers read from shared DB), but it was claudony-specific. Every other consumer would have needed to reimplement it. Now they just add the broadcaster module as a dependency.

The real lesson: the issue wasn't "which of three options?" It was "what does shared governance actually require?" Once you follow the implication — single source of truth is not optional — the three options collapse to one prerequisite and one notification mechanism. The gap that had been open for two months was a framing problem, not an engineering one.

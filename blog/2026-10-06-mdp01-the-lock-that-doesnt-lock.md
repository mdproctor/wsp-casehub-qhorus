---
layout: post
title: "The Lock That Doesn't Lock"
date: 2026-10-06
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [distributed-systems, concurrency, merkle-chain, architecture]
---

Every Java developer knows what `synchronized` does. You put it on a method, and only one thread enters at a time. Reliable. Understood. And completely invisible to a second JVM running the same code on another machine.

This is not a surprise to anyone who's thought about distributed systems for five minutes. But it becomes a very specific problem when you're staring at a method that protects a Merkle hash chain — a tamper-evidence audit trail where each entry's hash depends on the previous entry's hash — and realising that the protection evaporates the moment you add a second node.

## The question that forced the trace

We're designing the distributed qhorus mesh — transforming the communication layer from an embedded library into a standalone service that agents connect to over the network. The obvious architecture question: should writes to a channel be routed to a single owning node (like Kafka's partition leader), or can any node write to any channel and let PostgreSQL serialise the concurrent transactions?

I was leaning toward "just let PostgreSQL handle it." The database already serialises concurrent INSERTs. The sequence allocator already uses `REQUIRES_NEW` with MERGE. Why add a routing layer when the database does the hard work?

Claude traced the actual code paths to find out what breaks.

## Three things that break

**The Merkle chain.** `QhorusLedgerEntryRepository.save()` is `synchronized`. Inside that lock, it reads the current Merkle frontier for a channel, computes the next hash by appending the new entry, and replaces the frontier. Classic read-modify-write. Two JVMs calling this concurrently would both read the same frontier, compute different appended frontiers, and the last writer would silently corrupt the chain. The `synchronized` keyword is JVM-local — it serialises threads within one process and does nothing across processes.

**LAST_WRITE channels.** These are channels where each sender keeps only their latest message — used for status and presence. The dispatch path reads the last message, increments the version, and overwrites. The version field exists on `MessageEntity` but has no `@Version` annotation — Hibernate doesn't enforce it. Two nodes could read the same version, both increment to the same value, and one overwrites the other's content without detection.

**Correction count enforcement.** When an agent corrects a message, qhorus checks how many corrections already exist and rejects if the limit is exceeded. Count, check, insert — a textbook TOCTOU race. Two concurrent corrections from different nodes could both see count < max and both proceed.

Three other subsystems are safe: `Message.id` uses a PostgreSQL sequence (globally ordered by design), the sequence allocator's MERGE acquires a row lock that works across JVMs, and commitment state transitions are marginal — PostgreSQL's row-level UPDATE lock likely serialises them, though without an explicit `SELECT FOR UPDATE` it's not guaranteed.

## Belt and suspenders

The answer isn't "pick one." It's both.

**The hash ring routes writes to the channel owner.** In normal operation, only one node writes to any given channel. The existing `synchronized` works. No database lock contention, no extra latency. This is the Kafka model — partition leader handles all writes, deterministic routing via consistent hashing on the channel ID.

**Database-level locks catch the edge cases.** During ownership transfer — a node dies, the ring recalculates, a new node takes over — there's a brief window where both might write. Three surgical changes to the runtime add DB-level protection: `SELECT FOR UPDATE` on the Merkle frontier, `@Version` optimistic locking on `MessageEntity`, and `findByCorrelationIdForUpdate()` for commitment transitions. In normal operation these locks are uncontended and free. During edge cases they prevent corruption.

The hash ring is the performance guarantee. The database locks are the correctness guarantee. The two are independently verifiable — you can prove correctness from the DB locks alone, and prove performance from the hash ring alone.

## What this means for the mesh

The distributed mesh isn't a distributed database. It's a distributed application layer over a shared PostgreSQL. All nodes connect to the same database. "Partitioned" means write-ownership — which node does the INSERT — not data locality. Every node can read every table. The consistent hash ring determines the single writer per channel, and PostgreSQL LISTEN/NOTIFY (already built as `postgres-broadcaster`) handles cross-node subscriber fan-out.

This is a simpler model than most distributed messaging systems, and it's simpler because qhorus's workload is different. Agent channels are conversation-pace, not streaming-pace. A channel might see a few messages per minute, not thousands per second. At that scale, shared PostgreSQL is not a bottleneck — it's the simplest correct solution.

The spec is written, the safety-net changes are planned, and the first implementation phase is three surgical edits to existing code. The distributed topology — cluster manager, heartbeat, ownership transfer — comes in Phase 2. But the correctness foundation has to land first, because you can't verify the distributed layer's safety if the single-node safety doesn't exist at the database level.

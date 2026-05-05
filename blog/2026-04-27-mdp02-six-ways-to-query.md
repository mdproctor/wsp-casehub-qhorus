---
layout: post
title: "Six Ways to Query an Obligation"
date: 2026-04-27
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [ledger, normative-layer, jpa, testing]
---

The normative ledger already records everything — every speech act, every obligation opened and closed. What it lacked was a way to ask questions of it.

Six new query tools address that.

Three work at the obligation level. `get_obligation_chain` takes a `correlation_id` and returns computed enrichment: initiator, participants in encounter order, handoff count, elapsed seconds, resolution type, and live CommitmentStore state. The raw entries were already retrievable via `list_ledger_entries(correlation_id=X)` — what the chain tool adds is what can't be extracted from a single call: participant order, whether delegation occurred mid-chain, how long the whole thing took. `get_obligation_stats` aggregates across all COMMANDs on a channel — fulfilled, failed, declined, delegated, still open — and a fulfillment rate. `list_stalled_obligations` returns every COMMAND with no terminal sibling past a time threshold.

The other two are telemetry-facing. `get_causal_chain` is a compliance tool — given a `ledger_entry_id`, it walks `causedByEntryId` links upward to the root, oldest-first. `get_telemetry_summary` groups EVENT entries by tool name and computes count, average duration, and total tokens per tool.

`list_ledger_entries` gained `correlation_id` filtering and a `sort` parameter (`asc`/`desc`, default ascending).

None of this can be expressed as a single JPQL query — JPQL doesn't support recursive CTEs. The ancestor chain walk uses an iterative loop with a visited Set for cycle protection:

```java
while (currentId != null && !visited.contains(currentId)) {
    visited.add(currentId);
    MessageLedgerEntry entry = em.find(MessageLedgerEntry.class, currentId);
    if (entry == null || !channelId.equals(entry.channelId)) break;
    chain.add(entry);
    currentId = entry.causedByEntryId;
}
Collections.reverse(chain);
```

Stop at channel boundaries, stop on cycles, return oldest-first. Chain depth in practice is two or three hops.

The end-to-end scenario drives all nine message types across four channels — the insurance claim scenario from `docs/normative-layer.md`, compressed into test code. Writing it is where Claude and I hit something unexpected.

The original setup lived in `@BeforeEach`: create channels, drive the obligation lifecycle, then assert each query tool in separate `@Test` methods. Every test failed:

```
java.lang.IllegalArgumentException: Channel not found: e2e-claim-456
```

The channel was committed. It should have been visible. The mechanism: `@TestTransaction` wraps `@BeforeEach` and the test body in a single outer JTA transaction. When `LedgerWriteService.record()` fires with `REQUIRES_NEW`, it suspends that outer transaction, commits the ledger entry, then resumes. Something in the suspend/resume cycle disrupts the JPA EntityManager's view of entities persisted but not yet flushed in the outer transaction. After resumption, `channelService.findByName("e2e-claim-456")` returns empty.

The fix: put all setup inside the `@Test` method body. One contiguous transaction context — no suspension, no cache disruption.

I'd have spent an hour on this without the exact error message. Now it's in CLAUDE.md.

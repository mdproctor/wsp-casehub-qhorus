---
layout: post
title: "Phase 13: DB Independence and the Reactive Question"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [persistence, reactive, quarkus-ledger, store-pattern]
---

Phase 13 wasn't on the original roadmap. It landed because I wanted Qhorus to be properly portable before going further — and because the quarkus-workitems project had already worked out the right pattern.

The problem with the original code is simple to describe: services called Panache entity statics directly. `Channel.find("name", name)`, `Message.list(...)`, `Instance.findById(...)` — all woven through service methods. Swapping the backend without touching the services was impossible. Before a reactive migration, before a native image, before Quarkiverse submission, that had to change.

I'd already explored how quarkus-workitems solved this. They took the SWF SDK's persistence abstraction philosophy — `Store`, `scan(Query)`, KV semantics — and distilled it into something appropriate for a CDI container. No builder pattern, no explicit transaction management, no `Writer`/`Reader` split. Just `WorkItemStore` with `put`, `get`, `scan(WorkItemQuery)`. CDI does the wiring; `@Alternative @Priority(1)` handles test substitution.

We built the same pattern across all five Qhorus domains — channel, message, instance, data, watchdog. Five store interfaces, five JPA implementations, five in-memory implementations in a new `testing/` module, and Query value objects with a `matches()` predicate so in-memory stores apply the same filter logic as the JPQL. By the end, 646 tests in the runtime module, 67 pure-Java unit tests in `testing/`, and 4 happy-path tests in a new `examples/` module. 717 total.

The session also produced a cross-project comparison doc — SWF SDK, CaseHub, and quarkus-workitems laid out side by side, with a full assessment of what CaseHub should adopt and what it should keep. CaseHub's `CaseFileRepository` interface is close but not there; `Store` + `scan(CaseFileQuery)` would cover it without losing the per-key optimistic locking that justifies CaseHub's interface-based domain model.

## The quarkus-ledger Surprise

Part of the session was less planned. Mid-way through getting the worktree baseline clean, a `mvn clean install` failed on `LedgerHashChain` — class not found. The previous test runs had been passing because compiled `.class` files were still in `target/` from before the dependency update. Maven's incremental build doesn't care about classes removed from upstream jars; it only recompiles when source timestamps change. Tests: green. Clean build: broken. The lesson is to always `mvn clean install` after installing a locally-developed dependency.

Claude flagged the exact issue by running `jar tf` against the installed quarkus-ledger jar and grepping for the missing classes. Both `LedgerHashChain` and `ObservabilitySupplement` were gone — two separate API removals in the same dependency update. We also discovered four new abstract methods on `LedgerEntryRepository` that `AgentMessageLedgerEntryRepository` now had to implement. Once we understood what the new library actually contained, the fixes were straightforward: remove the old hash chain block, set `correlationId` directly on the entry (now a first-class field), implement the four new methods.

## The Reactive Question

After Phase 13 merged, I wanted to start the reactive migration — `Uni<T>` through the store interfaces, Hibernate Reactive replacing blocking Panache, `@Tool` methods returning `Uni<T>`. The store abstraction was supposed to make this seam clean. It did, except for one dependency: `AgentMessageLedgerEntry extends LedgerEntry`, which extends blocking `PanacheEntityBase`. You can't have a reactive entity inherit from a blocking one.

We worked through the options. Dual persistence units — blocking for ledger, reactive for everything else — is supported but architecturally impure. Breaking the JPA inheritance is cleaner but changes the ledger schema design. The right answer is for quarkus-ledger to offer both blocking and reactive support natively: plain `@Entity` POJOs (no Panache base class), a blocking `LedgerEntryRepository extends PanacheRepository`, and a new `ReactiveLedgerEntryRepository extends ReactivePanacheRepository`. Same entity, two persistence strategies. Standard Quarkus library design — the framework does the same thing for Hibernate ORM vs Hibernate Reactive.

So the reactive migration is paused until quarkus-ledger is fixed. I've written the briefing for quarkus-ledger Claude; the spec is clear. The deferral is the right call: better to get the foundation right than to build workarounds that accumulate debt.

## An Unexpected Conversation

The session ended somewhere I didn't expect. We were talking about when Claude-to-Claude communication should use Qhorus channels versus WorkItems, and the question kept expanding. The distinction I'd assumed — Qhorus for exploration, WorkItems for accountability — is real, but incomplete.

The deeper problem is that humans have no coherent view of what a multi-Claude system did or why. GitHub has the code. The ledger has telemetry. WorkItems has formal tasks. Qhorus channels have the reasoning. HANDOFF.md has session context. None of these give you the thread that runs through all of them: *goal → task → Claude conversation → decision → outcome → code*.

Two ideas worth developing. First: every Claude-to-Claude channel should open with a structured task context message — goalId, intent, initiator. The ledger captures it at channel open and close. Claudony aggregates these into human-readable summaries. Channels don't need to be WorkItems to have accountability; they need a "why" marker at the channel level.

Second: parent-child WorkItems as a form of Hierarchical Task Network. When multiple Claudes collaborate on a complex goal, the WorkItem hierarchy IS the coordination structure — each Claude knows what it owns, what the parent goal is, and what its children produced. The unified view humans see is the WorkItem tree annotated with the conversations that happened at each node. Qhorus for the nervous system; WorkItems for the skeleton.

Both ideas went into the idea log. Whether they shape Claudony's design is a future conversation — but they surfaced naturally from thinking through what Qhorus is actually for.

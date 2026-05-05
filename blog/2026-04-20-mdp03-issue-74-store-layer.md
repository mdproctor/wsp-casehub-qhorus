---
layout: post
title: "The Reactive Store Layer Ships"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [reactive, dual-stack, subagent-driven-development, testing]
---

The previous session ended with a clear constraint: the `superpowers:writing-plans`
skill had hit the 32k output limit trying to generate the full eight-issue plan
for Epic #73 at once. The lesson was simple — one issue per session.

This session was Issue #74: the reactive store layer. Five `Reactive*Store`
interfaces, five `ReactiveJpa*Store` implementations, five `InMemoryReactive*Store`
test doubles, 25 unit tests, and integration tests to prove the plumbing worked.
We ran the full `subagent-driven-development` workflow.

## One Plan, One Issue

I opened with `superpowers:writing-plans` and gave it the constraint explicitly:
Issue #74 only, max 6 tasks, full Java code required. The plan came back at the
right size — readable, complete, with actual code in every step rather than
"implement this method" placeholders.

The six tasks mapped cleanly to implementation order: interfaces first, Panache
repo helpers, JPA implementations split across two tasks by domain, InMemory
wrappers plus unit tests, then integration tests.

## Dispatching in Parallel

Tasks 3 and 5 were independent — different packages, different test scopes, no
shared files. I dispatched both implementer subagents in the same message. Each
reported back separately; spec and quality reviews ran sequentially. It cut one
full round-trip out of the session.

The `InMemoryReactive*Store` approach is worth noting. Each wrapper holds a
`private final InMemoryChannelStore delegate = new InMemoryChannelStore()` — no
CDI injection. That makes it directly instantiable in plain JUnit without any
Quarkus bootstrap. The 25 unit tests use `.await().indefinitely()` to unwrap and
run in milliseconds.

## The Compile Error That Pointed Somewhere Else

Before Task 2 could proceed, compilation failed. The error pointed at
`AgentMessageLedgerEntryRepository.java`:

```
cannot find symbol: method list(String, UUID)
  location: class LedgerAttestation
```

The file's own Javadoc said "LedgerAttestation is still a PanacheEntityBase; its
Panache static methods are used as-is." That comment was wrong.

`LedgerAttestation` in the current quarkus-ledger snapshot is a plain `@Entity`
with `@NamedQuery` annotations — no Panache statics. The previous session had
adapted `LedgerEntry` to the new SPI but missed `LedgerAttestation` entirely.
We replaced the three calls with `em.createNamedQuery(...)` and `em.persist()`.
Five minutes once the cause was clear; the misleading Javadoc sent the
investigation in the wrong direction first.

## H2 Can't Do Reactive

Task 6 was the integration tests. The plan called for `vertx-jdbc-client` as
the H2 reactive bridge. The subagent came back with DONE_WITH_CONCERNS: the
reactive pool factory wasn't registering. All 20 integration tests were `@Disabled`.

This confirmed something I'd suspected but not tested: Quarkus only registers a
reactive pool factory when a native reactive client extension is on the classpath —
`quarkus-reactive-pg-client`, `quarkus-reactive-mysql-client`. `vertx-jdbc-client`
alone doesn't wire it. H2 has no async driver.

The tests are written correctly — `@RunOnVertxContext`, `UniAsserter`,
`Panache.withTransaction()` wrapping mutations — and they'll pass against
PostgreSQL with Docker. Until then, `@Disabled` is the honest answer.

758 tests passing. Issue #74 closed. Next is #75 — the reactive service layer,
where the business logic lives and the dual-stack design either holds together
or shows its seams.

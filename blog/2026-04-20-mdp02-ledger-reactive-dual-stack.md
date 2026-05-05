---
layout: post
title: "Ledger Adaptation and the Dual-Stack Decision"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [reactive, quarkus-ledger, dual-stack, architecture]
---

The previous entry ended with a deferred question — reactive migration blocked
on quarkus-ledger needing dual blocking/reactive support. This session opened
with that blocker resolved: quarkus-ledger had shipped. `LedgerEntry` was now
a plain `@Entity`, and both a blocking `LedgerEntryRepository` and a new
`ReactiveLedgerEntryRepository` SPI were available.

Three concrete changes fell out of that. I brought Claude in to work through
the adaptation. `AgentMessageLedgerEntryRepository` — previously calling
`LedgerEntry.list(...)`, `entry.persist()`, and similar Panache statics —
had to be rewritten entirely. Since `LedgerEntry` was no longer a
`PanacheEntityBase`, those static methods were gone. We switched to
`EntityManager` injection directly: typed JPQL queries, `em.find()`,
`em.persist()`. More verbose than Panache statics, but it avoids the
return-type conflict you'd hit trying to implement both
`PanacheRepository<AgentMessageLedgerEntry, UUID>` and `LedgerEntryRepository`
in the same class — `listAll()` returns `List<AgentMessageLedgerEntry>` in
Panache and `List<LedgerEntry>` in the SPI, and Java won't resolve that.

We also added `ReactiveAgentMessageLedgerEntryRepository` — a new `@Alternative`
bean implementing the reactive SPI via
`PanacheRepositoryBase<AgentMessageLedgerEntry, UUID>`. Inactive by default.
A consumer wires it up via `quarkus.arc.selected-alternatives` when they have
a reactive datasource.

## The Reactive Boot Trap

Adding `quarkus-hibernate-reactive-panache` to the extension broke three tests.
The symptom: `FastBootHibernateReactivePersistenceProvider: Unable to find
datasource '<default>': No pool has been defined`. I tried
`quarkus.hibernate-reactive.active=false`. No effect.

Claude traced the actual mechanism: the Hibernate Reactive build step scans
for `PanacheRepositoryBase` implementations at build time and registers
reactive persistence units for any entities it finds. AOT compilation, not
CDI activation. Marking the reactive bean `@Alternative` suppresses CDI
injection but does nothing to the build step — the persistence unit was
already registered before the application started.

The fix was `quarkus.datasource.reactive=false`. That suppresses the runtime
pool acquisition even though the persistence unit was registered at build
time. Not obvious from the docs. Found by trial and error after several
wrong attempts.

## An Honest Audit

After the ledger commit, I asked whether it had followed the issue-workflow
conventions in CLAUDE.md: run Phase 1 before writing code, Phase 3 before
committing. It hadn't. Claude had created issue #68 directly via
`gh issue create` rather than using the skill, and committed without Phase 3.
Caught after the fact.

We ran a full retrospective over all 92 commits and 68 issues. Two epics had
time-based names — #32 was "Phase 9 — A2A compatibility endpoint..." and #36
was "Phase 10 — Human-in-the-loop controls" — both renamed to drop the phase
prefixes. Four commit clusters had no issue coverage: the initial scaffold,
the Quarkiverse restructuring, a quarkus-ledger supplement reconciliation,
and the comparative analysis documents. Four new issues (#69–#72) closed
those gaps.

## The H2 Question

With the ledger work done, I wanted to settle the reactive migration properly.
The previous entry had deferred it because H2 has no Vert.x reactive driver
— I'd treated that as a hard block.

Claude pushed back. There's `vertx-jdbc-client`, which wraps JDBC drivers
reactively using a thread pool — not true async, but it satisfies Hibernate
Reactive's interface requirements. Not a hard block. I'd overstated it.

The deeper question was architectural: migrate everything to reactive, or ship
both stacks? quarkus-ledger's answer is already clear — blocking
`LedgerEntryRepository` and reactive `ReactiveLedgerEntryRepository`, both
available, consumers choose. The extension is opinionated about SPI shape
and neutral on implementation. I wanted Qhorus to follow the same model.

## Approach C

The dual stack design runs top to bottom: reactive store interfaces, reactive
JPA implementations, reactive services, reactive MCP tools, reactive REST
resources. A single build property — `quarkus.qhorus.reactive.enabled=true`
— activates the whole reactive stack. Blocking remains the default.

For the MCP tool layer specifically, the shared code lives in a new abstract
base class `QhorusMcpToolsBase`: all response records, all validation helpers,
all entity-to-DTO mapping. No CDI, no `@Tool`. Then `QhorusMcpTools` and
`ReactiveQhorusMcpTools` each extend it, adding `@Tool` methods and service
injection. Identical logic, two activations.

Test contracts follow the same pattern. Abstract base classes define all
scenarios as `@Test` methods; blocking and reactive subclasses provide the
service via abstract factory methods. The reactive subclass unwraps `Uni` via
`.await().indefinitely()` in its factory — assertion code is identical.

The in-memory test doubles are particularly clean. `InMemoryReactiveChannelStore`
wraps `InMemoryChannelStore` via delegation, converting each method return to
`Uni.createFrom().item(...)`. One set of in-memory state, two interfaces,
no duplication.

Epic #73 covers the full implementation across eight child issues (#74–#81).

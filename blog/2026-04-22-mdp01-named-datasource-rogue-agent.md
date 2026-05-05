---
layout: post
title: "Named Datasources and a Rogue Agent"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [quarkus, persistence, named-datasource, spi, testing]
---

Claudony sent a message this week that was both a reasonable request and a
constraint I'd been meaning to address anyway. The ecosystem convention: each
library in the Quarkus AI Agent Ecosystem uses a named datasource matching its
artifact ID. Qhorus was using the default datasource, which was blocking
Claudony from having its own JPA persistence without schema collision.

The migration itself was straightforward — rename `quarkus.datasource.*` to
`quarkus.datasource.qhorus.*`, add the persistence unit config, point Flyway at
the named datasource. The constraint that made it interesting was
`AgentMessageLedgerEntry`.

## The Constraint in the Inheritance Chain

`AgentMessageLedgerEntry extends LedgerEntry` via JPA JOINED inheritance. Both
parent and child must be in the same persistence unit — standard JPA.
`LedgerEntry` is from `quarkus-ledger`; its package is
`io.quarkiverse.ledger.runtime.model`. Moving Qhorus entities to a named PU
without including the ledger package would cause Hibernate to fail at boot.

The clean answer would be for `quarkus-ledger` to define its own named PU.
That's coming — I flagged it to Claudony as a follow-up. For now: include
`io.quarkiverse.ledger.runtime` in the "qhorus" packages config. All ledger
base tables live in the Qhorus datasource until the inheritance is reworked.
There's an ADR capturing the coupling and the revisit trigger.

## The Library-Bean Problem

Migrating to a named datasource doesn't mean the default datasource goes away —
at least not in tests. `quarkus-ledger` ships CDI beans that inject
`@Default EntityManager`. Library code; can't be changed. If no default
datasource is configured, those beans fail at injection time.

The solution: keep the default datasource active in tests, but assign it a
packages config pointing at a config-only package with no `@Entity` classes:

```properties
quarkus.hibernate-orm.packages=io.quarkiverse.qhorus.runtime.config
```

That satisfies Quarkus's requirement that every active persistence unit declares
at least one package, without routing any application entities to the default PU.
I hadn't seen this scenario documented anywhere — it only surfaces when you embed
a library whose internals depend on the default EntityManager.

## The TestProfile Trap

Three tests failed after the migration: `SmokeTest`, `A2AGetTaskTest`,
`WatchdogEnabledTest`. All three use `@TestProfile` and cause Quarkus to restart
the application context. All three got:

```
Unable to find datasource 'qhorus' for persistence unit 'qhorus':
No pool has been defined for persistence unit qhorus
```

The named datasource was correctly configured in
`src/test/resources/application.properties`. Every other test passed.

The issue: when `@TestProfile` causes a context restart, the restarted instance
doesn't re-read the test properties file. It applies the profile's
`getConfigOverrides()` on top of the base compiled config — not on top of the
test properties. The full datasource block needs to go in `getConfigOverrides()`
of every profile that triggers a restart. That's not in the Quarkus docs.

## Claude Goes Off-Script

The migration was the planned work. What I didn't plan was `PendingReplyStore`.

During implementation, one of the dispatched subagents went significantly beyond
its task scope. It created a feature branch, added `PendingReplyStore` as a
sixth store SPI interface, wrote its JPA and in-memory implementations, contract
tests — then continued with the actual named datasource migration it had been
asked to do.

The thing is: `PendingReplyStore` was a genuine gap. `wait_for_reply` was the
only Qhorus feature with direct JPA access outside any store interface. I'd
flagged it to Claudony as a known SPI limitation. Claude closing it without being
asked was a net positive. The code is correct, the tests pass, and the SPI is
now fully sealed.

The root cause of the scope expansion: I'd told the subagent to "keep iterating
until all tests pass" with no explicit file scope constraint. That phrasing left
the door open for "fix anything that looks wrong." More specific prompts — here
are the files in scope, here is what's out of scope — would have kept it focused.
But in this case, the rogue work was worth keeping.

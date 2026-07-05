---
title: The Silence of the Wrong Superclass
date: 2026-07-05
tags: [jpa, cdi, hibernate, ledger, qhorus]
---

# The Silence of the Wrong Superclass

Three bugs surfaced today, and all three shared the same failure mode: JPA doing exactly what you told it to, silently, with no indication that what you told it was wrong.

The first was a CDI ambiguity. After extracting InMemory store implementations into a `persistence-memory` module, the old copies were left behind in `testing`. Both modules ship `@Alternative @Priority(1)` beans for every store interface. CDI sees two alternatives at the same priority — ambiguity. Consumers worked around it with increasingly baroque `exclude-types` lists in their application properties, some referencing classes that no longer exist. The fix was mechanical: delete the duplicates.

The second was a CDI producer that shouldn't have existed. `CrossTenantProducer` injected JPA stores by concrete class, wrapped them with a `@CrossTenant` CDI qualifier, and added an admin assertion that checked a hardcoded `true`. The qualifier was redundant — `CrossTenantChannelStore` and `ChannelStore` are separate interfaces with no shared type hierarchy. CDI already knows which one you want from the type alone. A qualifier on a unique type adds zero information; it only forces a producer pattern into existence to bridge the unqualified JPA beans to the qualified injection points. Removing the qualifier eliminated the producer, which eliminated the concrete-type injection, which eliminated the test-time failure when no datasource was configured.

The third was the real find. After the casehub-ledger API migration, `LedgerEntry` moved from `@Entity` in the runtime module to `@MappedSuperclass` in the API module. A new `JpaLedgerEntry` class in runtime became the `@Entity` with `@Inheritance(JOINED)`. But `MessageLedgerEntry extends LedgerEntry` still compiled — `@MappedSuperclass` is a valid JPA supertype. No error. No warning. It just quietly stopped being part of the JOINED inheritance tree.

The consequences were everywhere and nowhere. Queries using `FROM LedgerEntry` stopped returning `MessageLedgerEntry` instances. Attestation writes failed because the causal COMMAND entry couldn't be found. Cross-dtype sequence tests saw one entry instead of two. The symptom was always "expected 1, got 0" — never "inheritance is broken" or "entity not managed." The exception in the attestation path was caught and logged at WARN with a generic message that didn't include the actual error.

One line fix: `extends JpaLedgerEntry` instead of `extends LedgerEntry`. The subclass re-enters the JOINED inheritance tree, gets `@PrePersist` for `id` and `occurredAt`, and appears in polymorphic queries again.

The pattern worth remembering: when a library refactors an `@Entity` base class into `@MappedSuperclass` + `@Entity` subclass, every consumer that extended the original base class is silently broken. The code compiles, the entity persists to its own table, direct queries on the subclass work fine. Only polymorphic queries through the parent type fail — and they fail by returning empty, not by throwing.

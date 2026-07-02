---
layout: post
title: "The Roots, Not the Leaves"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [store-spi, domain-records, migration, test-hierarchy]
---

## The Roots, Not the Leaves

The Store SPI migration — moving every store interface from JPA entity types to immutable domain records — touched roughly 140 files across 12 modules. The scope wasn't surprising; the approach to handling it was the thing worth writing about.

The first attempt was leaf-chasing. Fix a file, run Maven, see 200 errors (Maven's hard cap on reported compilation failures), fix those files, run Maven again, see 200 more. Each batch revealed files hiding behind earlier failures. We were making progress but couldn't see the horizon.

The fix was to stop and think about the dependency graph. Thirteen abstract contract test base classes define method signatures that 37 concrete tests inherit. Fix a base, and every inheritor's signature mismatch resolves for free. Fix a leaf, and you still have to fix the base — plus every other leaf that shares it. The hierarchy looks like this: `ChannelStoreContractTest` sits in `persistence-memory/` and `testing/` (same contract, different packages). `InMemoryChannelStoreTest`, `InMemoryReactiveChannelStoreTest`, and `JpaChannelStoreTest` all extend it. Multiply that across seven store types and five service types.

Once we mapped the full tree, the work went fast. Fix all 13 bases first. Then the 37 concrete inheritors needed only signature changes — the test logic in the bases was already correct. The remaining ~28 standalone test files had no cascading structure to exploit, but the patterns were fully established by then: record type replaces entity type, `.field` becomes `.field()`, `new Entity()` becomes `Record.builder().build()`, `entity.field = x` becomes `record.toBuilder().field(x).build()`.

The migration pattern at the JPA store boundary is clean. Stores accept and return domain records. Inside, they convert: `Entity.fromDomain(record)` on writes, `entity.toDomain()` on reads. The JPA entity stays mutable — Hibernate needs that. The domain record is immutable — the rest of the codebase gets that. The seam is explicit and narrow.

Three modules compiled and tested clean by session end — runtime, persistence-memory, testing. Connector-backend has six test files remaining, same mechanical pattern. Slack-channel and examples modules haven't been checked yet but the conversion is fully mechanical at this point.

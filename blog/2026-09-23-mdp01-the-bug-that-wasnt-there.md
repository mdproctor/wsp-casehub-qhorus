---
layout: post
title: "The Bug That Wasn't There"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [module-boundaries, cdi, graphql, debugging]
---

# The Bug That Wasn't There

The issue said `StoredCommitmentAttestationPolicy` and `EvidentialChecker` had constructor mismatches — callers passing no args to constructors that now required five. Downstream consumers were blocked: CDI failures cascading through 30 unsatisfied dependencies, every integration test dead.

I went looking for the reported constructors. Both classes have no-arg constructors. Both compile. The described problem doesn't exist on main.

The actual breakage was elsewhere entirely. When the `CausalGraphApi` SPI was extracted from the runtime module in a prior commit, the graphql adapter — `CausalGraphService` — wasn't updated. It still imported three runtime-internal types: `MessageLedgerEntry`, `MessageLedgerEntryRepository`, and the runtime's own `CausalGraphService`. The graphql module depends on `casehub-qhorus-api` only. Those types aren't on its classpath.

The fix was straightforward: swap the runtime types for their API-level equivalents. `CausalGraphReader` replaces the runtime service. `LedgerReader` replaces the repository. `LedgerEntryView` replaces the entity. The graphql module already has six other services following this exact pattern — `AuditService` was the template.

The same prior commit also deleted `ChannelResource`, `SpaceResource`, and `CausalGraphResource` but left their 37 tests behind. Every test hit a 404 on endpoints that no longer exist. Those went too.

The interesting thing is how the symptoms pointed away from the cause. A downstream consumer sees CDI wiring failures and reports constructor mismatches — because that's what CDI error messages look like when a bean's dependency tree has an unresolvable link. The compilation error in graphql prevented the module from producing a valid bean archive, which cascaded into the CDI deployment failures the issue described. Same root cause, completely different symptom at the consumer level.

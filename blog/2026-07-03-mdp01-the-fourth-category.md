---
layout: post
title: "The Fourth Category"
date: 2026-07-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [spi, api-design, tier-structure]
---

Qhorus's api/ module had three kinds of interfaces: stores for data access, SPIs for consumer-provided policies, and gateway contracts for integration. Each sits in its own package — `api/store/`, `api/spi/`, `api/gateway/` — and the boundaries are clean because the consumer relationship is different for each. Stores are called. SPIs are provided. Gateway contracts are implemented.

Then casehub-blocks needed to dispatch messages.

The types were already in api/ — `MessageDispatch`, `DispatchResult`, `Channel`, `ChannelCreateRequest`. Everything a consumer needs to construct a dispatch call. But the method that actually enforces policy, writes the ledger entry, and fans out to backends lives in `MessageService`, which lives in runtime, which depends on JPA, Hibernate, the ledger, the commitment service, and half of Quarkus. A Tier 1 consumer cannot reference a Tier 3 class. The type mismatch is one interface away from solved, but that interface didn't exist.

The obvious placement — `api/spi/` — was wrong. The consumer-spi-placement protocol is explicit: `api/spi/` is for interfaces that consumers *provide* (custom policies, identity resolvers, attestation strategies). `MessageDispatcher` and `ChannelManager` are interfaces that consumers *call*. The dependency direction is inverted. Putting a call-target interface in the provider-interface package creates a category error that misleads every future reader about the contract.

I put them in their domain packages instead. `MessageDispatcher` sits in `api/message/` next to `MessageDispatch` and `DispatchResult`. `ChannelManager` sits in `api/channel/` next to `Channel` and `ChannelCreateRequest`. Maximum cohesion — the interface and the types it operates on are co-located.

The design review caught something I'd missed. `ChannelCreateRequest` still used `String` for `barrierContributors`, `allowedWriters`, and `adminInstances` — CSV strings parsed by `Channel.splitCsv()`. The `Channel` record already used `List<String>` for the same fields. That type mismatch was invisible when both types lived in the same module, but once `ChannelManager.setAllowedWriters(UUID, List<String>)` appeared as a Tier 1 contract, the `String` field on the request record was indefensible. We changed all three fields to `List<String>` and moved `splitCsv()` to the MCP tool boundary — the only place where CSV strings arrive from the outside world.

The other thing the review found was subtler. `findOrCreate` catches `PersistenceException` from a constraint violation and retries the lookup in the same transaction. On H2 this works — H2 lets you query after a failed statement. PostgreSQL marks the entire transaction rollback-only. Every subsequent command aborts with "current transaction is aborted." The tests are green; production would break. The fix is to catch outside the `REQUIRES_NEW` boundary so the failed transaction completes before the retry.

That PostgreSQL gotcha is the kind of thing that only surfaces when you actually look at what the transaction manager does after an exception, rather than what the code expects it to do. H2 is permissive about things PostgreSQL is strict about, and the gap is silent — no error at test time, no warning at compile time.

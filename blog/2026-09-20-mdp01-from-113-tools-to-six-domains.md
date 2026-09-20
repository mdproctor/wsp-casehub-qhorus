---
layout: post
title: "From 113 Tools to Six Domains"
date: 2026-09-20
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [mcpdomain, migration, graphql, mcp, tri-channel]
series: issue-451-mcpdomain-migration
---

Qhorus has been accumulating `@Tool` methods in a single 2,756-line class since the project started. `QhorusMcpTools` owned everything — channel configuration, message dispatch, artefact storage, commitment lifecycle, watchdog registration, presence tracking. One hundred and thirteen methods on one named MCP server, all loaded into the agent's context at connection time regardless of what it actually needed.

The platform moved to a different model: `@McpDomain` interfaces discovered lazily via `casehub_activate`. An agent working with channels activates `"channels"` and gets channel operations. One working with audit activates `"audit"`. The tri-channel generator turns each `@McpDomain` interface into a GraphQL resolver and a REST resource at compile time — a single interface definition, three exposure surfaces.

We started the migration with the smallest domains to lock in the pattern. `AgentsApi` covers instance registration, capability discovery, and presence — seven operations backed by a new `InstanceManager` interface that `InstanceService` now implements. `DataApi` covers artefact CRUD and the chunked upload lifecycle — eleven operations against a new `DataManager`. `GovernanceApi` already existed with one query; we expanded it to include commitment observability and watchdog management, adding six operations that inject `CommitmentReader` and `WatchdogStore` directly.

The pattern that emerged is mechanical: a `*Api` interface in the api module with `@McpDomain` and `@PlatformQuery`/`@PlatformMutation` annotations, a `*Service` implementation in the graphql module, CDI-free tests with Mockito. The graphql module depends only on the api module — it never touches runtime classes. Runtime services implement the api-layer manager interfaces, and CDI wires them at deployment time. The APT generator handles everything else.

The one complication worth noting: removing `@Tool` annotations from methods that fifty-plus test files call directly. The first instinct — make the de-annotated methods package-private — failed because the tests live in a different package. The pragmatic answer: keep the methods `public` but strip the `@Tool` annotation. Without `@Tool`, the method isn't registered on the MCP server. The Java method survives for test backward compatibility until `QhorusMcpTools` is deleted entirely.

Four batches remain. Audit is the most interesting — it needs a `LedgerReader` facade to expose the ledger query surface through the api layer, and some of the aggregation logic (obligation chain computation, telemetry summarisation) currently lives in the MCP tool methods and needs a proper home. The two channels batches are large (~50 operations) but straightforward delegation. The final batch deletes `QhorusMcpTools` and its 600-line base class.

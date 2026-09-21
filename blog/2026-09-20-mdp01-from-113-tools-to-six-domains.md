---
layout: post
title: "From 113 Tools to Seven Domains"
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

`AuditApi` was the most interesting domain — thirteen operations covering the ledger query surface, obligation chain computation, causal graph traversal, and telemetry aggregation. The aggregation logic had been living inside MCP tool methods and needed a proper home: `LedgerReader`, `CausalGraphReader`, and `ReviewerProvider` facades now own it at the api layer.

The two channels batches were the largest by count — fifty-seven operations across topics, membership, spaces, gateways, protocols, enforcement, routing, summaries, projections, and capacity thresholds. Each needed its own api-layer facade: `TopicManager`, `MembershipManager`, `SpaceManager`, `ChannelSummaryManager`, `ProjectionReader`, `ProtocolReader`, `RoutingDiagnostics`. The code is straightforward delegation, but the sheer surface area meant promoting a dozen result records from the old `QhorusMcpToolsBase` inner types to proper api-layer records.

The final step was stripping MCP exposure from `QhorusMcpTools` itself — removing `@McpServer`, fifty-two `@Tool` annotations, a hundred and ninety-five `@ToolArg` annotations, and `@WrapBusinessError`. The class is now an inert `@ApplicationScoped` bean with no MCP footprint. All tool discovery goes through seven `@McpDomain` interfaces: channels, messaging, governance, agents, data, audit, compliance.

That left eighty-seven test files still injecting `QhorusMcpTools` and calling its convenience methods. The `@WrapBusinessError` removal surfaced an interesting gotcha: the interceptor fires on all methods of the annotated class (CDI binding is class-level), but only wraps exceptions for `@Tool`-annotated methods. Remove `@Tool` while keeping `@WrapBusinessError`, and the interceptor activates but does nothing — exceptions pass through unwrapped. Twenty-three test files that asserted `ToolCallException` needed to switch to the raw exception types: `IllegalArgumentException` for invalid input, `IllegalStateException` for state violations like paused channels and ACL denials.

Deleting `QhorusMcpTools` entirely is deferred. The class is dead weight — zero MCP exposure, zero production value — but eighty-seven test files with twelve hundred call sites is a lot of mechanical rewriting. It'll happen, but as a separate cleanup pass. The migration goal is met: every operation is discoverable via `@McpDomain`, and the `DomainRegistrationTest` asserts all seven domains.

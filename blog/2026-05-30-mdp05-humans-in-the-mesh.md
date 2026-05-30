---
layout: post
title: "Humans in the mesh"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [casehub, qhorus, connector, mapper, native-image]
---

Qhorus has always been an agent-to-agent communication layer. COMMAND, RESPONSE, DONE — speech acts between agents, with the human in the loop only through Claudony's browser panel. This branch changes that. `ConnectorChannelBackend` bridges `InboundMessage` CDI events from casehub-connectors into Qhorus channels via `HumanParticipatingChannelBackend`, which means a text to a Twilio number, an email to an IMAP inbox, a Slack message — any of those can now become a Qhorus message and kick off an agent conversation.

The plumbing is straightforward: `ChannelConnectorBinding` is a JPA entity linking a Qhorus channel to an inbound/outbound connector pair. `ConnectorChannelBackend` observes `ChannelInitialisedEvent` synchronously on startup — sync is important, the cache is populated before any test body runs — and then observes `InboundMessage` asynchronously to route incoming messages to `gateway.receiveHumanMessage()`. Outbound goes back through `ConnectorService.send()`. V14 migration, store seam, InMemory alternative, contract tests — the standard Qhorus pattern.

The more interesting work was a design decision that revealed itself mid-implementation.

`QhorusEntityMapper.toChannelDetail()` was built with `@Inject ChannelBindingStore` — Option A, the mapper queries its own supplementary data. The `ChannelDetail` API record grew a `ConnectorBinding` nested type to expose this data to callers, and the mapper handled the lookup internally. I'd approved Option B in the brainstorm (caller supplies `Optional<ChannelConnectorBinding>`, mapper stays pure), but the pre-committed code had gone with Option A. When Claude reported what it found reading the files, the choice was clear: refactor.

The case against Option A isn't subtle. A mapper that issues queries hides N+1 behaviour behind what looks like a transformation — `toChannelDetail(ch, count)` is called inside a stream over all channels, and each call fires a `findByChannelId`. The extra store injection also makes the mapper harder to test: you need either a live datasource or a mock, whereas Option B lets you pass an empty `Optional` and the test is clean. We pulled `ChannelBindingStore` out of the mapper and into `QhorusMcpToolsBase`, where it lives alongside `QhorusEntityMapper`. The base class provides two overloads: a single-item path that does `findByChannelId()` per channel (fine for individual lookups), and a batch path that accepts a `Map<UUID, ChannelConnectorBinding>` pre-loaded by the caller. `list_channels` uses `bindingStore.findAll()` once before the stream and passes the map to the batch overload. The mapper itself took `Optional<ChannelConnectorBinding>` as a third parameter and nothing else.

There was one moment during the final code review where Claude came back with Critical findings about N+1 still being present. I checked the actual files on the remote branch — the batch overload was wired correctly, the `allBindings` map was being passed through. Claude had been reading from `main`. The branch had been squashed and the local project repo switched back to main during the verification step; the reviewer never switched to the feature branch. Not a bug in the implementation — a context failure in the review. Worth knowing.

Two other things from this branch worth keeping.

`@ObservesAsync` events in `@QuarkusTest` are unreliable — GE-20260513-b15933 has this — so the integration test was calling `InboundConnectorService.receive()` and waiting on async delivery with `timeout(2000)`. We dropped that entirely and call `backend.onInboundMessage(msg)` directly through the CDI proxy, which is synchronous and deterministic. The `fanOut` test still uses `timeout(1000)` because `ChannelGateway.fanOut()` dispatches `backend.post()` on a virtual thread — that async hop is real, not a test artifact.

The native image gap caught me off guard. `LedgerProcessor` in `casehub-ledger-deployment` has no `NativeImageResourcePatternsBuildItem` — it doesn't self-register its SQL files for native builds. `QhorusProcessor` only covered its own migrations, so ledger migrations would be absent from any native Qhorus binary. We added `includeGlob("db/ledger/migration/*.sql")` alongside the qhorus glob. There's no warning when SQL files are excluded; it's silent until Flyway runs at startup. It's now in the protocols and the garden.

PR #222 is open. 1,518 tests, 0 failures.

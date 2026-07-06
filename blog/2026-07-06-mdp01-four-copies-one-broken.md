---
layout: post
title: "Four Copies of the Same Parse, One of Them Broken"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [type-safety, CDI, maven, backoff]
series: issue-322-testing-persistence-memory-cdi-fix
---

When the review follow-up said "extract parseCorrelationUuid to a shared utility," I started by looking at where the duplication came from. Three copies in `MessageService`, `ReactiveMessageService`, and `ChannelGateway` — identical private static methods, each safely returning null for non-UUID strings. Standard code duplication, standard fix.

Then I found the fourth copy. `DeliveryBatchExecutor.toOutbound()` had the same conversion inline — `UUID.fromString(m.correlationId())` — but without the try-catch. A non-UUID correlation ID would throw `IllegalArgumentException` and crash the delivery batch. The other three methods handled this gracefully; this one didn't.

That changed the question. The duplication wasn't the problem — the type mismatch was. `OutboundMessage.correlationId` was `UUID`, but correlationIds are strings throughout the domain model: `Message`, `MessageDispatch`, `Commitment`. Every construction site was doing a lossy conversion — non-UUID strings silently became null, or in the `DeliveryBatchExecutor` case, threw.

We changed `OutboundMessage.correlationId` from `UUID` to `String`. All four parse sites disappeared. The latent bug disappeared. The cascade touched 30 files across `api/`, `runtime/`, `slack-channel/`, and `connector-backend/` — every one a mechanical type substitution the compiler enforced.

## The A2A case trap

The design review caught something I hadn't considered. With UUID keys, `sseStreams` in `A2AChannelBackend` was case-insensitive by nature — `UUID` normalises its hex digits to lowercase. With String keys, `"A1B2C3D4-..."` and `"a1b2c3d4-..."` are different map entries. A client sending an uppercase task ID to the SSE endpoint would never match the lowercase correlationId stored by `receive()`.

The fix: normalize to lowercase at both ingestion boundaries. `A2AResource.streamTask()` lowercases the validated task ID. `A2AChannelBackend.receive()` parses through `UUID.fromString().toString()` for UUID-format IDs, falling back to the raw string for non-UUIDs. Both produce canonical lowercase for valid UUIDs.

## The scope change that broke five modules

The second issue — `testing/pom.xml` declaring `persistence-memory` at compile scope — was straightforward to diagnose. The testing module exports two classes: `RecordingChannelBackend` and `MessageLedgerEntryTestFactory`. Neither imports anything from `persistence-memory`. Only an internal test (`CommitmentServiceTest`) needed `InMemoryCommitmentStore`, and that's a test-scope concern.

Changing compile to test scope was a one-line fix. The blast radius wasn't. Five sibling modules — `connector-backend`, `slack-channel`, and three examples — had been importing `InMemoryChannelStore` and friends via the transitive path through `testing`. The compiler flagged every one, but the count was non-obvious from the scope change alone.

## The broadcaster's hidden race

The exponential backoff for `PostgresChannelActivityBroadcaster` was supposed to be simple: replace a fixed 5-second retry delay with 1s→2s→4s→...→60s cap. But the design review surfaced two pre-existing issues in the reconnection logic.

First, the `closeHandler` registered on the PostgreSQL connection fires asynchronously when the old connection is closed. After a successful reconnection, closing the previous connection triggers its closeHandler — which calls `scheduleReconnect()` and tears down the healthy new connection. The fix: check `subscriberConnection != pgConn` in the handler before reconnecting.

Second, `stopListening()` in `@PreDestroy` closes the connection but doesn't prevent sleeping virtual threads from waking up and acquiring a new one. An `AtomicBoolean stopped` flag, checked at the top of `acquireAndListen()`, closes the gap.

Neither issue would have surfaced from tests — both require specific timing between concurrent reconnection attempts or shutdown ordering. The design review earned its keep on this one.

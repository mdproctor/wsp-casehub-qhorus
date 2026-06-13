---
layout: post
title: "Ten issues, one stream"
date: 2026-06-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [cdi, sse, a2a, type-safety, quarkus]
---

Ten issues closed in one batch. Most were mechanical — annotation fixes, null guards, one docs update, one method addition. But three of them kept throwing surprises, and one of those surprises turned into a fairly subtle production gotcha that probably lives in a lot of Quarkus SSE implementations.

The easiest catch was `#242` and `#234`. I looked at the code expecting to implement `saveAttestation` in the reactive ledger path and remove the `ManagedExecutor` bridge from the trust gate. Both were already there, done quietly during earlier sessions. Closed with comments. That's always a good way to start.

The type-safety migration (`#247/#246`) was bigger than it looked. Replacing `String allowedTypes` with `Set<MessageType>` in `ChannelCreateRequest` is a one-line change conceptually, but the cascading breaks were thorough — three telescoping `String`-accepting overloads in each service, `populateChannel()` in both blocking and reactive services, the MCP tools, `AutoChannelSpec` in the connector module, and about a dozen test files. The spec went through two review rounds before implementation; we caught `populateChannel()` being the actual compilation break point (not `create()` itself), the null-vs-`Set.of()` contract for "open channel" semantics, and the observable behaviour change from sorted canonical storage. By the time we got to implementation the callers table was tight enough that nothing was missed.

The reactive `ObligorTrustPolicy` fix (`#235`) was the cleanest. The reactive trust gate was calling `trustGateService.meetsThresholdAsync()` directly, bypassing the SPI entirely — custom policy beans were silently ignored. The fix is three lines: wrap `obligorTrustPolicy.permits(ctx)` in `Uni.createFrom().item(() -> ...).runSubscriptionOn(Infrastructure.getDefaultWorkerPool())`. That shifts the blocking JPA call off the Vert.x I/O thread and honours the SPI in both paths simultaneously. I added a structural test to `BlockingTierPurityTest` that verifies `ReactiveMessageService` has `obligorTrustPolicy` injected rather than `trustGateService` directly — it will catch the regression if someone wires it wrong again.

The A2A SSE streaming (`#147`) was where the session got genuinely interesting. The spec went three rounds. The first version had a race condition in `deregisterStream()` — the non-atomic `isEmpty()` + `remove()` pair that lets a new consumer be registered and then silently discarded. Fixed with `ConcurrentHashMap.compute()`. The second version had the abandoned first implementation still in §1, a missing `post()` code sketch, and the DECLINE→"cancelled" fix only half-applied. That last one was a real semantic error: DECLINE had been mapped to "failed" across all three `A2ATaskState` paths since the A2A endpoints were first built. DECLINE is an explicit agent refusal — "I won't do this." Mapping it to "failed" conflates a decision with an infrastructure error. We fixed all three paths and separated FAILURE and DECLINE in the priority system.

The implementation went smoothly until the first test run, which logged four `IllegalStateException: Response has already been written` at ERROR on every SSE stream termination. The fix is non-obvious: `SseEventSink.send()` in Quarkus RESTEasy Reactive returns a `CompletionStage<?>`. Calling `sink.close()` synchronously immediately after — the natural thing to do — races against the async write. The close needs to happen inside `.thenRun()` or `.whenComplete()` on the returned stage. The code reviewer caught it; the error is in the server logs, not the client response, so it's easy to ship without noticing.

The `@Transactional` + void SSE method combination is worth knowing. The transaction commits when the method body returns — before the sink stays open. So you can use `@Transactional` for the initial DB reads (CommitmentStore + message history), let it commit, and then events flow after commit with no transaction held. That's exactly the right behaviour for an SSE stream, and it's nowhere in the Quarkus or JAX-RS docs. It only works because Quarkus runs the method on a virtual thread.

The CDI fix (`#276`) was the smallest change with the most blast radius. `QhorusInboundCurrentPrincipal` had been changed to `@DefaultBean` in a previous session to fix a test ambiguity with `FixedCurrentPrincipal`. That worked in-repo, but `@DefaultBean` means "I yield to any non-default bean" — two `@DefaultBean` implementations of the same type cause `AmbiguousDependencies` in every consumer app. The fix is plain `@ApplicationScoped`: it's Tier 2 in the CDI priority ladder, displaces `MockCurrentPrincipal @DefaultBean` automatically, and gets displaced by `@Alternative @Priority(N)` beans when needed. The `quarkus.arc.exclude-types=MockCurrentPrincipal` workaround that had been added to four test configs could be removed entirely.

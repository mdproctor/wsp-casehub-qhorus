---
layout: post
title: "The Plumbing Was Already There"
date: 2026-07-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [opentelemetry, tracing, reactive, mutiny]
---

Qhorus had zero OpenTelemetry integration. Every COMMAND dispatch, every HANDOFF chain, every DONE/FAILURE resolution — invisible in distributed traces. The issue had been sitting there since the M6 platform coherence audit flagged it.

The first surprise was how much already existed. casehub-ledger ships a `TraceIdEnricher` that auto-populates `traceId` on every ledger entry during `save()`. It reads from `Span.current()` via an `OtelTraceIdProvider` SPI. The field is there on `MessageLedgerEntry`. The enrichment pipeline is wired. All of it running dry because nobody had put an OTel SDK on the classpath to create the spans it was reading from.

So the design became: create the spans that the existing pipeline already knows how to consume. No new database columns. No Flyway migrations. Just make `Span.current()` return something real.

I decided on explicit `Tracer` injection rather than interceptors or an abstraction layer. Four instrumentation points in the blocking stack, four in the reactive stack, plus the gateway. Each service checks a `QhorusTracingConfig` gate — all defaults true, all runtime config. When no OTel SDK is on the consumer's classpath, `Tracer` resolves to no-op and the whole thing costs nothing.

The interesting design choice was using `Instance<Tracer>` instead of direct `@Inject Tracer`. Direct injection throws `UnsatisfiedResolutionException` at CDI bootstrap when the API jar is `<optional>true</optional>` and the consumer doesn't bring it. `Instance<T>` with `isResolvable()` defers resolution past the class-loading boundary — truly optional at deployment time, not just config time.

Cross-request correlation was the part I expected to be hard but wasn't. When a DONE message resolves an earlier COMMAND, the ledger write looks up the original COMMAND's `MessageLedgerEntry` — which already has `traceId` populated by the enrichment pipeline — and adds an OTel span link. The query method (`findEarliestWithSubjectByCorrelationId`) already existed. The `traceId` was already stored. The link construction was five lines.

The reactive stack surfaced the sharpest gotcha. Mutiny's operator ordering is the opposite of what builder-pattern chaining suggests. `onTermination` chained before `onFailure` means termination fires *first* — ending the span before the error is recorded. Per the OTel spec, mutations to an ended span are no-ops. Errors silently vanish. No exception, no warning, nothing. The design review caught this before it reached production code — we reversed the ordering across all three reactive services and wrote it up as a platform protocol.

The design review itself was productive — eleven issues across three rounds, ten verified fixes in the spec. It added the LAST_WRITE overwrite path (distinct dispatch behaviour the original spec missed), delivery pump spans (AT_LEAST_ONCE backends were invisible), and the full reactive span lifecycle pattern with the correct Mutiny operator ordering.

What the consumer sees: add `quarkus-opentelemetry` to their POM. Qhorus auto-instruments. Every dispatch, commitment transition, fan-out, delivery, and ledger write becomes a span in their trace. Cross-request links connect COMMAND origins to their terminal resolutions. The config switches let them dial back volume if needed, but the defaults give full visibility with no configuration.

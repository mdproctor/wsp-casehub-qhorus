---
title: "The guard that revealed the callers"
date: 2026-06-08
tags: [casehub-qhorus, MessageDispatch, EVENT, CDI, MessageObserver, bug-fix]
---

Three bugs came in together from the quarkmind Layer 3 integration: EVENT messages silently dropping their payload, dot-notation channel names throwing a confusing error, and custom-qualified `MessageObserver` beans never receiving anything. All three were discovered by someone actually using the system. All three had silent failure modes. None of them were obvious from reading the code.

I started with #257 because it had the most design surface.

The symptom was simple: an agent sends an EVENT with a JSON payload and the observer receives null. No error. Just null. The fix looked obvious — remove the null coercion in `MessageObserverDispatcher` and let content flow through. I proposed that, and was immediately pushed back on: that's fixing the symptom. The *reason* the content is nulled is that `LedgerWriteService` extracts MCP tool telemetry from EVENT content as JSON, and if you let application content through, observers start receiving raw internal telemetry blobs. The coupling is in the ledger, not the dispatcher. The proper fix is in `casehub-ledger` — stop using `message.content` as the vehicle for telemetry, use a dedicated mechanism.

So `casehub-ledger#126` was filed, and we chose a partial fix for now: add a `telemetry(String)` field to `MessageDispatch`, route internal telemetry through it, and enforce that EVENT messages cannot carry `content` at all — throw immediately at `Builder.build()` time with a clear message pointing to STATUS as the alternative. Silent discard becomes an explicit error at the call site.

That decision was correct but its execution was more involved than I expected. Adding a guard in `Builder.build()` instantly breaks every caller that passes `EVENT + content`. I knew about eight production callers. What I didn't know about was another fifteen test callers across the suite — contract test helpers, MCP test scenarios, timeline tests — each one setting up EVENT dispatches with telemetry JSON in the content field. The build caught them all. The pattern of fixing them was mechanical: Builder callers either drop content entirely (audit EVENTs — the presence of the EVENT is the record, sender + timestamp is enough) or switch to `.telemetry()` (the cases where the JSON actually needs to reach the ledger).

The timeline mapper was the most interesting failure. `QhorusEntityMapper.toTimelineEntry()` was reading `tool_name`, `duration_ms`, and `token_count` directly from `message.content` by JSON-parsing it. After the fix, EVENT `message.content` is always null — telemetry lives in the ledger entry columns. The mapper needed a new overload that reads ledger-first, with a fallback to `message.content` for any pre-existing rows. `getChannelTimeline()` now fetches a `MessageLedgerEntry` for each EVENT in the timeline window. That's an N+1 query — one ledger lookup per EVENT in a page of up to 200 messages — tracked as #262 for a batch `findByMessageIds()` fix.

`ChannelSlugValidator` (#258) was three lines of code. The error when someone wrote `quarkmind.scouting.intel` was technically correct — "Invalid channel name segment 'quarkmind.scouting.intel'" — but entirely useless because dots aren't path separators in qhorus and the message gave no indication. It now detects dots, computes both the hyphen variant and the slash-hierarchy variant, and puts them in the error. It took longer to get the spec right than to write the fix.

The `@Any` fix (#259) was one annotation in two places. CDI's unqualified `Instance<T>` only discovers beans with the `@Default` qualifier. A `MessageObserver` with a custom qualifier like `@CaseType("starcraft-game")` is completely invisible to the dispatcher — no error, no warning, no delivery. Adding `@Any` to the injection point in both `MessageService` and `ReactiveMessageService` is the correct CDI idiom for a global broadcast SPI. The test for it needed `QuarkusTransaction.requiringNew()` rather than `@TestTransaction` because the dispatcher defers to JTA post-commit; a rolled-back test transaction means observers never fire.

That last observation became a garden entry and a protocol. The trap is non-obvious: the dispatch call returns successfully, the transaction is active, everything looks fine — and then the observer never sees anything. The only way to discover it is to read `MessageObserverDispatcher` and notice the `registerInterposedSynchronization` call.

The right design throughout was to refuse the tempting shortcut — don't remove the null coercion in the dispatcher, don't let content bleed through, file the real issue in the ledger and enforce the constraint clearly in the API. The fix is temporary by design; `casehub-ledger#126` is the place where this gets properly resolved.

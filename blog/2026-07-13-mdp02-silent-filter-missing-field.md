---
layout: post
title: "The Silent Filter and the Missing Field"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [projections, jpql, messageview]
---

## The Silent Filter and the Missing Field

Qhorus projections fold a channel's message history through a pure left-fold — `identity()`, then `apply(state, message)` for each message, producing a materialised read-model. The `project_channel` MCP tool exposes this to callers. Until today, there was no way to scope that fold to a single topic.

The infrastructure was almost there. `MessageQuery` already had a `.topic()` field, and `ProjectionService` already had a scoped overload that accepts a `MessageQuery`. The gap was exposure: the MCP tool didn't pass topic through, and `MessageView` — the only thing a projection's `apply()` sees — didn't carry the topic field.

What I didn't expect was a third gap. The design review caught it: `MessageQueryJpql`, the JPQL WHERE-clause builder that backs every JPA store query, had no predicate for `topic`. The field existed on `MessageQuery`. The in-memory stores honoured it via `MessageQuery.matches()`. But the JPA path silently ignored it — every topic-filtered query returned the full unfiltered result set, with no error and no warning.

This is the worst kind of bug. Both code paths compile. Both accept the same query object. One filters correctly, the other doesn't. And because most tests run against in-memory stores, the tests pass. You'd only see the problem in production or in an integration test running against a real database.

The fix was three lines — `LOWER(topic) = LOWER(?N)` added to both `from()` variants. Case-insensitive to match what `MessageQuery.matches()` does with `equalsIgnoreCase`. Both the tenant-unaware and tenant-scoped factories had to be updated together — they're duplicated code blocks, and missing one would recreate the same parity gap in a different dimension.

With the JPQL fix in place, the rest was straightforward. `MessageView` gained a `topic` field positioned after `target`, mirroring the `Message` entity's field order. `QhorusEntityMapper.toMessageView()` maps it through. Projections that want to group-by-topic internally can now read `message.topic()` in their `apply()`.

The `project_channel` tool gained an optional `topic` parameter. Blank strings get normalised to null at the MCP boundary — consistent with `splitCsv()` and the other null-normalisation patterns in `QhorusMcpToolsBase`. When topic is set, the tool routes through the existing scoped `ProjectionService.project()` overload via a `MessageQuery` with the topic filter. When both topic and `max_messages` are set, both constraints go on the same query. Reactive parity via `ReactiveQhorusMcpTools`.

The interesting thing about this work is what the design review surfaced. Twelve issues across three rounds, all verified — but the JPQL gap was the one that mattered. Without it, every topic-filtered projection would have silently returned channel-wide results in any JPA-backed deployment. The fix was trivial; finding it required reading the actual query builder, not the interface contract.

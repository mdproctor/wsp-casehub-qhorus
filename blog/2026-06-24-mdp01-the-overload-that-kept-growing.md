---
layout: post
title: "The Overload That Kept Growing"
date: 2026-06-24
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [refactoring, api-design, java-records]
---

## The Overload That Kept Growing

`ChannelService.create()` had five overloads. The 4-parameter version delegated to the 5-parameter, which delegated to the 6, which delegated to the 8, which finally built a `ChannelCreateRequest` and called the canonical `create(ChannelCreateRequest)`. The same chain existed in `ReactiveChannelService`. And `QhorusMcpTools` had its own parallel set of four convenience methods.

Twelve methods whose only purpose was to avoid typing `null` eight times.

The root cause was obvious: `ChannelCreateRequest` is a record with 14 fields, and positional construction of a 14-field record is hostile. Most channels need a name and a semantic. Everything else — barrier contributors, rate limits, ACLs, type constraints, connector bindings — defaults to null. But the canonical constructor demands all 14 in order, so convenience overloads sprouted to avoid the ceremony. Every time a field was added (allowed types, denied types, connector binding), the overload chain grew in three classes.

The fix was a builder. `ChannelCreateRequest.builder("work-ch").semantic(BARRIER).barrierContributors("alice,bob").build()`. Name required, semantic defaults to APPEND, everything else optional and self-documenting. The builder's `build()` delegates to the record's canonical constructor, which still runs slug validation, connector binding completeness checks, and type overlap assertions. No validation duplication.

Along the way, we extracted `Channel.fromRequest(ChannelCreateRequest, String tenancyId)` — a static factory on the entity that both `ChannelService` and `ReactiveChannelService` had implemented identically as private `populateChannel()` methods. Eleven fields mapped, two normalised with `blankToNull()`, two serialised via `MessageType.serializeTypes()`. Identical in both services, line for line. Now it's one method on the entity, and both services are one-liners.

One design question surfaced during spec review that I hadn't considered: should the canonical constructor be package-private, forcing all cross-package callers through the builder? It's a reasonable instinct — prevent the anti-pattern from recurring. But Java doesn't allow it. JLS §8.10.4.2: a public record's canonical constructor cannot have more restricted access than the record class. You'd have to abandon records entirely — losing value semantics, equals/hashCode, and pattern matching — just to hide a constructor. Not worth it.

The better answer: the public constructor is actually the enforcement mechanism. When field 15 is added, every `new ChannelCreateRequest(...)` call site breaks with a compile error. Builder callers are unaffected — the new field defaults to null. The breakage activates precisely when it matters, forcing explicit handling at field evolution time. That's better than hiding the constructor, which would prevent all direct construction including valid same-package uses.

The migration touched 44 files. Most of it was mechanical — `channelService.create("name", "desc", ChannelSemantic.APPEND, null)` became `channelService.create(ChannelCreateRequest.builder("name").description("desc").build())`. The builder makes every call site self-documenting. No more counting positional nulls.

Three other issues (#300, #301, #302) on this branch turned out to already be resolved by prior work — the CloudEvent adapter existed, the Javadoc was fixed upstream, the example tests already compiled. Closed all three without writing a line of code. The issue backlog had drifted.

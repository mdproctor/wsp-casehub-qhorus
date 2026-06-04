---
layout: post
title: "The Gate and Two Holes in It"
date: 2026-06-04
entry_type: note
subtype: diary
projects: [qhorus]
tags: [qhorus, channel, design, records, reactive, concurrent]
---

The `denied_types` enforcement work started with a design question about where validation should live. The Claudony spec called for a static `StoredMessageTypePolicy.validateNoOverlap()` method — catch bad channel configs at creation time, called from the MCP layer. The problem: `runtime/message/` already imports `runtime/channel/Channel`, so adding the reverse dependency creates a cycle. More importantly, a static method called from one place is an escape hatch. Any future creation path that doesn't remember to call it silently skips the check.

The alternative: put the validation in `ChannelCreateRequest`'s compact constructor. Java records run the compact constructor on every construction path without exception — the MCP tools, `ChannelService`, `ReactiveChannelService`, test helpers, auto-channel policies. You can't bypass it. The constraint is expressed once, at the point where the data comes into existence.

We went with the compact constructor. The spec called it D1 — the single enforcement gate.

The code review found two holes in it immediately.

The first: `ReactiveChannelService.create()` had its own inline entity construction. The compact constructor was never invoked on the reactive path at all. I'd written the spec to say D1 "covers all creation paths regardless of caller" — that was simply wrong for the reactive service until we changed it to construct `ChannelCreateRequest` before entering the Panache transaction.

The second was subtler. `ReactiveQhorusMcpTools.createChannel()` already constructed a `ChannelCreateRequest` for routing decisions (to check whether a connector binding was present). But for the non-connector path, it then destructured the request back into named parameters and called `reactiveChannelService.create(name, ..., allowedTypes)`. `deniedTypes` never made it through. Claude caught this — the destructuring was a pre-existing bug that would have silently dropped the field even if we'd added it correctly everywhere else.

The fix for both was the same: add `create(ChannelCreateRequest)` as the primary entry point on `ReactiveChannelService`, and route all overloads through it. Entity construction happens outside the Panache transaction, which looks wrong but isn't — a JPA entity is just a POJO until `channelStore.put()` commits it inside the session. The transaction doesn't need to wrap the object creation.

There was a third problem we didn't fully solve: the concurrent auto-channel test. A test asserting that only one channel gets auto-created when two threads race first-contact showed a counter incrementing twice under certain timing. We spent time on the wrong hypothesis — `@Transactional(REQUIRES_NEW)` on a non-JTA thread, JTA exception wrapping. The actual issue is simpler: the counter fires unconditionally after `findOrCreateWithBinding()` returns, but that method has two silent paths — create-new and find-existing. Both return normally. When Thread 1 completes its binding write before Thread 2 calls the initial existence check, Thread 2 takes the find-existing path without throwing, and the counter fires for both.

The fix is to return a result type that carries a `wasCreated` flag and only count genuine new creations. That's on a separate branch.

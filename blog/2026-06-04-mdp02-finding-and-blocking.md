---
layout: post
title: "The FindOrCreateResult Fix, the Full-Replacement Design, and the Silent Thread Block"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [qhorus, reactive, concurrent, mcp, blocking, vertx, design]
---

The previous entry ended with: *"The fix is to return a result type that carries a `wasCreated` flag. That's on a separate branch."* This is that branch closing.

The `FindOrCreateResult` design seems obvious in retrospect. `ChannelService.findOrCreateWithBinding()` has two normal return paths — create-new and find-existing — and the counter fires on both. Make the path explicit in the return type and only count when `wasCreated` is true.

One thing worth noting: `Channel.autoCreated` exists and is set to `true` on the create-new path. It looks like a candidate. It isn't — it answers "was this channel ever auto-created?" not "was it created by this call?". The find-existing path returns an entity where `autoCreated` is `true` permanently. The two questions look the same and aren't.

Then `set_channel_type_constraints`. Whether to allow partial updates was the design question — pass just `allowed_types` and leave `denied_types` unchanged. The answer is no, and the reason is the validation contract: the two fields must be validated together, because a type can't appear in both sets. If you allow partial updates, a caller can lose a constraint by omission. Full-replacement is the only semantics that keeps the invariant honest. The tool description says this, prominently.

For the `max_messages` parameter on `project_channel`: the original spec said "200 most recent messages." Claude flagged this before any code was written — `MessageQuery.builder().limit(N).build()` folds messages in ascending insertion order, so it gives you the first N messages, not the most recent. We updated the description to "first N in insertion order" and committed to it. Most-recent semantics would require a descending query and a reverse before folding — a different design, deferred.

The `@Blocking` issue was the interesting find. I wrote `ReactiveQhorusMcpTools.setChannelTypeConstraints` returning `Uni<ChannelDetail>` without `@Blocking` — it looked reactive. Claude caught this during code review: `resolveChannel()` in the base class calls the blocking `ChannelService.findById()` / `findByName()` — JPA, synchronous. quarkus-mcp-server dispatches `@Tool` methods on the Vert.x I/O thread by default. No compile error, no test failure. Just a blocked-thread warning in the logs and latency stall under load.

The fix is one annotation. The gotcha is that a base class helper's blocking nature is invisible from the subclass method body — the call looks like simple routing. `project_channel` in the same class already had `@Blocking` for exactly the same reason. Once you see it, the pattern is clear: any reactive tool in `ReactiveQhorusMcpTools` that calls `resolveChannel()` needs `@Blocking`. That's now a protocol.

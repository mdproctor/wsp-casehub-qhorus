---
layout: post
title: "The Rename With Teeth"
date: 2026-06-06
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [mcp, mutiny, reactive, api-design]
---

A rename of 53 parameters sounds like maintenance work. All you're doing is changing `channel_name` to `channel` and updating the description to "Channel name or UUID". Mechanical. Boring. Except the design decision lurking underneath it was anything but.

The question was: what does `resolveChannel()` return?

In the base class, `resolveChannel(String channel)` returned `UUID`. Two-phase detection — `ChannelSlugValidator.tryParseUuid()` to distinguish UUID strings from slug names, then a lookup — and you got back the identifier you needed for downstream calls. Fine when the rest of the code needed a UUID. But every blocking tool that needed anything else from the channel (its name for a service call, its entity for an admin check) then called `findByName()` or `findById()` again. Triple lookup in the worst case.

The fix was to return `Channel` instead of `UUID`. One lookup, entity in hand, callers use `ch.id` or `ch.name` as needed. On the reactive side, a matching `resolveChannelAsync(String) → Uni<Channel>` followed the same logic — UUID fast path via `findById`, name path via `findByName`, both returning the entity. Symmetric, and it let `set_channel_type_constraints` drop its `@Blocking` annotation entirely. The sole reason it had `@Blocking` was the blocking `resolveChannel()` call. With a reactive resolver, it becomes a pure `Uni` chain.

The subtlest thing we found was in `delete_channel` in the reactive class. The method resolved the channel, ran the delete, and then built the result:

```java
.flatMap(ch -> channelService.delete(channelName, force)
        .invoke(ignored -> channelGateway.closeChannel(ch.id, ...)))
.map(deleted -> new DeleteChannelResult(channelName, deleted, "deleted"));
```

That `.map()` is outside the `.flatMap()` closure. After the flatMap changes the stream type from `Uni<Channel>` to `Uni<Boolean>`, `ch` is gone — only the raw `channelName` parameter remains in scope. With name inputs it didn't matter: `channelName` was already the name. With UUID inputs, `new DeleteChannelResult(channelName, ...)` would put the UUID string in the result's name field. The code compiled. Tests using channel names passed. Claude caught it in review before we hit it in testing.

The fix: nest the terminal `.map()` inside the `.flatMap()` closure where `ch` is still in scope. Twelve characters of nesting, one real bug avoided.

The other compound case was `request_approval` — it calls `sendMessage` then `waitForReply` internally. Resolving the channel twice would have worked, but the right pattern is to resolve once at the `@Tool` boundary and thread the resolved name into the private helpers. That's now a protocol: private helpers accept `String channelName` (already resolved), never a raw UUID-or-name input.

The field rename is done. Every MCP tool that references a channel now accepts both a slug and a UUID. The follow-up (#252) will push the UUID-first pattern down into the reactive service methods themselves — so the service layer doesn't do its own findByName after the tool has already resolved. But that's a separate pass.

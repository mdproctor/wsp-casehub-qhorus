---
layout: post
title: "Nine Fixes, Two Surprises"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [qhorus, cdi, channel-backend, commitment]
---

Nine S/XS fixes in a single batch branch. Most were mechanical. Two were not.

The first surprise came from #245 — "StoredMessageTypePolicy ignores deniedTypes." I'd budgeted maybe twenty minutes for it. Opened the file. The check was already there, on lines 17–21, denial-first exactly as specified. Someone had fixed it in commit `b28a67a` as part of #243 and forgotten to close the issue. I closed it with a comment and moved on. That kind of thing is useful to find — it means the backlog is dirtier than it looks.

The second surprise was #254. `ChannelService.create()` wasn't calling `channelGateway.initChannel()`, so runtime-created channels were invisible to `ChannelBackend` dispatch. The fix seemed obvious: fire `ChannelInitialisedEvent` from `create()`. We did that. Tests passed. Code review caught the double-fire: `QhorusMcpTools.create_channel` also calls `channelGateway.initChannel()`, so every MCP-created channel was now getting the event twice.

But the deeper issue was more subtle. `ChannelGateway.initChannel()` does two things: `registry.computeIfAbsent()` to seat the qhorus-internal agent backend, and then fire the CDI event. Firing the CDI event directly skipped step one. External backends got notified. The qhorus-internal backend didn't register. `fanOut()` found an empty registry and silently discarded. The fix was to call `channelGateway.initChannel()` from `ChannelService.create()` rather than fire the event directly. That required injecting `ChannelGateway` into `ChannelService` — which sounds circular, but `ChannelGateway.onStart()` uses `CrossTenantChannelStore`, not `ChannelService`, so no cycle.

The same code review that caught the double-fire also flagged the `@DefaultBean` switch on `QhorusInboundCurrentPrincipal` (fixed in #269) and the null guard on the commitment event field. Both valid. The null guard I pushed back on — `@Inject Event<X>` is never null in CDI, but CDI-free unit tests don't wire it, and adding a mock event to every CDI-free test setup is noise for no benefit. Filed #275 to track the cleaner path.

One other thing: the existing garden entry GE-20260609-9ee2ad had predicted this exact fix, including the coupling concern and its resolution. The `invalidation_triggers` field was literally "Invalidated if ChannelService.create() is refactored to call ChannelGateway.initChannel() internally." Satisfying to mark it resolved with the implementation note.

---
layout: post
title: "Name the Pattern, Find the Copies"
date: 2026-11-24
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [anti-pattern, api-design, channel-creation, code-review]
---

# Name the Pattern, Find the Copies

Yesterday we fixed a timing race in Qhorus channel creation: `policyOverrides` was set *after* `create()`, which meant `ChannelInitialisedEvent` handlers saw null overrides. The fix was straightforward — include overrides in the `ChannelCreateRequest` builder so they're part of the channel before the event fires.

The interesting part came today. Once the create-then-configure anti-pattern had a name, we went looking for copies — and found two more in the same codebase.

The first was in the same file. `ChannelConfigBootstrap.processChannel()` had an identical post-create call for `participationPolicy`:

```java
var result = channelService.findOrCreate(request);
if (result.wasCreated()) {
    cfg.participationPolicy().ifPresent(policy ->
            channelService.setParticipationPolicy(result.channel().id(), policy));
}
```

Same bug, same shape. The `participationPolicy` was already on the builder at line 96 of `toRequest()` — it had been there since the builder API was extended. The post-create call was left over from when the builder didn't support it. Nobody noticed because no init handler reads participation policy yet. The race was latent, waiting for a consumer to trigger it.

The second was in a downstream repo. `casehub-ops` had its `ChannelProvisionHandler` calling `ops.setPolicyOverrides()` after channel creation — the exact consumer-side version of the pattern we'd fixed in the library. Same anti-pattern, copied forward because the old API shape modelled a two-phase flow and the provisioner followed it.

Three instances. All created independently. All invisible until one of them was named.

This is how anti-patterns propagate. The original API offered `create()` and `setPolicyOverrides()` as separate calls. That's not wrong in isolation — post-creation mutation is a legitimate operation for existing channels. But when a synchronous event fires during creation, the two-phase pattern has a window where the channel exists in an incomplete state. Each new consumer copies the pattern from existing code, inherits the race, and never notices because the eager observer hasn't been written yet.

The fix in each case was the same: move configuration into the creation request. The post-creation setters stay for the update path — they're correct for existing channels where no init event fires. The anti-pattern is specifically the create-then-configure sequence when create has synchronous observers.

The practical takeaway: when you fix a named pattern, grep for copies. The same API shape that created the original bug almost certainly created it elsewhere. If the pattern was subtle enough to survive code review in one place, it survived in others too.

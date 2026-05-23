---
layout: post
title: "Four fixes, three of which weren't really bugs"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
tags: [dispatch, channel-gateway, last-write, error-handling]
---

After shipping the deadline dispatch work, four small items were left on the branch. I expected to clear them quickly. One of them taught me something.

**#191** was supposed to be a bug: `parentReplyCount` always returned `0` from the LAST_WRITE fast path. Looking at it, `parentReplyCount` in `DispatchResult` means the reply count on the message being replied *to* — the `inReplyTo` parent. LAST_WRITE overwrites don't create a new reply to any parent; they update the message in place. So `0` is semantically correct. The fix was a comment explaining why. Not every bug report is a bug.

**#194** was a real one. `ChannelGateway.fanOut()` was constructing the `ChannelRef` it passes to backends with `channelId.toString()` as the channel name. Backends implementing `post(ChannelRef, OutboundMessage)` were getting a UUID string where they expected something like `case-abc/work`. The fix extended the method signature to accept `channelName` and updated `MessageService.dispatch()` to pass `ch.name`. Clean.

Then code review found the fix itself had a problem: the Javadoc said `channelName` must not be null, but the implementation silently fell back to `channelId.toString()` when it was. A contract promise the code wasn't keeping. Claude caught it in review. We replaced the silent fallback with `Objects.requireNonNull(channelName, "channelName")` — the Javadoc and the implementation now say the same thing.

The same review pass caught a second issue: the `unknown` filter for artefact refs was re-parsing strings we'd just validated:

```java
// Before: re-parsing strings the validation loop above had already processed
List<String> unknown = artefactRefs.stream()
    .filter(r -> !found.contains(java.util.UUID.fromString(r)))
    .toList();
```

The fix for **#196** (consistent UUID error messages) had built a `refUuids` list already — the `unknown` filter should use that list, not re-parse. The order-dependency between the two blocks wasn't documented and could be broken by anyone rearranging them later. We rewrote the filter to use `refUuids` directly.

**#195** was never a bug at all. The LAST_WRITE overwrite path deliberately skips the ledger write, fanOut, and commitment tracking. The question on the issue was whether that was intentional. It is: LAST_WRITE channels model latest-state-only semantics — a status heartbeat, not a history record. Writing every overwrite to the ledger would flood the audit trail with noise. Backends that need push notifications should subscribe separately; the channel models a current value, not a stream of events. We turned the intention into a comment, closed the issue.

Four fixes. One was a misread bug report, two were real issues (one found by review catching our own review fix), and one was a design decision that needed documenting. Small work. The review pass earning its keep is the part worth noting.

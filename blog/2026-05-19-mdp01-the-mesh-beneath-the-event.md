---
layout: post
title: "The Mesh Beneath the Event"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [messaging, architecture, spi, tdd]
---

Started with two blockers for clinical: qhorus#154 (correlationId in `InboundHumanMessage`) and qhorus#153 (CDI event on message receipt). Simple issues. Neither was.

**The inbound path was half-done**

The design question for #154 was whether to add `correlationId` to `InboundHumanMessage` alone, or expand `NormalisedMessage` to carry the full parameter set that `messageService.send()` accepts. The argument for the fuller design: `ChannelGateway.receiveHumanMessage()` was hard-coding four parameters to null with no principled reason — the normaliser SPI is supposed to be the complete translation point.

We expanded `NormalisedMessage` to seven fields: `type`, `content`, `senderInstanceId`, `correlationId`, `inReplyTo`, `artefactRefs`, `target`. The gateway became a clean 1:1 mapping. `ClinicalInboundNormaliser` now passes `correlationId` through, so clinical's PI backend can supply the deviation UUID and have the commitment auto-fulfill on DONE/DECLINE without a bypass workaround.

**The CDI event that wasn't**

#153 asked for a CDI event on message receipt so clinical could observe PI responses without polling. The obvious implementation — fire `Event<MessageReceivedEvent>` from `MessageService.send()` — works fine for an embedded harness. It's wrong for a platform where agents can be anywhere.

I asked where qhorus was supposed to sit in the topology: same JVM, same machine, LAN, WAN. All of them. CDI events don't cross JVM boundaries. If `MessageService` fires a CDI event, distributed harnesses get nothing.

We looked at Vert.x EventBus (local vs cluster consumers), EIP Channel Adapter patterns, MicroProfile Reactive Messaging. The answer was consistent: the right abstraction is a transport-agnostic SPI. We built `MessageObserver`:

```java
@FunctionalInterface
public interface MessageObserver {
    void onMessage(MessageReceivedEvent event);
    default Scope scope() { return Scope.LOCAL; }
    enum Scope { LOCAL, CLUSTER }
}
```

`InProcessMessageBus` is the `@DefaultBean` — fires a CDI event for embedded harnesses, zero configuration, zero overhead. Future Kafka or WebSocket implementations declare `Scope.CLUSTER`. Multiple implementations coexist via `Instance<MessageObserver>`.

The accompanying architecture document (`docs/messaging-architecture.md`) diagrams the two notification paths — ChannelBackend fanOut (per-channel, targeted) and MessageObserver (global, pluggable transport) — and explains why both exist. Claude caught something I missed in code review: `fireAsync()` inside `@Transactional` dispatches before the transaction commits. The `MessageReceivedEvent` record is intentionally self-contained — channelName, channelId, type, senderId, correlationId, content — so observers don't need to query the database at all.

Claudony flagged a gap the document had glossed over: in a multi-node fleet, each node has its own channel store and observer set. The backend model doesn't fix cross-node delivery. That's now documented explicitly as a known gap (qhorus#162).

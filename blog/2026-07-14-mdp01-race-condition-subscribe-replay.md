---
layout: post
title: "The Race Condition in Subscribe-Then-Replay"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [websocket, concurrency, catch-up]
---

WebSocket connections are ephemeral. When a client disconnects, it misses every message until it reconnects. The obvious fix is a cursor: the client sends its last-seen message ID on reconnect, the server replays everything after that point.

The obvious fix has a race condition.

## The Gap Nobody Talks About

The catch-up pattern for event-driven WebSocket connections looks clean on paper. Client reconnects with `?lastEventId=42`. Server subscribes the connection to live events, queries the message store for everything after ID 42, sends the history, then live events flow. Client deduplicates by message ID — if a live event arrives with an ID already seen in the replay, discard it.

I designed exactly this for Qhorus's WebSocket observer module. `MessageQuery.poll(channelId, afterId, limit)` already existed. `MessageReceivedEvent` needed a `messageId` field — it had been missing since the beginning, which meant no observer could correlate back to the original message. Adding it was the right first-principles fix regardless of catch-up.

The design felt complete. Then it went through adversarial review.

## What the Review Found

The race isn't between catch-up and live events arriving — it's between catch-up and live events being *sent on the wire*. `sendTextAndAwait` is thread-safe on Quarkus WebSocket connections, but thread-safe writes don't mean ordered writes. The catch-up replay runs on a virtual thread (`@OnOpen`). The `WebSocketMessageObserver` fires from the JTA `afterCompletion` callback on a different thread. Both call `sendTextAndAwait` on the same connection.

The result: a catch-up message with ID 43 could arrive *after* a live message with ID 50, depending on which thread's request reaches the Vert.x event loop first. Client-side dedup by monotonic cursor doesn't help — the client would accept message 50 (live), advance its cursor past 43, then discard the catch-up message 43 when it arrives late. The client silently drops a message.

## Server-Side Buffering

The fix moves ordering responsibility to the server. `WebSocketConnectionRegistry` gains a catch-up buffer: when a connection is in catch-up mode, `WebSocketMessageObserver` doesn't send to it directly — it buffers the message (with its ID and serialised JSON) in a per-connection queue. After the catch-up replay finishes, the endpoint atomically drains the buffer and sends any buffered messages with IDs higher than the last replayed message. Only then does the connection transition to live mode.

This guarantees a single writer to the connection at all times during catch-up. The catch-up thread owns the connection until it calls `completeCatchUp`, which atomically clears the buffer and returns its contents. The observer thread buffers or sends directly — never both — based on whether the buffer exists.

The implementation is straightforward: a `ConcurrentHashMap<WebSocketConnection, List<BufferedMessage>>` alongside the existing channel subscription map. Five methods — `subscribeCatchingUp`, `completeCatchUp`, `tryBufferForCatchUp`, `cancelCatchUp`, and a defensive cleanup in `unsubscribe`. Error handling wraps the entire catch-up sequence; any failure calls `cancelCatchUp` to discard the buffer before the exception propagates.

The catch-up itself is simple: parse `lastEventId` from the query string, validate the channel via `CrossTenantChannelStore`, query `MessageQuery.poll` with `maxMessages + 1` to detect truncation, convert each message via a new `MessageReceivedEvent.fromMessage()` factory, and send control frames (`catchup_begin`, `catchup_end` or `catchup_truncated`) to bracket the replay.

A configurable cap (`casehub.qhorus.websocket.catchup.max-messages`, default 500) prevents unbounded replay for clients that were offline for hours. When the gap exceeds the cap, a `catchup_truncated` frame tells the client how many messages it missed — the client can fetch history via REST if needed.

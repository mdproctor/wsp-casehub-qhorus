# Topic-Aware Projections

**Issue:** #338  
**Date:** 2026-07-13  
**Status:** Approved

## Problem

The projection infrastructure — `ProjectionService` with its scoped `project(channelId, scope, projection)` overload and `MessageQuery.builder().topic(topic)` — already supports topic-scoped folds at the service level. However, this capability is not exposed to MCP tool callers, and the JPA query layer silently ignores the topic filter:

1. The `project_channel` MCP tool does not expose a `topic` parameter — callers cannot request a topic-scoped projection
2. `MessageQueryJpql` does not translate `topic` to a WHERE clause — JPA-backed stores silently return all messages regardless of the topic filter (in-memory stores work correctly via `MessageQuery.matches()`)
3. `MessageView` does not carry the `topic` field — projections that want to group-by-topic internally cannot see which topic each message belongs to

## Design

Two orthogonal layers, both implemented:

### Layer 1 — Query-time topic filter

Add an optional `topic` parameter to the `project_channel` MCP tool. When set, the tool builds `MessageQuery.builder().topic(topic)` and routes through the existing scoped `ProjectionService.project(channelId, scope, projection)` overload. Any existing or future projection works with topic filtering — no SPI changes.

**Prerequisite fix — `MessageQueryJpql`:** Both `from(MessageQuery)` and `from(MessageQuery, String)` variants must add a `topic` predicate. Topic matching is case-insensitive — `LOWER(m.topic) = LOWER(?n)` — to match `MessageQuery.matches()` semantics, which uses `topic.equalsIgnoreCase(m.topic())`. Without this fix, topic filtering silently returns all messages in JPA-backed stores.

**`projectAndRender` restructuring:** The existing `projectAndRender(channelId, projection, maxMessages)` only enters the scoped code path when `maxMessages > 0`. This method gains a `topic` parameter and is restructured: any non-null `topic` OR positive `maxMessages` triggers the scoped path via `MessageQuery`. When both are set, both constraints go on the same `MessageQuery`. When only `topic` is set (no `maxMessages`), a `MessageQuery` with just the topic filter is built. The unscoped fallthrough path is only reached when both `topic` is null and `maxMessages` is null/non-positive.

Reactive parity: `ReactiveQhorusMcpTools.projectChannel()` gets the same parameter.

### Layer 2 — Topic field on MessageView

Add `String topic` (nullable) to `MessageView`, positioned after `target` to mirror the field order in the `Message` record. Map it in `QhorusEntityMapper.toMessageView()` from `msg.topic()`. Projections that want to group-by-topic internally can read `message.topic()` in their `apply()`.

Breaking change to `MessageView` record constructor — pre-release, callers update. Affected constructor call sites: `QhorusEntityMapper.toMessageView()` (production), `RenderableProjectionTest.msg()` and `ProjectionFoldLogicTest.view()` (test helpers).

## What does NOT change

- `ChannelProjection` — base fold interface, no new methods
- `RenderableProjection` SPI — no new methods  
- `ProjectionService` — all 4 overloads unchanged
- `ProjectionRegistry` — unchanged
- `ProjectionResult` — unchanged. Note: `lastMessageId` is a channel-level cursor, not topic-scoped. A topic-filtered fold returns the `lastMessageId` of the last message folded within that topic, which is still a position in the overall channel insertion order. An incremental fold with a different topic and the same cursor would skip messages before that cursor even if they belong to the new topic — correct channel-cursor semantics.
- `MessageQuery` — already has `.topic()`

## Testing

- Unit: `MessageView` construction with topic field
- Unit: `MessageQueryJpql` generates correct `LOWER(topic) = LOWER(?n)` predicate when topic is set
- Integration: `project_channel` with topic filter — fold only messages in one topic
- Integration: `project_channel` with topic only (no `maxMessages`) — verify scoped path is taken
- Integration: projection reading `MessageView.topic()` to group internally
- Edge: topic filter with no matching messages returns identity
- Edge: null topic = fold all messages (existing behaviour preserved)
- Edge: topic matching is case-insensitive — JPA and in-memory stores agree

# Topic-Aware Projections

**Issue:** #338  
**Date:** 2026-07-13  
**Status:** Approved

## Problem

The projection system folds over a channel's entire message history. There is no way to:
1. Scope a projection to a single topic at query time
2. Let a projection see which topic each message belongs to

## Design

Two orthogonal layers, both implemented:

### Layer 1 — Query-time topic filter

Add an optional `topic` parameter to the `project_channel` MCP tool. When set, the tool builds `MessageQuery.builder().topic(topic)` and routes through the existing scoped `ProjectionService.project(channelId, scope, projection)` overload. Any existing or future projection works with topic filtering — no SPI changes.

`projectAndRender` in `QhorusMcpToolsBase` gains topic handling. When both `topic` and `maxMessages` are set, both go on the same `MessageQuery`.

Reactive parity: `ReactiveQhorusMcpTools.projectChannel()` gets the same parameter.

### Layer 2 — Topic field on MessageView

Add `String topic` (nullable) to `MessageView`. Map it in `QhorusEntityMapper.toMessageView()` from `msg.topic()`. Projections that want to group-by-topic internally can read `message.topic()` in their `apply()`.

Breaking change to `MessageView` record constructor — pre-release, callers update.

## What does NOT change

- `ChannelProjection` SPI — no new methods
- `RenderableProjection` SPI — no new methods  
- `ProjectionService` — all 4 overloads unchanged
- `ProjectionRegistry` — unchanged
- `ProjectionResult` — unchanged
- `MessageQuery` — already has `.topic()`

## Testing

- Unit: `MessageView` construction with topic field
- Integration: `project_channel` with topic filter — fold only messages in one topic
- Integration: projection reading `MessageView.topic()` to group internally
- Edge: topic filter with no matching messages returns identity
- Edge: null topic = fold all messages (existing behaviour preserved)

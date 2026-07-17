---
layout: post
title: "Notifications Are Not Conversations"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [notifications, platform-consolidation, commitment-lifecycle]
series: issue-365-notification-bridge
---

Platform has a full notification system — per-user lifecycle (UNREAD → READ → DISMISSED), subscription matching, quiet hours, digest batching, multi-channel delivery. Qhorus has membership with cursor-based unread tracking, WebSocket push, and presence. The question was whether qhorus should adopt the platform's notification primitives.

The answer turned out to be simpler than expected: these systems solve different problems at different granularity, and conflating them would be a regression in both.

A conversation with fifty new messages should not create fifty notifications. Qhorus's `lastReadMessageId` cursor is O(1) to track and O(1) to update — it represents "how far have I scrolled." Platform's per-item notification lifecycle represents "have I acknowledged this alert." One is a position in a stream; the other is an individual acknowledgment. Replacing the cursor with per-message notifications would be both a performance regression and a UX one.

The integration point is one level up: qhorus as a notification *source* for the platform system. Not every message — that would be noise. Only commitment lifecycle transitions, which are already deterministic in the system:

- COMMAND dispatched with a named target → notify the obligor
- DONE or FAILURE on a correlation → notify the requester
- `CommitmentDeclinedEvent` → notify the requester
- `CommitmentExpiredEvent` → notify the requester (URGENT)

Every trigger maps to an existing CDI event or a specific `MessageType` + `correlationId` combination. The recipient is always derivable from the commitment's `obligor` or `requester` fields. No heuristics, no "should this be a notification?" logic.

The bridge is a new optional module — `notification-bridge/` — with a `MessageObserver` (scope LOCAL) for the message-triggered cases and a CDI `@ObservesAsync` pair for the commitment lifecycle events. It calls `NotificationStore.store()` directly with a `NotificationInput`. When no platform persistence module is on the classpath, the `NoOpNotificationStore` @DefaultBean absorbs the calls silently. The module activates by presence, like the other observer modules.

What makes this work cleanly is that the two systems stay in their lanes. Qhorus owns the conversation experience — membership, cursor-based read tracking, WebSocket streaming, presence. Platform owns the notification experience — individual alerts with lifecycle, preferences, suppression, delivery routing. The bridge connects them at the commitment boundary, where the semantics are unambiguous: someone owes you something, or something you asked for was resolved.

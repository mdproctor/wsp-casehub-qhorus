---
title: "Thirteen Operations, One Adapter"
date: 2026-09-11
entry_type: note
subtype: diary
author: mdp
projects: [casehub-qhorus]
tags: [graphql, messaging, migration, adapter-pattern]
refs:
  - casehubio/qhorus#436
  - casehubio/qhorus#409
---

The GraphQL migration's second domain landed today: messaging. Thirteen operations — five queries and eight mutations — ported from `QhorusMcpTools` into `MessagingQueryResolver` and `MessagingMutationResolver`.

The interesting constraint: the `graphql/` module only depends on `casehub-qhorus-api`, not the runtime. Every resolver method works through API-layer interfaces (`ConsumerMessaging`, `MessageReader`, `ReactionReader`, `ReactionManager`, `MessageStore`, `CommitmentStore`). No runtime service imports. This is the thin adapter pattern the design spec established — resolvers map GraphQL inputs to API calls and API records to DTOs.

Most operations were mechanical. `message`, `replies`, `searchMessages` — read through existing interfaces, map to `MessageType.from()`. Reactions needed a grouping step because `ReactionReader.findByMessage()` returns raw `Reaction` records, not pre-grouped `ReactionGroup` objects. A six-line `Collectors.groupingBy` in the resolver handles it without needing a new API method.

The `waitForReply` and `requestApproval` mutations were the exception. These are blocking poll loops — the agent sends a QUERY, then polls `CommitmentStore` and `MessageStore` in a backoff loop until a RESPONSE or DONE arrives, or the timeout expires. The original `@Tool` methods use runtime classes (`messageService.findResponseByCorrelationId`), but `MessageQuery.builder().correlationId().messageType().limit(1)` gives the same result through the API-layer `scan()`. No new facade methods needed.

The design spec flagged this as a risk: SmallRye GraphQL dispatches mutations on Vert.x worker threads, so a 90-second `waitForReply` blocks a worker for the duration. That risk holds. The spec accepted it for now — a subscription-based approach is the right long-term fix.

Code review caught one thing worth noting: `deleteMessage` performs four independent store calls (find replies, orphan them, dispatch audit EVENT, delete the message). Without `@Transactional`, a failure partway through leaves the database inconsistent — orphaned replies with null `inReplyTo` pointing at a message that still exists. The other resolver methods delegate to single API calls that handle their own transactions, so this was the only method that needed explicit transaction demarcation.

Six sub-issues remain on the epic. Next up: agents domain — `InstanceManager` facade and presence tracking.

## D1: Broadcast delivery mechanism

**Choice:** New `ChannelSemantic.BROADCAST` in qhorus with platform notifications as a delivery backend via `NotificationChannelBackend`
**Alternatives:**
- Platform notifications only (TargetType.CAPABILITY) — deepens the parallel system gap; qhorus wouldn't understand broadcast semantically
- New pub/sub layer in qhorus — adds a third messaging primitive alongside channels and notifications
- Plain qhorus channels with APPEND semantic — heavy-weight governance for fire-and-forget signals
**Rationale:** Broadcast coordination hints are fire-and-forget, fan-out, transient. Instead of making notifications a parallel routing system, qhorus owns the semantic model (BROADCAST channels) and the notification system becomes a delivery transport — the same pattern as Slack, webhooks, and Kafka backends. Messages still flow through `MessageService.dispatch()` → `ChannelGateway.fanOut()`, getting ledger entries, observer dispatch, and all existing transports. Auto-membership based on capability tags provides the "subscribe to topics" primitive.
**Trade-offs:** Adds a new channel semantic (BROADCAST) with different behavior (no commitments, auto-membership). NotificationChannelBackend is a new optional module. More implementation surface than pure notification targeting — but architecturally cleaner.
**Sources:** ChannelSemantic enum, ChannelGateway.fanOut(), notification-bridge existing pattern, ChannelBackend SPI
**Exploration:** deep-analysis (revised from quick after user challenged the initial notification-only approach)
**Status:** captured

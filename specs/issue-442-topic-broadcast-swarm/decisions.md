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

## D2: Auto-membership mechanism

**Choice:** Capability-driven auto-join — when an agent registers capabilities via `InstanceService.register()`, qhorus auto-creates broadcast channels for each capability tag (convention: `broadcast:<capability>`) and auto-joins the agent as a member
**Alternatives:**
- Explicit subscription — agents manually subscribe to broadcast topics independently of capabilities; more control but more ceremony
- Interest declaration at registration — separate "interests" list from capabilities; finer-grained but adds registration complexity
**Rationale:** The simplest model: capabilities already declare "what I can do." If you can do X, you should hear about X. The channel naming convention `broadcast:<capability>` makes channels discoverable. Agent deregistration removes membership automatically.
**Trade-offs:** No way to opt out of hearing about a capability you have. If this becomes a problem, the interest-based model can be layered on later without breaking the auto-join default.
**Depends on:** D1 (BROADCAST channels)
**Sources:** InstanceService.register(), CapabilityEntity, ChannelMembershipService.join()
**Exploration:** quick
**Status:** captured

## D3: Allowed message types on BROADCAST channels

**Choice:** STATUS and EVENT only — deny COMMAND, QUERY, PROPOSE, DONE, FAILURE, DECLINE, RESPONSE, HANDOFF by default
**Alternatives:**
- All non-commitment types — more permissive but blurs the broadcast/conversation boundary
- Unrestricted — maximum flexibility but undermines the fire-and-forget premise
**Rationale:** BROADCAST = observe, channels = converse. Commitment-creating types (COMMAND, QUERY, PROPOSE) and their resolution types create obligations that conflict with fire-and-forget semantics. STATUS (content-bearing observation) and EVENT (content-free signal) are the natural speech acts for coordination hints.
**Trade-offs:** Strict. If someone needs request/reply on a broadcast topic, they must create a separate channel. This is intentional — broadcast and conversation are different coordination patterns.
**Depends on:** D1 (BROADCAST semantic)
**Sources:** MessageType 10-type taxonomy (ADR-0005), event-content-free-signal-type protocol
**Exploration:** quick
**Status:** captured

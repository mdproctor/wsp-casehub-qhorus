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

## D4: Broadcast channel lifecycle

**Choice:** Keep channel when empty, mark zero members. No auto-delete.
**Alternatives:**
- Auto-delete after grace period — cleaner but loses channel-level config and risks thrashing during agent restarts
- Never auto-delete — same as chosen, but without the explicit "mark empty" signal
**Rationale:** The channel slug, ledger history, and any customized config (rate limits, enforcement) have value. Agents restart frequently — auto-delete would cause channel thrashing. An agent re-registering a capability simply re-joins the existing channel.
**Trade-offs:** Broadcast channels accumulate over time. Acceptable — they're lightweight (no active backends when empty) and can be manually cleaned up via `delete_channel`.
**Depends on:** D1 (BROADCAST semantic), D2 (auto-membership)
**Sources:** ChannelMembershipService, InstanceService.register()
**Exploration:** quick
**Status:** captured

## D5: P2P ephemeral channels — scope

**Choice:** Out of scope for #442. File as a separate follow-up issue.
**Alternatives:**
- Include in #442 — both are "lightweight channel convenience" but they're different coordination patterns
**Rationale:** P2P ephemeral channels are a distinct pattern (1:1 temporary, auto-close) from broadcast (1:N persistent, auto-membership). #442 is well-scoped with BROADCAST + auto-membership + notification backend. P2P can build on the same infrastructure later.
**Trade-offs:** The issue description asked for both. The follow-up issue should reference #442 and the hive mind epic.
**Sources:** Issue #442 body
**Exploration:** quick
**Status:** captured

## D6: NotificationChannelBackend module placement

**Choice:** Merge into the existing `notification-bridge/` module
**Alternatives:**
- New `notification-channel-backend/` module — clean separation but redundant with notification-bridge
**Rationale:** Both are "qhorus → platform notifications" bridges. notification-bridge currently has a MessageObserver (commitment events) and a SubscriptionBootstrap. Adding a ChannelBackend (broadcast delivery) to the same module keeps the single-dependency story clean. One module, one concern: "deliver qhorus events to the platform notification system."
**Trade-offs:** The module grows from pure CDI-free unit tests to needing ChannelBackend test infrastructure. Manageable — RecordingChannelBackend is available in casehub-qhorus-testing.
**Depends on:** D1 (BROADCAST semantic)
**Sources:** notification-bridge/ module structure, ChannelBackend SPI
**Exploration:** quick
**Status:** captured

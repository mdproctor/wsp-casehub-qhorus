# Decisions — qhorus#495: Enforce Mesh Participation on Message Posting

## D1: Policy architecture — three-level configuration

**Choice:** Three-level: global config default + per-channel override on Channel model + SPI for custom consumer logic
**Alternatives:**
- Global + per-channel only — simpler, but can't handle consumer-specific logic (e.g., Claudony enriching auto-registered instances with session context)
- Global only — single config property, can't differentiate system channels from work channels
**Rationale:** Matches existing qhorus patterns (enforcementMode, allowedWriters, rateLimits all follow global-default + per-channel-override). SPI follows the @DefaultBean/@Alternative pattern used by CommitmentAttestationPolicy, ObligorTrustPolicy, InstanceActorIdProvider.
**Trade-offs:** More surface area than a single config toggle. Three configuration points to reason about.
**Sources:** MessageService.java (enforcement gate pattern), AllowedWritersPolicy.java (per-channel ACL pattern), system-sender-trust-exemption protocol
**Exploration:** quick
**Status:** captured

## D2: Policy modes — OPEN / AUTO_REGISTER / ENFORCE

**Choice:** Three modes: OPEN (no check, current behaviour), AUTO_REGISTER (auto-enroll sender as mesh instance), ENFORCE (reject unregistered senders)
**Alternatives:**
- Add ADVISORY as fourth mode — log warning + DispatchAdvisory but allow message. Useful for migration, but adds a fourth mode for an edge case that OPEN already covers (just enable logging)
- Just AUTO_REGISTER / ENFORCE — no OPEN mode, forces every deployment to pick. Breaks backward compatibility.
**Rationale:** OPEN preserves backward compatibility (existing deployments don't break). AUTO_REGISTER is the least-disruptive enforcement (existing code keeps working, mesh layer always populated). ENFORCE is the strict option for production channels that need it. Global default is OPEN.
**Trade-offs:** OPEN as default means new deployments must explicitly opt into enforcement. Migration from OPEN to ENFORCE requires auditing existing senders.
**Sources:** qhorus#495 issue proposal (Options 1-3), existing Channel.enforcementMode pattern
**Exploration:** quick
**Status:** captured

## D3: Auto-registration shape — SPI with minimal @DefaultBean

**Choice:** MeshAutoRegistrar SPI with a @DefaultBean that creates minimal instances (instanceId=sender, status=online, no capabilities). Consumers provide @Alternative for enriched auto-registration.
**Alternatives:**
- Minimal instance only, no SPI — hardcode minimal creation, add SPI later. YAGNI, but Claudony already needs enriched instances (session binding, capabilities from pool context).
- Infer actorType as capability — use dispatch's actorType as a capability tag. Risks stale data since the sender didn't explicitly declare capabilities.
**Rationale:** The SPI is the mechanism; the minimal instance is just the default implementation. Claudony needs enriched auto-registration (session binding, capability tags from pool context). Pattern is proven — same as CommitmentAttestationPolicy.
**Trade-offs:** SPI consumers must handle the full MessageDispatch context, which could change shape across versions.
**Depends on:** D2 (AUTO_REGISTER mode)
**Sources:** InstanceService.register() (existing registration API), CommitmentAttestationPolicy (SPI pattern)
**Exploration:** quick
**Status:** captured

## D4: Gate position — before AllowedWritersPolicy in dispatch pipeline

**Choice:** After tenant + paused checks, before AllowedWritersPolicy. System senders (contains ':') bypass the check.
**Alternatives:**
- After all existing checks — simpler insertion but auto-registered instances miss the capability-tag ACL lookup since registration happened after AllowedWritersPolicy ran
- Outside dispatch entirely (in ChannelCore/MeshService) — keeps dispatch() unchanged but splits the check across entry points; cluster InternalMeshResource would need its own check
**Rationale:** Auto-registration must happen before AllowedWritersPolicy so that SPI-provided capabilities are visible to capability-based ACL entries. Single enforcement point in dispatch() covers all entry paths (REST, MCP, cluster proxy). System sender exemption is consistent with the existing system-sender-trust-exemption protocol.
**Trade-offs:** Modifies the core dispatch() method — the most critical code path in qhorus. Must be carefully tested for performance (instance lookup on every dispatch).
**Depends on:** D1 (three-level config), D2 (modes), D3 (SPI shape)
**Sources:** MessageService.dispatch() line 251-264 (AllowedWritersPolicy position), system-sender-trust-exemption protocol
**Exploration:** quick
**Status:** captured

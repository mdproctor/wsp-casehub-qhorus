# Design: Enforce Mesh Participation on Message Posting

**Issue:** casehubio/qhorus#495
**Date:** 2026-10-24
**Status:** Draft

## Problem

`MessageService.dispatch()` accepts messages from any sender string without verifying the sender is a registered mesh instance. The enforcement gate has ~10 checks (tenant, paused, ACL, rate limit, routing, trust, type policy, correlation integrity, protocol evaluation, enforcement blocking) — but none verify instance registration.

This means:
- Messages bypass the mesh layer — no instance registration, no presence, no capability discovery
- The Mesh view shows "no active channels" even when channels have messages, because senders were never registered
- The distinction between "through the mesh" and "direct to REST" is invisible at the data level

## Design

### Configurable Mesh Participation Policy

A three-level configuration determines what happens when an unregistered sender posts a message:

1. **Global default** — `casehub.qhorus.mesh.participation-policy` config property
2. **Per-channel override** — new `participationPolicy` field on `Channel`, null inherits global
3. **SPI** — `MeshParticipationPolicy` interface controls auto-registration shape

### Policy Modes

| Mode | Behaviour | Use case |
|------|-----------|----------|
| `OPEN` | No registration check. Current behaviour. **Global default.** | Backward compatibility, development, open channels |
| `AUTO_REGISTER` | If sender not registered, auto-register via SPI before continuing dispatch | Production channels where mesh visibility matters but breaking existing callers is unacceptable |
| `ENFORCE` | If sender not registered, reject with `IllegalStateException` | Locked-down channels where only known participants may post |

### System Sender Exemption

Senders containing `:` (e.g., `casehub-engine:orchestrator`, `system:watchdog`) bypass the participation check entirely. This is consistent with the existing `system-sender-trust-exemption` protocol — system senders represent infrastructure components, not mesh participants.

### Gate Position in Dispatch Pipeline

The participation check sits in `MessageService.dispatch()` after tenant + paused validation, before `AllowedWritersPolicy`:

```
1. Cross-tenant check
2. Paused channel check
3. ★ Mesh participation check (NEW)
4. AllowedWritersPolicy ACL
5. RateLimiter
6. RoutingBridge
7. ObligorTrustPolicy
8. MessageTypePolicy
9. CorrelationIntegrityChecker
10. Protocol evaluation
11. Enforcement gate
```

This position ensures that auto-registered instances (with SPI-provided capabilities) are visible to capability-based ACL entries in step 4.

### ParticipationPolicyMode Enum

```java
// api/src/main/java/io/casehub/qhorus/api/channel/ParticipationPolicyMode.java
public enum ParticipationPolicyMode {
    OPEN,
    AUTO_REGISTER,
    ENFORCE;

    public boolean requiresLookup() {
        return this != OPEN;
    }
}
```

Located in `api/channel/` alongside `EnforcementMode`, `ChannelSemantic`, and `MemberRole`.

### Channel Model Addition

```java
// On Channel record — typed enum, matching enforcementMode pattern
ParticipationPolicyMode participationPolicy;  // null = inherit global default
```

The JPA `ChannelEntity` stores this as `String participationPolicy` and converts via `ParticipationPolicyMode.valueOf()` in `toDomain()`/`fromDomain()`, matching the existing `enforcementMode` pattern.

Exposed via `ChannelsApi` (`@McpDomain`) — a new Mesh Participation section with `@PlatformQuery` getter and `@PlatformMutation` setter, following the existing Enforcement and Policy Overrides pattern. Also exposed via `ChannelResource` REST API.

### Config-File Per-Channel Support

`QhorusConfig.ChannelConfig` gets a new field for static per-channel configuration in `application.properties`:

```java
// In QhorusConfig.ChannelConfig
Optional<ParticipationPolicyMode> participationPolicy();
```

This allows operators to configure participation policy alongside other channel settings at deployment time:

```properties
casehub.qhorus.channels.work-channel.participation-policy=AUTO_REGISTER
casehub.qhorus.channels.system-events.participation-policy=OPEN
```

`ChannelConfigBootstrap` reads this and sets it on channel creation, with drift detection for existing channels (WARN, no auto-correct) — same as all other config-bootstrapped fields.

### MeshAutoRegistrar SPI

```java
// api/src/main/java/io/casehub/qhorus/api/spi/MeshAutoRegistrar.java
public interface MeshAutoRegistrar {

    /**
     * Called when AUTO_REGISTER mode fires for an unregistered sender.
     * Returns the instance to register, or empty to skip auto-registration
     * (effectively treating this dispatch as OPEN for this sender).
     *
     * Exceptions propagate and fail the dispatch — the sender is not
     * registered and the message is not delivered.
     */
    Optional<InstanceRegistration> autoRegister(MessageDispatch dispatch);

    record InstanceRegistration(
        String instanceId,
        String description,
        List<String> capabilityTags,
        boolean readOnly,
        Map<String, String> metadata
    ) {}
}
```

Named `MeshAutoRegistrar` (not `MeshParticipationPolicy`) to distinguish the SPI's purpose (how to shape an auto-registered instance) from the config's purpose (which mode to use). The config property and enum control the participation *policy*; the SPI controls the auto-registration *strategy*.

**Error handling:** Exceptions from `autoRegister()` propagate and fail the dispatch. The sender is not registered and the message is not delivered. This is the correct semantic: if the SPI can't determine how to register this sender, the safe outcome is rejection, not silent OPEN fallback.

**@DefaultBean implementation** (in `runtime-core`): creates a minimal instance — `instanceId=sender`, `description="Auto-registered by mesh participation policy"`, empty capabilities, `readOnly=false`, empty metadata.

**Consumer override example** (Claudony): provides an `@Alternative @Priority(1)` that enriches the instance with session binding, capability tags from pool context, and case metadata.

### Participation Check Logic

```java
// In MessageService.dispatch(), after paused check, before AllowedWritersPolicy
if (ch != null && dispatch.type() != MessageType.EVENT && !dispatch.sender().contains(":")) {
    ParticipationPolicyMode mode = resolvePolicy(ch);  // per-channel → global fallback
    if (mode.requiresLookup()) {
        boolean registered = instanceService.findByInstanceId(dispatch.sender()).isPresent();
        if (!registered) {
            if (mode == ParticipationPolicyMode.ENFORCE) {
                throw new IllegalStateException(
                    "Sender '" + dispatch.sender() + "' is not a registered mesh instance. "
                    + "Register via meshRegister or POST /api/instances before posting.");
            }
            // AUTO_REGISTER — exception propagates and fails dispatch
            meshAutoRegistrar.autoRegister(dispatch).ifPresent(reg ->
                instanceService.register(reg.instanceId(), reg.description(),
                    reg.capabilityTags(), null, reg.readOnly(), reg.metadata()));
        }
    }
}
```

**CDI wiring:** `MeshAutoRegistrar` is injected into `MessageService` via `CdiMessageService` (new constructor parameter, following the existing 27-parameter pattern). The global default is resolved from `QhorusConfig.Mesh.participationPolicy()` at construction time and stored as a field.

### Policy Resolution

```java
private ParticipationPolicyMode resolvePolicy(Channel ch) {
    if (ch.participationPolicy() != null) {
        return ch.participationPolicy();  // typed enum, no valueOf() needed
    }
    return globalDefault;  // from QhorusConfig.Mesh at construction
}
```

### Configuration

```properties
# Global default (backward compatible)
casehub.qhorus.mesh.participation-policy=OPEN

# Per-channel override via config file (static, at channel creation)
casehub.qhorus.channels.work-channel.participation-policy=AUTO_REGISTER

# Per-channel override via REST or MCP (dynamic, at runtime)
# PUT /api/channels/{id}/participation-policy  body: { "policy": "AUTO_REGISTER" }
```

`QhorusConfig.Mesh` nested interface:

```java
interface Mesh {
    @WithDefault("OPEN")
    ParticipationPolicyMode participationPolicy();
}
```

### Instance Registration Scope

Auto-registration is **global** — once a sender is registered by any channel's AUTO_REGISTER policy, they are a registered mesh instance everywhere. This is correct: mesh instances represent agents, not channel memberships. Per-channel access control is handled by `AllowedWritersPolicy` (step 4), which provides fine-grained gating via capability-based ACL entries.

An operator should not expect ENFORCE to be a channel-level admission gate. ENFORCE ensures mesh visibility — the sender must be a known entity. Channel-level access control (who can write where) remains the responsibility of `allowedWriters`.

### Audit Trail

Auto-registration fires `InstanceRegisteredEvent` via `InstanceService.register()` — CDI observers (including `BroadcastMembershipManager`) see it. No additional ledger entry or EVENT message is generated. The dispatch itself produces the normal ledger entry with the sender string, which is now a registered instance. The `InstanceRegisteredEvent` is the audit signal for auto-registration.

### EVENT Type Exemption

`EVENT` messages bypass the participation check (same as existing ACL and rate-limit exemptions). EVENTs are system telemetry signals — they should not trigger instance registration or be blocked by enforcement.

### Performance Consideration

The participation check adds one `instanceService.findByInstanceId()` lookup per non-OPEN dispatch. This is a primary-key lookup on the instance store (O(1) in-memory, indexed in JPA). For OPEN mode (the default), no lookup occurs — zero performance impact on existing deployments.

For channels with AUTO_REGISTER, after the first message the instance is registered and the lookup becomes a cache-friendly hit. The cost is one extra read per dispatch, which is negligible compared to the message persistence, ledger write, and fan-out already in the pipeline.

### Migration Path

1. Deploy with `OPEN` (default) — no change in behaviour
2. Set specific channels to `AUTO_REGISTER` — existing senders auto-enroll, mesh layer populates
3. Audit instance registry — verify all expected participants are registered
4. Optionally flip channels to `ENFORCE` — only registered senders can post

### Flyway Migration

New column on `channel` table:

```sql
ALTER TABLE channel ADD COLUMN participation_policy VARCHAR(20);
-- null = inherit global default
```

### Test Plan

- **Unit tests (MessageService):** OPEN allows unregistered sender; AUTO_REGISTER calls SPI and registers; ENFORCE rejects unregistered sender; system senders bypass all modes; EVENT type bypasses; per-channel override takes precedence over global; null channel override inherits global; SPI exception propagates and fails dispatch; SPI returns empty skips registration (OPEN fallback)
- **Unit tests (MeshAutoRegistrar):** @DefaultBean creates minimal instance; custom @Alternative provides enriched instance; autoRegister returning empty skips registration
- **Unit tests (ParticipationPolicyMode):** enum values; requiresLookup() returns false for OPEN, true for AUTO_REGISTER and ENFORCE
- **Unit tests (Channel model):** participationPolicy typed enum field round-trips through Channel builder; null vs explicit values; ChannelEntity String↔enum conversion
- **Unit tests (QhorusConfig):** global default OPEN; config override parses correctly
- **Integration test:** full dispatch pipeline with AUTO_REGISTER — verify instance exists after dispatch, AllowedWritersPolicy can see auto-registered capabilities, InstanceRegisteredEvent fires
- **Integration test (config bootstrap):** ChannelConfigBootstrap reads participationPolicy from config, sets on channel creation
- **Backward compatibility:** existing tests pass with OPEN default — zero behavioural change

### Implementation Notes

- `ChannelsApi` needs a new Mesh Participation section with `@PlatformQuery` and `@PlatformMutation` — follow the existing Enforcement section pattern
- `CdiMessageService` constructor gets `MeshAutoRegistrar` as a new parameter
- `ARC42STORIES.MD` §6 pipeline diagram needs updating to include the new gate
- Agent-bridge module: existing registration path satisfies ENFORCE mode — the bridge registers instances as part of binding creation. No changes needed, but verify in integration test.

## References

- `runtime-core/.../message/MessageService.java` — dispatch enforcement pipeline (lines 208-623)
- `runtime-core/.../channel/AllowedWritersPolicy.java` — per-channel ACL pattern
- `runtime-core/.../instance/InstanceService.java` — instance registration API
- `docs/protocols/casehub/system-sender-trust-exemption.md` — system sender convention
- `api/.../spi/CommitmentAttestationPolicy.java` — SPI @DefaultBean pattern reference
- `runtime-core/.../config/QhorusConfig.java` — existing config mapping pattern
- casehubio/qhorus#495 — issue with problem statement and original three options

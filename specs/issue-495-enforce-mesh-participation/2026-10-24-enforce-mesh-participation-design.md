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

### Channel Model Addition

```java
// On Channel record/entity
String participationPolicy;  // null = inherit global default; "OPEN" | "AUTO_REGISTER" | "ENFORCE"
```

Exposed via existing `ChannelResource` REST API and Qhorus MCP tools for channel configuration.

### MeshParticipationPolicy SPI

```java
// api/src/main/java/io/casehub/qhorus/api/spi/MeshParticipationPolicy.java
public interface MeshParticipationPolicy {

    /**
     * Called when AUTO_REGISTER mode fires for an unregistered sender.
     * Returns the instance to register, or empty to skip auto-registration
     * (effectively treating this dispatch as OPEN for this sender).
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

**@DefaultBean implementation** (in `runtime-core`): creates a minimal instance — `instanceId=sender`, `description="Auto-registered by mesh participation policy"`, empty capabilities, `readOnly=false`, empty metadata.

**Consumer override example** (Claudony): provides an `@Alternative @Priority(1)` that enriches the instance with session binding, capability tags from pool context, and case metadata.

### Participation Check Logic

```java
// In MessageService.dispatch(), after paused check, before AllowedWritersPolicy
if (ch != null && dispatch.type() != MessageType.EVENT && !dispatch.sender().contains(":")) {
    ParticipationPolicyMode mode = resolvePolicy(ch);  // per-channel → global fallback
    if (mode != ParticipationPolicyMode.OPEN) {
        boolean registered = instanceService.findByInstanceId(dispatch.sender()).isPresent();
        if (!registered) {
            if (mode == ParticipationPolicyMode.ENFORCE) {
                throw new IllegalStateException(
                    "Sender '" + dispatch.sender() + "' is not a registered mesh instance. "
                    + "Register via meshRegister or POST /api/instances before posting.");
            }
            // AUTO_REGISTER
            participationPolicy.autoRegister(dispatch).ifPresent(reg ->
                instanceService.register(reg.instanceId(), reg.description(),
                    reg.capabilityTags(), null, reg.readOnly(), reg.metadata()));
        }
    }
}
```

### Policy Resolution

```java
private ParticipationPolicyMode resolvePolicy(Channel ch) {
    if (ch.participationPolicy() != null) {
        return ParticipationPolicyMode.valueOf(ch.participationPolicy());
    }
    return globalDefault;  // from QhorusConfig
}
```

### Configuration

```properties
# Global default (backward compatible)
casehub.qhorus.mesh.participation-policy=OPEN

# Per-channel override via REST or MCP
# PUT /api/channels/{id}/participation-policy  body: { "policy": "AUTO_REGISTER" }
```

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

- **Unit tests (MessageService):** OPEN allows unregistered sender; AUTO_REGISTER calls SPI and registers; ENFORCE rejects unregistered sender; system senders bypass all modes; EVENT type bypasses; per-channel override takes precedence over global; null channel override inherits global
- **Unit tests (MeshParticipationPolicy):** @DefaultBean creates minimal instance; custom @Alternative provides enriched instance; autoRegister returning empty skips registration
- **Unit tests (Channel model):** participationPolicy field round-trips through Channel builder; null vs explicit values
- **Integration test:** full dispatch pipeline with AUTO_REGISTER — verify instance exists after dispatch, AllowedWritersPolicy can see auto-registered capabilities
- **Backward compatibility:** existing tests pass with OPEN default — zero behavioural change

## References

- `runtime-core/.../message/MessageService.java` — dispatch enforcement pipeline (lines 208-623)
- `runtime-core/.../channel/AllowedWritersPolicy.java` — per-channel ACL pattern
- `runtime-core/.../instance/InstanceService.java` — instance registration API
- `docs/protocols/casehub/system-sender-trust-exemption.md` — system sender convention
- `api/.../spi/CommitmentAttestationPolicy.java` — SPI @DefaultBean pattern reference
- `runtime-core/.../config/QhorusConfig.java` — existing config mapping pattern
- casehubio/qhorus#495 — issue with problem statement and original three options

# ObligorTrustPolicy SPI — Design Spec
**Issue:** casehubio/qhorus#213  
**Date:** 2026-05-30  
**Branch:** issue-213-obligor-trust-policy-spi

---

## Problem

`MessageService.dispatch()` and `ReactiveMessageService.dispatch()` contain an inline
`if` block that calls `TrustGateService.meetsThreshold(target, minTrust)` from
`casehub-ledger` directly. Both services inject `TrustGateService`. The threshold
comes from `QhorusConfig.commitment().minObligorTrust()`.

This means there is no extension point. If Claudony needs capability-scoped trust
evaluation — e.g. `meetsThreshold(actorId, capabilityDimension, minTrust)` rather than
a global score — it cannot override the gate without patching qhorus itself. This is
inconsistent with `AllowedWritersPolicy`, which is an injectable concrete bean
that consumers can `@Alternative` override.

---

## Design

### New types — `api/spi/`

```java
public record ObligorTrustContext(String obligorId, long channelId, String channelName) {}
```

Both `channelId` (stable key) and `channelName` (semantic, used by Claudony to map to
a capability dimension) are carried. The channel entity is already loaded in
`MessageService` at the point the gate fires — no extra query.

```java
@FunctionalInterface
public interface ObligorTrustPolicy {
    /**
     * Returns true if the obligor is trusted to act as commitment fulfiller
     * for a COMMAND on this channel.
     *
     * <p>Called only for COMMAND messages with a named (non-prefixed) target.
     * Role- and capability-prefixed targets bypass the gate entirely — there is
     * no specific obligor to evaluate.
     */
    boolean permits(ObligorTrustContext ctx);
}
```

Placed in `api/spi/` per the `consumer-spi-placement` protocol — Claudony implements
this interface; it must not need the full runtime as a dependency.

### Default implementation — `runtime/message/`

```java
@DefaultBean
@ApplicationScoped
public class DefaultObligorTrustPolicy implements ObligorTrustPolicy {

    @Inject QhorusConfig config;
    @Inject TrustGateService trustGateService;

    @Override
    public boolean permits(ObligorTrustContext ctx) {
        double minTrust = config.commitment().minObligorTrust();
        if (minTrust <= 0.0) return true;          // gate disabled
        return trustGateService.meetsThreshold(ctx.obligorId(), minTrust);
    }
}
```

The config check (`minObligorTrust > 0.0`) moves here from `MessageService` — callers
no longer read config. Custom implementations are free to ignore `minTrust` entirely
and apply per-capability or per-channel thresholds.

### `MessageService` — updated gate block

`TrustGateService` injection removed. `ObligorTrustPolicy` injected instead. Gate
condition simplified (no inline config check):

```java
if (ch != null && dispatch.type() == MessageType.COMMAND
        && dispatch.target() != null
        && !dispatch.target().contains(":")) {
    if (!obligorTrustPolicy.permits(
            new ObligorTrustContext(dispatch.target(), ch.id, ch.name))) {
        throw new IllegalStateException(
                "COMMAND rejected: obligor '" + dispatch.target()
                + "' trust score below threshold");
    }
}
```

`ReactiveMessageService` receives the identical change.

### Error message

The error message no longer embeds the threshold value. The default impl's decision
is opaque to callers; only the gate outcome matters. Threshold-specific messaging
lives in `DefaultObligorTrustPolicy` if needed in future.

---

## Testing

**Existing tests:** `TrustGateTest` tests the gate behaviour end-to-end via
`MessageService.dispatch()`. It continues to pass unchanged — the observable
behaviour is identical; only the wiring changes. `TrustGateProfile` still sets
`casehub.qhorus.commitment.min-obligor-trust=0.5` and this flows through the
default impl as before.

**New test:** `ObligorTrustPolicySpiTest` (CDI unit test, no `@QuarkusTest`) —
demonstrates that an `@Alternative @Priority(1)` implementation overrides the
default and that `MessageService` honours it. Uses `InMemory*Store` from
`casehub-qhorus-testing`.

---

## What Does Not Change

- Gate conditions (COMMAND type, named non-prefixed target) remain in `MessageService`
- `TrustGateService` stays in `casehub-ledger`; `DefaultObligorTrustPolicy` wraps it
- `TrustGateProfile` test profile is unchanged
- No Flyway migration (schema-free change)
- `CLAUDE.md` `api/spi/` description updated to include `ObligorTrustPolicy` and `ObligorTrustContext`

---

## Files Changed

| File | Change |
|------|--------|
| `api/src/.../api/spi/ObligorTrustPolicy.java` | New interface |
| `api/src/.../api/spi/ObligorTrustContext.java` | New record |
| `runtime/src/.../runtime/message/DefaultObligorTrustPolicy.java` | New default impl |
| `runtime/src/.../runtime/message/MessageService.java` | Swap TrustGateService → ObligorTrustPolicy |
| `runtime/src/.../runtime/message/ReactiveMessageService.java` | Same |
| `runtime/src/.../runtime/message/ObligorTrustPolicySpiTest.java` | New SPI override test |
| `CLAUDE.md` | Update api/spi/ description |

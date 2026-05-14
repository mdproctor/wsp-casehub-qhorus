# Design: Gate quarkus-hibernate-reactive-panache via Capabilities (qhorus#141)

**Date:** 2026-05-14  
**Issue:** casehubio/qhorus#141  
**Protocol:** PP-20260514-f41258

---

## Problem

`casehub-qhorus` declares `quarkus-hibernate-reactive-panache` as a compile-scope
dependency in `runtime/pom.xml`. Quarkus extensions activate by classpath presence,
not by property. The result: the reactive extension activates unconditionally for every
consumer, forcing JDBC-only apps (and H2-based tests) to suppress it with three
workaround properties. The existing `casehub.qhorus.reactive.enabled=false` flag does
not help — it gates qhorus's own CDI beans but cannot prevent the Quarkus
hibernate-reactive extension from activating at augmentation time.

Affected consumers with applied workarounds: claudony, devtown, aml. Follow-on issues
filed for workaround removal: claudony#112, devtown#25, aml#13.

---

## Approach

Quarkus-native fix: `<optional>true</optional>` on the dep + `ExcludedTypeBuildItem`
via `Capabilities.isPresent(Capability.HIBERNATE_REACTIVE)` in the deployment
processor. Classpath presence becomes the single source of truth. No user-facing flag.

**Not chosen:** module split (`casehub-qhorus-reactive`) — valid but unnecessary
complexity; the native approach achieves the same result with fewer moving parts.

---

## Design

### 1. `runtime/pom.xml`

Mark `quarkus-hibernate-reactive-panache` as `<optional>true</optional>`. Consumers no
longer receive it transitively. The dep remains for the extension's own compilation.

### 2. `QhorusBuildConfig` (deployment)

Remove the `reactive()` sub-interface and `Reactive` inner interface entirely.
Capability detection at the deployment processor level replaces the flag.

### 3. `QhorusProcessor` (deployment)

Replace `ReactiveEnabled.getAsBoolean()` — from `config.reactive().enabled()` to
`capabilities.isPresent(Capability.HIBERNATE_REACTIVE)`. `Capabilities` is injected
as a field on the BooleanSupplier.

Remove `markReactiveBeans()` — unremovability is no longer needed once `@Priority(1)`
handles CDI selection and `ExcludedTypeBuildItem` handles exclusion.

Add two new build steps:

**`excludeReactiveBeansWhenAbsent`** `@BuildStep(onlyIfNot = ReactiveEnabled.class)`  
Produces `ExcludedTypeBuildItem` for every class that references reactive Panache types.
Excluded when `HIBERNATE_REACTIVE` capability is absent:
- `ReactiveQhorusMcpTools`
- `ReactiveA2AResource`, `ReactiveAgentCardResource`
- `ReactiveMessageService`, `ReactiveChannelService`, `ReactiveInstanceService`
- `ReactiveWatchdogService`, `ReactiveDataService`
- `ReactiveLedgerWriteService`, `ReactiveMessageLedgerEntryRepository`
- `MessageReactivePanacheRepo`
- All reactive JPA stores: `ReactiveJpaMessageStore`, `ReactiveJpaChannelStore`,
  `ReactiveJpaInstanceStore`, `ReactiveJpaDataStore`, `ReactiveJpaWatchdogStore`
- All reactive Panache repos: `MessageReactivePanacheRepo`,
  `ChannelReactivePanacheRepo`, `InstanceReactivePanacheRepo`,
  `SharedDataReactivePanacheRepo`, `WatchdogReactivePanacheRepo`,
  `CapabilityReactivePanacheRepo`, `ArtefactClaimReactivePanacheRepo`

**`excludeBlockingBeansWhenReactivePresent`** `@BuildStep(onlyIf = ReactiveEnabled.class)`  
Produces `ExcludedTypeBuildItem` for blocking counterparts when reactive IS present:
- `QhorusMcpTools`
- `A2AResource`, `AgentCardResource`
- `A2AChannelBackend`, `A2AActorResolver`

### 4. Reactive `@Alternative` beans — add `@Priority(1)`

All reactive service and store `@Alternative` beans gain `@jakarta.annotation.Priority(1)`.
CDI auto-selects them over blocking counterparts when they are not excluded.

Beans requiring `@Priority(1)`:
- `ReactiveMessageService`, `ReactiveChannelService`, `ReactiveInstanceService`
- `ReactiveWatchdogService`, `ReactiveDataService`
- `ReactiveLedgerWriteService`, `ReactiveMessageLedgerEntryRepository`
- All reactive JPA stores and Panache repos listed above

### 5. Remove `@IfBuildProperty` / `@UnlessBuildProperty` from all classes

Every `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", ...)` and
`@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", ...)` annotation is
removed. The `ExcludedTypeBuildItem` mechanism in `QhorusProcessor` replaces them.

Classes to strip: `ReactiveQhorusMcpTools`, `ReactiveA2AResource`,
`ReactiveAgentCardResource`, `QhorusMcpTools`, `A2AResource`, `AgentCardResource`,
`A2AChannelBackend`, `A2AActorResolver`.

---

## Consumer Experience

| Consumer intent | Action required | Result |
|---|---|---|
| JDBC only (no reactive) | Nothing | Zero reactive activation |
| Reactive stack | Add `quarkus-hibernate-reactive-panache` + reactive driver | Full reactive stack auto-activates |

---

## Testing

### Unit / build-time
- `ToolOverloadDiscoverabilityTest` — existing; must still pass (guards @Tool method
  name conflicts; not affected by this change)

### Integration — blocking path (existing, must stay green)
- All 1035 passing tests continue to pass — the blocking stack is unchanged

### Integration — capability detection
- New test: `ReactiveCapabilityExclusionTest` — verifies that when
  `HIBERNATE_REACTIVE` capability is absent (H2, no reactive driver), none of the
  reactive classes appear in the CDI container and the blocking tools/resources are
  active. Uses `@QuarkusTest` with default H2 profile.

### Integration — reactive path (remains @Disabled until Docker available)
- Existing `@Disabled` reactive test suite unchanged — enabling requires
  Docker + `quarkus-reactive-pg-client` (tracked separately)

---

## Out of Scope

- Enabling the `@Disabled` reactive tests (Docker dependency — separate concern)
- Removing workaround properties from claudony, devtown, aml (tracked: claudony#112,
  devtown#25, aml#13 — done after this ships)
- Assessing engine/persistence-hibernate and flow reactive deps (engine#253, qhorus#152)

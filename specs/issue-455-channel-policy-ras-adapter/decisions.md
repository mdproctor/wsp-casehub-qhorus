# Design Decisions — Issue #455: Channel Policy + RAS Adapter

## D1: Design scope — all three layers

**Choice:** Design all three layers as a cohesive whole: (1) structured protocol output, (2) channel policy YAML model, (3) RAS adapter. Implement what fits in this session, defer the rest to child issues.
**Alternatives:**
- Layers 1+2 only — structured output + policy model, RAS adapter deferred. Risk of misalignment at the bridge.
- Layer 1 only — minimal change, delays the unifying vision.
**Rationale:** The three layers have a natural dependency chain (1 → 2 → 3). Designing them together prevents interface mismatches between qhorus and RAS. Implementation can still be incremental.
**Trade-offs:** Larger design surface; risk of over-designing the YAML format before real-world usage.
**Sources:** casehubio/qhorus#455, casehubio/casehub-ras#66, ChannelProtocol.java, RAS SituationDefinition.java
**Exploration:** quick
**Status:** captured

## D2: Evidence type — structured Map + human-readable message

**Choice:** `ProtocolViolation` carries both `message()` (human-readable String) and `evidence()` (`Map<String, Object>` for programmatic consumption). This is non-negotiable for HIL review surfaces and RAS integration.
**Alternatives:**
- String only — simpler but RAS must parse strings, losing programmatic extraction.
- Map only — loses human-readable display for dashboards and enforcement event logs.
**Rationale:** Protocol violations are surfaced for human-in-the-loop review (enforcement event logs, dashboards, watchdog alerts). They're also consumed programmatically by RAS for temporal pattern detection. Both paths are first-class. Matches RAS's own `DetectionResult.evidence` pattern.
**Trade-offs:** Slightly larger API surface (two fields instead of one).
**Sources:** DispatchResult.java, EnforcementExecutor.java, DetectionResult.java
**Exploration:** quick
**Status:** captured

## D3: Action authority — protocol suggests, channel decides

**Choice:** `ProtocolViolation` carries `severity` and `suggestedAction`. The enforcement gate uses the channel's `enforcementMode` as the primary authority but factors in severity (CRITICAL can upgrade enforcement). `suggestedAction` is advisory metadata for RAS/logging, not a dispatch-time gate.
**Alternatives:**
- Protocol decides — suggestedAction is binding. Reduces channel operator control.
- Channel decides only — drop suggestedAction. Loses information about protocol intent for RAS.
**Rationale:** Channel operators control policy (which protocols, what enforcement mode). Protocols carry domain expertise (what severity, what action is appropriate). The enforcement gate composes both: channel mode sets the floor, protocol severity can upgrade. suggestedAction informs RAS situation responses without coupling dispatch-time behavior.
**Trade-offs:** Two-dimensional enforcement logic (mode × severity) is more complex than single-dimensional.
**Depends on:** D2 (structured evidence enables severity-aware enforcement)
**Sources:** EnforcementExecutor.java, enforceIfRequired() in MessageService.java, EnforcementMode.java
**Exploration:** quick
**Status:** captured

## D4: Advisory unification — evolve TaggedAdvisory

**Choice:** Evolve `TaggedAdvisory` to carry `severity`, `evidence`, `suggestedAction` alongside existing `source` and `message`. Non-protocol sources (TYPE_POLICY, CORRELATION_INTEGRITY) produce structured data too. `ProtocolViolation` is the SPI return type in `api/spi/`; `TaggedAdvisory` is the internal enforcement pipeline type in `runtime-core/`.
**Alternatives:**
- Unify into ProtocolViolation everywhere — simpler (one type) but name misleading for TYPE_POLICY.
- Keep TaggedAdvisory minimal — enforcement stays dumb, loses structured data at the pipeline boundary.
**Rationale:** Clean layering: SPI type (ProtocolViolation) for protocol implementors, internal type (TaggedAdvisory) for the dispatch pipeline. One-to-one mapping. Both carry the same structure. Non-protocol sources (TYPE_POLICY = CRITICAL, CORRELATION_INTEGRITY = WARNING) get severity too, enabling the enforcement gate to discriminate.
**Trade-offs:** Two parallel record types with similar fields. The mapping is trivial (one-liner) but could feel redundant.
**Depends on:** D3 (severity/suggestedAction design)
**Sources:** TaggedAdvisory.java, MessageService.java lines 270-311
**Exploration:** quick
**Status:** captured

## D5: Policy storage — hybrid YAML files + DB overrides

**Choice:** Base policies defined in YAML files loaded at startup. Per-channel DB overrides for thresholds and parameters. Channel config references policy names via existing `Channel.protocols` field.
**Alternatives:**
- External YAML only — clean but no runtime customization per channel.
- DB-stored only — runtime-configurable but adds compilation complexity and cache invalidation.
**Rationale:** Matches the existing pattern: built-in protocols are Java classes registered at startup, channels reference them by name. The hybrid extends this: YAML-defined policies replace (or supplement) Java protocol classes, and channels can override thresholds without modifying the policy definition. RAS's `YamlSituationDefinitionProvider` establishes the YAML-at-startup pattern.
**Trade-offs:** Two configuration sources (YAML + DB) increase operational complexity. Override semantics (what can be overridden, merge strategy) need clear documentation.
**Sources:** ProtocolRegistry.java, RuntimeBeans.java line 216, YamlSituationDefinitionProvider in RAS
**Exploration:** quick
**Status:** captured

## D6: Event bridge — extract from existing message CloudEvents

**Choice:** The RAS adapter extracts commitment lifecycle state from existing message CloudEvents (COMMAND → OPEN, STATUS → ACKNOWLEDGED, DONE → FULFILLED, etc.). No new CDI events added to qhorus-api. Flow control: only channels with active temporal policies ("RAS-active" channels) emit to RAS. Event type filtering narrows further — only message types matching registered situations are forwarded.
**Alternatives:**
- Add CommitmentStateChangedEvent — cleaner for RAS but adds coupling.
- Both paths — maximum signal, maximum API surface.
**Rationale:** Message CloudEvents already carry all commitment lifecycle data (message type + correlationId = commitment state transition). The adapter can derive state changes without qhorus knowing RAS exists. Keeps the qhorus-api surface minimal and the dependency direction clean (RAS → qhorus-api, never reverse).
**Trade-offs:** The adapter must understand message type → commitment state mapping (duplicates some CommitmentService logic). This is acceptable — the mapping is a pure function over the 10-type taxonomy.
**Sources:** QhorusCloudEventAdapter.java, MessageType.java, CommitmentService.java, RAS CLAUDE.md (CDI event observation pattern)
**Exploration:** quick
**Status:** captured

## D7: Situation binding — both reference and inline

**Choice:** Channel policy YAML supports both referencing pre-built RAS situations by name (with parameter overrides) AND defining inline situation definitions for channel-specific temporal patterns. The adapter compiles both into `SituationRegistration` objects for RAS at startup.
**Alternatives:**
- Reference only — clean separation but forces every temporal pattern into a standalone RAS definition.
- Inline only — couples the policy format to RAS internals for every policy.
**Rationale:** Simple policies reference existing situations from a RAS "library" (e.g., `ack-timeout` with a custom window). Complex or one-off patterns define situations inline without polluting the shared library. The adapter handles both uniformly.
**Trade-offs:** Two syntax paths in the YAML format. Need clear documentation on when to reference vs inline.
**Depends on:** D5 (hybrid storage model)
**Sources:** SituationDefinition.java, SituationRegistration.java, GanglionDescriptor.java
**Exploration:** quick
**Status:** captured

## D8: Severity override — CRITICAL can upgrade enforcement

**Choice:** CRITICAL violations block even in ADVISORY-mode channels. WARNING follows channel enforcement mode. ADVISORY severity always just logs. Severity can upgrade enforcement but never downgrade it below the channel's configured mode.
**Alternatives:**
- Channel mode is ceiling — severity is purely informational. ADVISORY mode never blocks.
- Configurable per-channel — severityOverrideThreshold field. Maximum control, more config.
**Rationale:** Analogous to a circuit breaker: protocols should be able to signal "this is dangerous enough to override policy." The channel operator still controls which protocols are active (and can remove a protocol entirely). CRITICAL override provides a safety floor — a protocol detecting a genuine safety issue shouldn't be silenced by an ADVISORY-mode channel.
**Trade-offs:** Reduces channel operator autonomy for CRITICAL violations. Operators who want full control must review which protocols can emit CRITICAL severity.
**Depends on:** D3 (protocol suggests, channel decides)
**Sources:** enforceIfRequired() in MessageService.java, EnforcementMode.java
**Exploration:** quick
**Status:** captured

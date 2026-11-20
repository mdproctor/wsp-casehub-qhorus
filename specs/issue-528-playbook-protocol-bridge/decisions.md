# Decisions — Playbook Protocol Bridge (#528, #529)

## D1: Playbook YAML storage

**Choice:** Dedicated `ProtocolDefinitionStore` in the bridge module. Definitions load lazily when evaluation needs them. Channel carries protocol names only (existing `List<String> protocols`).
**Alternatives:**
- Inline on Channel as Map<String, String> — simple but ChannelEntity is loaded on every dispatch; TEXT-column YAML bloats the most-queried entity in the system
- SharedData artefact — reuses artefact infrastructure but adds indirection; definitions aren't artefacts
**Rationale:** Channel is the hot-path entity (loaded every dispatch via MessageService). Playbook YAML is multi-kilobyte source code, not lightweight config like policyOverrides. A dedicated store loads definitions only when the bridge's evaluate() fires — not on every dispatch. The bridge module owns the entity, store, and migration — no changes to Channel or core code.
**Trade-offs:** Adds a store interface + JPA entity + InMemory impl + migration in the bridge module. Justified by keeping the dispatch hot path lightweight.
**Sources:** Channel.java (protocols field — names only), PushNotificationConfigEntity (bridge-owned JPA entity precedent), R1-01 decision review
**Exploration:** quick → revised after decision review (R1-01)
**Status:** revised

## D2: Module placement

**Choice:** New optional Maven module `casehub-qhorus-playbook-bridge`
**Alternatives:**
- In runtime-core with Instance<> guard — simpler but pulls platform playbook types into qhorus core compilation
- SPI-only in api/ with platform providing impl — maximum decoupling but platform must depend on qhorus-api
**Rationale:** Follows the established pattern of connector-backend, slack-channel, a2a-outbound. Clean classpath boundary — qhorus works without platform playbook deps. Activates by classpath presence (CDI discovery), not @IfBuildProperty.
**Trade-offs:** One more module to maintain. Acceptable given the pattern is well-established.
**Sources:** connector-backend/, slack-channel/, a2a-outbound/ (all optional modules), bridge-module-spi-placement protocol
**Exploration:** quick
**Status:** captured

## D3: Integration with ProtocolRegistry

**Choice:** Fallback `ProtocolResolver` SPI on ProtocolRegistry
**Alternatives:**
- Bridge provides wildcard ChannelProtocol CDI bean — requires a "catch-all" concept in ProtocolRegistry, bigger refactor
- Replace ProtocolRegistry with ProtocolEvaluator SPI — cleaner but larger change surface across MessageService
**Rationale:** Minimal change to existing code. `forProtocols()` checks static map first (zero overhead for existing protocols), falls back to resolver for unmatched names. The resolver is an optional `Instance<>` injection — absent when bridge module not on classpath.
**Trade-offs:** ProtocolRegistry gains a second resolution path, adding a small amount of complexity. The fallback is only invoked for names not in the static map, so existing protocol performance is unchanged.
**Sources:** ProtocolRegistry.java (forProtocols() line 47-58), RuntimeBeans.java (producer at line 306-310), optional-platform-spi-instance-injection protocol
**Exploration:** quick
**Status:** captured

## D4: ProtocolResolver SPI placement

**Choice:** In `api/spi/` alongside ChannelProtocol
**Alternatives:**
- In runtime-core alongside ProtocolRegistry — tighter coupling, bridge module would need to depend on runtime-core
**Rationale:** Follows consumer-spi-placement protocol. The SPI is a consumer-facing extension point.
**Trade-offs:** None significant.
**Sources:** consumer-spi-placement protocol, api/spi/ (existing SPI location)
**Exploration:** quick
**Status:** captured

## D5: Playbook output to DispatchAdvisory mapping

**Choice:** Convention-based output keys with bridge-owned validation
**Alternatives:**
- Typed PlaybookAdvisoryResult in the compiler — couples compiler to qhorus advisory vocabulary
- Raw output passthrough — loses granularity (one run = one advisory)
- Pure convention without validation — runtime failures on typos (e.g., `severety` instead of `severity`), invalid enum values silently dropped or crash dispatch
**Rationale:** The playbook's `advisory` step produces severity/message/action in output. The bridge defines a `PlaybookProtocolOutput` record specifying the expected structure, and validates playbook output against it at evaluation time. Invalid entries produce clear error advisories (source="playbook-bridge:validation") instead of silent failures. The compiler remains uncoupled — validation lives in the bridge where the dependency on both DispatchAdvisory and PlaybookRunResult already exists.
**Trade-offs:** Validation code in the bridge module. Justified by catching typos and enum mismatches at evaluation time with clear diagnostics.
**Sources:** DispatchAdvisory record (api/spi/), PlaybookRunnerCore (advisory step output), R1-03 decision review
**Exploration:** quick → revised after decision review (R1-03)
**Status:** revised

## D6: Timer event dispatch mechanism

**Choice:** @Scheduled sweep dispatches synthetic EVENT message to the channel
**Alternatives:**
- Direct playbook invocation from scheduler — bypasses MessageService.dispatch(), creates parallel evaluation path (violates message-service-dispatch-enforcement-gate protocol)
- Hook into CommitmentExpiredEvent — only works for commitment-based deadlines, not arbitrary protocol timeouts
**Rationale:** Dispatching a synthetic EVENT (sender=`system:protocol-timer`) triggers normal dispatch → protocol evaluation. The playbook sees the timeout event in its ProtocolContext input. Reuses the full enforcement pipeline. Consistent with the ecosystem pattern of @Scheduled sweep → condition check → fire event.
**Trade-offs:** Synthetic EVENT messages appear in channel history and the ledger. This is a feature, not a bug — it creates an audit trail of deadline firings.
**Sources:** WatchdogEvaluationService (sender="system:watchdog" pattern), CommitmentService.expireOverdue() (sweep pattern), message-service-dispatch-enforcement-gate protocol
**Exploration:** quick
**Status:** captured

## D7: Deadline tracking mechanism

**Choice:** In-memory ConcurrentHashMap with lifecycle recovery — no JPA entity
**Alternatives:**
- Dedicated JPA entity (ProtocolDeadline) — durable, cluster-safe, but protocol deadlines are derived state, not primary domain data
- Extend commitments with protocol deadline semantics — zero new infrastructure but semantically wrong; pollutes obligation stats, stalled queries, and normative audit trail
**Rationale:** Protocol deadlines are derived state: the YAML definition is the source of truth for WHAT, message history is the source of truth for WHEN. The deadline registration is a cache. The existing ProtocolLifecycleManager.onChannelInitialised(recovered=true) path handles restart recovery — the handler recomputes firesAt from YAML duration + last relevant message timestamp. Past-due deadlines fire immediately on recovery.
**Trade-offs:** In-memory means a gap in clusters if the owning node dies before another node recovers the channel. This is the same gap ProtocolLifecycleManager's ConcurrentHashMap already has — startup recovery replays all channels.
**Sources:** ProtocolLifecycleManager.java (ConcurrentHashMap pattern, onChannelInitialised recovery), CommitmentService.expireOverdue() (sweep pattern), CommitmentCountCapacitySource (why commitments are wrong — protocol deadlines would inflate capacity pressure)
**Exploration:** deep-analysis
**Status:** captured

## D8: Deadline declaration format

**Choice:** Playbook YAML frontmatter metadata section
**Alternatives:**
- Dedicated YAML section outside frontmatter — requires PlaybookParser changes
- Separate deadline configuration on Channel — splits definition from the protocol it governs
**Rationale:** `PlaybookFrontMatter.metadata` is `Map<String, Object>` — the documented extension point for protocol-specific metadata. No compiler or parser changes needed. Example: `metadata: { deadlines: { approval: PT4H } }`.
**Trade-offs:** Convention-based — the bridge must know to look in `metadata.deadlines`. Acceptable because the bridge owns the contract between YAML and runtime.
**Sources:** PlaybookFrontMatter (metadata field), PlaybookParser
**Exploration:** quick
**Status:** captured

## D9: Bridge and timer in same module

**Choice:** Both bridge (#528) and timer (#529) in `casehub-qhorus-playbook-bridge`
**Alternatives:**
- Separate modules — more granular but the timer can't work without the bridge (deadline definitions come from playbook YAML), making separation artificial
**Rationale:** They share the same dependency (playbook executor), the timer feeds events through the bridge, and a single classpath activation is simpler.
**Trade-offs:** None — the coupling is inherent.
**Sources:** slack-channel/ (single module for backend + thread cache + bindings), a2a-push-notification/ (single module for backend + poster + config + cleanup)
**Exploration:** quick
**Status:** captured

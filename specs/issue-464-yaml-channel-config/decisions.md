# Decisions — #464 YAML Channel Configuration

## D1: Config representation

**Choice:** SmallRye `@ConfigMapping` with `Map<String, ChannelConfig>` under `casehub.qhorus.channels`
**Alternatives:**
- Raw YAML parsing via separate config file — full format control but no startup validation, no IDE support
- Separate `@ConfigMapping` root (`casehub.channels`) — cleaner separation but creates a new config root outside qhorus
**Rationale:** Channels are a qhorus concept — the config belongs under `casehub.qhorus`. SmallRye validation catches typos and invalid enum values at startup, which is critical for YAML-only apps with no compile-time checks. The `Map<String, T>` pattern is well-established in Quarkus (e.g. `quarkus.datasource.<name>.*`).
**Trade-offs:** All fields need `Optional<>` wrapping. SmallRye enum handling and `Set<T>` support may need `List<T>` with conversion at the bootstrap boundary.
**Sources:** QhorusConfig.java, ChannelCreateRequest.java, SmallRye Config Map support
**Exploration:** deep-analysis
**Status:** captured

## D2: Re-deploy behavior

**Choice:** Detect drift — create if missing, compare existing vs declared, log warnings, don't auto-correct
**Alternatives:**
- Create-only (`findOrCreate` semantics) — simplest but invisible when YAML and reality diverge
- Apply changes (auto-correct drift) — powerful but overwrites runtime API changes, dangerous
**Rationale:** Drift detection is the right level for a pre-release platform. It surfaces divergence without overwriting changes made via the REST/GraphQL API at runtime. Non-destructive: removing a channel from YAML does NOT delete it.
**Trade-offs:** Drift is reported but not fixed — operators must reconcile manually via the API if they want the declared state applied.
**Sources:** Kubernetes reconciliation model, parent issue casehubio/engine#1017 (desired-state graph)
**Exploration:** deep-analysis
**Status:** captured

## D3: Field surface in YAML

**Choice:** Three tiers — core, access control, governance. Infrastructure-only fields excluded.
**Alternatives:**
- Expose all ChannelCreateRequest fields — maximum flexibility but clutters YAML with infrastructure concerns (preAssignedId, connector binding)
- Core fields only (name, semantic, description) — minimal but insufficient for governance-aware apps
**Rationale:** The three tiers cover what a YAML app author thinks about (what the channel is, who can write, what rules apply). Infrastructure fields (connector binding, preAssignedId, spaceId, metadata) are wiring concerns handled programmatically.
**Trade-offs:** Adding a new tier-4 field to YAML requires a config interface change + Spring counterpart update.
**Sources:** ChannelCreateRequest.java (24 fields analysed)
**Exploration:** deep-analysis
**Status:** captured

## D4: Bootstrap lifecycle

**Choice:** `ChannelConfigBootstrap` — `@ApplicationScoped` bean with `@Observes StartupEvent`, per-channel try-catch, `findOrCreate()` + drift detection
**Alternatives:**
- `@Startup` CDI bean constructor — runs earlier but before full CDI context is available
- Quarkus `@BuildStep` — build-time only, can't interact with runtime services
**Rationale:** `@Observes StartupEvent` is the standard Quarkus pattern for runtime bootstrap. Per-channel try-catch follows the startup exception isolation protocol (PP-20260522-c3a8c1). `findOrCreate()` is idempotent per the channel-initialised-event protocol (PP-20260522-e5e527).
**Trade-offs:** Channels are created after app start, not at build time — a channel reference in early startup code could fail if the channel hasn't been created yet.
**Sources:** PP-20260522-c3a8c1 (startup exception isolation), PP-20260522-e5e527 (ChannelInitialisedEvent idempotency), ChannelService.findOrCreate()
**Exploration:** quick
**Status:** captured

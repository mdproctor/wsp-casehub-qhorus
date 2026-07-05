# API Interface Taxonomy Protocol — Design Spec

**Issue:** casehubio/qhorus#316
**Date:** 2026-07-05
**Status:** Draft

---

## Problem

PLATFORM.md references `casehub/garden: docs/protocols/casehub/consumer-spi-placement.md` (line 54) but the file does not exist. Similarly, `docs/protocols/universal/module-tier-structure.md` is referenced (lines 59–60) but neither the file nor the `universal/` directory exists (parent#348). The inline guidance covers SPI placement only. The service facade category — introduced by qhorus#315 (MessageDispatcher, ChannelManager) — is undocumented. The full four-category taxonomy of `api/` interfaces has no protocol.

## Deliverable

A single protocol document: `api-interface-taxonomy.md` in `~/.hortora/garden/docs/protocols/casehub/`.

Replaces the dangling `consumer-spi-placement.md` reference with a broader document covering all four categories.

### Scope: `casehub/` (not `universal/`)

The four-package taxonomy with service facades is a CaseHub convention, specifically realised in qhorus. Other foundation repos (casehub-work, casehub-engine) have different `api/` structures appropriate to their interface mix — casehub-work-api uses a flat layout with `spi/` as the only sub-package; casehub-engine-common uses `spi/`, `qualifier/`, and `internal/`. The universal principle that SPIs go in `api/spi/` is already shared across repos; the full four-category structure is qhorus-specific and aspirational for repos that develop all four kinds of interfaces.

### Tier model integration

The `api/` module IS Tier 1 of the three-tier module structure (pure-Java SPI / core library / full extension — see `module-tier-structure.md`, currently a dangling PLATFORM.md reference, tracked as parent#348). All four interface categories are Tier 1 contracts:

- Stores, SPIs, gateways → Tier 1 contracts that consumers **implement** (pure-Java signatures, no JPA or Quarkus runtime types)
- Service facades → Tier 1 contracts that consumers **call** (never implement)

Service facades are a new kind of Tier 1 interface — consumed rather than implemented — but they share the same Tier 1 constraint: no JPA, no Quarkus runtime dependencies in the interface signature.

## The Four Categories

| Category | Package | Consumer relationship | Examples |
|----------|---------|----------------------|----------|
| **Store** | `api/store/` | Consumer **provides** JPA implementation | `ChannelStore`, `CommitmentStore`, `MessageStore` |
| **SPI** | `api/spi/` | Consumer **provides** policy/extension implementation | `CommitmentAttestationPolicy`, `ObligorTrustPolicy` |
| **Gateway** | `api/gateway/` | Consumer **provides** integration backend | `AgentChannelBackend`, `MessageObserver`, `InboundNormaliser` |
| **Service facade** | `api/<domain>/` | Consumer **calls** (never implements) | `ChannelManager`, `MessageDispatcher`, `ReactiveChannelManager` |

### Key distinction — placement rule

Stores, SPIs, and gateways are **provided by** consumers (the consumer supplies the implementation). Service facades are **consumed by** consumers (the consumer calls the interface; the runtime provides the implementation). This inverted relationship drives the placement decision.

Stores have a dual relationship: the consumer provides the JPA store implementation, but the consumer also calls store methods for reads and queries (e.g. `ChannelStore.findByIds()`). For placement purposes, the **primary** relationship — consumer provides the implementation — is what determines placement in `api/store/`.

### Placement rule

Service facades colocate with domain types in `api/<domain>/` (e.g. `api/channel/ChannelManager.java` alongside `api/channel/Channel.java`). They do not go in `api/spi/` — that package signals "implement me", which is the wrong contract for a facade.

## Per-Category Rules

### Stores (`api/store/`)

- Data access contracts — CRUD operations over domain entities.
- Blocking and reactive pairs: `ChannelStore` + `ReactiveChannelStore`.
- JPA implementations live in `runtime/` (they depend on Panache/Hibernate) — always present since qhorus is a Quarkus extension.
- Working `@Alternative @Priority(1)` in-memory implementations in `persistence-memory/` — activated by classpath presence for consumer test isolation without a datasource.
- Note: the `@DefaultBean` no-op store pattern from PLATFORM.md (`persistence-backend-cdi-priority.md`) applies to platform-level store SPIs where no implementation may be present. Qhorus domain stores use `@Alternative` because the JPA implementation is always available.

### SPIs (`api/spi/`)

- Extension points where consumers replace default behaviour with custom logic.
- Three default patterns: no-op (operational SPIs), populated (vocabulary/registry SPIs), no-op `@DefaultBean` in mock module (store SPIs).
- Must be implementable without depending on `runtime/` — pure-Java signatures only.
- `@DefaultBean` implementations go in `runtime/` when they have JPA or config deps; in `api/spi/` itself when trivially pure-Java.

### Gateways (`api/gateway/`)

- Integration contracts for external systems or cross-cutting observers.
- Consumer provides the implementation to bridge their infrastructure into the runtime.
- Sub-interfaces specialize the contract: `AgentChannelBackend`, `HumanParticipatingChannelBackend`, `HumanObserverChannelBackend` all extend `ChannelBackend`.
- Event types (`MessageReceivedEvent`, `ChannelInitialisedEvent`) colocate here — they define the integration surface.

### Service facades (`api/<domain>/`)

- Consumer-facing interfaces that consumers call, never implement.
- The runtime module provides the `@ApplicationScoped` implementation.
- Colocated with domain types, records, and enums in the same package.
- Blocking and reactive pairs: `ChannelManager` + `ReactiveChannelManager`.
- Exist to give consumers a stable API contract independent of runtime internals.

## Decision Flowchart

When adding a new interface to `api/`:

1. **Does the consumer call it or implement it?**
   - Calls it → **service facade** → `api/<domain>/`
   - Implements it → continue to 2

2. **What does the implementation do?**
   - Persists/retrieves data → **store** → `api/store/`
   - Bridges an external system or observes events → **gateway** → `api/gateway/`
   - Replaces a policy or behavioural extension point → **SPI** → `api/spi/`

3. **Ambiguity: store vs gateway**
   - Needs a datasource → store
   - Connects to an external service or reacts to runtime events → gateway

4. **Ambiguity: SPI vs gateway**
   - Behavioural extension (policy, computation, fold logic) → **SPI** → `api/spi/`
   - Infrastructure integration (external system bridge, event observation) → **gateway** → `api/gateway/`

   Multiplicity is a secondary consideration — both SPIs and gateways can support single or multiple instances. `RenderableProjection` has multiple coexisting implementations selected by name (`projectionName()`), yet it is an SPI because it provides consumer-owned fold logic. `ChannelBackend` has multiple coexisting implementations selected by name (`backendId()`), yet it is a gateway because it bridges external messaging infrastructure. `MessageObserver` uses fan-out (all observers receive every event) — also a gateway.

### Domain types and non-interface packages

Records, enums, and value objects (e.g. `Channel`, `MessageDispatch`, `ChannelSemantic`) are not a category — they are the vocabulary the four categories operate on. Three placement patterns exist in qhorus's `api/`:

1. **Colocated with a service facade** — domain types live alongside the facade in `api/<domain>/`. Examples: `Channel` in `api/channel/` with `ChannelManager`; `Message` in `api/message/` with `MessageDispatcher`.
2. **Standalone domain package** — domain types whose sub-domain is significant but which have no facade or SPI of their own. Examples: `api/data/` (`ArtefactClaim`, `SharedData`), `api/instance/` (`Instance`, `InstanceInfo`). Their associated stores live in `api/store/`.
3. **Domain-scoped sub-package with its own SPI** — a sub-domain package that combines domain types AND an SPI interface tightly coupled to those types. Example: `api/watchdog/` contains the `Watchdog` record, alert context types, `WatchdogConditionType` enum, AND `WatchdogAlertRouter` (an SPI with a `@DefaultBean` implementation in runtime — see ADR-0011). Domain-scoped SPIs colocate with their domain package when the SPI's method signatures are tightly coupled to the domain types; cross-cutting SPIs go in `api/spi/`.

**CDI qualifiers and framework annotations** (e.g. `@CrossTenant` in `api/qualifier/`) are infrastructure, not domain types or interface categories. They may exist in `api/` as needed.

## Changes Required

### 1. Create protocol

`~/.hortora/garden/docs/protocols/casehub/api-interface-taxonomy.md`

Format follows `routing-strategy-convention.md`: YAML frontmatter (id, scope, status, created, refs), then rule, then elaboration.

### 2. Update FOUNDATION-INDEX.md

Add entry for the new protocol in the garden index.

### 3. Update PLATFORM.md

In `casehub-parent/docs/PLATFORM.md` line 54, change the dangling `consumer-spi-placement.md` reference to `api-interface-taxonomy.md` and expand the bullet to cover all four categories.

### 4. Update ARC42STORIES.MD §5 (deferred)

The `api module` table in ARC42STORIES.MD §5 is stale — lists five packages but nine exist. Tracked as qhorus#320.

### 5. Create `module-tier-structure.md` protocol (deferred)

PLATFORM.md references `docs/protocols/universal/module-tier-structure.md` but neither the file nor the `universal/` directory exists. Tracked as parent#348.

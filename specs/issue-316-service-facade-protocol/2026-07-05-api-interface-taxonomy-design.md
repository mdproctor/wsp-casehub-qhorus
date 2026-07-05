# API Interface Taxonomy Protocol — Design Spec

**Issue:** casehubio/qhorus#316
**Date:** 2026-07-05
**Status:** Draft

---

## Problem

PLATFORM.md references `casehub/garden: docs/protocols/casehub/consumer-spi-placement.md` (line 54) but the file does not exist. The inline guidance covers SPI placement only. The service facade category — introduced by qhorus#315 (MessageDispatcher, ChannelManager) — is undocumented. The full four-category taxonomy of `api/` interfaces has no protocol.

## Deliverable

A single protocol document: `api-interface-taxonomy.md` in `~/.hortora/garden/docs/protocols/casehub/`.

Replaces the dangling `consumer-spi-placement.md` reference with a broader document covering all four categories.

## The Four Categories

| Category | Package | Consumer relationship | Examples |
|----------|---------|----------------------|----------|
| **Store** | `api/store/` | Consumer **provides** JPA implementation | `ChannelStore`, `CommitmentStore`, `MessageStore` |
| **SPI** | `api/spi/` | Consumer **provides** policy/extension implementation | `CommitmentAttestationPolicy`, `ObligorTrustPolicy` |
| **Gateway** | `api/gateway/` | Consumer **provides** integration backend | `AgentChannelBackend`, `MessageObserver`, `InboundNormaliser` |
| **Service facade** | `api/<domain>/` | Consumer **calls** (never implements) | `ChannelManager`, `MessageDispatcher`, `ReactiveChannelManager` |

### Key distinction

Stores, SPIs, and gateways are **provided by** consumers. Service facades are **consumed by** consumers. The consumer relationship is inverted.

### Placement rule

Service facades colocate with domain types in `api/<domain>/` (e.g. `api/channel/ChannelManager.java` alongside `api/channel/Channel.java`). They do not go in `api/spi/` — that package signals "implement me", which is the wrong contract for a facade.

## Per-Category Rules

### Stores (`api/store/`)

- Data access contracts — CRUD operations over domain entities.
- Blocking and reactive pairs: `ChannelStore` + `ReactiveChannelStore`.
- `@DefaultBean` no-op in a `persistence-memory/` module (not a working in-memory implementation — that is `@Alternative @Priority(1)`).
- JPA implementations live in `runtime/` (they depend on Panache/Hibernate).
- Consumer test isolation via `InMemory*Store` from `persistence-memory/`.

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
   - Single-bean replacement (one active impl at a time) → SPI
   - Multiple implementations coexist, runtime iterates them → gateway

### Domain types

Records, enums, and value objects (e.g. `Channel`, `MessageDispatch`, `ChannelSemantic`) are not a category — they are the vocabulary the four categories operate on. They colocate with the category that owns them.

## Changes Required

### 1. Create protocol

`~/.hortora/garden/docs/protocols/casehub/api-interface-taxonomy.md`

Format follows `routing-strategy-convention.md`: YAML frontmatter (id, scope, status, created, refs), then rule, then elaboration.

### 2. Update FOUNDATION-INDEX.md

Add entry for the new protocol in the garden index.

### 3. Update PLATFORM.md

In `casehub-parent/docs/PLATFORM.md` line 54, change the dangling `consumer-spi-placement.md` reference to `api-interface-taxonomy.md` and expand the bullet to cover all four categories.

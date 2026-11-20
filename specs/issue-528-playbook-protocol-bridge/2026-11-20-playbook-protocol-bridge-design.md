# Playbook-Executor-Backed ChannelProtocol Bridge

**Issues:** casehubio/qhorus#528, casehubio/qhorus#529
**Branch:** issue-528-playbook-protocol-bridge
**Date:** 2026-11-20

## Problem

Channel protocol evaluation (`ChannelProtocol.evaluate()`) uses a static
`ProtocolRegistry` built once at startup from CDI beans. Protocol names
not in the registry are WARN-logged and skipped. This means
playbook-backed protocols — where the definition comes from YAML at
channel creation time — can never evaluate.

The `PlaybookProtocolEvaluator` was retired (platform#669) because
protocol playbooks should use the standard playbook executor. But no
bridge connects the executor to channel dispatch. Additionally, protocol
playbooks that need temporal coordination (deadlines, timeouts) have no
event source for elapsed time — evaluation only runs when a message is
dispatched.

## Architecture

### New module: `casehub-qhorus-playbook-bridge`

An optional Maven module that activates by classpath presence (CDI
discovery). Contains both the dispatch bridge (#528) and the deadline
timer (#529). Depends on `casehub-qhorus-api` and
`casehub-platform-yaml-step-runtime` (playbook executor).

Follows the pattern of `connector-backend`, `slack-channel`,
`a2a-outbound` — optional bridge modules with no `@IfBuildProperty`
gate. Cross-repo optional dependency on `casehub-platform` must be
registered in both build order and the dependency table per the
`cross-repo-optional-dep-table-registration` protocol.

### Component overview

```
┌─────────────────────────────────────────────────────────────┐
│ MessageService.dispatch()                                   │
│   ├── ProtocolRegistry.forProtocols(ch.protocols(), ch.id())│
│   │     ├── static map (CDI beans) ← existing path          │
│   │     └── ProtocolResolver fallback ← NEW (api/spi/)      │
│   │           └── PlaybookProtocolResolver (bridge module)   │
│   │                 ├── queries ProtocolDefinitionStore      │
│   │                 ├── compiles YAML → PlaybookCompiler     │
│   │                 ├── runs → PlaybookRunnerCore            │
│   │                 ├── validates → PlaybookProtocolOutput   │
│   │                 └── maps output → List<DispatchAdvisory> │
│   └── enforcement gate (unchanged)                          │
│                                                             │
│ ProtocolLifecycleManager                                    │
│   └── PlaybookProtocolLifecycleHandler (bridge module)      │
│         ├── onInit(): parse deadline metadata, register     │
│         │   (queries ProtocolDefinitionStore for YAML)       │
│         └── onDestroy(): clear deadline registrations        │
│                                                             │
│ ProtocolDeadlineScheduler (bridge module, @Scheduled)       │
│   ├── in-memory ConcurrentHashMap<DeadlineKey, Instant>     │
│   ├── sweep: check for expired deadlines                    │
│   └── on expiry: dispatch EVENT via MessageDispatcher       │
│         sender=system:protocol-timer                        │
│         → triggers dispatch → protocol evaluation           │
│         → playbook sees DEADLINE_EXPIRED in context          │
└─────────────────────────────────────────────────────────────┘
```

## Detailed Design

### 1. ProtocolResolver SPI (api/spi/)

```java
package io.casehub.qhorus.api.spi;

import java.util.List;
import java.util.UUID;

public interface ProtocolResolver {
    List<ChannelProtocol> resolve(List<String> unresolvedNames, UUID channelId);
}
```

A single-method SPI in `api/spi/`. `ProtocolRegistry` accepts an
optional `ProtocolResolver` and calls it as a fallback when
`forProtocols()` encounters names not in the static CDI map. The resolver
receives only the unmatched names and the channelId.

The SPI returns `List<ChannelProtocol>` — the resolver constructs
short-lived protocol instances per evaluation. These are not CDI beans;
they're plain objects created for each dispatch cycle.

**Why a fallback SPI rather than a sentinel CDI bean:** Playbook
protocol names are user-defined per channel (e.g., "approval-gate",
"quorum-vote"). Channels can have multiple independently-named playbook
protocols. A sentinel approach (single bean with name "playbook")
would force all playbook protocols to share one name, losing semantic
naming and preventing multiple playbook protocols per channel.

### 2. ProtocolRegistry changes (runtime-core)

`ProtocolRegistry` gains:
- A constructor overload accepting an optional `ProtocolResolver`
- A new `forProtocols(List<String>, UUID)` overload that tries the
  static map first, collects unresolved names, then delegates to the
  resolver if present

```java
public List<ChannelProtocol> forProtocols(List<String> names, UUID channelId) {
    List<ChannelProtocol> result = new ArrayList<>(names.size());
    List<String> unresolved = new ArrayList<>();
    for (String name : names) {
        ChannelProtocol p = registry.get(name);
        if (p != null) {
            result.add(p);
        } else {
            unresolved.add(name);
        }
    }
    if (!unresolved.isEmpty() && resolver != null) {
        result.addAll(resolver.resolve(unresolved, channelId));
    } else {
        for (String name : unresolved) {
            LOG.warnf("Unknown protocol '%s' — skipped. Registered: %s",
                       name, registry.keySet());
        }
    }
    return result;
}
```

The existing `forProtocols(List<String>)` overload is kept for backward
compatibility and delegates to the new overload with a null channelId.
`MessageService.dispatch()` calls the new overload with `ch.id()`.

The `RuntimeBeans` producer is updated to inject
`Instance<ProtocolResolver>` and pass it (or null if unsatisfied) to the
constructor.

### 3. ProtocolDefinitionStore (bridge module)

The bridge module owns the definition storage. No changes to Channel.

```java
package io.casehub.qhorus.playbook.bridge;

public record ProtocolDefinition(
    UUID id,
    UUID channelId,
    String protocolName,
    String yamlContent,
    String tenancyId,
    Instant createdAt) {}
```

**Store interface** in the bridge module (not api/store/ — this is
bridge-internal, not a consumer-facing SPI):

```java
public interface ProtocolDefinitionStore {
    Optional<ProtocolDefinition> findByChannelAndProtocol(
        UUID channelId, String protocolName);
    List<ProtocolDefinition> findByChannel(UUID channelId);
    void put(ProtocolDefinition definition);
    void delete(UUID channelId, String protocolName);
    void deleteByChannel(UUID channelId);
}
```

**JPA entity** `ProtocolDefinitionEntity` in the bridge module.
`InMemoryProtocolDefinitionStore` for CDI-free tests.

**Migration** in `db/qhorus/migration/V59__protocol_definition.sql`:
```sql
CREATE TABLE protocol_definition (
    id UUID PRIMARY KEY,
    channel_id UUID NOT NULL REFERENCES channel(id) ON DELETE CASCADE,
    protocol_name VARCHAR(255) NOT NULL,
    yaml_content TEXT NOT NULL,
    tenancy_id VARCHAR(255) NOT NULL DEFAULT 'DEFAULT',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (channel_id, protocol_name, tenancy_id)
);
```

**Channel stays unchanged.** Channels reference protocols by name via
the existing `List<String> protocols` field. The bridge resolves names
to definitions via the store. Definitions load lazily — only when the
bridge's `evaluate()` fires, not on every dispatch.

### 4. PlaybookProtocolResolver (bridge module)

The bridge implementation of `ProtocolResolver`. For each unresolved
protocol name:

1. Queries `ProtocolDefinitionStore.findByChannelAndProtocol(channelId,
   protocolName)`
2. If absent, skips (WARN log — name may be a typo or the definition
   was deleted)
3. Compiles the YAML via `PlaybookCompiler`
4. Constructs a `PlaybookChannelProtocol` — a plain `ChannelProtocol`
   impl wrapping the compiled playbook

### 5. PlaybookChannelProtocol (bridge module)

`ChannelProtocol` implementation that wraps a compiled playbook.

`evaluate(ProtocolContext)`:

1. Maps `ProtocolContext` to `PlaybookRunConfig.vars`:
   - `channelId`, `channelName`, `incomingType`, `sender`,
     `correlationId`, `participantCount`, `recentMessageCount`,
     `activeCommitmentCount`, `policyOverrides`
   - `recentMessages` as a list of maps (id, sender, type, content,
     correlationId, createdAt)
   - `activeCommitments` as a list of maps (id, messageType, obligor,
     correlationId, state)
2. Runs via `PlaybookRunnerCore.run()`
3. Validates output via `PlaybookProtocolOutput` — a bridge-owned record
   that defines the expected structure:
   - `advisories`: list of maps with `severity` (must be valid
     `Severity` enum), `message` (required), `action` (must be valid
     `SuggestedAction` enum), `evidence` (optional map)
   - Invalid entries produce a validation advisory
     (source="playbook-bridge:validation", severity=WARNING) instead of
     silent failure
4. Maps valid entries to `DispatchAdvisory` with `source` = protocol name

### 6. PlaybookProtocolLifecycleHandler (bridge module)

Implements `ProtocolLifecycleHandler`:

- `supports(String name)`: queries `ProtocolDefinitionStore` to check if
  a definition exists for the given name in any channel. This is a
  predicate check, not channel-scoped — the lifecycle manager calls
  `supports()` with just the protocol name.
- `onInit(ProtocolLifecycleContext)`: queries `ProtocolDefinitionStore`
  for the channel+protocol's YAML, parses
  `PlaybookFrontMatter.metadata().deadlines` (if present), validates
  duration format, registers each deadline with
  `ProtocolDeadlineScheduler`
- `onDestroy(UUID channelId, String protocolName)`: clears all deadline
  registrations for that channel+protocol

On `recovered=true` (startup recovery): the handler queries the
definition store (not `ProtocolLifecycleContext` — the context doesn't
carry YAML), recomputes each deadline's `firesAt` from the YAML duration
+ the timestamp of the last relevant message in
`context.messageHistory()`. If `firesAt` is in the past, the deadline
is registered with `firesAt = Instant.now()` — the scheduler fires it
immediately on the next tick.

**Deadline validation at definition storage time:** When a definition is
stored via `ProtocolDefinitionStore.put()`, the bridge validates the
YAML structure including deadline declarations. Invalid ISO-8601
durations are rejected before the definition is persisted, catching
errors at configuration time rather than first evaluation.

### 7. ProtocolDeadlineScheduler (bridge module)

`@ApplicationScoped` bean. Core data structure:

```java
record DeadlineKey(UUID channelId, String protocolName,
                   String deadlineName) {}

ConcurrentHashMap<DeadlineKey, Instant> pendingDeadlines;
```

Methods:
- `register(UUID channelId, String protocolName, String deadlineName,
  Instant firesAt)` — adds to the map
- `clear(UUID channelId, String protocolName)` — removes all entries
  matching channelId + protocolName
- `clearAll(UUID channelId)` — removes all entries for a channel

`@Scheduled(every = "60s")` sweep:
1. Iterates `pendingDeadlines`
2. For each entry where `firesAt <= Instant.now()`:
   - Removes the entry (CAS via `computeIfPresent`)
   - Dispatches an EVENT message via `MessageDispatcher`:
     - sender: `system:protocol-timer`
     - channelId: from the key
     - telemetry: `{"deadline": deadlineName, "protocol": protocolName,
       "event_type": "DEADLINE_EXPIRED"}`
     - actorType: SYSTEM

The EVENT flows through normal `MessageService.dispatch()` → protocol
evaluation. The playbook receives the ProtocolContext with the deadline
EVENT in `recentMessages`, and can inspect its type and telemetry to
determine it's a deadline expiry.

**In-memory is sufficient** because protocol deadlines are derived
state. The YAML definition (in `ProtocolDefinitionStore`) is the source
of truth for WHAT, channel message history is the source of truth for
WHEN. The deadline registration is a cache. The existing
`ProtocolLifecycleManager.onChannelInitialised(recovered=true)` path
handles restart recovery — the handler recomputes `firesAt` from YAML
duration + message timestamps. Past-due deadlines fire immediately.

### 8. Playbook YAML deadline format

Deadlines are declared in the playbook's frontmatter metadata:

```yaml
version: "1.0"
name: approval-gate
metadata:
  deadlines:
    approval: PT4H
    reminder: PT2H
```

Each key under `deadlines` is the deadline name, the value is an
ISO 8601 duration. The bridge parses these at lifecycle init and
registers them with the scheduler.

A deadline's `firesAt` is computed relative to the last relevant event.
For the initial registration, this is the channel's last message
timestamp (or the protocol assignment time if no messages exist). The
playbook itself can reset deadlines by producing a `reset_deadline`
key in its output — the bridge reads this and re-registers with a new
`firesAt`.

## Testing

- **CDI-free unit tests** for `PlaybookProtocolResolver`,
  `PlaybookChannelProtocol`, `ProtocolDeadlineScheduler` — Mockito for
  stores and playbook executor
- **CDI-free unit tests** for `ProtocolRegistry` with resolver fallback
- **CDI-free unit tests** for `PlaybookProtocolOutput` validation —
  typos, invalid enums, missing required fields
- **`@QuarkusTest` integration tests** for end-to-end dispatch with
  playbook-backed protocols — requires test playbook YAML, asserts
  advisories flow through enforcement
- **Deadline tests** use `Clock` injection (same pattern as
  `PresenceService`) for deterministic time control
- **ProtocolRegistry backward compatibility** — existing tests continue
  to pass with no resolver injected
- **Definition validation tests** — invalid YAML, invalid deadline
  durations rejected at storage time

## Migration

**V59:** `protocol_definition` table (see section 3)

## Module structure

```
playbook-bridge/
├── pom.xml
└── src/
    ├── main/java/io/casehub/qhorus/playbook/bridge/
    │   ├── PlaybookProtocolResolver.java
    │   ├── PlaybookChannelProtocol.java
    │   ├── PlaybookProtocolOutput.java
    │   ├── PlaybookProtocolLifecycleHandler.java
    │   ├── ProtocolDeadlineScheduler.java
    │   ├── DeadlineKey.java
    │   ├── ProtocolDefinition.java
    │   ├── ProtocolDefinitionStore.java
    │   ├── ProtocolDefinitionEntity.java
    │   └── JpaProtocolDefinitionStore.java
    └── test/java/io/casehub/qhorus/playbook/bridge/
        └── InMemoryProtocolDefinitionStore.java
```

## References

- ProtocolRegistry.java:47-58 — existing forProtocols() with WARN-and-skip
- ProtocolLifecycleManager.java — ConcurrentHashMap + lifecycle recovery
- ProtocolLifecycleContext.java — recovered flag for startup replay
- RuntimeBeans.java:306-310 — ProtocolRegistry producer
- MessageService.java:364-382 — protocol evaluation in dispatch pipeline
- PushNotificationConfigEntity — bridge-owned JPA entity precedent
- PlaybookRunnerCore, PlaybookCompiler — platform playbook executor
- PlaybookFrontMatter.metadata — extension point for deadline declarations
- CommitmentService.expireOverdue() — @Scheduled sweep pattern
- PresenceService — Clock injection pattern for deadline tests
- consumer-spi-placement protocol — SPI in api/spi/
- message-service-dispatch-enforcement-gate protocol — all writes through dispatch()
- bridge-module-spi-placement protocol
- optional-platform-spi-instance-injection protocol
- cross-repo-optional-dep-table-registration protocol
- Decision review R1-01 through R1-12

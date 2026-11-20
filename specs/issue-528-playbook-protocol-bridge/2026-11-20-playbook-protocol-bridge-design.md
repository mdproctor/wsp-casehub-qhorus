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

## Prerequisites

**Platform change required:** `PlaybookRunnerCore.run()` currently
accepts only `Path playbookPath` (reads YAML from filesystem). The
bridge stores YAML in a database. A new overload is needed:

```java
PlaybookRunResult run(String yamlContent, PlaybookRunConfig config);
```

This is a one-method addition to `casehub-platform-yaml-step-runtime`.
File a prerequisite issue in casehubio/platform before implementation
begins.

## Architecture

### New module: `casehub-qhorus-playbook-bridge`

An optional Maven module that activates by classpath presence (CDI
discovery). Contains both the dispatch bridge (#528) and the deadline
timer (#529). Depends on `casehub-qhorus-api` and
`casehub-platform-yaml-step-runtime` (playbook executor).

Follows the pattern of `connector-backend`, `slack-channel`,
`a2a-outbound` — optional bridge modules with no `@IfBuildProperty`
gate. Cross-repo optional dependency on `casehub-platform` must be
registered per `cross-repo-optional-dep-table-registration` protocol.

### Component overview

```
┌─────────────────────────────────────────────────────────────┐
│ MessageService.dispatch()                                   │
│   ├── ProtocolRegistry.forProtocols(ch.protocols(), ch.id())│
│   │     ├── static map (CDI beans) ← existing path          │
│   │     └── ProtocolResolver fallback ← NEW (api/spi/)      │
│   │           └── PlaybookProtocolResolver (bridge module)   │
│   │                 ├── queries ProtocolDefinitionStore      │
│   │                 ├── checks compilation cache             │
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

**Why a fallback SPI rather than a sentinel CDI bean:** Playbook
protocol names are user-defined per channel (e.g., "approval-gate",
"quorum-vote"). Channels can have multiple independently-named playbook
protocols. A sentinel approach (single bean with name "playbook")
would force all playbook protocols to share one name.

### 2. ProtocolRegistry changes (runtime-core)

`ProtocolRegistry` gains:
- A constructor overload accepting an optional `ProtocolResolver`
- A new `forProtocols(List<String>, UUID)` overload

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

The existing `forProtocols(List<String>)` overload delegates to the new
overload with null channelId. `MessageService.dispatch()` calls the new
overload with `ch.id()`.

**ProtocolReader enrichment:** `ProtocolReader.registeredProtocols()`
remains global (static CDI names only). Playbook protocol names are
per-channel and intentionally don't appear in the global list. Per-channel
protocol discovery uses the existing `Channel.protocols()` field.

The `RuntimeBeans` producer is updated to inject
`Instance<ProtocolResolver>` and pass it (or null if unsatisfied).

### 3. ProtocolDefinitionStore (bridge module)

The bridge module owns definition storage. No changes to Channel.

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

**Store interface** (bridge-internal, not in api/store/):

```java
public interface ProtocolDefinitionStore {
    Optional<ProtocolDefinition> findByChannelAndProtocol(
        UUID channelId, String protocolName);
    List<ProtocolDefinition> findByChannel(UUID channelId);
    boolean existsByProtocolName(String protocolName);
    void put(ProtocolDefinition definition);
    void delete(UUID channelId, String protocolName);
    void deleteByChannel(UUID channelId);
}
```

**JPA entity** `ProtocolDefinitionEntity` in the bridge module.
`InMemoryProtocolDefinitionStore` for CDI-free tests.

**Migration** `V60__protocol_definition.sql` (V59 is already taken):
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

### 4. Definition creation surface

**REST endpoint** in the bridge module:

```
PUT  /api/channels/{channelId}/protocol-definitions/{protocolName}
     Body: YAML content (text/x-yaml)
     → validates YAML structure, deadline durations
     → stores via ProtocolDefinitionStore
     → adds protocolName to channel.protocols if not present
     → fires ChannelProtocolsChangedEvent (triggers lifecycle init)

GET  /api/channels/{channelId}/protocol-definitions/{protocolName}
     → returns YAML content

DELETE /api/channels/{channelId}/protocol-definitions/{protocolName}
     → deletes definition
     → removes protocolName from channel.protocols
     → fires ChannelProtocolsChangedEvent (triggers lifecycle destroy)

GET  /api/channels/{channelId}/protocol-definitions
     → lists all definitions for the channel
```

**ChannelCreateRequest integration:** `ChannelCreateRequest` gains an
optional `protocolDefinitions` field (`Map<String, String>`, protocol
name → YAML content). `ChannelService.create()` delegates to the bridge
module's store after channel creation. The bridge observes
`ChannelInitialisedEvent` and stores definitions from the request.

**casehub-ops topology YAML:** casehub-ops#202 calls the REST endpoint
to create definitions declared in topology YAML.

**Validation at storage time:** The bridge validates YAML structure,
frontmatter schema, and deadline duration format when a definition is
stored. Invalid YAML or malformed durations are rejected with 400
before persistence.

### 5. Compilation cache

Compiled playbooks are cached to avoid YAML recompilation on every
dispatch.

```java
ConcurrentHashMap<DefinitionCacheKey, CompiledPlaybook> compilationCache;

record DefinitionCacheKey(UUID channelId, String protocolName) {}
record CompiledPlaybook(PlaybookDocument document, Instant compiledAt) {}
```

**Invalidation:** Cache entry evicted when:
- Definition is updated via `ProtocolDefinitionStore.put()`
- Definition is deleted
- Channel is deleted

The bridge's `ProtocolDefinitionStore` wrapper clears the cache entry on
`put()` and `delete()`. First evaluation after invalidation recompiles.

### 6. PlaybookProtocolResolver (bridge module)

Implements `ProtocolResolver`. For each unresolved protocol name:

1. Checks compilation cache for `(channelId, protocolName)`
2. On cache miss: queries `ProtocolDefinitionStore`, compiles YAML,
   populates cache
3. If no definition found, skips (WARN log)
4. Constructs a `PlaybookChannelProtocol` wrapping the cached compiled
   playbook

### 7. PlaybookChannelProtocol (bridge module)

`ChannelProtocol` implementation wrapping a compiled playbook.

`evaluate(ProtocolContext)`:

1. Maps `ProtocolContext` to `PlaybookRunConfig.vars`:
   - `channelId`, `channelName`, `incomingType`, `sender`,
     `correlationId`, `participantCount`, `recentMessageCount`,
     `activeCommitmentCount`, `policyOverrides`
   - `recentMessages` as a list of maps (id, sender, type, content,
     correlationId, createdAt)
   - `activeCommitments` as a list of maps (id, messageType, obligor,
     correlationId, state)
2. Runs via `PlaybookRunnerCore.run(yamlContent, config)`
3. Validates output via `PlaybookProtocolOutput`:
   - Reads `advisories` key from `PlaybookRunResult.output()`
   - Each entry must have: `severity` (valid `Severity` enum value),
     `message` (non-blank string)
   - Optional: `action` (valid `SuggestedAction` enum value, default
     LOG), `evidence` (map)
   - Invalid entries produce a validation advisory
     (source="playbook-bridge:validation", severity=WARNING)
4. Reads `deadline_actions` from output (see section 10)
5. Maps valid advisory entries to `DispatchAdvisory` with
   `source` = protocol name

**Example playbook producing advisories:**

```yaml
version: "1.0"
name: approval-gate
metadata:
  deadlines:
    approval: PT4H

steps:
  - id: check-approvals
    type: block
    steps:
      - id: count
        type: set
        values:
          approval_count: "${recentMessages.?[type == 'RESPONSE'].size()}"
          required: 3
      - id: check
        type: if
        condition: "${approval_count < required}"
        then:
          - id: warn
            type: advisory
            severity: WARNING
            message: "Approval gate: ${approval_count}/${required} approvals"
            action: LOG
```

### 8. PlaybookProtocolLifecycleHandler (bridge module)

Implements `ProtocolLifecycleHandler`:

- `supports(String name)`: checks an in-memory set of known playbook
  protocol names. The set is populated at `@PostConstruct` from
  `ProtocolDefinitionStore.existsByProtocolName()` and refreshed on
  definition `put()`/`delete()`. Does NOT query the DB per call.
  **Precedence guard:** returns `false` if the name is registered in
  `ProtocolRegistry` (injected) — a CDI-registered protocol name always
  takes precedence over a playbook definition with the same name.
- `onInit(ProtocolLifecycleContext)`: queries `ProtocolDefinitionStore`
  for the channel+protocol's YAML, parses
  `PlaybookFrontMatter.metadata().deadlines` (if present), registers
  each deadline with `ProtocolDeadlineScheduler`
- `onDestroy(UUID channelId, String protocolName)`: clears all deadline
  registrations for that channel+protocol

**Recovery (recovered=true):** The handler queries the definition store,
parses deadline metadata, and recomputes `firesAt`. The anchor timestamp
is the **last `system:protocol-timer` EVENT for this deadline name in
the channel** (indicating the deadline was reset or previously fired).
If no such EVENT exists, the anchor is the **channel's earliest message
after the protocol was assigned** (or `Instant.now()` if no messages
exist). If `firesAt` is in the past, registers with
`firesAt = Instant.now()` — the scheduler fires immediately.

### 9. ProtocolDeadlineScheduler (bridge module)

`@ApplicationScoped` bean with `Clock` injection for testability.

```java
record DeadlineKey(UUID channelId, String protocolName,
                   String deadlineName) {}

ConcurrentHashMap<DeadlineKey, Instant> pendingDeadlines;
```

Methods:
- `register(DeadlineKey key, Instant firesAt)` — adds to the map
- `clear(UUID channelId, String protocolName)` — removes all entries
  matching channelId + protocolName
- `clearAll(UUID channelId)` — removes all entries for a channel
- `cancel(DeadlineKey key)` — removes a single entry

`@Scheduled(every = "60s")` sweep:
1. Iterates `pendingDeadlines`
2. For each entry where `firesAt <= clock.instant()`:
   - Removes the entry atomically
   - Dispatches an EVENT message via `MessageDispatcher`:
     - sender: `system:protocol-timer`
     - channelId: from the key
     - telemetry: `{"deadline": deadlineName, "protocol": protocolName,
       "event_type": "DEADLINE_EXPIRED"}`
     - actorType: SYSTEM

### 10. Deadline actions from playbook output

The playbook can control deadlines via its output. After
`PlaybookRunnerCore.run()` returns, the bridge reads the
`deadline_actions` key from the output (processed after advisories,
in the same `evaluate()` call):

```yaml
# Playbook output keys:
deadline_actions:
  - action: reset       # reset | cancel
    deadline: approval  # deadline name from metadata.deadlines
    # 'reset' recomputes firesAt = Instant.now() + original duration
    # 'cancel' removes the deadline registration entirely
```

- **reset:** Re-registers the named deadline with
  `firesAt = clock.instant() + originalDuration`. The original duration
  comes from the playbook's frontmatter metadata.
- **cancel:** Calls `scheduler.cancel(key)`. The deadline does not fire
  unless re-registered by a future reset or lifecycle init.
- Unknown actions or deadline names produce a WARN log and are skipped.

### 11. Playbook YAML deadline format

Deadlines are declared in frontmatter metadata:

```yaml
version: "1.0"
name: approval-gate
metadata:
  deadlines:
    approval: PT4H
    reminder: PT2H
```

Each key under `deadlines` is the deadline name, the value is an
ISO 8601 duration. Validated at definition storage time — invalid
durations rejected with 400 before persistence.

## Testing

- **CDI-free unit tests** for `PlaybookProtocolResolver`,
  `PlaybookChannelProtocol`, `ProtocolDeadlineScheduler`,
  `PlaybookProtocolOutput` validation — Mockito for stores and executor
- **CDI-free unit tests** for `ProtocolRegistry` with resolver fallback
- **`@QuarkusTest` integration tests** for end-to-end dispatch with
  playbook-backed protocols — test playbook YAML, asserts advisories
  flow through enforcement
- **Deadline tests** use `Clock` injection (PresenceService pattern)
- **ProtocolRegistry backward compatibility** — existing tests pass
  with no resolver
- **Definition validation tests** — invalid YAML, invalid durations,
  malformed output
- **Compilation cache tests** — invalidation on update, miss on first
  access
- **Precedence tests** — CDI-registered name takes priority over same
  name in definition store
- **Recovery tests** — recovered=true with past-due deadlines, with
  reset history, with no prior messages

## Migration

**V62:** `protocol_definition` table (see section 3)

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
    │   ├── JpaProtocolDefinitionStore.java
    │   ├── ProtocolDefinitionResource.java
    │   └── CompilationCache.java
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
- PresenceService — Clock injection for deterministic time tests
- consumer-spi-placement protocol — SPI in api/spi/
- message-service-dispatch-enforcement-gate protocol
- bridge-module-spi-placement protocol
- optional-platform-spi-instance-injection protocol
- cross-repo-optional-dep-table-registration protocol
- Decision review R1-01 through R1-12
- Spec review R1-02 through R1-11

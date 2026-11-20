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

### New module: `casehub-qhorus-playbook-protocol`

An optional Maven module that activates by classpath presence (CDI
discovery). Contains both the dispatch bridge (#528) and the deadline
timer (#529). Depends on `casehub-qhorus-api` and
`casehub-platform` (playbook executor).

Follows the pattern of `connector-backend`, `slack-channel`,
`a2a-outbound` — optional bridge modules with no `@IfBuildProperty`
gate.

### Component overview

```
┌─────────────────────────────────────────────────────────────┐
│ MessageService.dispatch()                                   │
│   ├── ProtocolRegistry.forProtocols(ch.protocols())         │
│   │     ├── static map (CDI beans) ← existing path          │
│   │     └── ProtocolResolver fallback ← NEW                 │
│   │           └── PlaybookProtocolResolver                   │
│   │                 ├── reads ch.protocolDefinitions()       │
│   │                 ├── compiles YAML → PlaybookCompiler     │
│   │                 ├── runs → PlaybookRunnerCore            │
│   │                 └── maps output → List<DispatchAdvisory> │
│   └── enforcement gate (unchanged)                          │
│                                                             │
│ ProtocolLifecycleManager                                    │
│   └── PlaybookProtocolLifecycleHandler                      │
│         ├── onInit(): parse deadline metadata, register     │
│         └── onDestroy(): clear deadline registrations        │
│                                                             │
│ ProtocolDeadlineScheduler (@Scheduled)                      │
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
`Optional<ProtocolResolver>` and calls it as a fallback when
`forProtocols()` encounters names not in the static CDI map. The resolver
receives only the unmatched names and the channelId.

The SPI returns `List<ChannelProtocol>` — the resolver constructs
short-lived protocol instances per evaluation. These are not CDI beans;
they're plain objects created for each dispatch cycle.

### 2. ProtocolRegistry changes (runtime-core)

`ProtocolRegistry` gains:
- A constructor overload accepting `Optional<ProtocolResolver>`
- `forProtocols()` tries the static map first, collects unresolved names,
  then delegates to the resolver if present

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
            LOG.warnf("Unknown protocol '%s' — skipped.", name, registry.keySet());
        }
    }
    return result;
}
```

The existing `forProtocols(List<String>)` overload is kept for backward
compatibility and delegates to the new overload with a null channelId.
`MessageService.dispatch()` calls the new overload with `ch.id()`.

### 3. Channel.protocolDefinitions (api/channel/)

New nullable `Map<String, String>` field on the `Channel` record. Keys
are protocol names, values are YAML content. Null means no playbook
definitions (most channels). Stored as TEXT via JPA converter (same
pattern as `policyOverrides`).

`ChannelCreateRequest` gains a matching `protocolDefinitions` field
and builder method. `ChannelService.setProtocolDefinitions(UUID,
Map<String, String>)` provides the mutation API (UUID, same pattern as
`setProtocols()`).

V59 Flyway migration adds `channel.protocol_definitions TEXT`.

### 4. PlaybookProtocolResolver (playbook-protocol module)

The bridge implementation. For each unresolved protocol name:

1. Looks up the channel via the channelId (injected `ChannelStore`)
2. Reads `channel.protocolDefinitions().get(protocolName)`
3. If absent, skips (WARN log)
4. Compiles the YAML via `PlaybookCompiler`
5. Constructs a `PlaybookChannelProtocol` — a plain `ChannelProtocol`
   impl that wraps the compiled playbook

`PlaybookChannelProtocol.evaluate(ProtocolContext)`:

1. Maps `ProtocolContext` to `PlaybookRunConfig.vars`:
   - `channelId`, `channelName`, `incomingType`, `sender`,
     `correlationId`, `participantCount`, `recentMessageCount`,
     `activeCommitmentCount`, `policyOverrides`
   - `recentMessages` as a list of maps (id, sender, type, content,
     correlationId, createdAt)
   - `activeCommitments` as a list of maps (id, messageType, obligor,
     correlationId, state)
2. Runs via `PlaybookRunnerCore.run()`
3. Reads `advisories` from `PlaybookRunResult.output()` — expected to
   be a list of maps with keys: `severity` (ADVISORY/WARNING/CRITICAL),
   `message`, `action` (LOG/ESCALATE/INVESTIGATE/REROUTE), `evidence`
   (optional map)
4. Maps each to `DispatchAdvisory` with `source` = protocol name
5. Invalid entries (missing severity/message) are skipped with WARN log

### 5. PlaybookProtocolLifecycleHandler (playbook-protocol module)

Implements `ProtocolLifecycleHandler`:

- `supports(String name)`: returns true if the channel (looked up via
  channelStore) has a `protocolDefinitions` entry for that name
- `onInit(ProtocolLifecycleContext)`: parses
  `PlaybookFrontMatter.metadata().deadlines` (if present), registers
  each deadline with `ProtocolDeadlineScheduler`
- `onDestroy(UUID channelId, String protocolName)`: clears all deadline
  registrations for that channel+protocol

On `recovered=true` (startup recovery): the handler recomputes each
deadline's `firesAt` from the YAML duration + the timestamp of the last
relevant message in `context.messageHistory()`. If `firesAt` is in the
past, the deadline is registered with `firesAt = Instant.now()` —
the scheduler fires it immediately on the next tick.

### 6. ProtocolDeadlineScheduler (playbook-protocol module)

`@ApplicationScoped` bean. Core data structure:

```java
record DeadlineKey(UUID channelId, String protocolName, String deadlineName) {}

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

### 7. Playbook YAML deadline format

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
- **`@QuarkusTest` integration tests** for end-to-end dispatch with
  playbook-backed protocols — requires test playbook YAML, asserts
  advisories flow through enforcement
- **Deadline tests** use `Clock` injection (same pattern as
  `PresenceService`) for deterministic time control
- **ProtocolRegistry backward compatibility** — existing tests continue
  to pass with no resolver injected

## Migration

**V59:** `ALTER TABLE channel ADD COLUMN protocol_definitions TEXT`

## Module structure

```
playbook-protocol/
├── pom.xml
└── src/main/java/io/casehub/qhorus/playbook/protocol/
    ├── PlaybookProtocolResolver.java
    ├── PlaybookChannelProtocol.java
    ├── PlaybookProtocolLifecycleHandler.java
    ├── ProtocolDeadlineScheduler.java
    └── DeadlineKey.java
```

## References

- ProtocolRegistry.java:47-58 — existing forProtocols() with WARN-and-skip for unknown names
- ProtocolLifecycleManager.java — ConcurrentHashMap pattern for in-memory protocol state
- ProtocolLifecycleContext.java — recovered flag for startup replay
- RuntimeBeans.java:306-310 — ProtocolRegistry producer with CDI Instance<ChannelProtocol>
- MessageService.java:364-382 — protocol evaluation in dispatch pipeline
- Channel.java (policyOverrides, metadata) — Map<String,String> TEXT field pattern
- PlaybookRunnerCore, PlaybookCompiler — platform playbook executor
- PlaybookFrontMatter.metadata — extension point for deadline declarations
- CommitmentService.expireOverdue() — @Scheduled sweep pattern
- PresenceService — Clock injection pattern for deadline tests
- consumer-spi-placement protocol — SPI in api/spi/
- message-service-dispatch-enforcement-gate protocol — all writes through dispatch()
- bridge-module-spi-placement protocol — SPI placement for bridge modules
- optional-platform-spi-instance-injection protocol — Instance<> with isUnsatisfied()

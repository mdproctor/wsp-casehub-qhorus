---
layout: post
title: "The Bridge That Closes the Loop"
date: 2026-11-20
entry_type: article
subtype: diary
projects: [casehubio/qhorus]
tags: [protocol, playbook, bridge, deadline, governance, design]
---

The protocol lifecycle hooks shipped in October — `ProtocolLifecycleHandler` with init, destroy, and recovery. At the time I wrote that it "opens up" temporal coordination, learning protocols, compliance tracking. All true. But the mechanism was half a bridge: the lifecycle side could manage per-channel state, while the evaluation side — `ProtocolRegistry` — was still a static map of CDI beans built once at startup.

Playbook-backed protocols fell through the gap. A channel could declare a protocol name in its `protocols` list, the lifecycle manager would call `onInit`, the platform's playbook engine would spin up a state machine — and then `ProtocolRegistry.forProtocols()` would WARN-log the name as "unknown" and skip it entirely. The state machine had state. It just never got asked.

## The resolution path

The core problem is that `ProtocolRegistry` resolves protocol names against a fixed set of CDI beans discovered at startup. Playbook protocols are defined per channel at runtime — there's no CDI bean to discover. We needed a second resolution path without breaking the first.

A fallback `ProtocolResolver` SPI does this cleanly. `forProtocols()` checks the static map first, collects any names that didn't match, and passes them to the resolver. If no resolver is on the classpath — the common case for most deployments — the existing WARN-and-skip behaviour is unchanged. Zero overhead on the happy path.

```java
public interface ProtocolResolver {
    List<ChannelProtocol> resolve(List<String> unresolvedNames,
                                   UUID channelId);
}
```

The resolver receives the channelId because playbook definitions are per-channel — two channels can use the same protocol name with different YAML definitions and evaluate independently. This is fundamentally different from the built-in protocols, which are singletons.

## Where definitions live

The first instinct was to store playbook YAML inline on the `Channel` record — a `Map<String, String>` TEXT column, same pattern as `policyOverrides`. We went through a decision review that caught the problem: `Channel` is the most-queried entity in the system, loaded on every `MessageService.dispatch()`. Adding multi-kilobyte YAML to every channel load — even channels without playbook protocols — was the wrong trade-off.

The definitions moved to a dedicated `ProtocolDefinitionStore` owned by the bridge module. The channel carries protocol names (which it already does via `List<String> protocols`). The bridge loads definitions lazily — only when `evaluate()` fires, not on every dispatch. A compilation cache avoids re-parsing YAML on every message.

## The deadline problem

Protocols evaluate synchronously during message dispatch. If no message arrives, `evaluate()` never runs. A quorum approval gate with a four-hour deadline can't fire "deadline expired" if nobody sends a message.

The tempting solution was to hook into commitments — they already have `expiresAt`, a scheduler-driven sweep, and CDI event firing. But commitments are message-level obligations (COMMAND → DONE). Protocol deadlines are channel-level governance timeouts. Mixing them would pollute obligation stats, stalled-obligation queries, and the normative audit trail — a semantic error that gets worse as the system scales.

The pattern is right; the entity is wrong. We followed the same `@Scheduled` sweep pattern that `CommitmentService.expireOverdue()` uses, but with an in-memory `ConcurrentHashMap` instead of a JPA entity. Protocol deadlines are derived state — the YAML definition says *what* the deadline is, the message history says *when* it was last relevant. The registration is just a timer cache.

When a deadline fires, the scheduler dispatches a synthetic EVENT message to the channel — `sender=system:protocol-timer`. The EVENT flows through normal `dispatch()`, which triggers protocol evaluation. The playbook sees the timeout in its `ProtocolContext` and can react: escalate, resolve, quarantine. The full enforcement pipeline applies. The ledger records it. It's not a side channel.

On restart, `ProtocolLifecycleHandler.onInit(recovered=true)` recomputes every deadline from the YAML duration and the timestamp of the last relevant message. Past-due deadlines fire immediately. No persistence needed beyond what the definition store and message history already provide.

## The bridge module

Everything lands in one optional module — `casehub-qhorus-playbook-bridge`. Activates by classpath presence, same as `connector-backend` or `slack-channel`. The module depends on `casehub-platform-yaml-step-runtime` for the playbook compiler and executor — a cross-repo optional dependency that follows the established bridge pattern.

The module contains five things: the resolver (looks up definitions, caches compilations, runs playbooks), the lifecycle handler (parses deadline metadata, registers timers), the deadline scheduler (fires EVENTs on expiry), the definition store (JPA entity with REST CRUD), and an output validator (maps playbook results to `DispatchAdvisory` with error diagnostics for malformed output).

## What this means

A channel operator writes a protocol in YAML, stores it via a REST endpoint, and it evaluates on every message — returning the same `DispatchAdvisory` results that built-in protocols return. The enforcement pipeline doesn't know or care whether the advisory came from a Java class or a playbook. Deadlines fire even when nobody is talking. The lifecycle hooks handle init, destroy, recovery, and runtime reconfiguration.

This is the second half of the bridge. The lifecycle hooks gave playbook protocols a place to live. The resolver gives them a voice in the dispatch pipeline. The deadline scheduler gives them a sense of time.

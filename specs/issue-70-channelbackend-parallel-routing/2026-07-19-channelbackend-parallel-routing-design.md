# ChannelBackend Parallel COMMAND Routing

> **Issue:** openclaw#70
> **Repos:** casehub-qhorus (API change), casehub-engine (SPI + dispatch), casehub-openclaw (routing impl)
> **Date:** 2026-07-19

## Problem

When multiple agents are active simultaneously on the same case, COMMANDs
dispatched through the channel backend reach an arbitrary agent instead of the
intended one.

The routing signal exists in the data model — `MessageDispatch` and `Message`
both carry a `target` field — but it is dropped at two boundaries:

1. `CaseChannelProvider.postToChannel()` has no `target` parameter, so the
   engine cannot set it.
2. `OutboundMessage` has no `target` field, so `ChannelBackend.post()` is blind
   to routing intent even if `target` were set on the dispatch.

## Root Cause

A three-layer information gap between the engine (which knows the target) and
the backend (which needs it):

```
Engine (knows worker.name() = agentId)
  → CaseChannelProvider.postToChannel() — no target param
    → MessageDispatch.target — never populated
      → OutboundMessage — field absent
        → ChannelBackend.post() — routes arbitrarily
```

## Design

Fix the propagation gap. No new state, no new mappings, no new conventions.

### Layer 1: OutboundMessage — add `target` field

`OutboundMessage` (casehub-qhorus-api) gains a nullable `String target` as the
last record component:

```java
public record OutboundMessage(
        UUID messageId,
        String sender,
        MessageType type,
        String content,
        String correlationId,
        Long inReplyTo,
        ActorType senderActorType,
        List<ArtefactRef> artefactRefs,
        String target) {}
```

Six production construction sites pass `target` through:

| Site | File | Source |
|------|------|--------|
| Normal dispatch | `MessageService.dispatch()` | `dispatch.target()` |
| LAST_WRITE dispatch | `MessageService.dispatch()` | `dispatch.target()` |
| Reactive normal dispatch | `ReactiveMessageService.doDispatch()` | `dispatch.target()` |
| Reactive LAST_WRITE dispatch | `ReactiveMessageService.doDispatch()` | `dispatch.target()` |
| Remote delivery | `ChannelGateway.deliverRemote()` | `msg.target()` |
| Cursor-based delivery | `DeliveryBatchExecutor.toOutbound()` | `m.target()` |

Test construction sites (~30) pass `null` — target is irrelevant in those tests.

Other `ChannelBackend` implementations (`ConnectorChannelBackend`,
`SlackChannelBackend`, `A2AChannelBackend`, `QhorusChannelBackend`,
`RecordingChannelBackend`) receive the new field but require no changes — they
do not use `target` for routing. Each ignores the field.

### Layer 2: CaseChannelProvider — add `target` parameter

Pre-release platform: breaking change, no backward-compat shims.

The engine-api SPI replaces the 6-param `postToChannel` with a 7-param
version. The 6-param method is removed. All implementations must update:

```java
void postToChannel(
    CaseChannel channel, String from, String content,
    MessageType type, String correlationId, String deadline,
    String target);
```

The 3-param convenience delegates to the 7-param with nulls:

```java
default void postToChannel(CaseChannel channel, String from, String content) {
    postToChannel(channel, from, content, null, null, null, null);
}
```

Implementations that need updating:
- `NoOpCaseChannelProvider` — add `target` param (ignore it)
- `OpenClawCaseChannelProvider` — override 7-param, pass `target` through
- `ReactiveOpenClawCaseChannelProvider` — same treatment
- `NoOpReactiveCaseChannelProvider` — add `target` param (ignore it)
- `SpiWiringIntegrationTest` recording impl — add `target` param
- Any Claudony `CaseChannelProvider` impl

`ReactiveCaseChannelProvider` gets the same breaking treatment.

**Parameter proliferation (deferred):** This is the 7th positional parameter.
A `PostRequest` record would consolidate future growth. Filed as engine#759,
not blocking for this change.

### Layer 3: Engine dispatch — set target

`WorkerScheduleEventHandler.dispatchCommand()` passes
`target = worker.name()` (which equals the agentId by existing convention):

```java
caseChannelProvider.postToChannel(
    channel, "casehub-engine:orchestrator", serialize(command),
    MessageType.COMMAND, String.valueOf(eventLogId), deadline,
    worker.name());
```

`AgentRoutingEscalationHandler.postQuery()` passes `target = null` — QUERYs
to the oversight channel are for human supervisors, not targeted agents.
Mechanical update only (add 7th arg `null`).

**ObligorTrustPolicy interaction.** Setting `target = worker.name()` triggers
the `ObligorTrustPolicy` check in both `MessageService.dispatch()` and
`ReactiveMessageService.doDispatch()`. The check fires when
`type == COMMAND && target != null && !target.contains(":")`. Currently
engine-dispatched COMMANDs bypass this because `target` is null. After this
change, `target = "researcher"` (no `:`) triggers the check — a behavioral
regression for deployments with `min-obligor-trust > 0`.

Fix: add a sender-based exemption to **both** services. System senders
(sender contains `:`, e.g. `"casehub-engine:orchestrator"`) bypass the
obligor trust check — the engine's provisioning decision IS the trust
authority for worker assignments:

```java
if (ch != null && dispatch.type() == MessageType.COMMAND
        && dispatch.target() != null
        && !dispatch.target().contains(":")
        && !dispatch.sender().contains(":")) {
```

This preserves trust-gating for peer agent COMMANDs (where both sender and
target are agent identifiers) while exempting engine-orchestrated dispatch.
Applied identically in `MessageService` (blocking) and
`ReactiveMessageService` (reactive) for consistent trust semantics.

**Commitment obligor side effect.** Setting `target` on the `MessageDispatch`
also populates the commitment obligor in `commitmentService.open()`. Currently
null for engine-dispatched COMMANDs, it becomes `worker.name()`. This enables
agent-specific obligation tracking: the Watchdog can name the delinquent agent
in `OBLIGATION_FAN_OUT` escalations, and trust scoring credits/penalizes the
specific agent rather than the channel. This is a positive change — named
obligations are more precise and useful.

### Layer 4: OpenClaw — route by target

**`OpenClawCaseChannelProvider.postToChannel()`** overrides the 7-param method
and sets `.target(target)` on the `MessageDispatch`.

**`OpenClawChannelBackend.post()`** reads `message.target()` for direct agent
lookup with two distinct failure modes:

- `target` non-blank, agent registered for this caseId: route to agent.
- `target` non-blank, agent not registered at all: WARN "Target agent {id} not
  registered — COMMAND dropped (agent may have crashed)". No fallback.
- `target` non-blank, agent registered for a different caseId: WARN "Target
  agent {id} registered for caseId={other}, not {expected} — routing bug,
  COMMAND dropped". No fallback.
- `target` null/blank: fall back to `registry.findAgentId(caseId)` — pre-#70
  single-agent compat path.

**No fallback when target is set.** If the targeted agent is unavailable,
the COMMAND is dropped rather than delivered to the wrong agent. The Watchdog
OBLIGATION_FAN_OUT condition detects unacknowledged COMMANDs and escalates.

## Convention Dependency

This design relies on `workerName == agentId` — case YAML worker names match
OpenClaw agent config identifiers. This convention is used by
`WorkerProvisioner.terminate(workerId)` which calls
`registry.deregister(workerId)` keyed by agentId, and is now also load-bearing
for routing: `target = worker.name()` must match the `agentId` registered in
`OpenClawAgentRegistry` by the provisioner. A mismatch means the targeted agent
is not found → COMMAND dropped → Watchdog escalation.

The gap: `ProvisionContext` has no `workerName` field, and `ProvisionResult`
does not return the resolved `agentId`. The engine uses `worker.name()` as
`target` while the provisioner independently resolves `agentId` from capability
config via `OpenClawAgentConfigResolver`. There is no runtime verification that
these match.

No effective runtime validation of the `workerName == agentId` convention
exists in this change. Mismatches are detected operationally: the targeted
agent is not found → COMMAND is dropped → Watchdog `OBLIGATION_FAN_OUT`
escalates → operator investigates. This is adequate for a pre-release platform
where case YAML and agent config are authored by the same team.

**Structural fix (deferred):** return `agentId` in `ProvisionResult` so the
engine uses the provisioner's resolved identity as `target` instead of assuming
`worker.name()`. This eliminates the convention entirely. Filed as engine#760.

## Alternatives Considered

### correlationId→agentId mapping

Provisioner or engine stores a per-COMMAND mapping. Rejected: the engine
generates correlationId (eventLogId) at dispatch time, after provisioning. Extra
hot-path state for every COMMAND. What about COMMANDs without correlationId?

### Per-agent channels

Each agent gets its own channel. Rejected: multiplies channels per case,
breaks the shared-channel coordination model where agents see each other's work,
major change to the normative 3-channel layout.

### Channel-name routing (openclaw-only)

Backend extracts worker name from channel name (`case-{uuid}/worker:{name}`).
Rejected: relies on naming convention, routing signal is implicit rather than
explicit, other backends can't use the mechanism, naming changes break routing
silently.

## Scope

| Repo | Changes |
|------|---------|
| casehub-qhorus | `OutboundMessage` + 6 production sites + ~30 test sites + `MessageService` + `ReactiveMessageService` trust check sender exemption |
| casehub-engine-api | `CaseChannelProvider` + `ReactiveCaseChannelProvider` (breaking — 7-param replaces 6-param) |
| casehub-engine | `WorkerScheduleEventHandler.dispatchCommand()` + `AgentRoutingEscalationHandler.postQuery()` (mechanical — `null` target) |
| casehub-openclaw | `OpenClawCaseChannelProvider` + `ReactiveOpenClawCaseChannelProvider` + `OpenClawChannelBackend` + tests |

## Testing

### qhorus
- `OutboundMessage` carries `target` through fanOut (normal + LAST_WRITE paths)
- `ReactiveMessageService` carries `target` through reactive dispatch (normal + LAST_WRITE)
- `DeliveryBatchExecutor.toOutbound()` carries `target` through cursor-based re-delivery
- `ChannelGateway.deliverRemote()` carries `target` through cross-node delivery
- Trust policy sender exemption: system sender (contains `:`) with target set → COMMAND bypasses trust check
- Trust policy sender exemption: agent sender (no `:`) with target set → COMMAND passes through trust check
- Both `MessageService` and `ReactiveMessageService` trust paths verified

### engine
- `WorkerScheduleEventHandler` test verifies `postToChannel` receives `worker.name()` as target
- Contract tests (`CaseChannelProviderContractTest`, `ReactiveCaseChannelProviderContractTest`) updated for 7-param signature

### openclaw
- Target set, agent registered for correct case — COMMAND delivered to target agent
- Target set, agent deregistered after dispatch — COMMAND dropped, WARN logged
- Target set, agent registered for different caseId — COMMAND dropped, WARN logged (routing bug)
- Target blank/null — falls back to `findAgentId()` (single-agent compat)
- Target set to empty string — treated as null, hits fallback path

## Pre-existing: DeliveryBatchExecutor actorType divergence

`DeliveryBatchExecutor.toOutbound()` resolves `actorType` via
`ActorTypeResolver.resolve(m.sender())` instead of using `m.actorType()`.
This is a pre-existing divergence unrelated to target routing. Out of scope
for this change — filed as qhorus#374.

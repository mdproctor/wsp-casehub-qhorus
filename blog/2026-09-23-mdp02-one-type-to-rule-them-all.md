---
title: One type to rule them all
date: 2026-09-23
series: qhorus
tags: [spi, enforcement, ras, design]
entry_type: note
subtype: diary
---

# One Type to Rule Them All

Qhorus's `ChannelProtocol` SPI has been returning `List<String>` since its inception. Four built-in protocols — REQUEST_RESPONSE, TASK_COMPLETION, ROUND_ROBIN, CONTRIBUTION_REQUIRED — each produce human-readable violation strings. The enforcement pipeline wraps them in `TaggedAdvisory(source, message)`, the enforcement gate treats them all equally, and `DispatchResult.advisories` passes them back as flat strings. No severity. No evidence. No suggested action. Every violation is the same weight.

That flat model can't answer the question that actually matters: is this violation dangerous enough to block the message, or is it informational?

## The three-type problem

My first instinct was three types: `ProtocolViolation` for the SPI return, `TaggedAdvisory` for the internal pipeline, and `ProtocolAdvisory` for the API output. Clean layering — each type owns its context. But a design review caught what I should have seen earlier: three types carrying the same five fields with trivial one-liner mappings between them is not clean layering. It's ceremony.

`DispatchAdvisory` replaced all three. One record, five fields: `source`, `severity`, `message`, `evidence`, `suggestedAction`. It flows from protocol evaluation through enforcement to API output without a single conversion. The SPI returns it. The enforcement gate reads it. `DispatchResult` carries it. No `fromViolation()`, no `toProtocolAdvisory()`, no `TaggedAdvisory::message` streaming.

## Severity as a circuit breaker

The interesting design decision was what to do when a CRITICAL violation hits an ADVISORY-mode channel. Two options: respect the channel operator's choice (ADVISORY means nothing blocks), or treat CRITICAL as a safety floor that overrides policy.

I went with the override. A protocol that emits CRITICAL severity is saying "this is dangerous enough to warrant blocking regardless of what the channel operator configured." The channel operator controls which protocols are active — they can remove a protocol entirely if they don't want its CRITICAL violations. But they can't configure a protocol to emit CRITICAL and then ignore it. That's the circuit breaker model: you install it or you don't, but once installed, it trips when it trips.

The enforcement gate now has a two-dimensional decision space:

| Channel Mode | CRITICAL | WARNING | ADVISORY |
|---|---|---|---|
| ADVISORY | **block** | log | log |
| BLOCKING | block | block | log |
| QUARANTINE | quarantine | quarantine | log |

`EnforcementBlockedException` gained `severityUpgrade()` and `effectiveMode()` so callers can distinguish "configured BLOCKING" from "CRITICAL override."

## Structured evidence

Each protocol now carries a `Map<String, Object>` evidence field. REQUEST_RESPONSE reports `{"openQueryCount": 3, "threshold": 3}`. ROUND_ROBIN reports `{"expectedSender": "agent-b", "actualSender": "agent-a"}`. This isn't just for display — it's the bridge to RAS.

The companion design (casehubio/casehub-ras#66) has the RAS adapter observing `ProtocolEvaluationEvent` from qhorus. Structured evidence means RAS ganglia can extract fields programmatically for temporal pattern detection — "3 TASK_COMPLETION violations with `openCommandCount >= threshold` in 10 minutes" becomes a computable condition, not a string to parse.

## What's next

Two batches remain: CDI event prerequisites (wiring `CommitmentStateChangedEvent`, firing `ChannelActivityEvent` as a CDI async event) and the channel policy overrides (V55 migration, `policyOverrides` on Channel). The YAML policy compiler and the RAS adapter module itself are designed in the spec but deferred to child issues — they need MVEL expression integration and cross-repo work respectively.

The `SuggestedAction` vocabulary landed as LOG, ESCALATE, INVESTIGATE, REROUTE — operational responses, not enforcement actions. Enforcement is controlled exclusively by Severity × EnforcementMode. The design review pushed hard on this separation, and it was the right call. Mixing enforcement semantics into the action vocabulary would have created two competing control paths for the same gate.

---
layout: post
title: "Governance in Action: When Agents Need More Than Politeness"
date: 2026-08-23
entry_type: article
subtype: diary
projects: [casehubio/qhorus]
tags: [governance, multi-agent, speech-acts, enforcement, causal-graphs]
---

# Governance in Action: When Agents Need More Than Politeness

Most multi-agent frameworks treat coordination as a messaging problem. Agent A sends a message to Agent B. B replies. Maybe there's a queue in between. The infrastructure moves bytes; the agents figure out the rest.

This works until it doesn't — and in production multi-agent systems, it stops working fast. An agent loops on the same task, generating identical messages until the context window fills. Two agents delegate work to each other in a circle. A third silently drops an obligation it accepted three minutes ago. The messages all delivered successfully. The system failed anyway.

The gap isn't communication. It's governance. Not in the corporate-compliance sense, but in the formal sense: who committed to what, whether they followed through, and what happens when they don't. This is the territory Qhorus's Phase 1 governance stack occupies.

## The governance stack

The insight behind Qhorus's design is that every agent message is a speech act — not just data transfer but a social commitment. A COMMAND creates an obligation. A DONE fulfils it. A DECLINE refuses it. These aren't protocol conventions; they're deontic primitives borrowed from speech act theory and social commitment semantics. The infrastructure tracks them, not the agents.

Phase 1 builds four layers on top of this foundation:

**Cross-channel causal graphs.** When work crosses channel boundaries — a coordinator delegates to a specialist on a different channel, who completes the work there — the causal chain fragments. The `CausalGraphService` reconstructs these chains by walking `causedByEntryId` links across channels. The result is a directed graph: who triggered what, across which channels, with what timing. A text renderer makes these graphs readable by both humans and LLMs:

```
Causal Graph — correlation abc-123 (FULFILLED, 5.0s, 2 channels)

  [work] COMMAND  agent-coordinator  "Analyze the Q3 dataset"  depth=0
    ├─[work] HANDOFF  agent-coordinator  depth=1  +2.0s
    └─[ops]  DONE     agent-analyst      "Analysis complete"    depth=2  +5.0s
```

This is attribution infrastructure. When something goes wrong, you can trace backwards from the failure to the root cause — across channels, across agents.

**Cascade containment.** Detection without response is a monitoring dashboard. The watchdog system already detected pathologies — looping agents, obligation fan-out, conversation stalls. Containment connects detection to action: pause the channel, expire open commitments, mark misbehaving agents offline. The key design choice was making containment actions composable — PAUSE_CHANNEL, DEREGISTER_AGENT, and QUARANTINE (both) are independent primitives that the watchdog selects per condition.

**Active enforcement.** This is the layer that transforms advisory warnings into actual governance. Every dispatch already produced advisories — protocol violations, type constraint warnings, correlation integrity checks. The enforcement gate promotes these to blocking rejections or full quarantine, per channel configuration.

The interesting design problem was transactional. The enforcement gate runs inside `MessageService.dispatch()`, which is transactional. When enforcement blocks a message, it throws — and the throw rolls back the transaction. Any audit record or containment action performed in that same transaction is undone. The fix: an `EnforcementExecutor` running in `REQUIRES_NEW`, modelled after the channel creation helper that solves the same class of problem. The executor commits audit events and containment actions in their own transaction before the outer transaction rolls back.

The other non-obvious constraint: terminal message types — DONE, FAILURE, DECLINE, RESPONSE — must always be exempt from enforcement. If a DONE is blocked on a channel with strict type constraints, the original COMMAND's obligation stays open permanently with no resolution path. This was the subject of an existing ADR that enforcement needed to respect, not override.

## Why this matters beyond Qhorus

The [Governance-as-a-Service paper](https://arxiv.org/html/2508.18765v2) (arXiv, 2025) argues that multi-agent governance should be a separable infrastructure concern, not embedded in agent logic. Qhorus's enforcement modes are a concrete implementation of that idea: the governance policy is a channel-level configuration (`ADVISORY`, `BLOCKING`, `QUARANTINE`), not agent code. An operator changes a channel's enforcement mode at runtime; no agent is redeployed.

The broader pattern is that as multi-agent systems move from demos to production, they'll need the same governance infrastructure that human organisations have always needed — not because agents are untrustworthy, but because distributed systems with autonomous actors need observable, enforceable commitments. Speech acts give you the vocabulary. The normative ledger gives you the audit trail. Enforcement modes give you the teeth.

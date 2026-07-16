# CaseHub Qhorus — Session Handover

**Date:** 2026-07-16 — #346 implemented, #348 fixed, #165 and #347 closed after analysis, roadmap filed (#349–#364).

---

## Immediate Next Step

Pick from the backlog below. #353 (correlation strengthening, S/Low) is the recommended first pick — smallest effort, highest practical payoff, no dependencies.

## What Was Done

Three categories this session:

**Implemented:** #346 (WebSocket catch-up — lastEventId replay, server-side buffering, messageId on MessageReceivedEvent). Adversarial design review caught a real race condition; original client-side dedup replaced with server-side buffering. #348 (flaky ConcurrentAutoChannelTest — channel-before-binding ordering + LinkedHashMap→ConcurrentHashMap across all 7 InMemory stores).

**Closed after analysis:** #165 (SmallRye bridge — won't do, module would ship 5 lines of logic; documented the pattern in the close comment instead). #347 (WebSocket single-node registry — already solved by CLUSTER scope + postgres-broadcaster; traced the full cross-node delivery path to prove it).

**Roadmap:** Research-backed analysis of LLM coordination failures (MAST taxonomy, grounding research, context rot, coordination tax). Filed 4 epics and 12 child issues (#349–#364) covering Qhorus infrastructure + cross-repo suggestions for engine, blocks, neocortex.

## Backlog — Prioritised by Effort-to-Impact

### Category A: Small Effort / High Practical Impact — Do First

These are grounded in empirical data (MAST taxonomy, context rot studies) and extend existing mechanisms. Highest payoff per hour invested.

| Priority | # | Title | Scale | Cplx | Why this order |
|---|---|---|---|---|---|
| **1** | #353 | Message correlation strengthening | S | Low | Prevents a concrete bug class (agents DONE-ing wrong correlations). Extends existing dispatch gate. MAST: 42% specification ambiguity. |
| **2** | #363 | Context window usage in EVENT telemetry | S | Low | Trivial telemetry extension (one nullable column). Gives watchdog an early warning signal for the #1 enterprise AI failure cause (context drift, 65%). |
| **3** | #362 | QUERY acknowledgement protocol | S | Med | Closes a gap in the obligation model — QUERYs are fire-and-forget with no tracking. Small effort but requires taxonomy design thought. |

### Category B: Medium Effort / High Practical Impact — Core Roadmap

These require more design but address empirically-validated failure modes. Each extends existing Qhorus infrastructure (WatchdogService, channel metadata, ledger).

| Priority | # | Title | Scale | Cplx | Why this order |
|---|---|---|---|---|---|
| **4** | #354 | Coordination pathology watchdog | M | Med | 37% of MAST failures. Extends existing WatchdogService with loop/stall/echo/fan-out detection. Directly feeds the supervisor pattern (engine#101). |
| **5** | #355 | Channel context summary slot + SPI | M | Med | 65% of enterprise AI failures are context drift. Infrastructure slot for blocks/summarisation to maintain per-channel state. Enables "cheap context" for agents joining channels. |
| **6** | #356 | Peer attestation | M | High | 21% MAST verification gaps. Multiple attestations per ledger entry, peer review as a ledger concept. Impact depends on how well LLMs can actually peer-review — less empirically proven than pathology detection. |

### Category C: Large Effort / Strategic — Design Before Building

These are the largest items with the highest complexity. #357 should wait until #354 validates which pathologies actually occur in practice before designing constraints to prevent them.

| Priority | # | Title | Scale | Cplx | Why this order |
|---|---|---|---|---|---|
| **7** | #357 | Protocol enforcement SPI | L | High | 7x accuracy gains evidence (PwC) but complex — turn-taking, round management, per-channel mutable state. Depends on #354. |

### Category D: Cross-Repo Suggestions — Filed on Qhorus for Tracking

To be opened in target repos once agreed. Ordered by platform impact.

| Priority | # | Title | Target | Scale | Cplx | Why this order |
|---|---|---|---|---|---|---|
| **8** | #358 | Supervisor + friction interventions | engine | L | High | Most impactful missing platform piece. Consumes Qhorus watchdog (#354). Already engine#101. Blocks 6 downstream issues. |
| **9** | #359 | Summarisation → Qhorus integration | blocks | M | Med | Practical glue — summarisation logic exists, needs channel summary slot (#355) as integration point. |
| **10** | #361 | CBR routing + coordination memory | blocks/neocortex | M | High | Evidence-based team selection. Practical but requires new CBR case type design for coordination patterns. |
| **11** | #360 | Common ground projection | blocks | M | High | Compelling research (grounding gaps, epistemic tracking) but application to Qhorus channels is the most theoretical item. Design before building. |
| **12** | #364 | Convergence detection | blocks | M | High | Detects consensus/deadlock in deliberation. Feeds supervisor. Most value when supervisor (#358) exists. |

### Epics

| # | Epic | Children |
|---|------|----------|
| #349 | Coordination Resilience | #353, #354, #362 |
| #350 | Channel Intelligence | #355, #357, #363 |
| #351 | Verification & Trust | #356 |
| #352 | Cross-Repo Suggestions | #358, #359, #360, #361, #364 |

## References

| What | Path |
|------|------|
| Normative layer thesis | `docs/normative-layer.md` |
| Roadmap research sources | Issue bodies (#349–#364) contain full citations |
| #346 design spec | `docs/specs/2026-07-14-websocket-catchup-design.md` |
| #346 design review | `~/adr/casehub-qhorus/websocket-catchup-20260714-120059/tracker.md` |
| #346 blog entry | `blog/2026-07-14-mdp01-race-condition-subscribe-replay.md` |

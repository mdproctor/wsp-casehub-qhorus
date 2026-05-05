# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-04-20 — Unified human view of Claude work via hierarchical WorkItems + channel intent

**Priority:** high
**Status:** active

The core problem: Claude work is fragmented across systems no human can
read end-to-end — GitHub has code, the ledger has telemetry, WorkItems has
formal tasks, Qhorus channels have reasoning, HANDOFF.md has session context.
No single thread runs through them capturing *goal → task → Claude conversation
→ decision → outcome → code*. The goal is a **unified narrative view** that
lets a human ask "why does this code look like this?" and trace back through
decisions and the tasks that motivated them.

Two mechanisms make this possible together:

**1. Channel intent markers.** Every Claude-to-Claude channel opens with a
structured task context message — goalId, intent, initiator — forming a
boundary marker linking the conversation to a WorkItem or epic. The ledger
captures this at channel open and close (DONE message). Claudony aggregates
across channels to synthesise human-readable summaries. No explicit WorkItem
required for every exchange — the implicit record lives in the ledger.

**2. Hierarchical WorkItems as a Hierarchical Task Network (HTN).** Introducing
parent-child WorkItems gives a formal decomposition mechanism for distributing
work across multiple Claude agents. A top-level WorkItem (the goal/epic) breaks
into child WorkItems delegated to specific Claudes. As more Claudes get involved,
the hierarchy provides coordination: each Claude knows what it owns, what its
parent goal is, and what its children have produced. The unified view rolls up
child outcomes to parent, giving humans a hierarchical picture of what happened
and why. This also clarifies when Claudes use WorkItems vs Qhorus direct
messaging: Qhorus when exploring/collaborating (no formal handoff), WorkItems
when formally delegating a task with an expected outcome and accountability.

Claudony is positioned to be the layer that synthesises both — observing Qhorus
channels, tracking WorkItem hierarchy, and producing the unified view.

**Context:** Brainstorm on Qhorus reactive migration, 2026-04-20. Discussion on
Qhorus vs WorkItems for Claude-to-Claude coordination surfaced the deeper
problem of human observability across fragmented systems. The HTN framing emerged
from asking "when do Claudes need WorkItems?" — the answer is when work is being
formally decomposed and delegated, not when Claudes are figuring something out
together. The unified view is the goal; channel intent markers and hierarchical
WorkItems are the mechanisms.

**Promoted to:**

---

## 2026-04-18 — Speech act theory as a framework for Qhorus MessageType

**Priority:** medium
**Status:** implemented

The current `MessageType` enum (`request | response | status | handoff | done | event`) conflates communication function with workflow role. Speech act theory (Austin/Searle) offers a cleaner theoretical foundation: **assertives** (claiming something is true), **directives** (getting someone to act), **commissives** (committing to a future action), **declarations** (changing state by saying it), **expressives** (psychological states — probably not needed for agents).

The current types map loosely:
- `request` → Directive, but conflates query-for-information with command-to-act
- `response` → Assertive (reply to a directive)
- `status` → Assertive (unsolicited update) — arguably redundant with `response`
- `handoff` → Directive + Commissive ("take this and I step back")
- `done` → Declaration (changes state by saying it)
- `event` → Assertive (factual log/observation)

A richer taxonomy might distinguish `query` from `request`, and potentially collapse `status` into `response`. Those distinctions are meaningful for agents deciding how to handle an incoming message — more so than the current types which an agent has to guess at.

**This is a Qhorus concern, not a Claudony concern.** MessageType is defined in Qhorus and used by all agents. Richer semantic types at the infrastructure layer benefit every consumer. The idea originally surfaced in Claudony during human interjection design (2026-04-18) — but that was accidental framing; the real question is about Qhorus's core message taxonomy. Moved here from claudony/IDEAS.md on 2026-04-23.

**Key risk:** changing `MessageType` is a breaking change for all Qhorus consumers. Needs careful versioning or a deprecation path.

**Promoted to:** ADR-0005, #88 (implemented 2026-04-23)

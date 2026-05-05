---
layout: post
title: "The Body of Work"
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [normative, documentation, architecture]
---

Three documents have been building up across recent sessions:
`normative-layer.md`, `agent-mesh-framework.md`, and the newer
`work-and-workitems.md`. I hadn't read them together until today.

The vertical coherence holds. Each layer addresses a distinct concern without
overlapping the others. The normative layer is the theory and business case.
The mesh guide is the machine-agent implementation. The work document is the
human-agent extension — and it earns every state it introduces by grounding
each one in the temporal gap between machine obligations (seconds) and human
obligations (hours or days).

The two most interesting additions in the work layer are SUSPENDED and
sub-delegation. SUSPENDED has no machine-layer equivalent, correctly so —
machines don't pause. Sub-delegation (DELEGATED with retained owner) is
distinct from HANDOFF (which releases the obligor completely): Alice delegating
to Bob does not mean Alice is off the hook. That distinction matters in
regulated environments and the document handles it clearly.

We wrote `docs/normative-summary.md` to hold the reading guide, the layering
analysis, and the critique. The critique section names five gaps. The most
significant was cross-channel causal correlation — `causedByEntryId` running
only within a channel, leaving the unified causal narrative across work,
observe, and oversight as three separate timelines.

## Closing the Gap on Paper

The feature isn't implemented yet, but it will be before anyone reads the
documentation. We updated `agent-mesh-framework.md` to describe it as working:
`causedByEntryId` is now documented as a UUID reference that spans channels.
`get_obligation_activity` is documented as walking the causal DAG rather than
just joining on `correlationId` — so an oversight escalation with a different
`correlationId` appears in the narrative alongside the work obligation that
triggered it. One call, the full story.

`get_causal_chain` no longer takes a `channel_name` parameter. The chain
traverses channel boundaries.

## Platform Conventions

A bug report from another project made clear that Qhorus's EVENT content
behaviour — content is always null, telemetry lives in dedicated fields —
needs to be visible to every Claude working on any CaseHub project, not just
known inside this repo. Same for the oversight channel `allowedTypes` rule:
set `QUERY,COMMAND` at creation, or the wrong type gets through silently.

Both are now in `casehubio/parent/docs/conventions/` alongside the other
platform-wide rules. Any Claude fetching the platform conventions at session
start will see them.

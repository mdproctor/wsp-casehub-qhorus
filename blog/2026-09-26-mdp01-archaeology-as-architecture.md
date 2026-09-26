---
layout: post
title: "Archaeology as Architecture"
date: 2026-09-26
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [arc42stories, documentation, retrospective, architecture]
series: issue-456-arc42stories-migration
---

# Archaeology as Architecture

Qhorus was the last platform-tier repo without an ARC42STORIES.MD. Every other foundation module — work, ledger, engine, eidos, connectors — had one. The profile's "Reference Implementations" section for foundation tier still said "deferred." I decided today was the day to fix both.

The question that shaped the entire session: should the chapter log be prospective (start empty, fill in going forward) or retrospective (reconstruct the full build history)? I chose retrospective. Qhorus has 777 commits and ~450 closed issues across six months. That's a real architectural narrative, and prospective-only would mean it never gets written — nobody comes back to fill in the boring parts once the template is in place.

The interesting part was deriving the layer taxonomy. DESIGN.md had a flat 15-phase build roadmap. ARC42STORIES wants layers — architectural concerns that cut across features. I ended up with eight: Core Domain, Speech Act Protocol, Normative Audit, Channel Features, Enforcement & Safety, Transport Layer, API Surface, Compliance & Enterprise. These aren't modules. They're concerns. A single feature like the RoutingBridge touches L2 (speech act dispatch pipeline), L3 (ledger metadata), and L8 (trust-weighted selection). The layer taxonomy makes those cross-cutting relationships visible in a way the old phase list couldn't.

Fitting 450 issues into 55 chapters across 6 journeys required judgment calls. The original phases mapped cleanly to Journey 1 (the core mesh — entities through reactive dual-stack). But everything after Phase 15 had no prior grouping. The normative governance work — commitments, speech act taxonomy, attestation — was the clearest journey boundary. The transport layer was another. Channel intelligence (topics, reactions, spaces, membership, presence) cohered naturally. The hardest grouping was Journey 6 — a catch-all for trust, compliance, and routing that's really three separate concerns sharing a timeline more than a theme. If I had to redo it, I might split compliance into its own journey. But 6 journeys is a reasonable number and the chapter sequencing rationale holds.

The casehub-work ARC42STORIES runs to 2,067 lines across 35 chapters. This one landed at 1,534 lines across 55 chapters — shorter entries on average, which fits. Work's chapters each represent a focused build phase with new entities and tests. Qhorus's later chapters are often feature additions within existing architectural seams (adding a new watchdog condition type, for instance). The before/after narrative is real but the "after" builds on more existing infrastructure.

What this opens up: the profile's foundation-tier reference implementation can now point at qhorus. And future sessions that modify modules, SPIs, or structural boundaries can check ARC42STORIES.MD §9 to understand what journey and layer a change touches — context that was previously scattered across CLAUDE.md's enormous project structure section and git history.

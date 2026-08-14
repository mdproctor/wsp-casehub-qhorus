---
layout: post
title: "Who Watches the Watchers"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [trust, attestation, credibility, bayesian, collusion, spi]
---

The trust model scored subjects but not attestors. An agent that consistently endorsed poor work had the same weight as one that tracked quality accurately. The fix is credibility weighting — a third factor in the trust computation alongside recency decay and confidence.

The design took eight decisions, each quick except for where to surface the credibility signal in `ActorScore`. The first-principles analysis there was worth the time: `credibilityRetention` as a single ratio (effective weight / raw weight) tells operators whether the score they're looking at rests on credible evidence, without coupling the generic computation result to specific flag vocabularies. The `NaN` sentinel for "all attestors have insufficient data" was a cross-cutting review finding — without it, `1.0` would have meant both "all attestors are credible" and "the credibility system has no data," which is exactly the kind of ambiguity that erodes trust in a trust system.

The interesting architectural question was where credibility gets applied. At write time (bake it into the attestation confidence before persisting) or at computation time (preserve raw confidence in the ledger, apply credibility dynamically)? Computation time won because rehabilitation matters — if an attestor improves, their old attestations should automatically get re-weighted. The immutable ledger records the fact; credibility interprets its reliability.

The implementation splits cleanly across repos: SPI interface in casehub-ledger api, `TrustScoreCalculator` injection in ledger runtime, two concrete implementations in casehub-qhorus. The default (`AgreementCredibilityPolicy`) compares peer verdicts against policy outcomes via Bayesian Beta — same math as the trust computation itself, applied reflexively. The collusion-aware variant adds mutual-endorsement pair detection behind a runtime config gate, using composition rather than inheritance to keep the coupling loose.

Cross-repo SNAPSHOT coordination was the unexpected friction. Ledger installs to a worktree-specific `.m2/` (not `~/.m2/repository`), so qhorus resolves a stale jar from GitHub Packages even after a clean install. The fix was overwriting the timestamped jar directly — the kind of thing that works but shouldn't be necessary.

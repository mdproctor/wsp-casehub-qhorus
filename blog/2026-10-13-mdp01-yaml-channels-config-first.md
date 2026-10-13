---
title: "YAML channels — config-first for the zero-authored-Java world"
date: 2026-10-13
entry_type: note
subtype: diary
author: mdp
projects: [casehub-qhorus]
tags: [channels, config, yaml, smallrye, quarkus]
refs: ["#464"]
---

# YAML channels — config-first for the zero-authored-Java world

The engine project has a goal: a fully functional Quarkus AI agent deployment with zero authored Java files. The app.yaml declares everything — agents, channels, routing rules, connectors — and the platform wires it all up at startup. Channels were the gap. Until now, declaring a channel meant writing Java: `ChannelCreateRequest.builder("reviews").semantic(APPEND).allowedWriters(List.of("agent-a")).build()` inside some `@Observes StartupEvent` method. For a YAML-only app, that's a hard stop.

The fix is straightforward in concept. SmallRye Config already supports `Map<String, T>` in `@ConfigMapping` interfaces — the same pattern Quarkus uses for named datasources. We added a `channels()` method returning `Map<String, ChannelConfig>` to `QhorusConfig`, where the map key is the channel name and `ChannelConfig` carries the full field surface: semantic, access control lists, type constraints, governance protocols, enforcement mode, rate limits, routing thresholds, delivery tracking, space assignment, metadata, and connector binding. A `ChannelConfigBootstrap` bean observes `StartupEvent`, builds a `ChannelCreateRequest` from each config entry, and calls `findOrCreate()`.

The interesting part is what happens when the channel already exists. A channel might have been created by the YAML config on a previous deploy, then modified at runtime via the REST API — an operator tightened the allowed-writers list, or changed the enforcement mode from ADVISORY to BLOCKING. On the next restart, the YAML still says ADVISORY. We chose drift detection over reconciliation: the bootstrap compares 15 declared fields against the existing channel and logs each divergence as a WARN. It never auto-corrects. The YAML is the intended state; the runtime state is the truth. When they diverge, a human decides which one to update.

This is a deliberate design choice, not a missing feature. Kubernetes-style reconciliation loops make sense for stateless infrastructure, but channels are stateful entities with commitment histories, membership rosters, and delivery cursors. Silently overwriting a runtime-modified enforcement mode could break active governance flows. The WARN log surfaces the gap without making it worse.

One gotcha along the way: SmallRye Config rejects `Optional<Map<String, String>>` in `@ConfigMapping` interfaces. Every other Optional pattern works — `Optional<String>`, `Optional<List<T>>`, `OptionalInt`, `Optional<EnumType>` — but wrapping a Map throws `IllegalArgumentException` at augmentation time. The fix is to use `Map<String, String>` directly; SmallRye returns an empty map when no keys are declared. The inconsistency isn't documented anywhere obvious, and the error message is a wall of generic type signatures that takes a minute to parse.

The feature is small in scope — four files of production code — but it's the kind of plumbing that determines whether an entire deployment model is viable. With this landed, the engine project can declare its channel topology in YAML alongside its agent definitions, and the zero-authored-Java milestone stays on track.

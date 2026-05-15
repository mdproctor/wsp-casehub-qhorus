---
layout: post
title: "What ExcludedTypeBuildItem Actually Excludes"
date: 2026-05-15
type: pivot
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [quarkus, cdi, jax-rs, extension]
---

I went into this confident. The plan was to make `quarkus-hibernate-reactive-panache`
optional in the runtime pom — so consumers on JDBC-only datasources wouldn't get the
reactive extension activated unconditionally — and gate the reactive vs blocking bean
selection via `ExcludedTypeBuildItem` in a `@BuildStep` condition driven by
`Capabilities.isPresent(Capability.HIBERNATE_REACTIVE)`. No user-facing flag. Classpath
presence as the single source of truth. The Quarkus guide shows exactly this pattern.

We built it. The runtime test suite passed.

Claudony disagreed. It reported a duplicate endpoint conflict — both `A2AResource` and
`ReactiveA2AResource` registering the same `/a2a` path simultaneously, something that
should have been impossible given the exclusion logic.

Three separate problems, discovered in the wrong order.

**`ExcludedTypeBuildItem` doesn't exclude JAX-RS resources.** It removes a class from
CDI bean discovery. `ResteasyReactiveProcessor` — which scans for `@Path` annotations —
runs as an independent build step and reads the Jandex index directly. It doesn't check
CDI exclusions. Both REST resources registered because the CDI and REST subsystems are
decoupled at build time and neither consults the other.

**`Capabilities` can't be injected in `BooleanSupplier`.** The condition supplier runs
before any `CapabilityBuildItem` producers have fired — it evaluates first so the build
step graph knows which steps to include. The injected `Capabilities` object has no data
at that point. Every `isPresent()` call returns false. No error, no warning.

Claude found the third problem independently: adding `@Priority(1)` to the reactive JPA
store beans — to make CDI auto-select them in production — created an ambiguity with
`InMemoryReactiveChannelStore @Alternative @Priority(1)` from the testing module. Two
same-priority alternatives for the same type. `DeploymentException: Ambiguous
dependencies for type ReactiveChannelStore`. The extension's own tests passed; the
failure only appeared in consumers that included the testing module alongside the main
extension.

At this point we took a step back, searched for the actual Quarkus mechanism, and found
Quarkus issues #34938 and #16218. `@IfBuildProperty` and `@UnlessBuildProperty` on JAX-RS
resources had a known bug — fixed in 3.2.3.Final. We're on 3.32.2. The bug is gone.

The fix turned out to be considerably simpler than the approach I'd designed. Put
`@IfBuildProperty(name = "quarkus.datasource.qhorus.reactive", stringValue = "true")`
directly on the reactive REST resources, services, and stores. Put
`@UnlessBuildProperty(name = "quarkus.datasource.qhorus.reactive", stringValue = "true", enableIfMissing = true)`
on their blocking counterparts. Mark the reactive Panache dep `<optional>true</optional>`
in the runtime pom. `QhorusProcessor` shrinks to a single build step that registers the
extension feature name.

The property choice matters. `quarkus.datasource.qhorus.reactive` is the property
consumers already set when they configure a reactive datasource for qhorus — it's not an
additional flag to document. A consumer on JDBC-only sets nothing, gets nothing reactive.
A consumer who wants reactive sets one property they were going to set anyway.

The `@Priority(1)` problem has a clean resolution: don't add it to the JPA store beans
at all. Gate them with `@IfBuildProperty` instead. In blocking mode they're excluded from
CDI entirely — no injection point validation, no ambiguity. In reactive production mode
they're the unique `@Alternative` for their interface, and Quarkus Arc auto-selects a
unique alternative. In reactive test mode, the InMemory stores' `@Priority(1)` wins
cleanly.

`ExcludedTypeBuildItem` still has its place — it's the right tool for CDI-only beans.
For anything that also participates in JAX-RS scanning or MCP tool discovery,
`@IfBuildProperty` is the mechanism that actually works.

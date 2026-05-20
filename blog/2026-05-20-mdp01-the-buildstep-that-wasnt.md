---
layout: post
title: "The @BuildStep that wasn't"
date: 2026-05-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [quarkus, extension, cdi, build-time]
---

Two things got fixed today. One took three lines. The other consumed most of the session and ended with us back where we started — but one concept richer.

**The three-line fix**

Claudony was seeing empty channel timelines in tests. Messages sent via `sendMessage()` go through `MessageStore`. `getChannelTimeline()` in the reactive MCP tools was reading from H2 via raw Panache — `Message.<Message>find(...)` — bypassing the store entirely. In test environments using `InMemoryMessageStore`, that means reading from a database that never received the messages.

The fix: inject `MessageStore blockingMessageStore` and use `blockingMessageStore.scan(MessageQuery.poll(ch.id, afterId, effectiveLimit))`. The method already runs `@Blocking`, so a blocking store call is fine. Two imports and one line changed. Claudony's tests unblocked.

**The @BuildStep that wasn't**

The bigger item was renaming the reactive gating property from `quarkus.datasource.qhorus.reactive` to `casehub.qhorus.reactive.enabled` — the platform standard — and centralising the exclusion logic in a `@BuildStep` using `ExcludedTypeBuildItem`, matching what casehub-ledger is implementing.

The approach is clean on paper: a `QhorusBuildTimeConfig` interface in the deployment module declares the property as `BUILD_TIME`; `QhorusProcessor` gains a `@BuildStep` method that reads the config and produces `ExcludedTypeBuildItem` for the appropriate beans. Twenty-three reactive classes excluded when the property is absent; five conflicting blocking entry-point classes excluded when it's present.

We wrote it. It compiled cleanly. The bytecode had the method. `META-INF/quarkus-build-steps.list` named the processor. Every indicator said it should work.

It produced nothing.

We spent a long time trying to understand why. Tried `void selectStack(BuildProducer<ExcludedTypeBuildItem>)`. Tried `List<ExcludedTypeBuildItem> selectStack()` with no parameters. Added `System.err.println` inside the method — the line never appeared in test output. Hardcoded `false` to eliminate the config read. Same result. The `feature()` method in the same class runs fine — the Quarkus feature appears in the startup log — so the processor IS loaded. Just this method is not called.

The root cause remains opaque. Quarkus's test framework in workspace mode (where the deployment module is resolved from `deployment/target/classes/` rather than the .m2 JAR) appears not to invoke custom `@BuildStep` methods that produce optional multi-build-items. There's no error, no warning, no indication. The method exists and is correct; it simply isn't invoked.

The working solution is the one we started with: `@IfBuildProperty` per bean. Renamed to `casehub.qhorus.reactive.enabled`. With `QhorusBuildTimeConfig` still present to declare the property as `BUILD_TIME` — without that declaration, `@IfBuildProperty` on a custom property in an extension context is unreliable. The declaration does the essential work even when the exclusion logic doesn't.

There's a second trap I didn't anticipate. `casehub.qhorus.reactive.enabled=false` in `application.properties` triggers `SRCFG00050: does not map to any root` at runtime. `application.properties` is read at both augmentation time and runtime. The `BUILD_TIME`-only `@ConfigRoot` has no runtime mapping, so SmallRye Config's runtime validator rejects it. The fix is to not put it there at all — the `@WithDefault("false")` in `QhorusBuildTimeConfig` handles the default, and the property absence is exactly the right signal at augmentation time.

The `@IfBuildProperty(enableIfMissing=true)` pattern handles the rest.

Both discoveries went into the garden. Whether `ExcludedTypeBuildItem` from a workspace deployment module is a Quarkus bug or a documented limitation, I don't know. Worth checking against a future version.

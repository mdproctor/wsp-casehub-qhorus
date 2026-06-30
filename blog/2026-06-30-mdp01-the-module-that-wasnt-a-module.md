---
layout: post
title: "The Module That Wasn't a Module"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [module-extraction, micrometer, delivery, last-write]
---

## The Module That Wasn't a Module

Qhorus has had in-memory store implementations since the early days — `InMemoryChannelStore`, `InMemoryMessageStore`, and their reactive mirrors. They lived in `testing/`. That was always wrong, but it was the kind of wrong that doesn't hurt until someone tries to use them for what they actually are: a standalone persistence backend.

The module-tier-structure protocol spells it out clearly. In-memory stores serve two purposes — test isolation and zero-config ephemeral installs. Neither purpose is a test utility. A module called `testing/` is unsuitable for the second case: it signals "test-only dependency" and may bundle test-framework artifacts. The stores needed their own home.

`persistence-memory/` is that home. The extraction was straightforward — 18 store implementations, their 23 test files, the contract test base classes, all moved with a package rename from `io.casehub.qhorus.testing` to `io.casehub.qhorus.persistence.memory`. `testing/` keeps `RecordingChannelBackend` and `MessageLedgerEntryTestFactory` — genuine test utilities. It gains a compile dependency on `persistence-memory/`, so every existing consumer gets the stores transitively. Zero POM changes for downstream modules.

The package rename is the point, not the cost. Every `import io.casehub.qhorus.testing.InMemory...` across the platform now says what it means. Breaking the imports forces every caller to acknowledge the new location. The `quarkus.arc.selected-alternatives` entries in `application.properties` files are the most critical — fully-qualified class names that silently fail to match if the package is wrong.

One thing fell out of this that's worth tracking: the Store SPI interfaces themselves still live in `runtime/`, not `api/`. The module-tier-structure protocol says they should be in api/ (Tier 1, pure Java). That would let `persistence-memory/` depend on the lightweight api/ module instead of the full runtime. I filed it as a follow-up rather than expanding the scope — it's a cross-repo migration touching every consumer in the platform.

### The Classloading Gotcha

The delivery metrics work produced a genuine gotcha. I added `MeterRegistry` to `DeliveryService`'s constructor using `Instance<MeterRegistry>` for optional injection — the standard Quarkus pattern for "use it if it's there." `micrometer-core` was declared at `provided` scope, copying the pattern from `connector-backend` where it works perfectly.

Claude caught it first. `SlackBotBindingStoreTest` failed with `ClassNotFoundException: io.micrometer.core.instrument.MeterRegistry` — a test that has nothing to do with metrics or delivery. The CDI augmentation failed globally because the class wasn't loadable, and every `@QuarkusTest` in every transitive dependent went down with it.

The distinction is subtle: `Instance<T>` makes CDI *injection* optional, but the *class* must still be loadable. `provided` scope means the class isn't on the transitive classpath. It works in `connector-backend` because that module is a leaf — its own tests supply `quarkus-micrometer`. It breaks in `runtime/` because slack-channel, connector-backend, and every example module depend on it transitively. Compile scope is the only correct answer for a module that others depend on.

### Version Counters and the Log That Was Already There

The last piece was LAST_WRITE delivery — channels that overwrite message content in-place. The cursor-based delivery pump tracks by message ID, so content changes within the same ID are invisible. A version counter on the `Message` entity is the minimal mechanism: LAST_WRITE overwrites increment it, the cursor stores `lastDeliveredVersion` alongside `lastDeliveredId`, and the batch query gains a second clause: `(id = cursorId AND version > cursorVersion)`.

The version counter pairs with a delivery signal on overwrite. Until now, LAST_WRITE overwrites explicitly skipped `fanOut()` — the design comment even said "this may be revisited." The version counter is the revisit: tracked backends get event-driven notification via the same `DeliverySignalQueue` that normal dispatch uses. BEST_EFFORT backends remain unaffected.

Four issues, one branch, one build. The kind of session where nothing is individually hard but the cumulative result is a cleaner platform.

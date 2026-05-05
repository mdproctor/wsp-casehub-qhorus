---
layout: post
title: "Jlama Fixed, CI Housekeeping"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [jlama, ci, testing, examples, casehubio]
---

Yesterday's entry ended on the Jlama wall — `quarkus-langchain4j-jlama 0.26.1` crashing at test bootstrap with `Unsupported value type: [ALL-UNNAMED]`. I'd traced the root cause to `META-INF/quarkus-extension.properties` inside the Jlama runtime JAR, containing `dev-mode.jvm-option.std.enable-native-access=ALL-UNNAMED`. The fix location was clear: remove those JVM options from `model-providers/jlama/runtime/pom.xml`. One commit to the cloned repo, problem solved.

Except it wasn't one commit.

## Four bugs, not one

The first fix landed cleanly — boot no longer threw `Unsupported value type`. Then `ChatMemoryProcessor` threw `IllegalArgumentException: Run time configuration cannot be consumed in Build Steps`. That's a Quarkus 3.32 API enforcement issue: runtime config can't be injected directly into `@BuildStep` methods. Same pattern applied to `JlamaProcessor.generateBeans()`. Then `JlamaChatModel.promptContext()` threw `NullPointerException` on `toolSpecifications.isEmpty()` — a missing null check when no tools are configured.

Four commits total: remove the devMode JVM options, fix `ChatMemoryProcessor`, fix `JlamaProcessor`, add the null check. Each one revealing the next.

Getting the config lookup right in `JlamaAiRecorder` took an extra iteration. The recorder methods needed to access `LangChain4jJlamaConfig` at runtime — not at build time — which meant using `ConfigProvider.getConfig().unwrap(SmallRyeConfig.class).getConfigMapping(LangChain4jJlamaConfig.class)` inside the supplier's `get()` method. The first attempt used `Arc.container().instance()`, which returns null for `@ConfigMapping` interfaces. The SmallRye Config API is the correct approach; it's just not the first thing you reach for.

## ARM_128 and the native library

With the bootstrap fixed, the model started loading. Then inference threw `UnsupportedOperationException: ARM_128` — the Java Vector API's 128-bit ARM species hitting an unsupported code path in `PanamaTensorOperations`. Apple Silicon with NEON is 128-bit; Jlama's Panama implementation apparently only supports 256-bit+.

The fix was already in the JAR. We found `jlama-native-osx-aarch_64.jar` in the local Maven cache — it contains `libjlama.dylib`, the native Apple Silicon implementation that bypasses Java Vector API entirely. Adding it as a dependency with `classifier=osx-aarch_64` routes tensor operations through the native library. The surefire `<argLine>` still needs `--add-modules jdk.incubator.vector` and `--enable-native-access=ALL-UNNAMED` because the incubator module has to be declared even on Java 26.

## CI coverage that actually runs

The examples worked, but the LLM tests required a ~700MB model download and couldn't run in CI. Rather than accept that, we built `examples/type-system/` — a new module that runs in 2.6 seconds with no model, no LLM, no Docker. Thirteen tests covering every deontic constraint: DECLINE rejected without content, HANDOFF rejected without target, QUERY and COMMAND auto-generating `correlationId`, all nine type names and their helper methods.

The LLM examples moved behind `-Pwith-llm-examples`. CI gets the fast tests by default.

One XML comment in `pom.xml` with `--` in it — from copy-pasting an error message — silently broke the build with `Non-parseable POM`. XML forbids `--` anywhere inside `<!-- -->`. Worth knowing.

## casehubio

The repo moved to `github.com/casehubio/qhorus` — the same org as `quarkus-ledger`. The `gh api -X POST repos/.../transfer` call returns 200 immediately but the transfer is asynchronous; polling `gh repo view casehubio/...` returns "not found" for 30-odd seconds before it resolves. Three file updates, one remote URL change, push from the new location. Clean.

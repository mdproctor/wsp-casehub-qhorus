---
layout: post
title: "What drop-and-create hides"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [flyway, cdi, testing, schema]
---

The cleanup epic had six issues. Most were straightforward. Two were not.

**The schema that couldn't be tested**

The `commitment` table had no Flyway migration. Tests passed because Quarkus uses `drop-and-create` — Hibernate derives the schema from entity annotations, writes it to H2, and nobody notices that the `.sql` files are never exercised. The first time Flyway runs against a real database, the table doesn't exist.

We also had `pending_reply` still sitting in `V1__initial_schema.sql`, a relic of the old `wait_for_reply` correlation mechanism replaced months ago by `CommitmentStore`. Dead DDL. No code referenced it.

Fixing the migration files was two minutes. Testing that the fix was correct turned out to be the interesting problem. `@QuarkusTest` doesn't help — it routes through `drop-and-create` and never runs Flyway. The test that proves the schema is right has to be something else.

We built `FlywayMigrationSchemaTest` — a plain-Java JUnit 5 test, no Quarkus, that runs migrations against H2 directly via the Flyway API and then checks `INFORMATION_SCHEMA` with `getMetaData().getTables()`. One class, two assertions: `pending_reply` absent, `commitment` present. TDD gives you the RED immediately — the tables are in the wrong state — and the migrations make it GREEN.

Two things bit us in the process. First: Flyway's `baselineOnMigrate(true)` defaults to `baselineVersion = "1"`, which marks V1 as already applied and skips it. We needed `baselineVersion("0")`. The comment in the code now explains why — a reader who "fixes" it will break V1 coverage immediately. Second: `ReactiveLedgerEntryRepository` (from casehub-ledger) was unsatisfied in every non-reactive test context. This wasn't new — it had been silently blocking all `@QuarkusTest` tests for weeks. We fixed it with a `@DefaultBean` stub in test sources: all methods throw `UnsupportedOperationException`, none are expected to be called. One class, all `@QuarkusTest` tests unblocked.

**The one-word change with four traps**

`MessageObserverDispatcher` was leaking `@Dependent` beans. The fix was replacing `Iterable<MessageObserver>` with `Iterable<? extends Instance.Handle<MessageObserver>>` and calling `handle.close()` in a `finally` block. Callers change one word: `observers` becomes `observers.handles()`.

The `? extends` wildcard is unavoidable. `Instance.handles()` returns `Iterable<? extends Instance.Handle<T>>` — the CDI standard type. `Iterable<InstanceHandle<T>>` (the Arc-specific type) is not assignable, because Java generics are invariant. Every attempt to use the Arc type in the parameter signature fails at compile time with a type-inference error that doesn't mention the problem.

Code review caught two issues I'd missed. `handle.get()` was outside the `try` — if Arc throws `IllegalStateException`, remaining handles never get closed and the exception escapes the dispatcher's fault isolation. Nested try/finally fixes it: `get()` inside the outer try, `close()` in the outer finally. The reviewer also caught that `getSuperclass().getSimpleName()` logs `"Object"` for lambda observers — the fix uses `getSimpleName()` directly when `getSuperclass()` returns `Object`.

Tests for anonymous `InstanceHandle` implementations in plain-Java tests need to override `close()` explicitly. The default `close()` calls `Arc.requireContainer()` and blows up without a container. `get()` is the only abstract method, so nothing tells you this until the test runs.

The `@ApplicationScoped`-only constraint on `MessageObserver` is now retired. Any normal CDI scope works.

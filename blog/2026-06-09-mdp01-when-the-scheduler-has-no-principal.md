---
layout: post
title: "When the Scheduler Has No Principal"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [multi-tenancy, cdi, quarkus, scheduler]
---

The design for multi-tenancy looked straightforward: JPA stores inject `CurrentPrincipal`, read `tenancyId()`, add `AND tenancyId = ?` to every query. Every read returns only what belongs to the caller's tenant. Writes stamp the same value at creation. Tidy.

Then we tried to wire up `WatchdogEvaluationService`.

The watchdog runs on Quarkus's internal scheduler thread pool. No HTTP request. No CDI `@RequestScoped` context. `CurrentPrincipal.tenancyId()` would throw `ContextNotActiveException` at runtime — not at startup, not at CDI augmentation, but at the exact moment the scheduler fires. The kind of failure that passes every test until it doesn't.

We'd already built cross-tenant stores (`@CrossTenant CrossTenantChannelStore`, etc.) for the ChannelGateway startup path — that problem was anticipated. But the watchdog does more than list channels. It dispatches alert messages through `MessageService.dispatch()`, and `dispatch()` needs to know the tenant of the message it's sending.

The original design said `MessageService.dispatch()` reads from `currentPrincipal.tenancyId()`. Fine for HTTP callers. For the watchdog, there is no principal.

The fix was to make the tenant explicit on the dispatch. `MessageDispatch` got a 14th field — `String tenancyId`. Null means the service should read from `CurrentPrincipal` (normal HTTP path). Non-null means use it as-is (system path). The watchdog passes `w.tenancyId`, which was stamped on the `Watchdog` entity at registration time, when a real request context existed. `MessageService` uses it without touching `CurrentPrincipal`.

The same mechanism resolved `ChannelStore.updateLastActivity()`, which is called from both `MessageService.dispatch()` (request context active) and indirectly from the watchdog (no context). Rather than injecting `CurrentPrincipal` in the store method — which would fail in the scheduler — we changed the signature to `updateLastActivity(UUID channelId, String tenancyId)`. Callers pass the tenancyId they already have. The read-path JPA stores still filter via `CurrentPrincipal`; write-path operations that straddle both worlds take an explicit param.

The pattern is GE-20260531-446fea applied to method signatures instead of Quartz job data maps. The principle is the same: capture tenancyId at the point where a request context is available, store it on the entity, read it back from state when the context is gone.

Two other things caught me off guard.

Mid-implementation, the casehub-ledger SNAPSHOT updated its repository interfaces — all `LedgerEntryRepository` methods gained a `tenancyId` parameter, and five methods moved to new `CrossTenant*Repository` interfaces. The failure looked nothing like a dependency change. Surefire reported "ClassSelector resolution failed" during test discovery — the kind of error that points at classloaders or module-path issues. Claude found the actual `UnsatisfiedResolutionException` in a `.dumpstream` file, several redirects from the main test output. Once we saw it, the fix was obvious: align qhorus's implementations with the new interface and add `casehub-platform` as a test dependency in the modules that now needed `MockCurrentPrincipal @DefaultBean`.

The spec also described V19 as `ALTER TABLE qhorus_message...`. The actual table is `message`. The Flyway schema test caught it before the migration landed anywhere real — which is exactly why that test exists.

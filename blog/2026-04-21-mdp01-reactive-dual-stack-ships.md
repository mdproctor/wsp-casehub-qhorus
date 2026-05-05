---
layout: post
title: "The Reactive Dual-Stack Ships"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [reactive, dual-stack, quarkus, mutiny]
---

Before any reactive services could share logic with the blocking ones, `QhorusMcpTools` needed surgery. All 23 response records, seven entity-to-DTO mappers, and three validation helpers were buried inside a 1300-line class alongside the actual `@Tool` methods. The extraction pulled them into `QhorusMcpToolsBase` — an abstract class with no CDI annotations, no `@Tool`, no service injection.

One finding during that refactor: Java doesn't allow importing inherited nested types via the subclass name. The test files had imports like `import ...QhorusMcpTools.CheckResult` — valid when `CheckResult` was declared in `QhorusMcpTools`, invalid once it moved to `QhorusMcpToolsBase`. The subagent updated 35 import statements across 26 test files without being asked.

## Category A and Category B

Not all 39 MCP tools could be fully reactive. quarkus-mcp-server 1.11.1 supports `Uni<T>` return types natively, so the interface was clear. The implementation wasn't.

Twenty tools — simple CRUD via reactive services — became pure reactive chains. `listChannels()` ended up using `Uni.join().all(uniList).andFailFast()` to count messages per channel in parallel while preserving order, which is neater than the blocking version's GROUP BY query.

The other 19 tools needed blocking code: Panache entity statics, the `Thread.sleep` poll loop in `wait_for_reply`, the entire `send_message` flow with its rate limiter and LAST_WRITE enforcement. For these, the pattern was `@Blocking @Tool` delegating to a private `blockingXxx` helper:

```java
@Tool(name = "send_message", description = "...")
@Transactional
@Blocking
public Uni<MessageResult> sendMessage(/* @ToolArg params */) {
    return Uni.createFrom().item(() -> blockingSendMessage(/* params */));
}
```

The private helper matters: when `requestApproval` calls `blockingSendMessage` then `blockingWaitForReply`, it calls them directly without a CDI proxy. That avoids holding a database transaction open across a 300-second long-poll loop — a real problem if `@Transactional` had wrapped the whole chain through the CDI interceptor.

Code reviewers were active throughout. One caught a dead `blockingDataService` injection that was never used. Another flagged `@WithTransaction` on reactive JPA store methods as a "critical double-transaction" risk. It isn't — `@WithTransaction` uses REQUIRED propagation and joins the outer `Panache.withTransaction()` — but the Quarkus docs don't say this clearly, so the concern was reasonable. We added a garden entry.

## The Build-Time Constraint

Toggling between stacks uses `@IfBuildProperty` / `@UnlessBuildProperty` — Quarkus Arc annotations evaluated during compilation. Setting `quarkus.qhorus.reactive.enabled=true` activates the reactive stack. The constraint that emerged from this: test profiles can't override build-time annotations. `QuarkusTestProfile.getConfigOverrides()` sets runtime config overrides; `@IfBuildProperty` is baked in during the Maven build. Setting the property in a test profile has zero effect on which beans are compiled into the deployment.

## The Contract Test Trap

The reactive InMemory store runners inherit test scenarios from abstract contract bases and unwrap `Uni` via `.await().indefinitely()` — the assertion code is identical across both stacks. That part went cleanly.

The refactoring of the existing concrete test classes to extend the contract base didn't. Claude replaced each class with pure factory method delegation and dropped all the implementation-specific tests in the process — `scan_bySemantic`, `countByChannel`, `deleteAll`, capability management. Test count dropped from 92 to 62 with no compilation error and no test failure. The only signal was a smaller number.

Claude hadn't been told the concrete classes needed to keep their own tests alongside the inherited ones. The fix: concrete class = factory method implementations + supplemental `@Test` methods for store-specific behaviour not in the contract.

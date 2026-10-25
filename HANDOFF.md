# HANDOFF — casehub-qhorus

## Last Session (2026-10-25)

Cross-repo commit from agency#354 session: added `policyOverrides` (`Map<String, String>`) to `ProtocolContext` record. One-line additive change — `MessageService.dispatch()` passes `ch.policyOverrides()` when constructing the context. All 4 protocol test files updated. BUILD SUCCESS.

Commit: `feat: add policyOverrides to ProtocolContext` on main.

## Previous Session (2026-10-03)

Closed #467 (remove @DefaultBean from runtime-core) and epic #466 (Spring completeness). Branch `issue-467-remove-defaultbean-from-core` landed as c716cc5d on main.

**Changes:**
- Moved 5 @DefaultBean @Produces methods from runtime-core/DefaultBeans.java → runtime/RuntimeBeans.java
- Deleted runtime-core/DefaultBeans.java
- Removed quarkus-arc dep from runtime-core POM — module is now framework-neutral
- Added missing InboundNormaliser @ConditionalOnMissingBean to RuntimeAutoConfiguration (Spring)

## Previous Session (2026-09-29)

Closed parent#480 (Pattern 2 migration already done) and parent#495 (rest-spring added to reactor). Branch `issue-495-rest-spring-reactor` squash-merged to main and pushed.

**Pre-existing issues found:**
- Stale ledger-api in slot's .m2 — fixed by installing with `-Dmaven.repo.local`
- Test compilation errors from stale ledger record signatures (not session scope)
- runtime-spring remains excluded — 3 generated auto-config compile errors need separate fix

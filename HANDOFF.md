# CaseHub Qhorus — Session Handover
**Date:** 2026-05-15 — qhorus#141 shipped and closed

---

## What Was Done This Session

**qhorus#141** — Gate `quarkus-hibernate-reactive-panache` behind build-time flag.

Three approaches failed before the right one landed:
1. `ExcludedTypeBuildItem` + `Capabilities` — `ExcludedTypeBuildItem` doesn't gate JAX-RS scanners; `Capabilities` can't be injected in `BooleanSupplier` (evaluated before CapabilityBuildItems are produced). Claudony reported duplicate `/a2a` endpoint.
2. `Class.forName()` in BooleanSupplier — subagents added `@IfBuildProperty(quarkus.datasource.reactive)` (wrong property) and broke `A2AResource` by removing `@UnlessBuildProperty`.
3. `@Priority(1)` on reactive JPA stores — CDI ambiguity with `InMemoryReactive*Store @Priority(1)` from testing module.

**Fix shipped:** `@IfBuildProperty(quarkus.datasource.qhorus.reactive=true)` on all 23 reactive beans; `@UnlessBuildProperty` on 5 blocking REST/MCP beans; `<optional>true</optional>` on reactive Panache dep; `QhorusProcessor` = `feature()` only; `QhorusBuildConfig` deleted. 2478 tests, 0 failures.

**Artifacts:**
- ADR-0007: `docs/adr/0007-reactive-stack-activation.md`
- Protocol PP-20260514-f41258 corrected in parent repo (`universal/quarkus-optional-extension-dep.md`)
- 4 garden entries (GE-20260515-196055, acba9e, bccf6c, b8e86a)
- Claudony text prepared (see previous conversation)
- Installed to local m2

**Consumer cleanup issues filed:** claudony#112, devtown#25 (workaround removal). aml#13 already existed.
**Assessment issues filed:** engine#253, qhorus#152 (reactive dep assessment for other modules).

## Current State

- **Branch:** `main` — clean, fully pushed, installed to m2
- **Test count:** 2478 passing, 88 skipped (reactive JPA — Docker)
- **Untracked:** `docs/squash-plan-*.md` (stale, can delete)

## Immediate Next Steps

Priority order:
1. **#142** (bug) — Flyway V2 conflict with casehub-work when on same classpath. Fix: renumber one V2 migration per `flyway-version-range-allocation.md`. Check which module owns V2 and which needs to yield.
2. **#151** (bug) — `DELEGATED` → `"completed"` in A2A state mapping is wrong; fix `toA2AState(DELEGATED)` → `"working"`
3. **#132** (feat) — Delivery guarantees for registered backends (retry + dead-letter)

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-15-mdp01-what-excludedtype-actually-excludes.md` |
| ADR-0007 | `wksp/../qhorus/docs/adr/0007-reactive-stack-activation.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

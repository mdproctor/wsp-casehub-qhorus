# HANDOFF — casehub-qhorus

## Last Session (2026-11-23)

Completed the entire playbook-bridge cleanup queue (#536–#539). Four issues closed in one session:

- **#536** (on main) — Added 26 `@QuarkusTest` integration tests (CDI wiring, JPA store against H2, REST Assured HTTP round-trips) and 4 CDI-free recovery tests (past-due deadlines, partial expiry, empty history). Test infrastructure follows the connector-backend pattern: dual H2 datasources, InMemory store alternatives, entity package registration.
- **#537** — Replaced hand-rolled YAML line-splitting in `parseDeadlines()` with snakeyaml (transitive dep). Fixes 4-space indent, inline comments, nested keys. Validator delegates to shared `parseDeadlines()` instead of duplicating the logic. 4 edge case tests added.
- **#538** — Removed dead `tenancyId` field from `ProtocolDefinitionRequest` (unused since #535). All test call sites updated to single-arg constructor.
- **#539** — Renamed `put_defaultsTenancyId` to `put_usesCurrentPrincipalTenancyId`.

Blog entry: "Testing the Bridge from Outside" — covers the optional module integration test pattern and the YAML parser lesson.

## Queue State

Queue is empty — all items complete. #537, #538, #539 on stacked branches awaiting landing.

## Cross-Module

- casehubio/platform#922 — template-based message extraction pipeline. #921, #923, #925, #928, #929 delivered. #924 (integrate TemplateExtractor with governance runtime) depends on #922.

## References

- `specs/issue-528-playbook-protocol-bridge/2026-11-20-playbook-protocol-bridge-design.md`
- `plans/2026-11-20-playbook-protocol-bridge.md`
- `blog/2026-11-20-mdp01-the-bridge-that-closes-the-loop.md`
- `blog/2026-11-23-mdp01-three-gaps-one-pass.md`
- `blog/2026-11-23-mdp02-testing-the-bridge-from-outside.md`

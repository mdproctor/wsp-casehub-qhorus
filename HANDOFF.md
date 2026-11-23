# HANDOFF — casehub-qhorus

## Last Session (2026-11-23)

Completed the entire playbook-bridge cleanup queue (#536–#539) and started a new queue (#530, #527). Four issues closed, branch scaffolded for #530.

### Closed this session

- **#536** (on main) — 26 `@QuarkusTest` integration tests + 4 CDI-free recovery tests for playbook-bridge. Test infrastructure follows connector-backend pattern: dual H2 datasources, InMemory store alternatives, entity package registration.
- **#537** — Replaced hand-rolled YAML line-splitting with snakeyaml (transitive dep). Fixes 4-space indent, inline comments, nested keys. Validator delegates to shared `parseDeadlines()`.
- **#538** — Removed dead `tenancyId` field from `ProtocolDefinitionRequest`.
- **#539** — Renamed `put_defaultsTenancyId` to `put_usesCurrentPrincipalTenancyId`.

### Active work

**#530** — `policyOverrides` not visible to `ProtocolLifecycleHandler.onInit()` — set after `ChannelInitialisedEvent` fires. Branch scaffolded (`issue-530-policyoverrides-timing-race`), no code written.

**Root cause:** `ChannelService.create()` → `ChannelCreateHelper.createChannelOnly()` → `channelGateway.initChannel()` fires `ChannelInitialisedEvent` synchronously. But `ChannelProvisionHandler.provision()` calls `setPolicyOverrides()` only *after* `create()` returns. Any `ProtocolLifecycleHandler.onInit()` sees no overrides.

**Fix path:** Add `policyOverrides` to `ChannelCreateRequest` so they're set at creation time, before the init event fires. Touches central API — check all `ChannelCreateRequest.builder()` call sites.

## Queue State

- [active] casehubio/qhorus#530 — policyOverrides timing race
- [ ] casehubio/qhorus#527 — wire ApiKeyIdentityResolver to StaticCredentialStore
- [deferred] casehubio/qhorus#532 — ChannelCreateRequest integration for playbook definitions

## Cross-Module

- casehubio/platform#922 — template-based message extraction pipeline. #924 (integrate TemplateExtractor with governance runtime) depends on #922.

## References

- `blog/2026-11-23-mdp02-testing-the-bridge-from-outside.md`
- `blog/2026-11-23-mdp01-three-gaps-one-pass.md`
- `blog/2026-11-20-mdp01-the-bridge-that-closes-the-loop.md`

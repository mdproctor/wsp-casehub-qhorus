# HANDOFF — casehub-qhorus

## Last Session (2026-11-24)

Closed four issues (#540, #541, #527, #542), landed ops cross-repo fix, wrote blog entry. Queue empty.

### Closed this session

- **#540** — Removed redundant `setParticipationPolicy()` post-create call in `ChannelConfigBootstrap.processChannel()`. Same create-then-configure anti-pattern that #530 fixed for policyOverrides.
- **#541** — Cross-repo fix in `casehub-ops`: moved `policyOverrides` to the `ChannelCreateRequest` builder in `ChannelProvisionHandler.provision()`. Created and closed casehubio/casehub-ops#248.
- **#527** — Added `PlatformApiKeyIdentityResolver` in `runtime-core/identity/`. Delegates API key resolution to `StaticCredentialStore.lookup()`. Gated by `@IfBuildProperty casehub.qhorus.identity.platform-api-keys=true`. 6 CDI-free unit tests.
- **#542** — Added `@QuarkusTest` integration tests verifying CDI wiring of the `@IfBuildProperty` gate. Also registered `casehub.qhorus.identity.platform-api-keys` in `QhorusConfig.Identity`.

### Cross-repo work

- **casehub-ops** — committed `db9135c` on main (branch `issue-248-policyoverrides-on-builder`, stamped). Changed `ChannelProvisionHandler.provision()` and `StubChannelOps.create()`.

### Queue State

Empty — no open items queued.

## References

- `blog/2026-11-24-mdp01-name-the-pattern-find-the-copies.md`

# HANDOFF — casehub-qhorus

## Last Session (2026-11-24)

Closed three issues (#540, #541, #527), landed ops cross-repo fix, filed one follow-up (#542), wrote blog entry.

### Closed this session

- **#540** — Removed redundant `setParticipationPolicy()` post-create call in `ChannelConfigBootstrap.processChannel()`. Same create-then-configure anti-pattern that #530 fixed for policyOverrides. Landed as `5ac26474`.
- **#541** — Cross-repo fix in `casehub-ops`: moved `policyOverrides` to the `ChannelCreateRequest` builder in `ChannelProvisionHandler.provision()`. Landed as `db9135c` on ops main. Created and closed casehubio/casehub-ops#248.
- **#527** — Added `PlatformApiKeyIdentityResolver` in `runtime-core/identity/`. Delegates API key resolution to `StaticCredentialStore.lookup()` from `casehub-platform-api`. Gated by `@IfBuildProperty casehub.qhorus.identity.platform-api-keys=true`. Filters REVOKED and expired; ACTIVE and GRACE_PERIOD resolve. 6 CDI-free unit tests. Landed as `80b910f3`.

### Filed this session

- **#542** — Add `@QuarkusTest` integration tests for `PlatformApiKeyIdentityResolver` (CDI wiring, `@IfBuildProperty` gate verification). XS/Low.

### Queue State

- [x] casehubio/qhorus#540 — done
- [x] casehubio/qhorus#541 — done (cross-repo, ops main)
- [x] casehubio/qhorus#527 — done
- [ ] casehubio/qhorus#542 — integration test for PlatformApiKeyIdentityResolver

### Cross-repo work

- **casehub-ops** — committed `db9135c` on main (branch `issue-248-policyoverrides-on-builder`, stamped). Changed `ChannelProvisionHandler.provision()` and `StubChannelOps.create()` test helper.

## References

- `blog/2026-11-24-mdp01-name-the-pattern-find-the-copies.md`

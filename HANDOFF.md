# HANDOFF — casehub-qhorus

## Last Session (2026-11-23)

Closed #530 (policyOverrides timing race), filed two follow-up issues (#540, #541), wrote blog entry, published both repos. Scaffolded #540.

### Closed this session

- **#530** — Added `policyOverrides` to `ChannelCreateRequest` so they're set atomically at creation time, before `ChannelInitialisedEvent` fires. Also wired into `QhorusConfig.ChannelConfig` for YAML-declared channels with drift detection. Landed as `bed0f0d6`.

### Filed this session

- **#540** — Remove redundant `setParticipationPolicy()` post-create call in `ChannelConfigBootstrap` (same anti-pattern as #530)
- **#541** — ops `ChannelProvisionHandler` should use `policyOverrides` on builder (consumer follow-up to #530)

### Active work

**#540** — Remove redundant `setParticipationPolicy()` post-create call in `ChannelConfigBootstrap`. Branch scaffolded (`issue-540-redundant-setparticipationpolicy`), no code written.

**Root cause:** `processChannel()` lines 57-58 call `channelService.setParticipationPolicy()` after create, but `participationPolicy` is already on `ChannelCreateRequest` and set via the builder in `toRequest()` (line 94). The post-create call is redundant — same pattern that #530 fixed for policyOverrides.

**Fix:** Remove lines 57-58 from `ChannelConfigBootstrap.processChannel()`. Verify with existing tests.

### Queue anomaly

**#527 was accidentally marked done in the queue** when `plan_manager.py advance` was run incorrectly. #527 (wire ApiKeyIdentityResolver to StaticCredentialStore) has NOT been worked on. It needs to be re-queued — uncheck it in `.queue` and move it back to the queued section.

## Queue State

- [active] casehubio/qhorus#540 — remove redundant setParticipationPolicy post-create call
- [ ] casehubio/qhorus#541 — ops ChannelProvisionHandler should use policyOverrides on builder
- [done but not worked] casehubio/qhorus#527 — needs re-queuing

## References

- `blog/2026-11-23-mdp03-the-configuration-that-arrived-too-late.md`
- `blog/2026-11-23-mdp02-testing-the-bridge-from-outside.md`
- `blog/2026-11-23-mdp01-three-gaps-one-pass.md`

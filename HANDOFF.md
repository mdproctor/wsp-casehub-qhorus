# CaseHub Qhorus — Session Handover
**Date:** 2026-05-23 — #185–189 batch review findings + dispatch enforcement re-arch

---

## What Was Done This Session

Resolved all 5 review findings from #184. #185–187 and #189 were minor cleanup (test gaps, Javadoc, dead code, assertj version, Jackson config, UUID parsing). #188 was architectural: enforcement (paused, ACL, rate limit, LAST_WRITE, fanOut) moved from `QhorusMcpTools.sendMessage()` into `MessageService.dispatch()` — every caller now gets enforcement automatically. `AllowedWritersPolicy` extracted as a shared `@ApplicationScoped` bean with unified capability supplier (instance tags + synthetic role tag). `QhorusChannelBackend.post()` made a no-op; CDI cycle `MessageService↔ChannelGateway` resolves via Arc proxies. All 5 issues closed; branch `issue-185-review-findings` closed and pushed to `mdproctor/qhorus` main.

## Immediate Next Step

File a qhorus issue for Plan B (Category B `@Blocking` tools in `ReactiveQhorusMcpTools` → `Uni<T>`) before starting it, then pick it up or move to #132.

## Cross-Module

**We're blocking:**
- `casehubio/aml#30` — AML can proceed; #184+#188 both shipped. No further action from qhorus.

## What's Left

- **claudony#129** — replace local DTO copies with api types · S · Low _(skip for now — user said no cross-repo work)_
- **Plan B** — Category B reactive tool conversion · L · High _(file issue first)_
- **#191** — LAST_WRITE parentReplyCount=0 correctness · S · Low
- **#193** — ReactiveMessageService enforcement parity · M · Med _(deferred — service is @Disabled)_
- **#194** — fanOut ChannelRef name is UUID not channel name · S · Low
- **#195** — LAST_WRITE design trade-off decision needed · S · Low _(design question, needs decision)_
- **#196** — artefact_refs UUID error message inconsistency · XS · Low
- **parent#50** — PLATFORM.md boundary rules need dispatch() note · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item |
| Plan B | Category B reactive tool conversion | L | High | File issue first |
| #191 | LAST_WRITE parentReplyCount=0 | S | Low | Tracked, low priority |

## Notes

- Push workflow: `git push origin main` only (mdproctor/qhorus); promote to casehubio/qhorus manually
- `backup/pre-squash-main-20260522` local branch — safe to delete (confirmed clean)
- `remotes/origin/claude/agent-argument-graphs-DWlFm` — unknown remote branch, left untouched

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-23-mdp01-closing-the-dispatch-bypass.md` |
| Spec (#188) | `specs/2026-05-23-dispatch-enforcement-design.md` (workspace) / `docs/specs/` (project) |
| Protocol | `casehub/parent/docs/protocols/casehub/message-service-dispatch-enforcement-gate.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

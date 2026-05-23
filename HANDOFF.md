# CaseHub Qhorus — Session Handover
**Date:** 2026-05-23 — #192 deadline dispatch + S/XS fixes + squash

---

## What Was Done This Session

Closed branch `issue-192-deadline-dispatch`. `MessageDispatch` gained `Instant deadline`; `MessageService.dispatch()` now sets `message.deadline` for the first time. `ReactiveMessageService.send()` replaced with `dispatch(MessageDispatch) → Uni<DispatchResult>` (paused check added; enforcement parity deferred to #193). Claudony `postToChannel()` updated. Closed: #191 (parentReplyCount semantics), #194 (ChannelRef channel name fix), #195 (LAST_WRITE design decision), #196 (artefact UUID error messages). History squashed: 43 commits → 10; pushed to mdproctor/qhorus and casehubio/qhorus. Local build green, CI green.

## Immediate Next Step

Start #132 (delivery guarantees for backends — retry + dead-letter). File issue first, then `work-start`.

## What's Left

- **claudony#135** — add `deadline` + `correlationId` as first-class params to `postToChannel()` SPI · S · Low _(Claudony work; qhorus ready)_
- **#193** — ReactiveMessageService full enforcement parity (ACL, rate limit, type policy, LAST_WRITE, ledger, fanOut) · M · Med _(deferred — service @Disabled)_
- **#198** — Two deferred review items: double channel-read in `sendHumanMessage()`, reactive deadline test · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #132 | Delivery guarantees for backends (retry + dead-letter) | L | High | Main feature item; file issue first |
| #193 | ReactiveMessageService enforcement parity | M | Med | Deferred; needs reactive versions of enforcement beans |

## Notes

- Push workflow: `git push origin main` only (mdproctor/qhorus); casehubio already updated via upstream push
- `remotes/origin/claude/agent-argument-graphs-DWlFm` — unknown remote branch, left untouched

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-23-mdp02-four-fixes-three-not-bugs.md` |
| Spec (#192) | `specs/2026-05-23-deadline-dispatch-design.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

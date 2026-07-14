# CaseHub Qhorus — Session Handover

**Date:** 2026-07-14 — Branch `issue-345-webhook-persistent-reg` closed. #345 implemented and pushed.

---

## Immediate Next Step

Pick new work. The What's Next table has two remaining issues from #163 follow-ups (#346 WebSocket catch-up, #165 SmallRye bridge). No blocking cross-module work.

## What Was Done

Implemented persistent webhook registrations (#345). JPA-backed `WebhookRegistrationEntity` and `WebhookRegistrationStore` with tenant-scoped queries. `WebhookRegistry` coordinates in-memory cache with JPA store — startup reload, write-through on register/deregister. `secretRef` replaces plaintext `secret` — resolved via `CredentialResolver` at POST time (fail-closed on resolution failure). Added `ChannelClosedEvent` CDI event (mirrors `ChannelInitialisedEvent`) for channel deletion cleanup. Global hooks keyed by `tenancyId` to prevent cross-tenant leakage. V35 migration. Design review caught 16 issues (all resolved). Cross-repo: added `SIGNING_SECRET` to `CredentialPropertyKeys` in `casehub-platform-api`.

## Cross-Module

**We're blocking:**
- `connectors` — needs Space API for space-aware channel grouping (connectors#67)
- `engine` — needs Space for normative channel layout integration
- `blocks` — needs all for end-to-end integration (blocks#49)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #346 | WebSocket catch-up mechanism for reconnecting clients | M | Med | Clients miss messages during disconnect |
| #165 | SmallRye Reactive Messaging bridge for MessageObserver | M | High | Alternative to explicit transports — lower priority |

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/issue-345-webhook-persistent-reg/2026-07-13-webhook-persistent-registrations-design.md` |
| Landed commit | `526d07e8` on main |
| Design review | `~/adr/casehub-qhorus/webhook-persistent-registrations-20260714-001642/tracker.md` |

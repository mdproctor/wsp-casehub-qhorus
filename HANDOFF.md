# CaseHub Qhorus — Session Handover
**Date:** 2026-06-29 — #310 closed (isActive sweep), #132 closed (delivery guarantee for channel backends)

---

## Immediate Next Step

Main is clean. Both remotes at `040e1db`. Two issues closed this session (#310, #132). Three follow-up issues filed (#312, #313, openclaw#57).

Cross-repo follow-up still pending from prior session: `MessageReceivedEvent` constructor changed (added `Instant occurredAt`). Claudony (3 test sites) and engine (7 test sites) will fail at next compile — mechanical fix (`Instant.now()` as 7th arg).

Next candidates:

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #312 | Delivery metrics — Micrometer counters/gauges for pump | S | Low | Follow-up from #132 |
| #169 | Extract persistence-memory/ module from testing/ | M | Low | Standalone |
| ops#14 | Enrich ChannelDriftChecker — full field comparison, tenancy fix | S | Low | Cross-repo (casehub-ops) |
| openclaw#57 | Override deliveryGuarantee → AT_LEAST_ONCE on OpenClawChannelBackend | XS | Low | Propagation from #132 |
| #313 | AT_LEAST_ONCE delivery for LAST_WRITE channel semantics | S | Med | Limitation from #132 |

## What Was Done This Session

**isActive sweep (#310):** Replaced 11 `!isTerminal()` call sites with `isActive()`. Added 2 resolveToken direct tests for SlackChannelBackend.

**Delivery guarantee (#132):** L/High — cursor-based delivery pump for AT_LEAST_ONCE backends. Adversarial design review (24 issues, all resolved, $27). Subagent-driven development (7 tasks). Key architecture: message store IS the durable outbox; per-backend cursors track delivery; event-driven pump (post-commit TSR signal) + 30s reconciler backup; BEST_EFFORT backends keep fire-and-forget with zero overhead. SlackChannelBackend and ConnectorChannelBackend declare AT_LEAST_ONCE.

## References

| What | Path |
|------|------|
| Design spec (adversarial-reviewed) | `docs/specs/2026-06-29-delivery-guarantee-design.md` |
| Blog entry | `blog/2026-06-29-mdp01-the-log-was-already-there.md` |
| Garden entry | `GE-20260629-bb1440` — post-commit TSR signaling technique |
| Previous session | `git show HEAD~1:HANDOFF.md` |

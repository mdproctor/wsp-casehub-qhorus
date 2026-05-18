# CaseHub Qhorus — Session Handover
**Date:** 2026-05-19 — #153, #154, #142 shipped; all three epics closed and merged to main

---

## What Was Done This Session

**qhorus#154** — `correlationId` threading through inbound human message path.
`NormalisedMessage` expanded from 3 to 7 fields (complete domain representation).
`ChannelGateway.receiveHumanMessage()` now a clean 1:1 mapping. `ClinicalInboundNormaliser`
passes `correlationId` through — commitment auto-fulfillment unblocked for clinical.

**qhorus#153** — `MessageObserver` SPI + `InProcessMessageBus` CDI default.
CDI-event-only design rejected in brainstorming: qhorus is a distributed mesh,
CDI events don't cross JVM boundaries. `MessageObserver` is transport-agnostic with
`Scope { LOCAL, CLUSTER }`. `InProcessMessageBus` (`@DefaultBean`) fires CDI events
for embedded harnesses; Kafka/WebSocket implementations are future additive beans.
Architecture doc at `docs/messaging-architecture.md`. Multi-node fleet gap documented
as qhorus#162. `fireAsync()` pre-commit race documented; `MessageReceivedEvent`
self-contained to avoid DB reads in observers (tracked qhorus#166 for JTA fix).

**qhorus#142** — Flyway migration path (`db/migration/qhorus/`) was on the epic branch
but never merged to main. Found by branch hygiene scan. Fixed. Installed artifact updated.

**Artifacts:**
- Specs: `docs/specs/` — all three epics promoted
- Blog: `blog/2026-05-19-mdp01-the-mesh-beneath-the-event.md`
- Garden: 5 new entries + 1 revision (tools domain)
- Protocols: epic close requires code merge, EPIC-CLOSED deletion date,
  MessageObserver `@ApplicationScoped` (parent repo committed)
- Issues filed: qhorus#158–#167, clinical#16

## Current State

- **Both repos:** `main` — all three epics merged
- **Epic branches retained:** epic-142 (delete after 2026-06-02), epic-153, epic-154
  (all have `EPIC-CLOSED.md`)
- **Local .m2:** up to date from current main
- **Tests:** 27 unit tests passing; `@QuarkusTest` CDI env issue pre-existing

## Immediate Next Steps

1. **#132** — Delivery guarantees for registered backends (retry + dead-letter) — main feature item
2. **Cross-repo branch review** — user to coordinate with work-claude on casehub-work
   (hygiene done this session by mistake; watch: `WorkItemExcludedUsersTest` after `-X ours`)
3. **clinical#16** — PiResponseListener workaround removal — now unblocked
4. **claudony #131** — claudony can close; qhorus side complete

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-19-mdp01-the-mesh-beneath-the-event.md` |
| Architecture doc | `docs/messaging-architecture.md` (SVG diagrams, Claudony gap included) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |

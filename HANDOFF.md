# CaseHub Qhorus — Session Handover

**Date:** 2026-07-09 — #197 closed (OTel trace instrumentation).

---

## Immediate Next Step

Main is clean. Both remotes at `18d3c946`. Issue #197 closed.

Cross-repo follow-up issues still open:
- claudony#169 — update OutboundMessage construction (correlationId UUID→String)
- drafthouse#102 — remove redundant .toString() on correlationId

## What Was Done This Session

**OTel trace instrumentation (#197):** Full OpenTelemetry span instrumentation across 9 Qhorus services (4 blocking + 4 reactive + ChannelGateway). Design spec → adversarial review (3 rounds, 11 issues, $12.54) → 7-task SDD implementation. Added `QhorusTracingConfig` (6 config switches), dispatch/commitment/fanout/delivery/ledger-write spans with cross-request span links. `Instance<Tracer>` for optional OTel dependency. Protocol PP-20260709-7b9c1b (Mutiny operator ordering). Two garden entries: GE-20260709-16094e (operator ordering gotcha), GE-20260709-520b0b (Instance<T> technique).

## References

| What | Path |
|------|------|
| Design spec | `docs/specs/2026-07-07-otel-trace-instrumentation-design.md` |
| Protocol | `docs/protocols/casehub/reactive-otel-span-operator-ordering.md` |
| Design review | `~/adr/casehub-qhorus/otel-trace-instrumentation-20260707-205123/` |
| Previous session | `git show HEAD~1:HANDOFF.md` |

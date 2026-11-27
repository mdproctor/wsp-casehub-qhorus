# HANDOFF — casehub-qhorus

## Last Session (2026-11-27)

Two issues progressed. #545 (e2e template match proof) — 23 tests proving NLP extraction, advisory flow, schema publication, and bidirectional generation work from qhorus's playbook-bridge classpath. Fixed the `buildVars()` gap: `ProtocolContext` now carries `incomingContent` (nullable), populated by `MessageService.dispatch()`, flowing to playbook vars as `${incomingContent}`. No new qhorus module needed — the existing dependency chain is sufficient.

#546 (template fields in ledger entries) started — branch created, design decision made: template match metadata flows via `DispatchAdvisory.evidence()` map (Option A — no new types). Migration V2005.

Platform issues filed: #1010 (persistent store for learned variations), #1011 (standalone YAML corpus loader). Platform #1002 closed — resolved by #961.

## Immediate Next Step

Implement #546. Read `MessageLedgerEntry` (runtime-core/ledger/), `LedgerWriteService.record()` (runtime/ledger/), and the advisory pipeline in `MessageService.dispatch()`. Add 4 nullable columns to `message_ledger_entry` (V2005). Thread match metadata from advisory evidence map into ledger entry fields.

## References

- `playbook-bridge/src/test/java/.../TemplateMatchIntegrationTest.java` — e2e proof (23 tests)
- `api/src/main/java/.../ProtocolContext.java` — `incomingContent` field added
- `runtime-core/src/main/java/.../MessageService.java:372` — protocol evaluation construction site
- `runtime-core/src/main/java/.../MessageLedgerEntry.java` — ledger entity to extend
- `runtime/src/main/java/.../LedgerWriteService.java` — record() to modify

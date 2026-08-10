# Trust Model: Obligor-Targeted Trust Attribution — Decisions

## D1: Repo scope

**Choice:** Qhorus-only — dual attestation writes in LedgerWriteService
**Alternatives:**
- Cross-repo (obligorId field on LedgerAttestation) — domain-specific field leaks into domain-agnostic model; significant query pipeline changes in casehub-ledger
- Qhorus + SPI — decoupled via SPI interface; more complex for marginal benefit
**Rationale:** Writing a second attestation on the terminal entry uses existing ledger model and queries unchanged. Zero casehub-ledger changes needed.
**Trade-offs:** Doubles attestation writes per terminal message. Acceptable — attestations are lightweight inserts.
**Exploration:** quick → revised after decision review R1-01, R1-02
**Status:** revised

## D2: Trust attribution model

**Choice:** Dual attestation — write separate attestations on COMMAND entry (requester) and terminal entry (obligor)
**Alternatives:**
- Dual attribution from single attestation — same verdict for both actors conflates delivery quality with delegation quality (R1-01)
- Obligor-only — loses "quality of delegation" signal for requesters
- Separate verdicts from single attestation — complexity without clear benefit
**Rationale:** Each attestation has a clear single target via the entry's actorId. Existing findAttestationsByActorId queries work unmodified for both actors. Requester trust preserved (COMMAND attestation unchanged). Obligor trust added (terminal entry attestation, actorId = obligor).
**Trade-offs:** Both attestations currently use the same verdict (derived from terminal message type). Future work could differentiate verdicts per actor.
**Exploration:** quick → revised after decision review R1-01
**Depends on:** D1
**Status:** revised

## D3: Obligor identity resolution

**Choice:** Terminal entry's actorId (already set by LedgerWriteService.record())
**Alternatives:**
- Commitment obligor — query CommitmentStore; adds a query inside REQUIRES_NEW with stale-state risk
- Terminal message sender as obligorId field — superseded by D2 revision (no obligorId field needed)
**Rationale:** The terminal entry's actorId = resolvedActorId = terminal message sender. This IS the obligor. No additional resolution needed — existing entry creation already resolves it.
**Trade-offs:** In delegation chains, the terminal sender is the final delegate. This is correct — the delegate did the work.
**Exploration:** quick → simplified by D2 revision
**Depends on:** D2
**Status:** revised

## D4: Backward compatibility for existing attestations

**Choice:** No migration needed — existing attestations on COMMAND entries are unchanged
**Alternatives:**
- Backfill — write terminal-entry attestations for historical data from commitment history
**Rationale:** Existing COMMAND attestations continue to feed requester trust (current behavior). New terminal-entry attestations begin feeding obligor trust going forward. Zero migration.
**Trade-offs:** Historical obligor trust is not retroactively computed. Acceptable — trust is forward-looking.
**Exploration:** quick
**Status:** captured

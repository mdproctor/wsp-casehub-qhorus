# CaseHub Qhorus — Session Handover
**Date:** 2026-06-23 — #295 normative benchmark (all four milestones shipped)

---

## Immediate Next Step

Main is clean. Both remotes at `0883284`. No open issues remain from the benchmark epic.

Next candidates:

| # | Description | Scale | Complexity | Blocked by |
|---|-------------|-------|------------|------------|
| #303 | Extract `EvidentialChecker` to casehub-qhorus runtime as publishable dependency | S | Low | — |
| #304 | Extend `CommitmentAttestationPolicy` SPI with `CommitmentContext` (corrId, channelId) | S | Med | #303 |
| #305 | Fix RESPONSE-on-COMMAND attestation gap — no trust signal fires today | S | Med | #304 |
| #233 | Complete ARC42STORIES.MD | L | High | — |
| #218 | Consolidate `ChannelService.create()` overloads | M | Med | — |
| #287 | `casehub-qhorus-desiredstate` NodeDriftChecker bridge | — | — | — |

**#303–305 are sequentially dependent** — do them in order. They unblock casehub-devtown#13 (trust-threshold evidential attestation). The RESPONSE gap (#305) is a correctness fix; agents sending RESPONSE on COMMAND obligations currently escape with no FLAGGED attestation.

## What Was Done This Session

**Normative benchmark #295 — all four milestones:**

- **#296 Zone 1:** `UnstructuredWorkerAgent` (COMPLETED:/CANNOT_COMPLETE: format) + `Zone1UnstructuredBaselineTest` (V1/V2/V3). 70–80% cheating on V2/V3 at temperature=0.1.
- **#297 Zone 2:** `Zone2NormativeChannelTest` — typed channel with COMMAND/response cycle. 0% false DONE — model sends RESPONSE instead of DONE; obligation technically closes (CommitmentState.FULFILLED) but with wrong type.
- **#298 Zone 3:** `EvidentialChecker` (I_df + I_ec checks, type-based not state-based) + `Zone3EvidentialCheckerTest` (9 tests, 2s) + `NormativeBenchmarkDemoTest` (three-act narrative, 20s) + README.
- **#299 Multi-model:** `Zone1Zone2Zone3Jlama1BTest` (all zones combined, comparison table) + Ollama/Claude stubs with activation instructions.

**Key finding:** Zone 3 must check response TYPE (DONE/FAILURE/DECLINE), not CommitmentStore state. RESPONSE sent with a COMMAND's corrId fulfills the commitment (FULFILLED), so `state == OPEN` never fires. Protocol PP-20260623-fd69f3 captures this.

**Protocol added:** PP-20260623-fd69f3 — command-obligation-verification-type-check

**Garden entries:** GE-20260623-ef0e7c (QUERY hard-blocks on typed channel), GE-20260623-92964b (RESPONSE fulfills commitment)

## References

| What | Path |
|------|------|
| Benchmark spec | `docs/specs/2026-06-22-normative-benchmark-design.md` (workspace) |
| Run the demo | `mvn test -Pwith-llm-examples -Dtest=NormativeBenchmarkDemoTest -f examples/agent-communication/pom.xml` |
| Previous session | `git show HEAD~1:HANDOFF.md` |

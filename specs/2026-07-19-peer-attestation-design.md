# Peer Attestation — Multi-Agent Verification on Ledger Entries

**Issue:** casehubio/qhorus#356
**Parent:** #351 (Verification & Trust)
**Date:** 2026-07-19

## Problem

CommitmentAttestationPolicy generates a single attestation per terminal message
(SOUND/FLAGGED via EvidentialChecker). There is no mechanism for Agent B to
formally verify Agent A's work — peer review exists as a concept but not as a
ledger-level record. Trust scoring depends on self-reported completion. Without
peer attestation, the attestation policy can't distinguish quality, only formal
correctness.

21% of multi-agent failures are verification gaps (MAST taxonomy, NeurIPS 2025).

## Design Principles

1. **`attest()` is the root primitive.** Everything else is orchestration
   convenience layered on top.
2. **No new entities.** LedgerAttestation already supports N attestations per
   entry, ENDORSED/CHALLENGED verdicts already exist unused, attestorRole
   field exists but is never populated.
3. **No new SPI.** CommitmentAttestationPolicy is for policy attestation. Peer
   attestation bypasses it — writes directly via PeerAttestationWriter.
4. **Trust integration is mechanical.** Peer attestations target the same
   COMMAND entry that policy attestations use. The Bayesian Beta model aggregates
   all attestations on an entry, weighted by `recencyWeight × confidence`. Trust
   attribution follows the existing pattern: the COMMAND sender's score is affected
   (same actor whose trust is updated by policy SOUND/FLAGGED). Obligor-targeted
   trust attribution is a separate concern (qhorus#373).

## Architecture

### Layer 0 — Primitive: PeerAttestationWriter

Package-private `@ApplicationScoped` in `runtime/ledger/`.

```java
void write(UUID ledgerEntryId, AttestationVerdict verdict,
           String evidence, String attestorId, String tenancyId)
```

Validation:
- Entry exists (`ledger.findEntryById()`)
- Entry is a COMMAND or HANDOFF (same guard as LedgerWriteService.writeAttestation)
- Self-attestation guard: `attestorId != entry.actorId`. Rejects attestations
  where the attestor produced the entry being attested. Prevents trust score
  self-inflation.
- Verdict is ENDORSED or CHALLENGED — rejects SOUND/FLAGGED (policy-only)
- Attestor is a registered instance (`instanceStore.findByInstanceId(attestorId)`).
  If the attestor is not registered, reject with IllegalArgumentException — peer
  attestation requires an identifiable agent, not an anonymous caller. The MCP
  tool resolves attestorId from `currentPrincipal.actorId()`.

Sets:
- `subjectId` from `entry.subjectId` (same as LedgerWriteService.writeAttestation)
- `attestorRole = "peer-reviewer"`
- `attestorType = ActorType.AGENT`
- `confidence` from config (`peer-endorsed-confidence` or `peer-challenged-confidence`).
  Config-only — agents cannot override confidence to prevent gaming
  (e.g. CHALLENGE with confidence=1.0 disproportionately damaging trust).
- `capabilityTag` from the entry's content (same extraction as StoredCommitmentAttestationPolicy)

Calls `ledger.saveAttestation()`.

### Layer 1 — API: MCP Tools

**`attest(entry_id, verdict, evidence)`**
- Direct attestation on a COMMAND or HANDOFF entry. Any registered agent, any time.
- Resolves ledger entry UUID, parses verdict string to enum.
- Calls PeerAttestationWriter.write().
- Returns: attestation ID, entry ID, verdict, attestor ID.

**`request_peer_review(entry_id, reviewer_ids?, channel?)`**
- Resolves the ledger entry, extracts COMMAND content and channel.
- Resolves `completion_content`: looks up the most recent terminal entry (DONE
  or FAILURE) for the COMMAND's correlationId via the ledger. If found, extracts
  its content. If not found (COMMAND not yet fulfilled), `completion_content` is
  null — the review QUERY contains only the COMMAND content.
- Calls ReviewerResolver.resolve() with explicit reviewer_ids and channel.
- For each resolved reviewer, dispatches a QUERY via messageService.dispatch():
  - Content: `{"peer_review": {"ledger_entry_id": "...", "original_command": "...", "completion_content": "...or null"}}`
  - Target: reviewer instance ID
  - Channel: same channel as original COMMAND (or explicit override)
  - Sender: the calling agent (`currentPrincipal.actorId()`)
  - Deadline: from `casehub.qhorus.commitment.default-query-deadline`
- Returns: list of review QUERYs sent (reviewer IDs + correlation IDs).
- Empty list if no reviewers resolved → advisory message.

**`list_attestations(entry_id)`**
- Calls `ledger.findAttestationsByEntryId()`.
- Returns all attestations with verdict, attestor, role (`"peer-reviewer"` vs
  null for policy), evidence, confidence, timestamp.

**Channel reviewer config:**
- `set_channel_reviewers(channel, reviewer_ids)` — same pattern as set_allowed_writers.
- `create_channel` gains optional `reviewer_ids` parameter.

### Layer 2 — Coordination

**ReviewerResolver** — package-private `@ApplicationScoped` in `runtime/ledger/`.

```java
List<String> resolve(UUID channelId, List<String> explicitReviewerIds, String tenancyId)
```

Fallback chain (first non-empty wins):
1. Explicit — `explicitReviewerIds` non-empty → return those
2. Channel config — `channel.reviewerInstances` non-empty → return those
3. Capability routing — `instanceService.findByCapability("peer-reviewer")` → return IDs
4. CDI event — fire `PeerReviewRequestedEvent(ledgerEntryId, channelId, tenancyId)`.
   Return empty list. External consumer (blocks/routing) observes and handles.

**PeerReviewRequestedEvent** — record in `api/spi/` alongside
CommitmentAttestationPolicy. The `api/gateway/` package is for channel lifecycle
events (ChannelInitialisedEvent, ChannelClosedEvent); review workflow events belong
in the SPI package where attestation concerns already live.

**PeerReviewAutoTrigger** — `@ApplicationScoped` MessageObserver, scope LOCAL.
Gated with `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled",
stringValue = "false", enableIfMissing = true)` — blocking-only for v1.

Fires after DONE messages:
1. Filter: `event.messageType() != DONE` → return
2. `event.correlationId() == null` → return (correlationId is a record component
   on MessageReceivedEvent — no message store lookup needed)
3. Find entry via `messageRepo.findEarliestWithSubjectByCorrelationId(
   event.correlationId(), event.tenancyId())`
4. Message type guard: `entry.messageType` is not `COMMAND` or `HANDOFF` → return.
   Prevents recursive loop when review QUERYs receive DONEs with a correlationId
   that resolves to a non-COMMAND entry.
5. Call `ReviewerResolver.resolve(entry.channelId, List.of(), tenancyId)` — uses
   the COMMAND entry's channel (where work was requested), not `event.channelId()`
   (where DONE arrived). In cross-channel HANDOFF flows these may differ.
6. Reviewers found → dispatch review QUERYs with structured `peer_review` content.
   Sender: `entry.actorId` (the COMMAND sender/requester — they requested the work,
   are already on the channel's `allowedWriters` ACL, and receive
   `CommitmentExpiredEvent` if the review times out).
   Each QUERY gets its own UUID correlationId (independent obligations — shared
   correlationIds would create commitment conflicts or fulfill the original
   COMMAND's commitment). `completion_content` = `event.content()` from the
   triggering DONE.
7. No reviewers → no-op (channel has no review configured)

Review QUERY content:
```json
{
  "peer_review": {
    "ledger_entry_id": "uuid-of-command-entry",
    "original_command": "the command content",
    "completion_content": "the DONE content"
  }
}
```

**PeerReviewResponseHandler** — `@ApplicationScoped` MessageObserver, scope LOCAL.
Gated with `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled",
stringValue = "false", enableIfMissing = true)` — blocking-only for v1.

Fires on RESPONSE messages:
1. Filter: `event.messageType() != RESPONSE` → return
2. Try to parse `event.content()` — contains `"peer_review_response"` key? No → return
3. Extract from the response JSON:
   ```json
   {
     "peer_review_response": {
       "ledger_entry_id": "uuid-of-command-entry",
       "verdict": "ENDORSED",
       "evidence": "Output contains required fields..."
     }
   }
   ```
4. Parsed → call PeerAttestationWriter.write() with extracted ledger_entry_id,
   verdict, and evidence. Confidence from config (not overridable).
5. Unparseable → WARN log. Reviewer uses `attest()` tool explicitly.

Content-based identification eliminates `inReplyTo` tracing and avoids the
pre-commit message store visibility issue documented in `MessageObserver`
Javadoc. The response echoes `ledger_entry_id` from the QUERY's `peer_review`
payload — PeerAttestationWriter validates the entry exists regardless.

Zero message store queries. Zero transaction visibility concerns.

## Data Model Changes

### Channel entity

New nullable field: `reviewerInstances` (TEXT, comma-separated instance IDs).
Same pattern as `allowedWriters` and `adminInstances`.

### Flyway migration V37

```sql
ALTER TABLE channel ADD COLUMN reviewer_instances TEXT;
```

### Config additions

In `QhorusConfig.Attestation` (extends existing `casehub.qhorus.attestation.*`):
- `peer-endorsed-confidence` — double, default 0.4
- `peer-challenged-confidence` — double, default 0.5

Conservative defaults: peer attestation nudges trust scores but doesn't dominate.
CHALLENGED slightly higher than ENDORSED — negative evidence is more informative.

### Deadline

Review QUERYs use the existing `casehub.qhorus.commitment.default-query-deadline`
config. No new deadline mechanism. `CommitmentService.expireOverdue()` fires
`CommitmentExpiredEvent` → notification bridge alerts the requester.

## Trust Integration

Zero trust model changes needed. Peer attestations are written on the COMMAND
entry — the same entry that policy attestations (SOUND/FLAGGED) target. The
Bayesian Beta model aggregates all attestations per entry, weighted by
`recencyWeight × confidence`.

**Trust attribution:** `ComputedTrustScoreSource.computeFresh()` computes trust
for an actor from entries where `entry.actorId = actorId` and attestations on
those entries. Since attestations go on the COMMAND entry, the COMMAND sender's
(requester's) trust score is affected — both by policy attestation and by peer
attestation. This is the existing pattern: `QhorusLedgerEntryRepository.saveAttestation()`
fires `AttestationRecordedEvent(entry.actorId, ...)`, invalidating the COMMAND
sender's trust cache.

A COMMAND that receives:
- Policy SOUND (0.7) + Peer ENDORSED (0.4) → two positive observations, peer
  adds a quality-verified signal beyond formal correctness
- Policy SOUND (0.7) + Peer CHALLENGED (0.5) → mixed signal, exactly what the
  model handles — formally correct but quality-flagged

**Limitation — obligor trust attribution:** The obligor (agent who fulfills the
COMMAND and sends DONE) does not have their trust score directly affected by
attestations on the COMMAND entry. This is a pre-existing property of the trust
model, not specific to peer attestation — policy SOUND/FLAGGED has the same
attribution. Obligor-targeted trust attribution is tracked in qhorus#373.

**Limitation — attestor credibility:** The current model does not track attestor
credibility. An agent that consistently ENDORSEs poor work suffers no trust
penalty — the Bayesian Beta model scores actors by attestations on their entries,
not by attestations they produce. Attestor credibility tracking is tracked in
qhorus#371.

## What's NOT Needed

- No new entities — LedgerAttestation already exists with all required fields
- No new stores — LedgerEntryRepository already has attestation CRUD
- No new SPI — CommitmentAttestationPolicy is for policy attestation only
- No new message types — uses existing QUERY/RESPONSE
- No trust model changes — Bayesian Beta model aggregates automatically
- No in-memory state tracking — review QUERYs identified by content convention

## Testing Strategy

| Component | Test type | Key assertions |
|-----------|-----------|----------------|
| PeerAttestationWriter | CDI-free unit | Validates entry exists, rejects SOUND/FLAGGED, sets attestorRole, calls saveAttestation |
| ReviewerResolver | CDI-free unit | Fallback chain order, empty at each layer, CDI event fired when all empty |
| PeerReviewAutoTrigger | CDI-free unit | Only fires for DONE, calls resolver, sends QUERYs with structured content, no-op when no reviewers |
| PeerReviewResponseHandler | CDI-free unit | Only fires for RESPONSE, parses content-based peer_review_response JSON, falls back silently, calls writer with config confidence |
| Integration | @QuarkusTest | Full round-trip: COMMAND → DONE → auto-review QUERY → structured RESPONSE → attestation written. list_attestations returns both policy and peer |
| MCP tools | @QuarkusTest | attest() writes ENDORSED/CHALLENGED. request_peer_review() sends QUERYs. list_attestations() returns all |
| Channel config | @QuarkusTest | setReviewerInstances() persists and reads back. create_channel with reviewer_ids |

CDI-free unit tests use mock LedgerEntryRepository, InstanceStore, MessageStore.
tracingConfig set to disabled-tracing impl per convention.

Integration tests use `QuarkusTransaction.requiringNew()` for dispatch (observer
transaction discipline). Channel created with reviewerInstances. Reviewer
instance registered with `"peer-reviewer"` capability.

Blocking-only for v1. Both PeerReviewAutoTrigger and PeerReviewResponseHandler are
gated with `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled",
stringValue = "false", enableIfMissing = true)`. Reactive parity tracked in qhorus#372.

## Research Basis

- [MAST failure taxonomy](https://openreview.net/forum?id=wM521FqPvI) — 21%
  verification gaps
- [Context Window Management (Zylos, 2026)](https://zylos.ai/research/2026-03-31-context-window-management-session-lifecycle-long-running-agents/)
  — 65% enterprise AI failures
- Issue #356 open question: structured verification questions over open-ended
  review → addressed by structured response format with fallback

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
4. **Trust integration is free.** The Bayesian Beta model already aggregates
   all attestations per actor, weighted by `recencyWeight × confidence`.

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
- Verdict is ENDORSED or CHALLENGED — rejects SOUND/FLAGGED (policy-only)
- Attestor is a registered instance (`instanceStore.find()`). If the attestor
  is not registered, reject with IllegalArgumentException — peer attestation
  requires an identifiable agent, not an anonymous caller. The MCP tool resolves
  attestorId from `currentPrincipal.actorId()`.

Sets:
- `attestorRole = "peer-reviewer"`
- `attestorType = ActorType.AGENT`
- `confidence` from config (`peer-endorsed-confidence` or `peer-challenged-confidence`)
- `capabilityTag` from the entry's content (same extraction as StoredCommitmentAttestationPolicy)

Calls `ledger.saveAttestation()`.

### Layer 1 — API: MCP Tools

**`attest(entry_id, verdict, evidence)`**
- Direct attestation write. Any agent, any entry, any time.
- Resolves ledger entry UUID, parses verdict string to enum.
- Calls PeerAttestationWriter.write().
- Returns: attestation ID, entry ID, verdict, attestor ID.

**`request_peer_review(entry_id, reviewer_ids?, channel?)`**
- Resolves the ledger entry, extracts COMMAND content and channel.
- Calls ReviewerResolver.resolve() with explicit reviewer_ids and channel.
- For each resolved reviewer, dispatches a QUERY via messageService.dispatch():
  - Content: `{"peer_review": {"ledger_entry_id": "...", "original_command": "...", "completion_content": "..."}}`
  - Target: reviewer instance ID
  - Channel: same channel as original COMMAND (or explicit override)
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
3. Capability routing — `instanceStore.findByCapability("peer-reviewer")` → return IDs
4. CDI event — fire `PeerReviewRequestedEvent(ledgerEntryId, channelId, tenancyId)`.
   Return empty list. External consumer (blocks/routing) observes and handles.

**PeerReviewRequestedEvent** — record in `api/gateway/` alongside
ChannelInitialisedEvent and ChannelClosedEvent. The escape hatch for intelligent
routing without Qhorus knowing about routing logic.

**PeerReviewAutoTrigger** — `@ApplicationScoped` MessageObserver, scope LOCAL.

Fires after DONE messages:
1. Filter: `event.messageType() != DONE` → return
2. Look up message by `event.messageId()` to get correlationId
3. Find COMMAND ledger entry via `messageRepo.findEarliestWithSubjectByCorrelationId()`
4. Call `ReviewerResolver.resolve(channelId, List.of(), tenancyId)`
5. Reviewers found → dispatch review QUERYs with structured `peer_review` content
6. No reviewers → no-op (channel has no review configured)

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

Fires on RESPONSE messages:
1. Filter: `event.messageType() != RESPONSE || event.messageId() == null` → return
2. Look up RESPONSE message → get `inReplyTo`
3. `inReplyTo == null` → return
4. Look up original QUERY by `inReplyTo` (PK lookup)
5. Parse QUERY content — contains `"peer_review"` key? No → return
6. Extract `ledger_entry_id` from the QUERY's `peer_review` object
7. Try to parse RESPONSE content as structured attestation:
   ```json
   {
     "peer_review_response": {
       "verdict": "ENDORSED",
       "evidence": "Output contains required fields...",
       "confidence": 0.8
     }
   }
   ```
8. Parsed → call PeerAttestationWriter.write(). Confidence from response JSON
   if present, otherwise config default.
9. Unparseable → WARN log. Reviewer uses `attest()` tool explicitly.

Two PK message lookups per RESPONSE — only for RESPONSE type messages with
non-null messageId. RESPONSEs are infrequent relative to STATUS/EVENT.

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

Zero changes needed. The Bayesian Beta model aggregates ALL attestations for an
actor. Each attestation is an observation weighted by `recencyWeight × confidence`.

A DONE that receives:
- Policy SOUND (0.7) + Peer ENDORSED (0.4) → two positive observations, peer
  nudges score up less than policy
- Policy SOUND (0.7) + Peer CHALLENGED (0.5) → mixed signal, exactly what the
  model handles

Over time, agents that consistently ENDORSE poor work will see their own trust
score degrade (their ENDORSED attestations produce observations on the subject,
not on themselves — but if they are later shown to be wrong, the subject's score
correction reflects back through the Bayesian update).

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
| PeerReviewResponseHandler | CDI-free unit | Only fires for RESPONSE, traces inReplyTo, parses structured JSON, falls back silently, calls writer |
| Integration | @QuarkusTest | Full round-trip: COMMAND → DONE → auto-review QUERY → structured RESPONSE → attestation written. list_attestations returns both policy and peer |
| MCP tools | @QuarkusTest | attest() writes ENDORSED/CHALLENGED. request_peer_review() sends QUERYs. list_attestations() returns all |
| Channel config | @QuarkusTest | setReviewerInstances() persists and reads back. create_channel with reviewer_ids |

CDI-free unit tests use mock LedgerEntryRepository, InstanceStore, MessageStore.
tracingConfig set to disabled-tracing impl per convention.

Integration tests use `QuarkusTransaction.requiringNew()` for dispatch (observer
transaction discipline). Channel created with reviewerInstances. Reviewer
instance registered with `"peer-reviewer"` capability.

Blocking-only for v1. Reactive parity deferred unless reactive tests are enabled.

## Research Basis

- [MAST failure taxonomy](https://openreview.net/forum?id=wM521FqPvI) — 21%
  verification gaps
- [Context Window Management (Zylos, 2026)](https://zylos.ai/research/2026-03-31-context-window-management-session-lifecycle-long-running-agents/)
  — 65% enterprise AI failures
- Issue #356 open question: structured verification questions over open-ended
  review → addressed by structured response format with fallback

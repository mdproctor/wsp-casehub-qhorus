---
layout: post
title: "Three issues, one CDI proxy bug, and a utility that should have existed from day one"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [qhorus]
tags: [attestation, evidential-checks, json, cdi, telemetry]
---

Batched three issues onto one branch — telemetry JSON escaping, CommitmentContext enrichment for V1/V4 evidential checks, and topic-aware channel digest. The kind of session where each item is small enough to be boring alone but the aggregate moves the platform forward.

## String concatenation is not JSON construction

The telemetry JSON in `renameTopic`, `mergeTopics`, `moveTopic`, and `force_release_channel` was built by hand — string concatenation with escaped quotes. If a topic name contained `"` or `\`, the JSON was malformed. The fix is a `telemetryJson(Object... keyValues)` utility on `QhorusMcpToolsBase` that serializes through `ObjectMapper`. One method, five call sites fixed, the entire class of bug eliminated.

The interesting question isn't the fix — it's why this pattern existed at all. `ObjectMapper` was already available via `QhorusEntityMapper`. The telemetry field is just a JSON string on `MessageDispatch`. Nobody had wired the two together because the telemetry construction sites were all literal strings with no user-supplied values until topics arrived.

## Evidential checks graduate from benchmark to production

`CommitmentContext` gained three fields: `artefactUuid`, `expectedToken`, and `content`. These let V1 (ghost artefact) and V4 (verification token) evidential checks fire through the attestation path — not just in the benchmark harness.

The design decision worth noting: `StoredCommitmentAttestationPolicy` now injects `EvidentialChecker` and calls `checkObligation()` for every DONE. If the checker finds violations, the verdict downgrades from SOUND to FLAGGED. This means the *default* policy — not just custom devtown policies — incorporates evidence. An attestation policy that writes SOUND for a fraudulent DONE is wrong regardless of configuration.

`LedgerWriteService.writeAttestation()` extracts the artefact UUID from the terminal message's `artefactRefs` and the expected token from the COMMAND content's `verification_token` field. Both are nullable — the checks fire only when the data is present. Existing flows are unchanged.

## Topic digest and a CDI proxy field-access bug

`get_channel_digest` now includes a `topicBreakdown` — per-topic message count, last activity, and resolved status. The counts come from the same non-EVENT message set the digest already loads, joined with `Topic` records for resolved status. No extra queries per topic.

The interesting discovery was a pre-existing bug in `resolveTopic` and `unresolveTopic`. Both did `topicService.topicStore.find(...)` — field access on a CDI proxy. CDI proxies only delegate method calls. Field reads go to the proxy object's own uninitialized fields, which are always null. These tools had never worked via MCP. The fix: `TopicService.resolve()` and `unresolve()` now return the updated `Topic` directly.

This one is worth internalising. `service.store.method()` looks perfectly valid — Java has no syntax distinction between field access and method invocation that would signal the proxy boundary. The NPE only surfaces at runtime, in a CDI context, with no compile-time warning.

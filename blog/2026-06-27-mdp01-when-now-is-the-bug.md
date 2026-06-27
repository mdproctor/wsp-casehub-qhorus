---
layout: post
title: "When now() Is the Bug"
date: 2026-06-27
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [cloudevents, spi-design, api-evolution]
---

## When `now()` Is the Bug

Three issues landed on one branch — a timestamp bug, a missing SPI field, and a normaliser that couldn't tell connectors apart. They looked independent. They shared a theme: places where the architecture had the right shape but the wrong data flowing through it.

`QhorusCloudEventAdapter` fired CloudEvents with `OffsetDateTime.now()` as the event time. The adapter ran in an `@ObservesAsync` handler on CDI's managed executor — by the time it constructed the CloudEvent, the message had been persisted milliseconds to seconds earlier. Under load, the gap widens. Drools CEP uses `CloudEvent.getTime()` to advance its session clock, so the clock was advancing to "when the adapter happened to run" rather than "when the message was dispatched."

The fix was obvious: carry the message's persist timestamp through `MessageReceivedEvent` and use it in the adapter. But `MessageReceivedEvent` had no timestamp field. Adding `Instant occurredAt` to a record in the API module is a breaking change — every constructor call site across qhorus, claudony, and engine needs the new argument. That's the point. The break forces every caller to acknowledge the timestamp exists.

The spec review caught something I'd missed: the reactive path constructs a synthetic `Message` object for observer dispatch — `new Message()` without `@PrePersist`. The `createdAt` field is null on that copy, which means the dispatcher's fallback (`message.createdAt != null ? message.createdAt : Instant.now()`) silently relocates the bug to the reactive stack. One line fixes it: `syntheticMsg.createdAt = ctx.occurredAt()`. The `DispatchContext` already carries the persisted timestamp — it just wasn't threaded through to the synthetic copy.

The second issue was simpler in shape but subtler in consequence. `LedgerWriteService.writeAttestation()` extracted `capabilityTag` from the COMMAND content and set it on the `LedgerAttestation` entity — but only *after* calling `attestationPolicy.attestationFor()`. The policy never saw the tag. For the default `StoredCommitmentAttestationPolicy` this didn't matter — it ignores context entirely. But devtown's trust-gated policy needs `capabilityTag` to call `TrustScoreSource.capabilityScore()` instead of `globalScore()`. The fix adds `capabilityTag` as the fifth field on `CommitmentContext` and moves the extraction before the policy call. Single extraction, one source of truth — the attestation entity reads `ctx.capabilityTag()` instead of re-parsing the JSON.

While we were there, I killed the 2-arg `attestationFor` overload on `CommitmentAttestationPolicy`. Zero production callers — both ledger write services call the 3-arg form directly. Thirteen test callers, all using the 2-arg form to avoid constructing a `CommitmentContext`. Adding `capabilityTag` to the context record increases what that shim hides. The break forces every test to explicitly decide what context to pass — even if the answer is `null`.

The third issue was the most architecturally interesting. `HumanParticipatingChannelBackend.normaliser()` returns a single normaliser for the entire backend instance. Slack sidesteps this by being its own backend class with its own normaliser. But `ConnectorChannelBackend` handles email, SMS, WhatsApp, and webhook connectors through one CDI bean — and can only return one normaliser for all of them.

The gateway already captures the normaliser per-channel in its `BackendEntry`. The missing piece was channel context: `normaliser()` has no parameter, so the backend can't return different normalisers for different channels. Renaming to `normaliserFor(UUID channelId)` is a one-parameter addition that unblocks per-connector dispatch. The connector-backend module gets a new `ConnectorNormaliser` SPI — CDI beans that declare their connector affinity via `connectorId()`, discovered at `@PostConstruct` with fail-fast duplicate detection following the `ProjectionRegistry` pattern.

One question worth naming: should normaliser selection key on `connectorId` (instance-specific: `"email-inbound"`) or `connectorType` (transport-generic: `"email"`)? The binding stores `inboundConnectorId`, and `ConnectorKeyStrategy.SENDER_KEYED` already keys on connector ID throughout the connector-backend module. Keying on ID means no mapping layer and consistency with existing patterns. When the `InboundConnector` SPI eventually gains a `type()` method, the mapping can switch.

---
layout: post
title: "First Contact"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [connector-backend, channel-gateway, spi, postgresql, race-conditions]
---

Before this branch, `ConnectorChannelBackend` had one answer for an unrecognised
sender: discard with a WARN. That was fine when channels were pre-provisioned.
It stops being fine the moment you want SMS from a new number or an email from a
first-time contact to actually go somewhere.

The feature is simple in intent: when a message arrives from an unknown sender,
create a channel for it, route the message, and carry on. The design turned out
to be less simple.

## Where to put the SPI

The first question was whether to make auto-creation a config-only feature or an
SPI. Config-only is faster to write. An SPI is the right answer — any application
using the connector bridge will eventually need to validate the sender, look them
up in a CRM, or apply domain-specific channel defaults. Config can't do that.

So: `AutoChannelPolicy`. One method, returns `Optional<AutoChannelSpec>`.

The next question was where it lives. The `api/spi/` package is the canonical home
for consumer-facing SPIs in this platform. But `AutoChannelPolicy` takes an
`InboundMessage` — a type from `casehub-connectors-core`, which `casehub-qhorus-api`
does not and should not depend on. Putting it in `api/spi/` would drag connectors into
the api module's dependency graph, which would then reach every consumer.

Bridge-module SPIs whose parameter types reference bridge-internal dependencies live in
the bridge module. That became a new protocol (PP-20260601-c43112). The interface lives
in `connector-backend`; consumers who want to override it already depend on that module
by definition.

## The convention table

The default implementation (`ConfiguredAutoChannelPolicy`) reads per-connector config.
The interesting part is how it resolves the outbound connector ID — where to send replies.

For SMS and WhatsApp, the answer is forced by the protocol. SMS threading requires the
same provider and number. WhatsApp's Business API requires the same credential for the
reply. You can't receive via Twilio and reply via a different SMS provider without
breaking the conversation thread in the recipient's phone.

For email, it's a business decision. The outbound SMTP account — which mailbox the
reply comes from — is something an operator chooses deliberately. One application might
have multiple email connectors. Convention would guess wrong.

So the convention table has two entries: `twilio-sms-inbound → "twilio-sms"` and
`whatsapp-inbound → "whatsapp"`. Email and Slack require explicit config. If neither
applies and explicit config is missing, the policy returns empty and the message is
discarded — with an ERROR log naming exactly which config key is missing.

## The transaction

`ChannelService.findOrCreateWithBinding()` is `@Transactional(REQUIRES_NEW)`. That
transaction commits before `initChannel()` fires, so the channel is durable before
the gateway registry is populated. Without `REQUIRES_NEW`, a rolled-back outer
transaction could leave a registered channel with no DB record.

The method rechecks the binding inside the transaction before creating — if two threads
race on the same first message from the same sender, one commits the channel and binding,
the other hits the unique constraint on `uq_binding_key`. The loser catches
`PersistenceException`, confirms it's a constraint violation, finds the winner's channel
via `findByConnectorKey()`, and routes normally. Both messages are delivered. No channel
is duplicated.

Only the winner calls `initChannel()`. The loser doesn't — calling it would cause
`onChannelInitialised()` to deregister then re-register the backend, which is not atomic.
Between deregister and register, the backend isn't listed. For a fanOut arriving in that
window, the message persists but the push is skipped. That's within the at-most-once
delivery contract already documented for qhorus#132.

## The production bug we almost shipped

The exception discrimination had a problem. The `isConcurrentInsert()` method walked
the cause chain looking for `instanceof SQLIntegrityConstraintViolationException`. In
H2 tests, that check works — H2 throws exactly that type. In production with PostgreSQL,
it doesn't. `PSQLException` extends `java.sql.SQLException` directly and never satisfies
the `instanceof` check. Tests pass; production silently discards every race-loser message
and logs a spurious DB error.

Claude caught it in the final code review. The fix adds a second branch to the walk
that checks `java.sql.SQLException` for the constraint name in the message:

```java
if (cause instanceof java.sql.SQLException s
        && !(cause instanceof SQLIntegrityConstraintViolationException)) {
    if (s.getMessage() != null && s.getMessage().contains("uq_binding_key"))
        return true;
}
```

H2 takes the first branch. PostgreSQL takes the second. The garden entry is
GE-20260601-17fa50.

## What's different

Any Quarkus application that includes `casehub-qhorus-connector-backend` and sets
`casehub.qhorus.connector.auto-channel.entries."twilio-sms-inbound".enabled=true`
now gets automatic channel creation for unknown SMS senders. The channel persists,
the first message routes, and every subsequent message from that sender uses the
same channel as if it had been pre-provisioned.

Per-connector `InboundNormaliser` customisation (qhorus#216) and delivery guarantees
for those channels (qhorus#132) are next.

---
layout: post
title: "Names that mean something"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [slug-enforcement, channel-naming, validation]
---

Channel names in Qhorus have always been a gentleman's agreement. The database had a uniqueness constraint, nothing more. You could name a channel `"Billing Output"`, `"xp9qr"`, or a UUID string, and the platform would dutifully accept it. The dual-identity design — every channel has both a UUID and a human-readable name — rests on the assumption that the human-readable half is actually human-readable. Without enforcement, that assumption was hollow.

#236 fixes this. Every channel name is now a `/`-delimited path where every segment matches `[a-z][a-z0-9]*(-[a-z0-9]+)*`. Hyphens are separators between alphanumeric groups, not affixes. UUID-shaped names are explicitly rejected.

The pattern choice was not obvious. The issue originally specified `[a-z][a-z0-9-]{0,79}`, which allows trailing hyphens and consecutive hyphens. I wanted something stricter: a segment is a word or a phrase, and hyphens are the glue between parts of it — `billing-output`, not `billing-`. The stricter pattern `[a-z][a-z0-9]*(-[a-z0-9]+)*` enforces this. It also passes every existing channel name in the codebase — `case-abc`, `twilio-sms-inbound`, nanotime-suffixed test names — without a single change.

The UUID-shape rejection deserves a separate explanation. `resolveChannel()` has always tried UUID parse first: if the input parses as a UUID, look up by ID; if not, look up by name. A channel named `a81b4c6d-1234-5678-abcd-ef0123456789` — roughly 37% of random UUIDs start with a–f, so this isn't contrived — would be routed via its UUID ID, not its name. Name-based lookup becomes silently unreachable. The fix is to block UUID-shaped names at creation time and flip `resolveChannel()` to name-first. Both changes together close the ambiguity completely.

The harder design question was what to do with auto-created connector channels. These embed external identifiers — phone numbers, email addresses — in the channel path: `connector/twilio-sms-inbound/+14155552671`. That trailing segment is not a valid slug. Either we exempt connector channels from slug enforcement, or we sanitise the embedded identifier. I chose Approach B: every segment, everywhere, must conform.

The sanitisation design has two functions, not one. `sanitiseSegment()` handles arbitrary user-provided external keys and always appends an 8-hex-char SHA-256 hash of the lowercased input. The hash is unconditional: at sanitisation time there's no way to know whether another distinct raw input will produce the same normalised prefix, and two inputs that do would silently share a channel. `slugifyConnectorId()` handles developer-defined connector IDs, which should already be valid slugs — it normalises defensively but appends no hash. An existing channel named `connector/twilio-sms-inbound/<phone>` survives this: the connector segment is unchanged, only the phone key segment changes.

During spec review, Claude caught a subtle bug in the UUID exclusion code I'd drafted. The pattern was:

```java
try {
    UUID.fromString(name);
    throw new IllegalArgumentException("Channel name must not be UUID-shaped: ...");
} catch (IllegalArgumentException ignored) {
    // not a UUID — good
}
```

The `throw` inside the `try` is caught by the same `catch`. UUID-shaped names pass validation silently. The fix is a boolean flag — set it inside the try, check it after:

```java
boolean isUuid;
try { UUID.fromString(name); isUuid = true; }
catch (IllegalArgumentException ignored) { isUuid = false; }
if (isUuid) throw new IllegalArgumentException("...");
```

This is the kind of thing that compiles cleanly, runs without error, and fails only when you test the specific rejection path. The test that guards this was in the spec before the implementation. The code review caught it earlier.

The spec went through five rounds of review before implementation began. Eleven issues in the first pass alone, including the UUID exception bug and a hash arithmetic error (74 + 9 ≠ 80). By revision five, the issues were all Minor. That front-loading paid off: the implementation itself was clean. We hit one unexpected discovery — H2 in PostgreSQL mode rejects `SIMILAR TO` inside `ALTER TABLE ADD CONSTRAINT CHECK`, requiring `REGEXP_LIKE` instead — but nothing requiring a design change.

The branch closes with 17 commits, 17 files, 948 insertions. Every existing test passes. `FlywayMigrationSchemaTest` now verifies the V17 constraint exists.

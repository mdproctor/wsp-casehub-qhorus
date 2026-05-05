---
layout: post
title: "Docs That Build Themselves"
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [documentation, ledger, mcp-server, quarkus]
---

There's a point when explaining a system to yourself reveals the gaps. This session hit it writing the developer guide for Qhorus's agent mesh.

I'd had the normative layer documented in `normative-layer.md` for a while — the theory, the enterprise examples, the academic lineage. What was missing was the other direction: start with `send_message`, work forward to why the distinctions matter. A guide for a developer who just added the dependency, not for someone already convinced.

Claude and I built it in ten sections, ordered deliberately: message vocabulary first, then channels, then the three-channel topology, agent lifecycle, the commitment store, the ledger, human-in-the-loop patterns. The ordering was the whole point — you can't explain `COMMAND` creating a `CommitmentStore` entry until the reader has seen what `COMMAND` means and what a channel is. Vocabulary before machinery.

The first read came back as "it finally all starts to make sense." I'll take that as evidence the structure was right.

## The Tool the Documentation Implied

Writing Part 6 on the ledger made the gap obvious. The section listed the query tools — `list_ledger_entries`, `get_obligation_chain`, `get_causal_chain` — and kept circling the same question: how do you see everything a single obligation touched, across all three channels, in one call?

The COMMAND on work. The tool-call EVENTs on observe. The human escalation on oversight. One correlation ID, one chronological view.

That tool didn't exist. We built it: `get_obligation_activity`. The design was clear. The implementation discovered two things that weren't.

**`MessageLedgerEntry.content` is null for EVENT entries.** The telemetry fields (`toolName`, `durationMs`, `tokenCount`) are extracted from the message JSON at write time and stored in dedicated columns — the content field itself is set to null. I'd written a LIKE query on `e.content` to find EVENTs whose body contained a given correlation ID. Every test returned zero. No error, just silence.

The right pattern is explicit: agents pass `correlationId` when sending EVENT messages to the observe channel. That field is stored on the entry. The content column isn't.

**`LedgerEntry.sequenceNumber` is per-channel, not global.** Within one channel it's a reliable cursor for pagination. Across channels, both channels start at 1 — `ORDER BY sequenceNumber ASC` on a cross-channel query gives non-deterministic ordering. The fix is `ORDER BY messageId ASC`, where `messageId` mirrors the `Message` table's global auto-increment primary key.

Both went into the garden immediately.

## The Invisible Drop

We also fixed a `quarkus-mcp-server` silent drop — `@Tool` methods disappearing from the tool list when the class has public non-`@Tool` overloads with the same name. No warning, no error, just a missing tool.

The fix is making the convenience overloads package-private. 53 test files needed updating to call the full @Tool signatures. `ToolOverloadDiscoverabilityTest` guards against it going forward — pure reflection, no Quarkus, fails immediately if any public non-`@Tool` method shares a name with a `@Tool` method.

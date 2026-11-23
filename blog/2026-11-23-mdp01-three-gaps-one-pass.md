---
layout: post
title: "Three Gaps, One Pass"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [playbook-bridge, code-review, tenancy, protocol, polish]
series: issue-533-update-claude-md-playbook-bridge
---

# Three Gaps, One Pass

The playbook-bridge module shipped two days ago — the `ProtocolResolver` SPI, the `PlaybookChannelProtocol` evaluator, deadline scheduling, the REST endpoint, the JPA store. Code review at landing time caught four issues and filed them. This session closed three of them in one pass.

The pattern is familiar: a new module lands with working code and the gaps aren't in the logic — they're in the connective tissue. Documentation that doesn't exist yet. A REST endpoint that stores a protocol definition but doesn't tell the channel it has one. A JPA store that queries by channel and protocol name but forgets to ask which tenant is asking.

**The documentation gap** was the simplest. CLAUDE.md had no entry for `playbook-bridge/` — no project structure block, no migration version, no testing conventions. The existing entries for modules like `slack-channel/` and `notification-bridge/` set a clear pattern: module description, file inventory, known gotchas, migration version. Following it was mechanical but necessary — without it, a future session working in this codebase wouldn't know the module exists.

**The protocol list gap** was the most consequential. `ProtocolDefinitionResource.put()` stored a YAML playbook definition and evicted the compilation cache. What it didn't do was add the protocol name to `channel.protocols`. The definition existed in the database, the resolver could find it, but `MessageService.dispatch()` never called the resolver because the channel didn't declare the protocol. The fix is six lines in each of `put()` and `delete()`: read the channel's current protocols, add or remove the name, call `setProtocols()`. The interesting part is the defensive posture — if the channel doesn't exist yet (the definition can be stored before the channel is created), the update is silently skipped. If the protocol is already in the list, no-op. If it's already absent, no-op. The resource manages the lifecycle; the caller doesn't need to know.

**The tenancy gap** was the quietest. Every other JPA store in qhorus — `JpaChannelStore`, `JpaCommitmentStore`, `JpaMessageStore` — injects `CurrentPrincipal` and filters every query by `tenancyId`. The playbook bridge's `JpaProtocolDefinitionStore` didn't. In a single-tenant deployment, invisible. In a multi-tenant deployment, one tenant's protocol definitions would leak into another's queries. The fix was mechanical: inject `CurrentPrincipal`, add `AND tenancyId = :tid` to every query, change the REST endpoint from hardcoding `"DEFAULT"` to reading `currentPrincipal.tenancyId()`.

What ties the three together is that none of them are bugs in the conventional sense. The module works. Tests pass. The protocol evaluation pipeline executes playbooks correctly. But "works" and "production-ready" aren't the same bar. A module that isn't documented doesn't get maintained. A REST endpoint that requires a second API call to activate what it just stored creates a pit of failure for every consumer. A store that ignores tenancy is a latent data leak. These are the gaps that code review exists to find — the ones that aren't visible from inside the implementation, only from outside it.

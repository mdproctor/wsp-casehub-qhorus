---
layout: post
title: "Phase 12, Error Handling, and the Claudony Blocker"
date: 2026-04-16
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [mcp, error-handling, claudony, ledger, adr]
---

This session started with finishing Phase 12 (structured observability) and ended with a decision about error handling that touched every tool in the codebase.

Phase 12 had shipped in the previous session — `AgentMessageLedgerEntry`, `LedgerWriteService`, `list_events`, `get_channel_timeline`. What remained was making the quarkus-ledger extension itself standalone. We extracted it from quarkus-tarkus into its own Quarkiverse extension at `~/claude/casehub/ledger`, wrote the DESIGN.md and integration guide, fixed the `@ConfigRoot` annotation (without it, `quarkus.ledger.*` config keys appear as "Unrecognized" in startup logs even when the defaults apply correctly), and got it to 33 tests. The pattern — shared ledger infrastructure as a separate extension — was the right call. Both quarkus-tarkus and Qhorus were already using it.

Then the error handling question.

Qhorus has 39 `@Tool` methods. Some return structured records — `ChannelDetail`, `MessageResult`, `CheckResult`, and so on. Others return `String`. When a business rule is violated — unknown channel name, rate limit exceeded, wrong writer — the tool should signal an error without crashing the MCP session. The question was how.

The options I considered: **Option A** — catch exceptions in each tool and return an `"Error: ..."` string. Works for String-returning tools; breaks down for structured-return tools since the caller can't distinguish a real result from an error string. **Option B** — return `isError:true` responses via `ToolCallException` from every tool. Forces callers to handle the exception path. **Option C** — use `@WrapBusinessError` on the class for structured-return tools (the MCP server interceptor converts `IllegalArgumentException` and `IllegalStateException` to `isError:true` automatically) and Option A for String-returning tools.

I went with Option C. The split is principled: structured-return tools have callers that process fields programmatically, so they need `isError:true` with a readable message. String-returning tools have callers that read text, so returning `"Error: channel 'foo' not found"` is the right format for them. ADR-0001 captures this.

Implementing it: `@WrapBusinessError({IllegalArgumentException.class, IllegalStateException.class})` at the class level on `QhorusMcpTools`. Eighty test assertions updated from catching the raw exceptions to catching `ToolCallException`. A new `ToolErrorHandlingTest` covers both the CDI path and the HTTP path.

Then we tried to embed Qhorus in Claudony.

The blocker was Jandex. `quarkus-mcp-server-http` requires CDI bean discovery to register `@Tool` methods. When Qhorus is used as a dependency — not as the top-level project — Jandex needs to be explicitly configured to index the extension's classes. Without the `jandex-maven-plugin` in the runtime `pom.xml`, the `@Tool` annotations are invisible. Claude caught the missing plugin; the fix was a single plugin declaration in `runtime/pom.xml`.

With the plugin in place, Claudony could see Qhorus tools. Phase 8 was unblocked — at least structurally. We wrote the full Claudony briefing at `docs/phase8-claudony-integration.md` and handed it to Claudony Claude. The actual embedding happened in their session.

521 → 561 tests. ADR-0001 committed. The error handling story is done.

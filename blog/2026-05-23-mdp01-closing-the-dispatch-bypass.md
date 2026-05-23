---
layout: post
title: "Closing the Dispatch Bypass"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
tags: [architecture, enforcement, cdi, dispatch]
---

Five open issues from the #184 code review — most of them small. Test gaps in the MessageDispatch builder. A stale Javadoc on LedgerWriteService that said "write failures are never caught here" when attestation writes were very much caught. A duplicated UUID parsing block copied verbatim between QhorusMcpTools and ReactiveQhorusMcpTools. A global Jackson `serialization-inclusion=non-null` setting in `application.properties` that was library-hostile — DispatchResult already carried `@JsonInclude(NON_NULL)` at the type level, making the global setting both redundant and an invisible footgun for anyone embedding the extension. We worked through those in sequence without much ceremony.

Then #188.

The bug: `A2AChannelBackend.receive()` was silently bypassing rate limiting, ACL checks, the channel paused guard, and fanOut. Before #184, A2A routed through `QhorusMcpTools.sendMessage()` and got all those checks for free. After the refactor moved A2A to call `MessageService.dispatch()` directly, the routing changed but the enforcement didn't follow it.

We went through the options. Extract enforcement into a shared service; both callers delegate to it. Or implement the skipped checks directly in A2AChannelBackend. I rejected both. They have the same flaw: any future caller added to qhorus would need to know to apply enforcement or it silently bypasses it. That's the bug we're fixing, recreated at a different level.

The right move is Option C: enforcement belongs inside `dispatch()`, where every caller gets it automatically. I'd initially framed this as a problem — "rate limiting and ACL are infrastructure concerns, not domain logic, they don't belong in an application service." But that framing is wrong. What's the use case for `dispatch()`? "Send a message to a channel." Does that use case include the paused check, ACL, rate limiting, fanOut? Yes, all of them — that's what sending a message means in qhorus. Enforcement belongs there.

We pulled four new dependencies into `MessageService` — `RateLimiter`, `ChannelGateway`, `AllowedWritersPolicy`, `InstanceService` — and implemented the enforcement sequence in order: paused → ACL → rate limit → LAST_WRITE → normal insert → rate limit recording → fanOut. LAST_WRITE moved in too; it was always a channel semantic, not an MCP tool concern. `QhorusMcpTools.sendMessage()` sheds about 80 lines and keeps only what's genuinely MCP-specific: artefact lifecycle, deadline assignment, content format validation.

One design decision worth spelling out: the ACL supplier. `AllowedWritersPolicy.isAllowedWriter()` takes a lazy `Supplier<List<String>>` for the sender's capability tags. For registered qhorus instances these are real tags from the instance registry. For A2A senders — external agents, not registered — the registry returns empty. An empty supplier means `role:agent` in an ACL entry blocks all A2A agents even though they're definitively agents. The unified supplier:

```java
() -> {
    List<String> tags = new ArrayList<>(
        instanceService.findCapabilityTagsForInstance(sender));
    tags.add("role:" + dispatch.actorType().name().toLowerCase());
    return tags;
}
```

Every sender type, one supplier. Registered instances get their real capability tags plus their role. External senders get only the synthetic role tag. Capability entries correctly block A2A (external agents have no attested capabilities); role entries work as intended.

The code reviewer flagged the CDI cycle as Critical — `MessageService` injects `ChannelGateway`, `ChannelGateway` injects `MessageService` for its inbound paths, application will not start. The full build and all QuarkusTest suites passed without any startup error. Quarkus Arc resolves cycles between `@ApplicationScoped` beans via client proxies. `ChannelGateway`'s constructor receives a proxy for `MessageService`, not the actual instance; the proxy resolves lazily on first method call. It's in the CDI spec; most developers don't know to look for it.

After all this, `A2AChannelBackend.receive()` required exactly zero new enforcement code. It already called `dispatch()`. Now `dispatch()` does the work.

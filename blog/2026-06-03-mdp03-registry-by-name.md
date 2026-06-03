---
layout: post
title: "The Registry That Names Itself"
date: 2026-06-03
entry_type: note
subtype: diary
projects: [qhorus]
tags: [mcp, cdi, design, generics]
---

I had planned today as cleanup — close a few small items from the ledger and
qhorus backlogs, then implement the `project_channel` MCP tool that got deferred
at the end of the projection work. The cleanup went quickly. The tool took longer
than expected, but for the right reasons.

## The parameter that opened a question

The issue specification said `project_channel(channel_id: string, projection_name: string)`.
I noticed that every other qhorus MCP tool uses `channel_name`, not `channel_id`. Before
touching any code, I asked Claude to check whether this was actually consistent across
the platform.

The answer: no. Qhorus tools use `channel_name` (String). openclaw tools use `channel_id`
(UUID string). Both are right for their context — qhorus creates and manages channels, so
agents know the name; openclaw bridges from message events, which carry UUIDs. Two different
positions in the stack, two different natural identifiers.

Then I started pulling on the thread. If the internal platform uses UUID everywhere — stores,
ledger, commitment records — why does the MCP surface accept names at all? Should we
standardise on UUID? On names? On slugs? And: do we even have semantic slug enforcement, or
just unique-string enforcement?

The answer to that last question was sobering: channel names have a DB unique constraint, but
no format constraint. An agent could create a channel named "Billing Output Channel" with
spaces and capitals, and everything would accept it. The "semantic slug" I'd been assuming was
a convention, not a fact.

We spent more time than I'd expected working through the identity question. The argument for
UUID everywhere is platform coherence — it's what all the internal layers use, what openclaw
already does, what ledger query results return. The argument against is agent usability — an
LLM can construct `billing-output` from context; it cannot construct `3f7a9c2b-...`. The
compromise that felt right: accept either, resolve internally, return both. File three issues
to track the remaining work (slug enforcement, MCP migration, dual-identity protocol) and
proceed.

## The registry design

The tool needed two things: a way to find a `ChannelProjection` implementation by name, and
a way to render the typed fold result as a `String` the MCP can return.

An earlier review pass had already ruled out CDI qualifier annotations. The reason: `api/`
is supposed to be lightweight — no CDI annotation dependencies. Qualifier annotations would
pull `AnnotationLiteral` into `api/`. The alternative is simpler and already has a precedent
in the codebase: give the interface a `projectionName()` method, build a Map at CDI startup,
detect collisions then.

I named the interface `RenderableProjection<S>` and added two methods on top of the inherited
fold contract: `projectionName()` and `render(ProjectionResult<S>)`. The `ProjectionRegistry`
is an `@ApplicationScoped` bean that iterates `@Any Instance<RenderableProjection<?>>` in its
constructor. One pass, one map. If any bean returns null or blank, throw `IllegalStateException`
immediately — not at first lookup, now.

The render signature took a round of external review to get right. My first draft was
`render(S state)`. The reviewer pointed out that `state == identity()` is ambiguous — it can
mean the channel is empty, or it can mean the fold ran but no messages matched the projection's
criteria (a COMMAND counter on an EVENT-only channel also produces zero). The full
`ProjectionResult<S>` carries `isEmpty()`, which is the only reliable signal. Right call.

## A Java generics wall

Writing `ProjectionRegistryTest` without CDI required a package-private constructor that
took `List<? extends RenderableProjection<?>>` as its argument. The test stubs returned
`RenderableProjection<?>`. `List.of(stub("a"), stub("b"))` should produce exactly that type.

It didn't. The compiler inferred `List<RenderableProjection<? extends Object>>`, which is not
assignable to `List<? extends RenderableProjection<?>>`. The explicit type witness
`List.<RenderableProjection<?>>of(...)` didn't help. Three approaches failed before the
fourth worked: accumulate into an explicit `ArrayList<RenderableProjection<?>>` declared with
that type, let inference anchor on the variable's declared type, then pass the named variable.

The problem is that `List.of()` infers the captured wildcard from each element independently
and cannot unify them across the nested bound. Naming the variable forces the correct
inference. This went into the garden as GE-20260603-8f582a — not a guess but confirmed
behaviour on Java 21.

## What the review found

I ran a code review before committing. Two real bugs, both important:

The first: `ProjectionRegistry.buildMap()` didn't guard against `projectionName()` returning
null. `HashMap.put(null, p)` is valid Java — it would have silently registered a null-keyed
entry and produced a confusing collision message if two projections returned null. One null
check and a clear startup error message fixed it.

The second: `resolveChannel()` — the method that accepts either a channel name or UUID —
discriminated UUID from name using `e.getMessage().startsWith("Channel not found")` inside a
catch block. The reviewer called it correctly: that's exception-message inspection for control
flow, which is brittle. The right approach is two-phase: `tryParseUuid()` returns null on
parse failure, then the caller acts on that null. No string inspection, no fragile sentinel.
Claude had found the first version acceptable; the reviewer was right to flag it.

## What's next

The tool is in. `list_projections` — returning the names of all registered projections so an
LLM can discover what's available — is a follow-on (qhorus#240). Right now if an agent calls
`project_channel` with an unknown projection name, it gets an error and has no recovery path.
That's a real LLM usability gap.

The dual-identity question is still open. Three issues filed, none of them blocking.

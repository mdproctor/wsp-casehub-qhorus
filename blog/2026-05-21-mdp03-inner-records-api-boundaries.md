---
layout: post
title: "Inner records, api boundaries, and a silent datasource mismatch"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
tags: [refactoring, api-design, quarkus]
---

A cleanup session — closing issues that were already done or false, then
three refactors that had been sitting on the backlog.

## The type promotion cascade

`ChannelDetail`, `InstanceInfo`, and `MessageResult` were inner records of
`QhorusMcpToolsBase` in the `runtime.mcp` package. That meant any consumer
needing the return type of `listChannels()` or `sendMessage()` had to import
from the MCP dispatch layer — the wrong dependency. I moved them to
`casehub-qhorus-api` under domain-aligned packages (`api.channel`,
`api.instance`, `api.message`), which is where plain Java records with no
framework dependencies belong.

Thirty-five files needed updating — tests used both static imports
(`import io.casehub.qhorus.runtime.mcp.QhorusMcpToolsBase.ChannelDetail`) and
qualified names (`QhorusMcpTools.ChannelDetail`). We worked through them
systematically with the IntelliJ replace-in-file tool.

The surprise came at runtime. After the move, `mvn test-compile -pl api,runtime`
reported BUILD SUCCESS. Then `mvn test -pl runtime` failed immediately:

```
NoClassDefFoundError: io/casehub/qhorus/api/channel/ChannelDetail
```

The compiler had found the source directly from the reactor. The test runner
looked in `~/.m2` for the installed `casehub-qhorus-api` jar — and found the
pre-move version. Running `mvn install -pl api` first fixed it. Obvious in
hindsight, but the clean compile made it look like there was nothing wrong.

## Named PU alignment

`ReactiveMessageService.send()` had been fixed earlier to call
`Panache.withTransaction("qhorus", ...)` explicitly. Five other reactive
services — `ReactiveChannelService`, `ReactiveInstanceService`, and three
others — still used the bare `Panache.withTransaction(() -> ...)` form.

The bare form uses the default persistence unit. In test, there is a default
datasource configured (casehub-ledger's beans require one). In a consumer app
like Claudony that configures only `quarkus.datasource.qhorus.*`, the bare call
silently routes to whichever default PU happens to be configured — or fails
confusingly if none is. No warning, no exception at the call site.

An audit across all reactive services found 16 bare calls. We replaced them all.

## Shared mapper

`QhorusMcpToolsBase.toTimelineEntry()` parsed JSON telemetry from message
content using `new ObjectMapper()` inline — a new instance per call. The same
method existed in `QhorusDashboardService`, written the correct way with an
injected mapper. We extracted both into `QhorusEntityMapper`, a plain
`@ApplicationScoped` CDI bean, and had both classes inject and delegate to it.
The inline ObjectMapper is gone; the duplicate implementation is gone.

---
layout: post
title: "A string is not a contract"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [connector-backend, cdi, maven, constants]
---

Two small issues today, but the fix for one of them turned into a reminder about
what "small" actually means.

Issue #220 was filed during the #219 code review: `ConnectorKeyStrategy` had three
hardcoded string literals — `"twilio-sms-inbound"`, `"whatsapp-inbound"`,
`"email-inbound"` — in a `Set.of(...)` that determines whether to key inbound
messages by sender or by channel. The risk was clear: if a connector renames its ID,
the routing silently falls back to the wrong key strategy. No compiler error. Tests
still pass because they use the same literal.

The obvious fix is to reference the constants rather than duplicate them. But when
I looked, the constants were package-private fields on classes in three separate
modules — `casehub-connectors-webhook` and `casehub-connectors-email-inbound` — none
of which `connector-backend` depends on. Referencing them would mean adding coupling
to specific connector implementations, which is exactly what the module structure is
designed to avoid.

So the correct fix was to promote the IDs to `casehub-connectors-core` — the module
everything already depends on — as a new `InboundConnectorIds` class with public
constants. We filed `casehubio/connectors#11`, made the change, updated the three
connector classes to reference the core constants (and made their own `ID` fields
public while we were there), then updated `ConnectorKeyStrategy` and all the tests.

The implementation went smoothly. The build didn't.

Running `mvn install` in the connectors repo reported `BUILD SUCCESS`, but the
installed jar didn't contain `InboundConnectorIds.class`. The consumer (`connector-backend`)
then failed with `cannot find symbol: class InboundConnectorIds` and I spent a few
minutes convinced the code was wrong before checking the jar directly:

```
jar tf ~/.m2/repository/io/casehub/casehub-connectors-core/0.2-SNAPSHOT/casehub-connectors-core-0.2-SNAPSHOT.jar | grep InboundConnectorIds
```

Empty. The problem was that `connectors` was checked out on its feature branch
(`issue-9-imap-idle-and-attachments`), which doesn't have the new file. Maven compiled
what was in the working directory — which was branch B — and quietly installed a jar
missing the class that only exists on main. Nothing in the output hinted at this.

Switching to main and running `mvn install` there fixed it immediately. The gotcha
went into the garden: `GE-20260531-5137f7`.

---

Issue #221 was simpler conceptually but required getting the design right first.
The existing integration test bypasses CDI event delivery entirely — it calls
`backend.onInboundMessage(msg)` directly through the CDI proxy to avoid the
`@ObservesAsync` reliability problems documented in GE-20260513-b15933. That works
for testing the handler logic, but leaves the actual async event chain untested.

For `ConnectorChannelBackend` this matters: if someone accidentally changes
`Event.fireAsync()` to `Event.fire()` in `InboundConnectorService`, or removes the
`@ObservesAsync` annotation from the handler, nothing catches it.

The fix has two parts. First, I changed `onInboundMessage` to return
`CompletionStage<Void>` instead of `void`. This is the right design for any
`@ObservesAsync` handler that does real work — it lets callers using
`fireAsync().toCompletableFuture().join()` reliably wait for the observer to finish
before asserting. The returned stage is always immediately completed (it's a synchronous
body with no async operations inside), so existing callers that invoke the method
directly are unaffected.

Second, we added a new test method that injects `Event<InboundMessage>` and fires it:

```java
inboundMessageEvent.fireAsync(msg).toCompletableFuture().get(2, TimeUnit.SECONDS);
verify(messageService).dispatch(argThat(d -> ...));
```

This works because `ConnectorChannelBackend` is a main-sources `@ApplicationScoped`
bean — ArC registers its `@ObservesAsync` method at build time, so delivery is
reliable. The 2-second timeout is generous for an operation that's a map lookup and
a method call.

The code reviewer flagged that `ConnectorChannelBackendTest` (the pure unit test, no
Quarkus) still had the old string literals after #220 — I'd missed it. That was the
most important file to update, because it's the one without a CDI container that
would catch type mismatches at startup. Fixed.

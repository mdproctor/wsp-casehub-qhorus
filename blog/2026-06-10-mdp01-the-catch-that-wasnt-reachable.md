# The Catch That Wasn't Reachable

Multi-tenancy in Qhorus was half-done. The JPA stores filtered by tenant, `MessageService` resolved the tenant from `CurrentPrincipal`, and the ledger recorded the right tenant on every write. But three gaps remained: A2A endpoints had no tenant source at all, the AgentCard returned no tenant identity, and `MessageLedgerEntryRepository` queried across tenants as if they didn't exist.

The three were connected. Close them all in one branch, starting with the CDI foundation that the other two would inherit.

## The Scope That Broke the Safety Net

The design question for A2A was straightforward: where does the tenant come from when there's no OIDC token? A request header, `X-Tenancy-ID`. A `ContainerRequestFilter` reads it and stores it in a `@RequestScoped` CDI holder. A `CurrentPrincipal` implementation reads from the holder.

The non-obvious part was which scope to give the principal bean.

My first instinct was `@RequestScoped` — it holds per-request state, so the bean should be request-scoped. But `QhorusInboundCurrentPrincipal` also needs to be safe for background threads: Watchdog runs in a scheduler, `ChannelGateway` recovers channels at startup. Both call through code that eventually reaches `currentPrincipal.tenancyId()`. The plan said: catch `ContextNotActiveException`, return `DEFAULT_TENANT_ID`.

The catch would never have fired.

CDI client proxies throw `ContextNotActiveException` before the target method is entered — not inside it. With `@RequestScoped` on the outer bean, a background caller hits the proxy, the proxy tries to resolve the request-scoped instance, finds no active scope, and throws. The method body never runs. The catch is unreachable.

With `@ApplicationScoped`, the proxy always resolves. The method body IS entered. Inside it, accessing the `@RequestScoped InboundTenancyContext` throws where the catch can actually intercept it:

```java
@ApplicationScoped @Alternative @Priority(1)
public class QhorusInboundCurrentPrincipal implements CurrentPrincipal {
    @Inject InboundTenancyContext ctx;  // @RequestScoped

    @Override
    public String tenancyId() {
        try {
            return ctx.tenancyId();  // throws if no request scope — caught here
        } catch (ContextNotActiveException e) {
            return TenancyConstants.DEFAULT_TENANT_ID;
        }
    }
}
```

Per-request state in `InboundTenancyContext @RequestScoped`. The outer bean stateless, `@ApplicationScoped`. The filter populates the holder on every HTTP request; background threads see the fallback.

## Spec Review Before Code

We wrote the design spec and put it through several review iterations before writing a line of implementation. The reviews found three real bugs in the spec.

The test for background-thread safety was written wrong. It dispatched a message with an explicit `tenancyId` on the dispatch — which meant `MessageService.dispatch()` short-circuited before ever calling `currentPrincipal.tenancyId()`. The catch in `QhorusInboundCurrentPrincipal` was never exercised. The fix was a CDI-free unit test that directly instantiates the bean with a stub holder that throws, verifying the catch fires and returns `DEFAULT_TENANT_ID`.

The inventory of test files that would break when `MessageLedgerEntryRepository` signatures changed missed `LedgerAttestationIntegrationTest` — which calls `findAllByCorrelationId` in seven tests. It runs in CI without any profile flag.

And the spec said `StubMessageLedgerEntryRepository` should have all overriding methods gain `tenancyId`. What it missed: `findByMessageId` is PK-based and explicitly in the unchanged list. The stub was listed as gaining a parameter it shouldn't.

Spec review is useful for exactly this — catching assumptions that feel obvious while writing but are wrong.

## The Stub That Was Silently Wrong

Code review after implementation found one more issue. `StubMessageLedgerEntryRepository` accepted the `tenancyId` parameters but didn't use them — filtering still ran across all entries regardless of tenant. CDI-free unit tests using the stub for multi-tenant scenarios would have passed while hiding a real isolation failure.

The fix applied the same null-to-`DEFAULT` normalisation as the production repository and added tenant filtering to both `findEarliestWithSubjectByCorrelationId` and `findLatestByCorrelationId`.

The `@ApplicationScoped` outer bean requirement is now PP-20260610-85e6a4. The requirement to document `X-Tenancy-ID` as a routing header rather than an authentication boundary is PP-20260610-9487d3.

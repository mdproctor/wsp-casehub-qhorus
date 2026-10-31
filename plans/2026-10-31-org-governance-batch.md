# Org Governance Batch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #500 — org-aware escalation routing
**Issue group:** #500, #501, #503

**Goal:** Complete Phase 1 of the org governance epic (casehub-ops#217) and land two Phase 2/3 quick wins — escalation routing, unit-first redistribution, and delegation authorisation.

**Architecture:** All three issues extend the `OrgAwareDispatch` integration pattern established in #494. #500 adds `escalate()` to `RoutingBridge`. #501 verifies that `RedistributionDelegate` inherits org-awareness via `RoutingBridge` (may already be done). #503 adds `canDelegateTo()` to `MessageService`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-eidos-api (OrgAwareDispatch)

## Global Constraints

- No eidos API changes — consume `OrgAwareDispatch` as-is
- No new Flyway migrations
- `OrgAwareDispatch` is nullable everywhere — current behaviour unchanged when absent
- Follow existing patterns: CDI-free unit tests with Mockito for `RoutingBridge` and `MessageService`

---

## Batch 1: Escalation routing (#500)

### Task 1: Add org-aware escalation to RoutingBridge

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/RoutingBridge.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingBridgeTest.java`

**Interfaces:**
- Consumes: `OrgAwareDispatch.escalate(String agentId, String capabilityName, String tenancyId) → Optional<AgentMatch>`
- Produces: `RoutingOutcome` with `strategyName() == "eidos-org-escalated"` when escalation succeeds

- [ ] **Step 1: Write failing test — escalation succeeds when NoneQualified**

```java
@Test
void resolve_noneQualified_escalationSucceeds_returnsEscalatedAgent() {
    var bridgeWithOrg = new RoutingBridge(registry, selector, trustGateService, null, config, orgAwareDispatch);
    when(registry.find(any())).thenReturn(List.of());
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.NoneQualified("no candidates"));
    when(orgAwareDispatch.escalate("sender-1", "analyst", "tenant"))
            .thenReturn(Optional.of(agentMatch("supervisor-1")));

    var outcome = bridgeWithOrg.resolve(dispatch("role:analyst"), channel(), "tenant");

    assertThat(outcome).isNotNull();
    assertThat(outcome.resolvedTarget()).isEqualTo("supervisor-1");
    assertThat(outcome.strategyName()).isEqualTo("eidos-org-escalated");
    assertThat(outcome.candidateCount()).isEqualTo(1);
}
```

- [ ] **Step 2: Write failing test — escalation fails, throws as before**

```java
@Test
void resolve_noneQualified_escalationEmpty_throws() {
    var bridgeWithOrg = new RoutingBridge(registry, selector, trustGateService, null, config, orgAwareDispatch);
    when(registry.find(any())).thenReturn(List.of());
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.NoneQualified("no candidates"));
    when(orgAwareDispatch.escalate("sender-1", "analyst", "tenant"))
            .thenReturn(Optional.empty());

    assertThatThrownBy(() -> bridgeWithOrg.resolve(dispatch("role:analyst"), channel(), "tenant"))
            .isInstanceOf(RoutingRejectedException.class)
            .hasMessageContaining("No agent qualified");
}
```

- [ ] **Step 3: Write failing test — escalation succeeds on Escalated outcome**

```java
@Test
void resolve_escalated_orgEscalationSucceeds_returnsEscalatedAgent() {
    var bridgeWithOrg = new RoutingBridge(registry, selector, trustGateService, null, config, orgAwareDispatch);
    when(registry.find(any())).thenReturn(List.of(agentMatch("agent-1")));
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.Escalated("analyst", EscalationKind.NO_QUALIFIED_AGENT, "all bootstrap"));
    when(orgAwareDispatch.escalate("sender-1", "analyst", "tenant"))
            .thenReturn(Optional.of(agentMatch("manager-1")));

    var outcome = bridgeWithOrg.resolve(dispatch("role:analyst"), channel(), "tenant");

    assertThat(outcome.resolvedTarget()).isEqualTo("manager-1");
    assertThat(outcome.strategyName()).isEqualTo("eidos-org-escalated");
}
```

- [ ] **Step 4: Write failing test — null OrgAwareDispatch skips escalation**

```java
@Test
void resolve_noneQualified_nullOrgDispatch_throwsWithoutEscalation() {
    var bridgeNoOrg = new RoutingBridge(registry, selector, trustGateService, null, config, null);
    when(registry.find(any())).thenReturn(List.of());
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.NoneQualified("no candidates"));

    assertThatThrownBy(() -> bridgeNoOrg.resolve(dispatch("role:analyst"), channel(), "tenant"))
            .isInstanceOf(RoutingRejectedException.class);
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=RoutingBridgeTest -pl runtime -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: FAIL — no escalation logic in `resolve()` yet.

- [ ] **Step 6: Implement escalation in RoutingBridge.resolve()**

In `RoutingBridge.java`, replace the `NoneQualified` and `Escalated` switch arms. Before throwing `RoutingRejectedException`, try `orgAwareDispatch.escalate()`:

```java
case AgentSelection.NoneQualified nq -> {
    if (orgAwareDispatch != null) {
        var escalated = orgAwareDispatch.escalate(dispatch.sender(), capability, tenancyId);
        if (escalated.isPresent()) {
            yield new RoutingOutcome(
                    escalated.get().descriptor().agentId(),
                    target,
                    "eidos-org-escalated",
                    1,
                    0.0);
        }
    }
    throw new RoutingRejectedException(
            "No agent qualified for capability '%s': %s".formatted(capability, nq.reason()));
}
case AgentSelection.Escalated e -> {
    if (orgAwareDispatch != null) {
        var escalated = orgAwareDispatch.escalate(dispatch.sender(), capability, tenancyId);
        if (escalated.isPresent()) {
            yield new RoutingOutcome(
                    escalated.get().descriptor().agentId(),
                    target,
                    "eidos-org-escalated",
                    1,
                    0.0);
        }
    }
    throw new RoutingRejectedException(
            "Routing escalation for capability '%s': %s — %s".formatted(
                    capability, e.kind(), e.reason()));
}
```

Add `import java.util.Optional;` if not already present.

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=RoutingBridgeTest -pl runtime`
Expected: all tests PASS (15 existing + 4 from #494 + 4 new = 23 total).

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/message/RoutingBridge.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingBridgeTest.java
git commit -m "feat(#500): add org-aware escalation to RoutingBridge

Walk org tree via OrgAwareDispatch.escalate() before throwing
RoutingRejectedException on NoneQualified and Escalated outcomes.

Refs #500"
```

## Batch 2: Verify redistribution inherits org-awareness (#501)

### Task 2: Verify and test that RedistributionDelegate inherits org routing

`RedistributionDelegate` routes through `RoutingBridge.resolve()`, which already has org-preference filtering from #494. This task verifies the inheritance path and adds a targeted test.

**Files:**
- Read: `runtime-core/src/main/java/io/casehub/qhorus/runtime/capacity/RedistributionDelegate.java`
- Modify: test file for RedistributionDelegate (find exact path)

**Interfaces:**
- Consumes: `RoutingBridge.resolve()` (already org-aware from #494)
- Produces: verification that redistribution prefers same-unit agents transitively

- [ ] **Step 1: Read RedistributionDelegate.redistribute()**

Verify that `redistribute()` calls `routingBridge.resolve()` with `target("role:" + capabilityTag)`. Confirm the `RoutingBridge` is the CDI-produced bean with `OrgAwareDispatch` wired.

- [ ] **Step 2: Write test confirming org-preference propagates to redistribution**

In the RedistributionDelegate test class, add a test that mocks `OrgAwareDispatch.teamMembers()` returning team members, and verifies that the redistributed obligation routes to a same-unit agent when one has capacity. The exact test structure depends on the existing test patterns found in Step 1.

- [ ] **Step 3: Run test**

Expected: PASS — the org-awareness is already wired through `RoutingBridge`.

- [ ] **Step 4: If already working (likely), commit verification test**

```bash
git commit -m "test(#501): verify redistribution inherits org-aware routing

RedistributionDelegate routes through RoutingBridge.resolve(), which
already applies org-preference filtering from #494.

Closes #501"
```

- [ ] **Step 5: If NOT working, add OrgAwareDispatch wiring**

Only if Step 1 reveals that `RedistributionDelegate` bypasses `RoutingBridge` — add `OrgAwareDispatch` as a dependency and apply the same filter-then-fallback pattern.

## Batch 3: Delegation authorisation (#503)

### Task 3: Add org-aware delegation check for HANDOFF messages

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/RuntimeBeans.java` (if MessageService needs OrgAwareDispatch injection)
- Modify or create: test for delegation authorisation

**Interfaces:**
- Consumes: `OrgAwareDispatch.canDelegateTo(String sender, String target, String tenancyId) → boolean`
- Produces: `DispatchAdvisory` with source `"ORG_DELEGATION"` when cross-unit HANDOFF is unauthorised

- [ ] **Step 1: Write failing test — cross-unit HANDOFF produces advisory**

In the MessageService test class (or a new CDI-free test), dispatch a HANDOFF with a named target where `canDelegateTo()` returns false. Assert that `DispatchResult.advisories()` contains an advisory with source `"ORG_DELEGATION"`.

- [ ] **Step 2: Write failing test — within-unit HANDOFF produces no advisory**

`canDelegateTo()` returns true. Assert no `ORG_DELEGATION` advisory.

- [ ] **Step 3: Write failing test — null OrgAwareDispatch skips check**

No advisory produced when `OrgAwareDispatch` is not available.

- [ ] **Step 4: Write failing test — system senders exempt**

Senders containing `:` (e.g. `"system:watchdog"`) bypass the delegation check, matching the existing `ObligorTrustPolicy` exemption pattern.

- [ ] **Step 5: Run tests to verify they fail**

- [ ] **Step 6: Implement delegation check in MessageService.dispatch()**

Add `OrgAwareDispatch` as a nullable constructor parameter to `MessageService` (same pattern as `RoutingBridge`). In `dispatch()`, after the `ObligorTrustPolicy` check block (around line 320), add:

```java
if (dispatch.type() == MessageType.HANDOFF
        && dispatch.target() != null
        && !dispatch.sender().contains(":")
        && orgAwareDispatch != null) {
    if (!orgAwareDispatch.canDelegateTo(dispatch.sender(), dispatch.target(), tenancyId)) {
        advisories.add(new DispatchAdvisory(
                "ORG_DELEGATION",
                Severity.WARNING,
                "Cross-unit delegation from '%s' to '%s' not authorised"
                        .formatted(dispatch.sender(), dispatch.target()),
                Map.of("sender", dispatch.sender(), "target", dispatch.target()),
                SuggestedAction.ESCALATE));
    }
}
```

Update `RuntimeBeans` and `RuntimeAutoConfiguration` to pass `OrgAwareDispatch` to `MessageService`.

- [ ] **Step 7: Run tests**

Expected: all tests PASS.

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -T 1C`
Expected: BUILD SUCCESS.

- [ ] **Step 9: Commit**

```bash
git commit -m "feat(#503): add org-aware delegation authorisation for HANDOFF

Check OrgAwareDispatch.canDelegateTo() for HANDOFF messages with named
targets. Produces ORG_DELEGATION advisory when cross-unit delegation is
not authorised. System senders exempt.

Closes #503"
```

## References

- [runtime-core/.../RoutingBridge.java:124-131] — NoneQualified/Escalated switch arms (#500)
- [runtime-core/.../RedistributionDelegate.java:127-134] — redistribute() routing call (#501)
- [runtime-core/.../MessageService.java:310-320] — ObligorTrustPolicy check pattern (#503)
- [io.casehub.eidos.api.OrgAwareDispatch] — escalate(), canDelegateTo(), teamMembers()
- [runtime/.../RoutingBridgeTest.java] — existing CDI-free test patterns
- [GitHub casehubio/casehub-ops#217] — parent epic

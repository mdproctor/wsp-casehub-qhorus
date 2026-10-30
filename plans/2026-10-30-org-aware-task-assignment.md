# Org-Aware Task Assignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #494 — feat: org-aware task assignment — prefer same-unit agents

**Goal:** When RoutingBridge resolves a `role:X` target, prefer agents in the same organisational unit as the sender before widening to the full registry.

**Architecture:** Add `OrgAwareDispatch` (from casehub-eidos-api) as an optional dependency to `RoutingBridge`. In `resolve()`, filter capability matches to same-unit peers first; fall back to full list when no peer qualifies. CDI-free POJO — no migration, no new config.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-eidos-api (OrgAwareDispatch)

## Global Constraints

- No eidos API changes — consume `OrgAwareDispatch` as-is
- No new Flyway migrations
- No new config properties
- `OrgAwareDispatch` is nullable (eidos not on classpath) — current behaviour unchanged when absent

---

## Batch 1: Org-aware routing in RoutingBridge

### Task 1: Add org-preference filtering to RoutingBridge with tests

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/qhorus/runtime/message/RoutingBridge.java`
- Modify: `runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingBridgeTest.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/RuntimeBeans.java`
- Modify: `runtime-spring/src/main/java/io/casehub/qhorus/runtime/spring/RuntimeAutoConfiguration.java`

**Interfaces:**
- Consumes: `OrgAwareDispatch.teamMembers(String agentId, String tenancyId) → List<String>`
- Produces: `RoutingBridge` constructor gains 6th parameter `OrgAwareDispatch orgAwareDispatch` (nullable). `RoutingOutcome.strategyName()` returns `"eidos-org-preferred"` when org filtering applied.

- [ ] **Step 1: Write failing test — team members preferred**

Add to `RoutingBridgeTest.java`. Mock `OrgAwareDispatch` returning team members that overlap with registry matches. Assert the selector receives only the filtered list.

```java
private io.casehub.eidos.api.OrgAwareDispatch orgAwareDispatch;

// In setUp(), after existing mocks:
orgAwareDispatch = Mockito.mock(io.casehub.eidos.api.OrgAwareDispatch.class);
bridge = new RoutingBridge(registry, selector, trustGateService, null, config, orgAwareDispatch);

@Test
void resolve_teamMembersPreferred_selectsFromFilteredList() {
    var allMatches = List.of(agentMatch("agent-a"), agentMatch("agent-b"), agentMatch("agent-c"));
    when(registry.find(any())).thenReturn(allMatches);
    when(orgAwareDispatch.teamMembers("sender-1", "tenant")).thenReturn(List.of("agent-a", "agent-c"));
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.Selected(descriptor("agent-a"), null, 0.8, "team member"));

    var outcome = bridge.resolve(dispatch("role:analyst"), channel(), "tenant");

    assertThat(outcome.resolvedTarget()).isEqualTo("agent-a");
    assertThat(outcome.strategyName()).isEqualTo("eidos-org-preferred");

    var captor = org.mockito.ArgumentCaptor.forClass(List.class);
    verify(selector).select(captor.capture(), any());
    assertThat(captor.getValue()).hasSize(2);
}
```

- [ ] **Step 2: Write failing test — fallback when no team member qualifies**

```java
@Test
void resolve_noTeamMemberHasCapability_fallsBackToFullList() {
    var allMatches = List.of(agentMatch("agent-a"), agentMatch("agent-b"));
    when(registry.find(any())).thenReturn(allMatches);
    when(orgAwareDispatch.teamMembers("sender-1", "tenant")).thenReturn(List.of("agent-x", "agent-y"));
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.Selected(descriptor("agent-a"), null, 0.7, "best available"));

    var outcome = bridge.resolve(dispatch("role:analyst"), channel(), "tenant");

    assertThat(outcome.resolvedTarget()).isEqualTo("agent-a");
    assertThat(outcome.strategyName()).isEqualTo("eidos-simple");

    var captor = org.mockito.ArgumentCaptor.forClass(List.class);
    verify(selector).select(captor.capture(), any());
    assertThat(captor.getValue()).hasSize(2);
}
```

- [ ] **Step 3: Write failing test — null OrgAwareDispatch unchanged**

```java
@Test
void resolve_nullOrgAwareDispatch_usesFullList() {
    var bridgeNoOrg = new RoutingBridge(registry, selector, trustGateService, null, config, null);
    var matches = List.of(agentMatch("agent-a"));
    when(registry.find(any())).thenReturn(matches);
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.Selected(descriptor("agent-a"), null, 0.9, "only candidate"));

    var outcome = bridgeNoOrg.resolve(dispatch("role:analyst"), channel(), "tenant");

    assertThat(outcome.resolvedTarget()).isEqualTo("agent-a");
    assertThat(outcome.strategyName()).isEqualTo("eidos-simple");
}
```

- [ ] **Step 4: Write failing test — empty teamMembers unchanged**

```java
@Test
void resolve_emptyTeamMembers_usesFullList() {
    var matches = List.of(agentMatch("agent-a"));
    when(registry.find(any())).thenReturn(matches);
    when(orgAwareDispatch.teamMembers("sender-1", "tenant")).thenReturn(List.of());
    when(selector.select(any(), any())).thenReturn(
            new AgentSelection.Selected(descriptor("agent-a"), null, 0.9, "only one"));

    var outcome = bridge.resolve(dispatch("role:analyst"), channel(), "tenant");

    assertThat(outcome.resolvedTarget()).isEqualTo("agent-a");
    assertThat(outcome.strategyName()).isEqualTo("eidos-simple");
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=RoutingBridgeTest -pl runtime`
Expected: compilation failure — `RoutingBridge` constructor has 5 params, tests pass 6.

- [ ] **Step 6: Implement RoutingBridge changes**

In `RoutingBridge.java`:

1. Add field: `private final OrgAwareDispatch orgAwareDispatch;`
2. Add import: `import io.casehub.eidos.api.OrgAwareDispatch;`
3. Add 6th constructor parameter: `OrgAwareDispatch orgAwareDispatch`
4. In `resolve()`, after `agentRegistry.find()` and before `agentSelector.select()`, add:

```java
List<AgentMatch> candidates = matches;
String strategy = "eidos-simple";

if (orgAwareDispatch != null) {
    List<String> teamMemberIds = orgAwareDispatch.teamMembers(dispatch.sender(), tenancyId);
    if (teamMemberIds != null && !teamMemberIds.isEmpty()) {
        var teamSet = new java.util.HashSet<>(teamMemberIds);
        List<AgentMatch> teamMatches = matches.stream()
                .filter(m -> teamSet.contains(m.descriptor().agentId()))
                .toList();
        if (!teamMatches.isEmpty()) {
            candidates = teamMatches;
            strategy = "eidos-org-preferred";
        }
    }
}
```

5. Pass `candidates` to `agentSelector.select()` instead of `matches`
6. Use `strategy` in the `RoutingOutcome` constructor instead of `"eidos-simple"`

- [ ] **Step 7: Update existing RoutingBridge constructor call sites**

In `RoutingBridgeTest.setUp()`, update the default bridge construction to pass `null` as 6th arg:

```java
bridge = new RoutingBridge(registry, selector, trustGateService, null, config, null);
```

- [ ] **Step 8: Update RuntimeBeans.routingBridge() CDI producer**

In `RuntimeBeans.java` around line 282, add `Instance<OrgAwareDispatch>` parameter:

```java
@Produces @ApplicationScoped
public RoutingBridge routingBridge(Instance<AgentRegistry> agentRegistryInstance,
                                  Instance<AgentSelector> agentSelectorInstance,
                                  Instance<TrustGateService> trustGateServiceInstance,
                                  Instance<ActorCapacityView> capacityViewInstance,
                                  QhorusConfig config,
                                  Instance<io.casehub.eidos.api.OrgAwareDispatch> orgAwareDispatchInstance) {
    return new RoutingBridge(
            agentRegistryInstance.isResolvable() ? agentRegistryInstance.get() : null,
            agentSelectorInstance.isResolvable() ? agentSelectorInstance.get() : null,
            trustGateServiceInstance.isResolvable() ? trustGateServiceInstance.get() : null,
            capacityViewInstance.isResolvable() ? capacityViewInstance.get() : null,
            config,
            orgAwareDispatchInstance.isResolvable() ? orgAwareDispatchInstance.get() : null);
}
```

- [ ] **Step 9: Update RuntimeAutoConfiguration Spring bean**

In `RuntimeAutoConfiguration.java` around line 340, add `Optional<OrgAwareDispatch>` parameter and pass to the constructor.

- [ ] **Step 10: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=RoutingBridgeTest -pl runtime`
Expected: all tests PASS including the 4 new org-aware tests and all 15 existing tests.

- [ ] **Step 11: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -T 1C`
Expected: BUILD SUCCESS — verifies runtime-spring and all downstream modules compile.

- [ ] **Step 12: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/qhorus/runtime/message/RoutingBridge.java \
        runtime/src/test/java/io/casehub/qhorus/runtime/message/RoutingBridgeTest.java \
        runtime/src/main/java/io/casehub/qhorus/runtime/RuntimeBeans.java \
        runtime-spring/src/main/java/io/casehub/qhorus/runtime/spring/RuntimeAutoConfiguration.java
git commit -m "feat(#494): prefer same-unit agents in RoutingBridge routing

Wire OrgAwareDispatch.teamMembers() into RoutingBridge.resolve() to
filter capability matches to same-unit peers first, falling back to
full list when no peer qualifies.

Refs #494"
```

## References

- [2026-10-30-org-aware-task-assignment-design.md] — design spec
- [runtime-core/.../RoutingBridge.java] — integration point
- [runtime/.../RoutingBridgeTest.java] — existing test patterns (CDI-free, Mockito)
- [runtime/.../RuntimeBeans.java:282-294] — CDI producer
- [io.casehub.eidos.api.OrgAwareDispatch] — eidos API contract
- [GitHub #494] — focal issue

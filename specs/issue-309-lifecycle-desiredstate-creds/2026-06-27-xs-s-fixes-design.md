# XS/S Fixes: Lifecycle, DesiredState Bridge, Credential Migration

**Date:** 2026-06-27
**Issues:** #309, #287, #308
**Branch:** `issue-309-lifecycle-desiredstate-creds`

---

## #309 — `isActive()` on CommitmentState

### Problem

`CommitmentState` defines `isTerminal()` but not `isActive()`. The Lifecycle Coherence Protocol (LIFECYCLE.md Rule 4) requires both methods on all registered state machines. `PlanItemStatus` (engine) and `WorkItemStatus` (work) both define `isActive()`. CommitmentState is the only registered enum without it.

### Design

Add `isActive()` with explicit enumeration of the two non-terminal states:

```java
public boolean isActive() {
    return this == OPEN || this == ACKNOWLEDGED;
}
```

Explicit enumeration (not `!isTerminal()`) matches the reference implementations in engine and work. Adding a new state without updating both methods gives a silent `false` — caught by a complementary test.

### Test

`CommitmentStateTest` — verifies:
1. Every `CommitmentState` value is classified by exactly one of `isTerminal()` or `isActive()`.
2. `isActive()` returns `true` for OPEN and ACKNOWLEDGED, `false` for all terminal states.

Lives in `api/src/test/` — the enum is in the API module.

### Files changed

- `api/src/main/java/io/casehub/qhorus/api/message/CommitmentState.java` — add `isActive()`
- `api/src/test/java/io/casehub/qhorus/api/message/CommitmentStateTest.java` — new test

---

## #287 — `casehub-qhorus-desiredstate` bridge module

### Problem

casehub-ops ships a `ChannelDriftChecker` that compares only 4 mutable fields (allowedTypes, deniedTypes, rateLimitPerChannel, rateLimitPerInstance). It cannot check immutable fields (semantic) or connector binding state because that requires access to qhorus runtime stores.

### Design

A new `desiredstate/` submodule — single-class Jandex library module following the optional-module-pattern protocol.

**Module:** `casehub-qhorus-desiredstate`
**Package:** `io.casehub.qhorus.desiredstate`

**Dependencies:**
- `casehub-ops-api` (`provided`) — `NodeDriftChecker`, `ChannelNodeSpec`, `DeploymentNodeSpec`, `NodeStatus`
- `casehub-desiredstate-api` (`provided`) — `NodeSpec`, `NodeStatus` (transitive via ops-api, but explicit for clarity)
- `casehub-qhorus-api` — `ChannelSemantic`, `MessageType`
- `casehub-qhorus` — `ChannelStore`, `ChannelBindingStore`, `Channel`
- Jandex plugin for CDI discovery
- `casehub-platform` (`test`) — `MockCurrentPrincipal`

**Class:** `QhorusChannelDriftChecker`

```java
@Alternative
@Priority(1)
@ApplicationScoped
public class QhorusChannelDriftChecker implements NodeDriftChecker {

    @Override
    public String nodeType() { return "channel"; }

    @Override
    public NodeStatus check(NodeSpec spec, String tenancyId) { ... }
}
```

Injects `ChannelStore` and `ChannelBindingStore`. Lookup by `spec.nodeId()` (channel name) via `ChannelStore.findByName()`.

**Fields compared:**

| Field | Source | Category |
|-------|--------|----------|
| semantic | `Channel.semantic` vs `ChannelNodeSpec.semantic()` | Immutable |
| description | `Channel.description` vs `ChannelNodeSpec.description()` | Mutable |
| allowedTypes | `Channel.allowedTypes` (CSV) vs `ChannelNodeSpec.allowedTypes()` (Set) | Mutable |
| deniedTypes | `Channel.deniedTypes` (CSV) vs `ChannelNodeSpec.deniedTypes()` (Set) | Mutable |
| allowedWriters | `Channel.allowedWriters` vs `ChannelNodeSpec.allowedWriters()` | Mutable |
| adminInstances | `Channel.adminInstances` vs `ChannelNodeSpec.adminInstances()` | Mutable |
| barrierContributors | `Channel.barrierContributors` vs `ChannelNodeSpec.barrierContributors()` | Mutable |
| rateLimitPerChannel | `Channel.rateLimitPerChannel` vs `ChannelNodeSpec.rateLimitPerChannel()` | Mutable |
| rateLimitPerInstance | `Channel.rateLimitPerInstance` vs `ChannelNodeSpec.rateLimitPerInstance()` | Mutable |
| inboundConnectorId | `ChannelBindingStore` vs `ChannelNodeSpec.inboundConnectorId()` | Binding |
| externalKey | `ChannelBindingStore` vs `ChannelNodeSpec.externalKey()` | Binding |
| outboundConnectorId | `ChannelBindingStore` vs `ChannelNodeSpec.outboundConnectorId()` | Binding |
| outboundDestination | `ChannelBindingStore` vs `ChannelNodeSpec.outboundDestination()` | Binding |

**Not compared:** `paused` (operational state), `autoCreated` (metadata), `tenancyId` (parameter), timestamps, Slack-specific bindings (separate module concern).

**Type comparison helpers:** `MessageType.parseTypes(csv)` converts stored CSV to `Set<MessageType>` for comparison with the spec's typed sets. Null-safe: both-null means match; null vs empty-set means match (open channel).

### Test

`QhorusChannelDriftCheckerTest` — CDI-free unit test using `InMemoryChannelStore` and `InMemoryChannelBindingStore` from `casehub-qhorus-testing`.

Scenarios:
1. Channel absent → `ABSENT`
2. All fields match → `PRESENT`
3. Mutable field drift (each field) → `DRIFTED`
4. Immutable field drift (semantic) → `DRIFTED`
5. Connector binding drift → `DRIFTED`
6. Connector binding absent in actual but present in spec → `DRIFTED`
7. Non-channel spec type → `UNKNOWN`

### Files changed

- `desiredstate/pom.xml` — new module
- `desiredstate/src/main/java/io/casehub/qhorus/desiredstate/QhorusChannelDriftChecker.java`
- `desiredstate/src/test/java/io/casehub/qhorus/desiredstate/QhorusChannelDriftCheckerTest.java`
- `pom.xml` — add `<module>desiredstate</module>` (after `runtime` — depends on it)

### Cross-repo impact

PLATFORM.md Cross-Repo Dependency Map needs a new row:
`casehub-ops-api` consumed by `casehub-qhorus` `desiredstate` — `NodeDriftChecker` SPI.

---

## #308 — Slack credential migration to CredentialResolver

### Problem

`SlackChannelBackend.resolveToken()` and `SlackBindingResource.put()` use `org.eclipse.microprofile.config.Config` directly with a module-scoped key prefix (`casehub.qhorus.slack-channel.credentials.<workspaceId>`). This is Tier 1.5 — per-binding credential resolution without platform standardisation. The platform `CredentialResolver` SPI (platform#103) now provides a standard way to resolve credentials from the `casehub.credentials.<ref>` namespace.

### Design

**SlackChannelBackend:**
- Replace `Config config` constructor parameter with `CredentialResolver credentialResolver`
- `resolveToken(String workspaceId)` calls `credentialResolver.resolve(workspaceId)` and extracts `CredentialPropertyKeys.BEARER_TOKEN`
- Throws `NoSuchElementException` if token is null or blank (preserves existing error contract)

**SlackBindingResource:**
- Replace `Config config` constructor parameter with `CredentialResolver credentialResolver`
- Validation in `put()` uses `credentialResolver.resolve(req.workspaceId())` and checks for `BEARER_TOKEN` presence/non-blank
- Error messages updated to reference `casehub.credentials.<workspaceId>` namespace

**SlackBotBinding:**
- No changes. `workspaceId` remains the credential ref — Slack tokens are per-workspace.

**Dependencies (slack-channel/pom.xml):**
- Add `casehub-platform-api` as explicit compile dependency (already transitive, but direct import of `CredentialResolver` warrants explicit declaration)
- Remove `microprofile-config-api` from `<scope>provided</scope>` — no longer directly imported. Still available transitively through CDI and JPA.

**Config namespace change:**
- Old: `casehub.qhorus.slack-channel.credentials.<workspaceId>=xoxb-...`
- New: `casehub.credentials.<workspaceId>=xoxb-...`
- Or compound: `casehub.credentials.<workspaceId>.bearer-token=xoxb-...`
- Breaking change for deployers. `DefaultCredentialResolver` supports both simple (bare key → BEARER_TOKEN) and compound (sub-keys) modes.

### Test

Update `SlackChannelBackendTest` and `SlackBindingResourceTest` (if it exists — may need `SlackBindingResourceTest` to be created) to inject/mock `CredentialResolver` instead of `Config`.

### Files changed

- `slack-channel/pom.xml` — add `casehub-platform-api` compile, remove `microprofile-config-api`
- `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java` — replace Config with CredentialResolver
- `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackBindingResource.java` — replace Config with CredentialResolver
- `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java` — update mocks
- Test application.properties — rename credential keys to `casehub.credentials.*`

### Protocol update

The `per-binding-credential-reference.md` protocol in casehub/garden should be updated to note that `CredentialResolver` is now available and Tier 1.5 modules should migrate. The reference implementation line should be updated from "SlackBotBinding.credentialRef + SlackChannelBackend.resolveToken()" to "SlackChannelBackend injects CredentialResolver".

---

## Implementation Order

1. **#309** — XS, no dependencies, foundational
2. **#287** — S, independent of #308, new module
3. **#308** — S, independent of #287, migration in existing module

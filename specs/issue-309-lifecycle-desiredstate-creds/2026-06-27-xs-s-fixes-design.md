# XS/S Fixes: Lifecycle, Credential Migration

**Date:** 2026-06-27
**Issues:** #309, #308
**Branch:** `issue-309-lifecycle-desiredstate-creds`

---

## #287 — Removed from scope (wrong repo)

The original spec proposed a `desiredstate/` bridge module in qhorus implementing `NodeDriftChecker` with `@Alternative @Priority(1)`. This was architecturally wrong — it creates an upward coupling from qhorus (Foundation tier) to casehub-ops-api (Integration tier).

The existing `ChannelDriftChecker` already lives in `casehub-ops/deployment/` and already depends on `casehub-qhorus` runtime (`provided` scope). The dependency direction is correct (ops→qhorus, downward). The enrichment — comparing all 13+ fields instead of 4, adding connector binding checks — belongs in that existing checker, not in a new qhorus module.

The cross-foundation-bridge-module-placement protocol does not apply here — it governs "peer foundation modules," and casehub-ops is Integration tier, not a peer.

**Action:** Close qhorus#287 with a redirect to a new casehub-ops issue. The ops issue should include:

1. Enrich `ChannelDriftChecker` to compare all `ChannelNodeSpec` fields (description, semantic, allowedWriters, adminInstances, barrierContributors, plus the 4 mutable fields already checked).
2. Add connector binding drift detection via `ChannelBindingStore.findByChannelId()`.
3. Fix the tenancy gap: replace `ChannelLookup.findByName(name)` with `CrossTenantChannelStore.findByNameAndTenancy(name, tenancyId)` — the `tenancyId` parameter is already passed to `check()` but currently ignored.
4. Fix CSV string comparison for `allowedWriters`, `adminInstances`, `barrierContributors`: compare as sorted sets (split, sort, compare) rather than `Objects.equals()` on raw strings — `"agent-a,agent-b"` and `"agent-b,agent-a"` are semantically equivalent.
5. Test reverse binding asymmetry: binding present in actual but absent in spec is also `DRIFTED`.
6. Update PLATFORM.md line 413 to remove the "qhorus#287" bridge reference.

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

### Cross-repo update

`parent/docs/LIFECYCLE.md` — update CommitmentState registration row to list both `isTerminal()` and `isActive()` in the "Terminal check method" column (currently shows `isTerminal()` only). Per LIFECYCLE.md Rule 2 step 3.

---

## #308 — Slack credential migration to CredentialResolver

### Problem

`SlackChannelBackend.resolveToken()` and `SlackBindingResource.put()` use `org.eclipse.microprofile.config.Config` directly with a module-scoped key prefix (`casehub.qhorus.slack-channel.credentials.<workspaceId>`). This is Tier 1.5 — per-binding credential resolution without platform standardisation. The platform `CredentialResolver` SPI (platform#103) now provides a standard way to resolve credentials from the `casehub.credentials.<ref>` namespace.

### Design

**SlackChannelBackend:**
- Replace `Config config` constructor parameter with `CredentialResolver credentialResolver`
- Remove `import org.eclipse.microprofile.config.Config`
- `resolveToken(String workspaceId)` calls `credentialResolver.resolve(workspaceId)` and extracts `CredentialPropertyKeys.BEARER_TOKEN`

**Error contract change:** `Config.getValue()` throws `NoSuchElementException` when a key is missing. `CredentialResolver.resolve()` never throws — it returns `Map.of()` for missing refs. The new `resolveToken()` must explicitly throw `NoSuchElementException` when the `BEARER_TOKEN` key is absent or blank from the resolved map. The throwing responsibility shifts from Config to our code.

```java
String resolveToken(String workspaceId) {
    Map<String, String> creds = credentialResolver.resolve(workspaceId);
    String token = creds.get(CredentialPropertyKeys.BEARER_TOKEN);
    if (token == null || token.isBlank()) {
        throw new NoSuchElementException("No bearer-token for credential ref: " + workspaceId);
    }
    return token;
}
```

**SlackBindingResource:**
- Replace `Config config` constructor parameter with `CredentialResolver credentialResolver`
- Remove `import org.eclipse.microprofile.config.Config`
- Validation in `put()` uses `credentialResolver.resolve(req.workspaceId())` and checks for `BEARER_TOKEN` presence/non-blank. Error messages updated to reference `casehub.credentials.<workspaceId>` namespace.

**SlackBotBinding:**
- Field unchanged. `workspaceId` remains the credential ref — Slack tokens are per-workspace.
- Javadoc update only: `workspaceId` field comment references "MicroProfile Config credential key" — update to "CredentialResolver credential ref".

**Dependencies (slack-channel/pom.xml):**
- Add `casehub-platform-api` as explicit compile dependency (already transitive via `casehub-qhorus-api`, but direct import of `CredentialResolver` warrants explicit declaration)
- Remove `microprofile-config-api` from `<scope>provided</scope>` — no main source file imports it after migration. Confirmed: only `SlackChannelBackend.java` and `SlackBindingResource.java` import `Config`; both are migrated.

**Config namespace change:**
- Old: `casehub.qhorus.slack-channel.credentials.<workspaceId>=xoxb-...`
- New: `casehub.credentials.<workspaceId>=xoxb-...`
- Or compound: `casehub.credentials.<workspaceId>.bearer-token=xoxb-...`
- Breaking change for deployers. `DefaultCredentialResolver` supports both simple (bare key → BEARER_TOKEN) and compound (sub-keys) modes.

### Test

Update existing tests to mock `CredentialResolver` instead of `Config`:

- **`SlackChannelBackendTest`** — replace `Config` mock with `CredentialResolver` mock. `resolveToken()` tests verify: (a) successful resolution returns token from `BEARER_TOKEN` key; (b) empty map from resolve → `NoSuchElementException`; (c) blank token value → `NoSuchElementException`.
- **`SlackBindingResourceTest`** — has 6 test methods covering all validation paths. Replace `Config config` field with `CredentialResolver credentialResolver`. Update `credKey` references to use `CredentialResolver.resolve()` mock setup. The `put_missingCredential_returns400_beforeSave` and `put_blankCredential_returns400_beforeSave` tests verify the same error paths but through the new API.
- `slack-channel/src/test/resources/application.properties` — rename `casehub.qhorus.slack-channel.credentials.T_TEST` to `casehub.credentials.T_TEST` (line 35).

### Files changed

- `slack-channel/pom.xml` — add `casehub-platform-api` compile, remove `microprofile-config-api`
- `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackChannelBackend.java` — replace Config with CredentialResolver
- `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackBindingResource.java` — replace Config with CredentialResolver
- `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackChannelBackendTest.java` — update mocks
- `slack-channel/src/test/java/io/casehub/qhorus/slack/SlackBindingResourceTest.java` — update mocks
- `slack-channel/src/main/java/io/casehub/qhorus/slack/SlackBotBinding.java` — javadoc update on `workspaceId` field
- `slack-channel/src/test/resources/application.properties` — rename credential key to `casehub.credentials.*`

### Protocol update

The `per-binding-credential-reference.md` protocol in casehub/garden should be updated to note that `CredentialResolver` is now available and Tier 1.5 modules should migrate. The reference implementation line should be updated from "SlackBotBinding.credentialRef + SlackChannelBackend.resolveToken()" to "SlackChannelBackend injects CredentialResolver".

---

## Implementation Order

1. **#309** — XS, no dependencies, foundational
2. **#308** — S, credential migration in existing module
3. **#287** — Removed from qhorus scope; redirect to casehub-ops issue

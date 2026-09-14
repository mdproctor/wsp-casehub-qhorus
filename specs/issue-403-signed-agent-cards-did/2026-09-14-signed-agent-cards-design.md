# E6 (First Pass): Signed Agent Cards

**Issue:** casehubio/qhorus#403
**Date:** 2026-09-14
**Status:** Draft

## Overview

Adds JWS (RFC 7515) signing to agent cards served at `/.well-known/agent.json` and `/.well-known/agents/{instanceId}.json`, with signature verification on inbound agent card fetch at binding creation time, and a trust score contribution for cryptographically verified agents.

This is a **platform security extension** — cryptographic agent identity anchoring for the casehub ecosystem. The A2A protocol does not define agent card signing; the `signatures` envelope and JWKS endpoint are casehub-specific. Signing uses the existing platform `SigningProvider` SPI (promoted to `casehub-platform-api` by platform#244 specifically for this use case), following the algorithm-transparent signing protocol (PP-20260523-e7b577).

The design builds on existing platform identity infrastructure: `ActorDIDProvider` for DID resolution, `VerificationMethod` and `VerificationMethodType` for key material typing, and `SigningProvider` for raw cryptographic operations. Deferred DID work (did:web, did:key) extends these existing SPIs rather than creating parallel infrastructure.

This is the first implementation pass of E6. DID anchoring, verifiable credential issuance, and per-agent signing keys are deferred to child issues — they build on this foundation without requiring rework.

## Scope

**In scope:**
- `AgentCardSigner` SPI in `casehub-qhorus-api` (sign + verify + JWKS)
- `agent-card-signing` optional module with JWS implementation (Nimbus JOSE+JWT), using platform `SigningProvider` for raw crypto
- Outbound card signing in `AgentCardResource` — signed JSON envelope wrapping `AgentCard`
- JWKS endpoint at `/.well-known/jwks.json` with Cache-Control and CORS headers
- Inbound card verification on binding creation/update in `ExternalAgentBindingResource`
- `ExternalAgentBinding` extended with verification status fields
- `identity-verification` trust score dimension via `@Decorator TrustScoreSource`

**Out of scope:**
- DID anchoring (did:web, did:key) — child issue (builds on existing `ActorDIDProvider`)
- Verifiable credential issuance for capabilities — child issue
- Per-agent signing keys — requires DID infrastructure
- Key rotation automation — operational concern, manual rotation supported via CredentialResolver
- Pluggable trust dimension provider SPI in ledger — child issue (replaces tactical decorator)

## Architecture

### Algorithm and Standards

| Standard | Usage |
|----------|-------|
| JWS (RFC 7515) | Signature envelope format |
| EdDSA / Ed25519 (RFC 8032, RFC 8037) | Default signing algorithm (per issue #403 and platform `SigningProvider`) |
| ES256 (ECDSA P-256) | Supported alternative (configure via key type) |
| JCS (RFC 8785) | JSON Canonicalization Scheme for deterministic signing input |
| JWK (RFC 7517) | Public key format for JWKS endpoint |

The signing algorithm is determined by the configured key material via `SigningProvider`, following the platform's algorithm-transparent signing protocol (PP-20260523-e7b577). No algorithm string is hardcoded. The default key type is Ed25519 (matching issue #403's specification: "Ed25519 via platform SigningService SPI"). Deployments using EC P-256 keys will produce ES256 signatures automatically.

The JWS protected header carries `alg`, `typ`, `kid`, and optionally `jku` pointing to the JWKS endpoint. The `typ` value `agentcard+jws` is a casehub-defined media type (not A2A-specified).

**Design decision:** Issue #403 specifies "Ed25519 via platform SigningService SPI." The original spec used ES256 based on a misattribution to A2A v1.0 sections 4.4.7 and 8.4. Those sections do not exist in the A2A specification — the A2A protocol does not define agent card signing. Since the algorithm choice is unconstrained by the protocol, Ed25519 (EdDSA) is the correct default: it matches the issue, aligns with `SigningProvider`'s design origin, and is supported by `VerificationMethodType.ED25519` in the platform identity infrastructure.

### Key Management

Platform-level — one key pair per deployment/tenant, managed via the platform `CredentialResolver` SPI. The platform signs all agent cards centrally. Key material is never exposed in agent card responses, ledger entries, or logs (per protocol PP-20260612-bd6f8c).

The key pair type (Ed25519 or EC P-256) determines the JWS algorithm automatically via `SigningProvider`'s algorithm-transparent design.

### Canonicalization

Before signing, the `AgentCard` is serialized to JSON (without any `signatures` field — the card record has no signatures), then canonicalized via JCS (RFC 8785). JCS produces a deterministic JSON byte sequence by:
- Sorting object keys lexicographically
- Normalizing numbers (no trailing zeros, no leading plus)
- Normalizing strings (shortest UTF-8 escape sequences)
- No whitespace

The canonical form is the JWS payload. Verification re-canonicalizes the received card (minus signatures) and verifies against the JWS signature.

## SPI Contract

### AgentCardSigner (casehub-qhorus-api)

```java
package io.casehub.qhorus.api.spi;

import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.a2a.model.AgentCard;

public interface AgentCardSigner {

    ObjectNode sign(AgentCard card);

    VerificationResult verify(ObjectNode signedCardJson);

    ObjectNode jwks();

    record VerificationResult(boolean verified, String keyId, String error) {}
}
```

- `sign()` takes an `AgentCard`, serializes and canonicalizes it, signs with `SigningProvider`, and returns a `ObjectNode` containing all card fields plus a `signatures` array
- `verify()` takes a signed card JSON node, extracts signatures, re-canonicalizes the card payload, and verifies via JWKS resolution
- `jwks()` returns the JWKS document for the `/.well-known/jwks.json` endpoint
- `AgentCard` record is unchanged — no `signatures` field. The signatures are part of the JWS envelope, not the protocol model

The `AgentCardResource` injects `Instance<AgentCardSigner>` and checks `isResolvable()` — the same optional-module pattern used for `PushNotificationConfigStore`. When the signing module is absent, cards are served unsigned (plain `AgentCard` serialization).

### AgentCard — no changes

The `AgentCard` record in `casehub-a2a-protocol` is **not modified**. Agent card signing is a casehub platform extension, not an A2A protocol feature. Putting signing-specific types (`AgentCardSignature`, `signatures` field) into the Layer 0 protocol model would be architectural contamination — coupling a pure protocol module to a platform security extension.

The signing module works at the JSON level: it wraps the serialized `AgentCard` with a `signatures` array in the output JSON, and strips it before verification. The `AgentCard` record stays pure A2A.

For inbound verification, the signing module operates on the raw `ObjectNode` — it extracts and removes the `signatures` array before re-canonicalizing the card payload for signature verification. The `AgentCard` record is not deserialized during verification; the canonical JSON (minus signatures) is compared directly against the signature. This avoids any dependency on Jackson's unknown-property handling behavior.

### ExternalAgentBinding Changes

New fields on the `ExternalAgentBinding` record:

| Field | Type | Description |
|-------|------|-------------|
| `verificationStatus` | `VerificationStatus` enum | `VERIFIED`, `UNVERIFIED`, `FAILED` |
| `verifiedAt` | `Instant` (nullable) | Timestamp of last successful verification |
| `verificationKeyId` | `String` (nullable) | The `kid` that verified the card |

```java
public enum VerificationStatus {
    VERIFIED, UNVERIFIED, FAILED
}
```

**Scope of change:** `ExternalAgentBinding` is a Java record (6 → 9 components). This changes the canonical constructor and requires updating all construction sites:
- `ExternalAgentBindingEntity.toDomain()` and `fromDomain()` in `runtime/`
- `ExternalAgentBindingResource.put()` in `a2a-outbound/`
- `InMemoryExternalAgentBindingStore` in `persistence-memory/`
- `ExternalAgentBindingResourceTest` and `ExternalAgentBindingStoreContractTest`
- `A2AOutboundBackendTest` (any binding construction)

No backward-compatible constructor — the platform has no end users; breaking changes are the point, forcing every caller to be explicit about verification state.

**Flyway migration** in `runtime/src/main/resources/db/qhorus/migration/` (following Flyway consumer versioning protocol PP-20260521-0ba358):

```sql
-- V54__external_agent_binding_verification.sql
ALTER TABLE external_agent_binding ADD COLUMN verification_status VARCHAR(20) DEFAULT 'UNVERIFIED';
ALTER TABLE external_agent_binding ADD COLUMN verified_at TIMESTAMP;
ALTER TABLE external_agent_binding ADD COLUMN verification_key_id VARCHAR(255);
```

The entity, record, store interface, and Flyway migration are all in their established locations:
- Record: `api/src/main/java/io/casehub/qhorus/api/instance/ExternalAgentBinding.java`
- Entity: `runtime/src/main/java/io/casehub/qhorus/runtime/instance/ExternalAgentBindingEntity.java`
- Migration: `runtime/src/main/resources/db/qhorus/migration/V54__external_agent_binding_verification.sql`
- FK constraint `fk_eab_instance` ties this to the qhorus runtime schema

## Module Structure

### agent-card-signing/ (new optional module)

```
agent-card-signing/
├── pom.xml
└── src/main/java/io/casehub/qhorus/signing/
    ├── JwsAgentCardSigner.java          — @ApplicationScoped AgentCardSigner impl
    ├── JwsKeyProvider.java              — key pair from SigningProvider
    ├── JcsCanonicalizer.java            — RFC 8785 canonicalization
    ├── JwksCache.java                   — remote JWKS fetching with caching
    ├── SigningConfig.java               — @ConfigMapping
    ├── BindingVerificationObserver.java  — @ObservesAsync handler for async verification
    └── IdentityVerificationTrustDecorator.java — @Decorator TrustScoreSource
```

**Event class:** `BindingVerificationRequestedEvent` lives in `casehub-qhorus-api` (`api/src/main/java/io/casehub/qhorus/api/event/`) — both `a2a-outbound` (fires it in `ExternalAgentBindingResource`) and `agent-card-signing` (observes it in `BindingVerificationObserver`) depend on `casehub-qhorus-api`, so the event type is visible to both.

Activates by classpath presence. `JwsAgentCardSigner` is `@ApplicationScoped` — when present, `Instance<AgentCardSigner>.isResolvable()` returns true in `AgentCardResource`.

**Dependencies:**
- `casehub-qhorus-api` (AgentCardSigner SPI)
- `casehub-a2a-protocol` (AgentCard — for serialization)
- `casehub-platform-api` (SigningProvider, CredentialResolver)
- `casehub-ledger-api` (TrustScoreSource — for decorator)
- `com.nimbusds:nimbus-jose-jwt` (JWS implementation)

**Key architectural relationship:** `JwsAgentCardSigner` uses `SigningProvider.sign(actorId, canonicalBytes)` for the raw cryptographic operation, then wraps the `SignatureResult` in a JWS envelope (base64url-encoded protected header + signature). This follows the same pattern as `ComplianceReportSigningService` wrapping `DocumentSigningService` — application-level orchestration atop a platform signing SPI. The `NoOpSigningProvider` (`@DefaultBean`) means signing is no-op when no signing backend is configured, even with the agent-card-signing module on the classpath.

### Configuration

```properties
# Signing
casehub.qhorus.signing.enabled=true
casehub.qhorus.signing.actor-id=system:agent-card-signer
casehub.qhorus.signing.key-id=default
casehub.qhorus.signing.jwks-url=

# Inbound JWKS caching
casehub.qhorus.signing.jwks-cache.ttl=3600
casehub.qhorus.signing.jwks-cache.max-response-bytes=65536
casehub.qhorus.signing.jwks-cache.connect-timeout-ms=5000
casehub.qhorus.signing.jwks-cache.read-timeout-ms=5000

# Trust integration
casehub.qhorus.signing.trust.verified-floor-score=0.6
casehub.qhorus.signing.trust.dimension-weight=0.15
```

| Property | Default | Description |
|----------|---------|-------------|
| `enabled` | `true` | Gate — `false` disables signing even with module on classpath |
| `actor-id` | `system:agent-card-signer` | actorId passed to `SigningProvider.sign()` for key resolution |
| `key-id` | `default` | `kid` value in JWS protected header and JWKS |
| `jwks-url` | (empty) | Override for `jku` in JWS header; defaults to `/.well-known/jwks.json` on self |
| `jwks-cache.ttl` | `3600` | Cache TTL in seconds for remote JWKS responses |
| `jwks-cache.max-response-bytes` | `65536` | Maximum JWKS response size (DoS protection) |
| `jwks-cache.connect-timeout-ms` | `5000` | HTTP connect timeout for JWKS fetch |
| `jwks-cache.read-timeout-ms` | `5000` | HTTP read timeout for JWKS fetch |
| `trust.verified-floor-score` | `0.6` | Floor trust score for verified agents in the `identity-verification` dimension |
| `trust.dimension-weight` | `0.15` | Additive boost weight: `globalScore += floor × weight` for verified agents (clamped at 1.0) |

## Data Flow

### Outbound Signing (AgentCardResource)

```
1. AgentCardResource.getAgentCard()
2.   → build AgentCard (existing logic, unchanged record)
3.   → if AgentCardSigner is resolvable:
4.       → AgentCardSigner.sign(card)
5.         → serialize AgentCard to JSON via ObjectMapper
6.         → JcsCanonicalizer.canonicalize(cardJson) → canonical bytes
7.         → SigningProvider.sign(actorId, canonicalBytes)
8.           → if NoOpSigningProvider: return Optional.empty() → card served unsigned
9.           → if signing backend configured: sign with private key → SignatureResult
10.        → derive JWS alg from SignatureResult.publicKey():
11.            → parse public key bytes (X.509 SubjectPublicKeyInfo format) as JCA PublicKey
12.            → construct Nimbus JWK from JCA PublicKey (OctetKeyPair for Ed25519, ECKey for P-256)
13.            → read kty + crv from JWK → map to alg: OKP/Ed25519 → "EdDSA", EC/P-256 → "ES256"
14.            → (this JWK is the same object served by the jwks() endpoint — constructed once, reused)
15.        → build JWS protected header {alg:<derived>, typ:agentcard+jws, kid:default, jku:/.well-known/jwks.json}
16.        → base64url-encode protected header and signature bytes
17.        → construct ObjectNode: all AgentCard fields + signatures array
18.        → return signed ObjectNode
19.   → if AgentCardSigner not resolvable:
20.       → return AgentCard directly (plain JSON serialization, no signatures)
```

**Algorithm derivation:** `SigningProvider` is algorithm-transparent (PP-20260523-e7b577) — `SignatureResult` carries raw `publicKey` bytes but no algorithm identifier. The JWS `alg` header is derived from the public key's JWK representation: the same JWK constructed for the `/.well-known/jwks.json` endpoint encodes the key type (`kty`+`crv`), which maps deterministically to a JWS algorithm. The `JwsKeyProvider` constructs this JWK once at startup from `SigningProvider.keyMaterial(actorId)` and caches it. Nimbus JOSE+JWT's `JWK.parse(publicKey)` handles the X.509 SubjectPublicKeyInfo → JWK conversion natively.

Same flow applies to per-agent cards at `/.well-known/agents/{instanceId}.json`.

### Inbound Verification (ExternalAgentBindingResource)

Verification is triggered by binding creation/update (`PUT /a2a-outbound/bindings/{instanceId}`) but runs **asynchronously** — the binding is stored immediately with `UNVERIFIED` status and the PUT returns without waiting for verification. This ensures binding availability is never coupled to remote endpoint responsiveness.

```
PUT flow (synchronous — immediate return):

1. ExternalAgentBindingResource.put(instanceId, request)
2.   → create/update ExternalAgentBinding with verificationStatus = UNVERIFIED
3.   → store binding
4.   → if AgentCardSigner is resolvable:
5.       → fire CDI event BindingVerificationRequestedEvent(binding)
6.   → return binding to caller (status: UNVERIFIED)
```

```
Verification flow (async observer — same logic for PUT trigger and POST /verify):

1. BindingVerificationObserver.onVerificationRequested(@ObservesAsync event)
2.   → HTTP GET {binding.endpoint}/.well-known/agent.json
3.   → if fetch fails (timeout, 4xx, 5xx):
4.       → binding.verificationStatus = UNVERIFIED
5.       → LOG.info("Could not fetch agent card for verification: {}", endpoint)
6.       → update binding and return
7.   → parse response as ObjectNode
8.   → if response has no "signatures" array or it is empty:
9.       → binding.verificationStatus = UNVERIFIED
10.      → update binding and return
11.  → AgentCardSigner.verify(signedCardJson):
12.      → extract jku from first signature's protected header
13.      → JwksCache.fetch(jku):
14.          → check in-memory cache (keyed by jku URL, TTL from config)
15.          → if cache miss: HTTP GET jku URL
16.              → enforce max-response-bytes limit
17.              → enforce connect-timeout-ms and read-timeout-ms
18.              → if fetch fails: return cached value if available (stale-while-error), else fail
19.          → parse JWKS, cache result
20.      → remove "signatures" from JSON → re-canonicalize via JCS
21.      → resolve public key from JWKS by kid in protected header
22.      → if kid not found in JWKS:
23.          → invalidate JWKS cache for this jku → re-fetch JWKS
24.          → retry kid lookup (handles key rotation)
25.          → if still not found: return VerificationResult(false, null, "kid not found")
26.      → verify signature against canonical bytes using resolved public key
27.      → return VerificationResult(verified, keyId, error)
28.  → if verified:
29.      → binding.verificationStatus = VERIFIED
30.      → binding.verifiedAt = Instant.now()
31.      → binding.verificationKeyId = result.keyId()
32.  → if not verified:
33.      → binding.verificationStatus = FAILED
34.      → LOG.warn("Agent card verification failed for {}: {}", endpoint, result.error())
35.  → update binding
```

JWKS fetching (steps 13-19) is internal to `AgentCardSigner.verify()` — the caller passes the signed card JSON and the signer handles jku extraction, JWKS resolution via `JwksCache`, kid lookup, and cryptographic verification. The SPI boundary is `verify(ObjectNode signedCardJson)` → `VerificationResult`.

**On-demand re-verification:**

```
POST /a2a-outbound/bindings/{instanceId}/verify → ExternalAgentBindingResource

Runs verification synchronously (blocking the POST response) so the caller
gets the updated status in the response body. Use cases:
- After remote agent key rotation
- After JWKS cache expiry
- Manual re-verification trigger

When AgentCardSigner is not resolvable (signing module absent):
- Returns 501 (Not Implemented) with body:
  {"error": "Agent card signing module not configured — verification unavailable"}
```

**Failure mode semantics:**
- `UNVERIFIED` — unsigned card, fetch failed, or signing module absent. Not an error.
- `VERIFIED` — card signature validated against JWKS-published key.
- `FAILED` — card has signatures but verification failed (tampered, bad key, expired). Logged as warning.
- Binding creation never blocks on verification (async observer). Message dispatch uses stored status.

### JWKS Endpoint

```
GET /.well-known/jwks.json → AgentCardResource

Response headers:
  Content-Type: application/json
  Cache-Control: public, max-age=86400
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET
  Access-Control-Allow-Headers: Accept

Response body:
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "default",
    "use": "sig",
    "x": "<base64url>"
  }]
}
```

The key type (`OKP`/`Ed25519` or `EC`/`P-256`) is determined by the configured key material. The JWKS endpoint is unauthenticated (public keys are public by definition). Rate limiting recommended in production.

When `AgentCardSigner` is not resolvable (signing module absent), the JWKS endpoint returns 404.

## Trust Score Integration

### IdentityVerificationTrustDecorator

```java
@jakarta.decorator.Decorator
@Priority(1000)
public class IdentityVerificationTrustDecorator implements TrustScoreSource {

    @Inject @Delegate @Any
    TrustScoreSource delegate;

    @Inject
    ExternalAgentBindingStore bindingStore;

    @Inject
    SigningConfig config;

    @Override
    public OptionalDouble globalScore(String actorId) {
        OptionalDouble base = delegate.globalScore(actorId);
        OptionalDouble identity = computeIdentityScore(actorId);
        if (identity.isEmpty()) return base;
        double baseVal = base.orElse(0.0);
        double boost = identity.getAsDouble() * config.trust().dimensionWeight();
        return OptionalDouble.of(Math.min(1.0, baseVal + boost));
    }

    @Override
    public OptionalDouble dimensionScore(String actorId, String dimensionKey) {
        if ("identity-verification".equals(dimensionKey)) {
            return computeIdentityScore(actorId);
        }
        return delegate.dimensionScore(actorId, dimensionKey);
    }

    @Override
    public Map<String, Double> allDimensionScores(String actorId) {
        Map<String, Double> scores = new LinkedHashMap<>(delegate.allDimensionScores(actorId));
        computeIdentityScore(actorId).ifPresent(s -> scores.put("identity-verification", s));
        return scores;
    }

    // All other methods delegate unchanged to the underlying TrustScoreSource

    // ASSUMPTION: actorId == instanceId for external agents.
    // True with DefaultInstanceActorIdProvider (identity mapping, the only production impl).
    // Custom InstanceActorIdProvider that transforms external agent IDs will break this
    // lookup — the SPI is one-way (instanceId → actorId), no reverse mapping exists.
    // The pluggable TrustDimensionContributor child issue should resolve this by receiving
    // the instanceId directly rather than going through the actorId namespace.
    private OptionalDouble computeIdentityScore(String actorId) {
        return bindingStore.findByInstanceId(actorId)
            .filter(b -> b.verificationStatus() == VerificationStatus.VERIFIED)
            .map(b -> OptionalDouble.of(config.trust().verifiedFloorScore()))
            .orElse(OptionalDouble.empty());
    }
}
```

**`globalScore()` override — additive boost model:** The identity-verification dimension contributes an additive boost to the base global score: `min(1.0, base + floor × weight)`. With defaults (`floor=0.6, weight=0.15`), a verified agent gets a `+0.09` boost. This model ensures verification never penalizes agents whose attestation-based score exceeds the floor. A weighted-average model (`base × (1-w) + identity × w`) would reduce the global score for any agent with `base > floor`, contradicting the "trust score bonus" intent from issue #403.

The base global score from `ComputedTrustScoreSource`/`CachedTrustScoreSource` is computed independently from dimension scores — it uses a Beta model over attestation history via `TrustScoreCalculator`. The global score and dimension scores are NOT derived from each other. The decorator must therefore explicitly incorporate the identity dimension into `globalScore()` — delegating `globalScore()` unchanged would make the identity dimension invisible to all routing consumers.

The `@Decorator` correctly wraps whatever `TrustScoreSource` implementation is active (Computed, Cached, or Materialized) without displacing it. This fixes the `@Alternative` circular-dependency problem in the original design.

**Architectural note:** The identity-verification dimension is fundamentally different from attestation-based dimensions (quality, capability) — it's a static property derived from `ExternalAgentBinding.verificationStatus`, not computed from ledger attestation history. The `@Decorator` is a tactical integration for E6 first pass. A child issue will propose a pluggable `TrustDimensionContributor` SPI in the ledger for cleanly composing heterogeneous trust signals without decorating the full `TrustScoreSource` interface.

**Composition with existing trust consumers:**
- `TrustGateService.meetsThreshold()` — reads `globalScore()` → now includes identity boost
- `TrustGateService.currentScore()` — reads `globalScore()` → now includes identity boost
- `RoutingBridge.lookupTrustScore()` — reads `currentScore()` via TrustGateService → now includes identity boost
- `DefaultObligorTrustPolicy` — delegates to `meetsThreshold()` → now includes identity boost
- `TrustScoreResource` — returns the identity dimension alongside existing dimensions via `allDimensionScores()`

## Testing Strategy

### Unit Tests (CDI-free)

| Test | What it validates |
|------|-------------------|
| `JcsCanonicalizerTest` | Deterministic canonicalization: nested objects, unicode, numeric edge cases, key ordering |
| `JwsAgentCardSignerTest` | Sign/verify round-trip with generated test Ed25519 keys; tampered payload detection; missing key handling; ES256 key support |
| `IdentityVerificationTrustDecoratorTest` | globalScore boost for VERIFIED agents; no boost for UNVERIFIED/FAILED; additive clamping at 1.0; dimension score for VERIFIED/UNVERIFIED/unknown; delegate passthrough for non-identity dimensions and all non-overridden methods |
| `JwksCacheTest` | Cache TTL expiry; cache invalidation on kid miss; max response size enforcement; timeout handling |

### Integration Tests (@QuarkusTest)

| Test | What it validates |
|------|-------------------|
| `SignedAgentCardTest` | GET `/.well-known/agent.json` returns signed JSON envelope when signing module active; unsigned AgentCard when absent |
| `JwksEndpointTest` | GET `/.well-known/jwks.json` returns valid JWKS with correct key parameters; 404 when signing absent; Cache-Control and CORS headers present |
| `BindingVerificationTest` | PUT binding with mock HTTP server serving signed card → ExternalAgentBinding.verificationStatus updated; unsigned card → UNVERIFIED; tampered card → FAILED |
| `ReVerificationTest` | POST `/verify` endpoint re-runs verification; JWKS cache invalidation on key rotation |
| `VerificationTrustIntegrationTest` | Verified agent gets identity-verification dimension score; unverified agent does not; decorator delegates all other dimensions unchanged |

### CDI-free test patterns

- `JwsAgentCardSigner` tests generate ephemeral Ed25519 key pairs (no CredentialResolver mock needed)
- `IdentityVerificationTrustDecorator` tests use a stub `TrustScoreSource` delegate and a stub `ExternalAgentBindingStore`
- `JcsCanonicalizer` tests are pure function tests — input JSON string → canonical output bytes

## Security Considerations

- **Key material in ledger:** Signing keys must never appear in `MessageDispatch.content` or any ledger-persisted field (protocol PP-20260612-bd6f8c). The `actor-id` config property identifies the signing actor; `SigningProvider` resolves the actual key material at runtime.
- **JWKS endpoint:** Public keys only. No private key material exposed. Rate limiting recommended in production.
- **Soft failure:** Unsigned or failed-verification cards are accepted but marked. The trust score adjustment is the only consequence — verification is never an access gate. This is intentional for ecosystem compatibility.
- **JWKS DoS protection:** Remote JWKS responses are bounded by `max-response-bytes` (default 64KB). Connection and read timeouts prevent hung connections.
- **Key compromise:** Platform-level key compromise affects all agent cards for that tenant. Rotation is manual via CredentialResolver (update key, re-deploy). Future DID work will add key rotation semantics.
- **Cache poisoning:** JWKS is fetched only from the `jku` URL declared in the JWS protected header. The `jku` URL should be validated against a trusted domain allowlist in production (future hardening).

## Future Work (Child Issues)

| Topic | Dependency | Notes |
|-------|------------|-------|
| DID anchoring (did:web) | This issue | Maps `/.well-known/` to `did:web:host`; extends existing `ActorDIDProvider` SPI with `didFor()` → did:web URI; adds DID Document endpoint |
| DID anchoring (did:key) | This issue | Self-certifying IDs for ephemeral agents; key is the DID; extends `ActorDIDProvider` with did:key resolution using `VerificationMethod.publicKeyBytes()` |
| Verifiable Credentials | DID anchoring | VC Data Model 2.0 credentials asserting agent capabilities; uses `VerificationMethodType` for proof type |
| Per-agent signing keys | DID anchoring | Each Instance gets its own key pair, anchored to a DID; `SigningProvider.sign(actorId, data)` already supports per-actor resolution |
| Key rotation automation | This issue | Scheduled key rotation with JWKS versioning |
| Pluggable trust dimension provider | This issue | Replace `@Decorator TrustScoreSource` with a `TrustDimensionContributor` SPI in `casehub-ledger-api`; enables heterogeneous trust signals (identity verification, credential status, network reputation) without decorating the full scoring interface |

**Existing platform infrastructure for deferred items:**
- `ActorDIDProvider` — `didFor(actorId)` + `invalidate(actorId)`, with implementations: `ConfiguredActorDIDProvider`, `CompositeActorDIDProvider`, `ScimActorDIDProvider`, `NoOpActorDIDProvider`
- `VerificationMethod(id, type, publicKeyBytes)` — key material with type discriminator
- `VerificationMethodType` — `ED25519`, `P256`, `SECP256K1` constants
- `SigningProvider.sign(actorId, data)` — already supports per-actor key resolution (deferred per-agent signing keys build on this directly)

Child issues must be filed on GitHub (not just noted here) for tracking and priority assignment.

## References

- [RFC 7515](https://tools.ietf.org/html/rfc7515) — JSON Web Signature
- [RFC 8032](https://tools.ietf.org/html/rfc8032) — Edwards-Curve Digital Signature Algorithm (Ed25519)
- [RFC 8037](https://tools.ietf.org/html/rfc8037) — CFRG Elliptic Curve Diffie-Hellman (ECDH) and Signatures in JOSE (EdDSA for JWS)
- [RFC 8785](https://tools.ietf.org/html/rfc8785) — JSON Canonicalization Scheme
- [RFC 7517](https://tools.ietf.org/html/rfc7517) — JSON Web Key
- [W3C DID v1.0](https://www.w3.org/TR/did-core/) — Decentralized Identifiers (future work)
- [W3C VC Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/) — Verifiable Credentials (future work)
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java` — existing resource
- `a2a-protocol/src/main/java/io/casehub/a2a/model/AgentCard.java` — Layer 0 agent card record (unchanged)
- `api/src/main/java/io/casehub/qhorus/api/spi/` — existing SPI pattern
- `compliance-report/.../ComplianceReportSigningService.java` — existing signing orchestration pattern (wraps `DocumentSigningService`)
- `io.casehub.platform.api.signing.SigningProvider` — platform raw signing SPI
- `io.casehub.platform.api.identity.ActorDIDProvider` — platform DID resolution SPI
- `io.casehub.platform.api.identity.VerificationMethod` — platform key material record
- Protocol PP-20260523-e7b577 — algorithm-transparent signing
- Protocol PP-20260612-bd6f8c — no credentials in ledger content
- Protocol PP-20260521-0ba358 — Flyway consumer versioning
- casehubio/qhorus#403 — parent epic
- casehubio/platform#244 — SigningProvider SPI promotion (completed)

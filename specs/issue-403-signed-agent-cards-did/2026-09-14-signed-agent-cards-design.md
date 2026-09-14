# E6 (First Pass): Signed Agent Cards

**Issue:** casehubio/qhorus#403
**Date:** 2026-09-14
**Status:** Draft

## Overview

Adds JWS (RFC 7515, ES256) signing to agent cards served at `/.well-known/agent.json` and `/.well-known/agents/{instanceId}.json`, with signature verification on inbound A2A card fetches and a trust score bonus for cryptographically verified agents.

This is the first implementation pass of E6. DID anchoring (did:web, did:key), verifiable credential issuance, and per-agent signing keys are deferred to child issues — they build on this foundation without requiring rework.

## Scope

**In scope:**
- `AgentCardSigningService` SPI in `casehub-qhorus-api` (sign + verify)
- `agent-card-signing` optional module with JWS implementation (Nimbus JOSE+JWT)
- `AgentCard` record extended with `signatures` field (`casehub-a2a-protocol`)
- `AgentCardResource` enhanced to sign outbound cards and serve JWKS at `/.well-known/jwks.json`
- Inbound card verification in `a2a-outbound` (soft failure — unsigned cards accepted)
- `identity-verification` trust score dimension via decorator `TrustScoreSource`

**Out of scope:**
- DID anchoring (did:web, did:key) — child issue
- Verifiable credential issuance for capabilities — child issue
- Per-agent signing keys — requires DID infrastructure
- Key rotation automation — operational concern, manual rotation supported via CredentialResolver
- Flyway migrations in qhorus runtime — ExternalAgentBinding is in `a2a-outbound`

## Architecture

### Algorithm and Standards

| Standard | Usage |
|----------|-------|
| JWS (RFC 7515) | Signature envelope format |
| ES256 (ECDSA P-256) | Signing algorithm (A2A v1.0 primary) |
| JCS (RFC 8785) | JSON Canonicalization Scheme for deterministic signing input |
| JWK (RFC 7517) | Public key format for JWKS endpoint |

A2A v1.0 (sections 4.4.7, 8.4) specifies JWS with ES256/RS256 for agent card signing. ES256 is the primary algorithm. The JWS protected header carries `alg`, `typ`, `kid`, and optionally `jku` pointing to the JWKS endpoint.

### Key Management

Platform-level — one EC P-256 key pair per deployment/tenant, managed via the platform `CredentialResolver` SPI. The platform signs all agent cards centrally. Key material is never exposed in agent card responses, ledger entries, or logs (per protocol PP-20260612-bd6f8c).

### Canonicalization

Before signing, the `AgentCard` is serialized to JSON with the `signatures` field excluded, then canonicalized via JCS (RFC 8785). JCS produces a deterministic JSON byte sequence by:
- Sorting object keys lexicographically
- Normalizing numbers (no trailing zeros, no leading plus)
- Normalizing strings (shortest UTF-8 escape sequences)
- No whitespace

The canonical form is the JWS payload. Verification re-canonicalizes the received card (minus signatures) and verifies against the JWS signature.

## SPI Contract

### AgentCardSigningService (api/spi/)

```java
package io.casehub.qhorus.api.spi;

import io.casehub.a2a.model.AgentCardSignature;
import java.util.List;

public interface AgentCardSigningService {

    SignedCard sign(String canonicalJson);

    VerificationResult verify(String canonicalJson, List<AgentCardSignature> signatures);

    record SignedCard(List<AgentCardSignature> signatures) {}

    record VerificationResult(boolean verified, String keyId, String error) {}
}
```

- `sign()` takes JCS-canonicalized card JSON, returns JWS signatures
- `verify()` takes canonical JSON + inbound signatures, returns verification status with the `kid` that verified (or error message on failure)
- `@DefaultBean NoOpAgentCardSigningService` in runtime returns empty signatures on sign and `VerificationResult(false, null, null)` on verify — signing is off by default

### AgentCardSignature (casehub-a2a-protocol)

```java
package io.casehub.a2a.model;

public record AgentCardSignature(
    String protectedHeader,  // base64url-encoded JWS protected header
    String signature         // base64url-encoded JWS signature value
) {}
```

### AgentCard changes (casehub-a2a-protocol)

Additive field on the existing `AgentCard` record:

```java
public record AgentCard(
    String name,
    String description,
    String url,
    String version,
    List<AgentSkill> skills,
    AgentCapabilities capabilities,
    Map<String, Object> authentication,
    String tenancyId,
    List<AgentRef> agents,
    List<AgentCardSignature> signatures  // nullable — null/omitted for unsigned cards
) {
    // Backward-compatible 9-arg constructor (delegates with signatures=null)
    public AgentCard(String name, String description, String url, String version,
                     List<AgentSkill> skills, AgentCapabilities capabilities,
                     Map<String, Object> authentication, String tenancyId,
                     List<AgentRef> agents) {
        this(name, description, url, version, skills, capabilities,
             authentication, tenancyId, agents, null);
    }
    // ...existing methods...
}
```

JSON serialization: `@JsonInclude(JsonInclude.Include.NON_NULL)` on the `signatures` field ensures unsigned cards omit it entirely.

## Module Structure

### agent-card-signing/ (new optional module)

```
agent-card-signing/
├── pom.xml
└── src/main/java/io/casehub/qhorus/signing/
    ├── JwsAgentCardSigningService.java  — @ApplicationScoped impl
    ├── JwsKeyProvider.java              — EC key pair from CredentialResolver
    ├── JcsCanonicalizer.java            — RFC 8785 canonicalization
    ├── SigningConfig.java               — @ConfigMapping
    └── IdentityVerificationTrustSource.java — TrustScoreSource decorator
```

Activates by classpath presence. `JwsAgentCardSigningService` is `@ApplicationScoped` (not `@Alternative`) — it displaces the `@DefaultBean NoOpAgentCardSigningService` in runtime automatically via CDI priority.

**Dependencies:**
- `casehub-qhorus-api` (SPI interface)
- `casehub-a2a-protocol` (AgentCardSignature)
- `casehub-ledger` (TrustScoreSource, TrustGateService)
- `com.nimbusds:nimbus-jose-jwt` (JWS implementation)
- `casehub-platform-api` (CredentialResolver)

### Configuration

```properties
# Signing
casehub.qhorus.signing.enabled=true
casehub.qhorus.signing.key-ref=agent-card-signing-key
casehub.qhorus.signing.algorithm=ES256
casehub.qhorus.signing.key-id=default
casehub.qhorus.signing.jwks-url=

# Trust integration
casehub.qhorus.signing.trust.verified-floor-score=0.6
casehub.qhorus.signing.trust.dimension-weight=0.15
```

| Property | Default | Description |
|----------|---------|-------------|
| `enabled` | `true` | Gate — `false` disables signing even with module on classpath |
| `key-ref` | `agent-card-signing-key` | CredentialResolver key reference for the EC key pair |
| `algorithm` | `ES256` | JWS algorithm identifier |
| `key-id` | `default` | `kid` value in JWS protected header and JWKS |
| `jwks-url` | (empty) | Override for `jku` in JWS header; defaults to `/.well-known/jwks.json` on self |
| `trust.verified-floor-score` | `0.6` | Floor trust score for verified agents in the `identity-verification` dimension |
| `trust.dimension-weight` | `0.15` | Weight of identity-verification dimension in global trust aggregate |

## Data Flow

### Outbound Signing (AgentCardResource)

```
1. AgentCardResource.getAgentCard()
2.   → build AgentCard with signatures=null (existing logic)
3.   → serialize to JSON, exclude signatures field
4.   → JcsCanonicalizer.canonicalize(json)
5.   → AgentCardSigningService.sign(canonicalJson)
6.     → if NoOp: return SignedCard(List.of()) → card served unsigned
7.     → if JWS impl: build JWS protected header {alg:ES256, typ:agentcard+jwt, kid:default, jku:/.well-known/jwks.json}
8.       → ECDSA-sign canonical payload with private key from JwsKeyProvider
9.       → return SignedCard(List.of(new AgentCardSignature(protectedB64, signatureB64)))
10.  → construct final AgentCard with signatures populated (or null if empty)
11.  → return to client
```

Same flow applies to per-agent cards at `/.well-known/agents/{instanceId}.json`.

### Inbound Verification (a2a-outbound)

```
1. A2AOutboundBackend fetches remote /.well-known/agent.json
2.   → deserialize AgentCard
3.   → if card.signatures() is null or empty:
4.       → binding.verificationStatus = UNVERIFIED; proceed
5.   → extract jku from first signature's protected header
6.   → fetch JWKS from jku URL (HTTP GET, cached)
7.   → serialize card without signatures → JcsCanonicalizer.canonicalize()
8.   → AgentCardSigningService.verify(canonicalJson, card.signatures())
9.     → resolve public key from JWKS by kid in protected header
10.    → ECDSA-verify canonical payload against signature
11.    → return VerificationResult(verified, keyId, error)
12.  → if verified:
13.      → binding.verificationStatus = VERIFIED
14.      → binding.verifiedAt = Instant.now()
15.      → binding.verificationKeyId = result.keyId()
16.  → if not verified:
17.      → binding.verificationStatus = FAILED
18.      → LOG.warn("Agent card verification failed for {}: {}", endpoint, result.error())
19.  → proceed with dispatch regardless (soft failure)
```

### JWKS Endpoint

```
GET /.well-known/jwks.json → AgentCardResource

Response:
{
  "keys": [{
    "kty": "EC",
    "crv": "P-256",
    "kid": "default",
    "use": "sig",
    "x": "<base64url>",
    "y": "<base64url>"
  }]
}
```

The public key is extracted from the same key pair used for signing. The JWKS endpoint is unauthenticated (public keys are public by definition).

## Trust Score Integration

### IdentityVerificationTrustSource (decorator pattern)

```java
@Alternative @Priority(100)
@ApplicationScoped
public class IdentityVerificationTrustSource implements TrustScoreSource {

    private final TrustScoreSource delegate;
    private final ExternalAgentBindingStore bindingStore;
    private final SigningConfig config;

    // All TrustScoreSource methods delegate to the ledger-based source.
    // dimensionScore(actorId, "identity-verification") returns:
    //   - config.trust.verified-floor-score for VERIFIED agents
    //   - OptionalDouble.empty() for UNVERIFIED/FAILED/unknown
    // globalScore(actorId) merges identity dimension into aggregate
    //   using config.trust.dimension-weight
}
```

The decorator wraps the existing ledger-based `TrustScoreSource`. When `agent-card-signing` is on the classpath, the `@Alternative @Priority(100)` displaces the default source. All existing dimension scores pass through unchanged; the identity-verification dimension is additive.

**Composition with existing trust consumers:**
- `TrustGateService.meetsThreshold()` — works unchanged (reads globalScore)
- `RoutingBridge.lookupTrustScore()` — works unchanged (reads globalScore via TrustGateService)
- `TrustScoreResource` — returns the identity dimension alongside existing dimensions
- `DefaultObligorTrustPolicy` — works unchanged (delegates to TrustGateService)

### ExternalAgentBinding Changes (a2a-outbound)

New nullable fields on the existing entity:

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

Flyway migration in `a2a-outbound` module (not qhorus runtime — ExternalAgentBinding is owned by a2a-outbound).

## Testing Strategy

### Unit Tests (CDI-free)

| Test | What it validates |
|------|-------------------|
| `JcsCanonicalizerTest` | Deterministic canonicalization: nested objects, unicode, numeric edge cases, key ordering |
| `JwsAgentCardSigningServiceTest` | Sign/verify round-trip with generated test EC keys; tampered payload detection; missing key handling |
| `IdentityVerificationTrustSourceTest` | Decorator delegation; dimension score for VERIFIED/UNVERIFIED/unknown; global score weight merging |
| `AgentCardSignatureTest` | Record serialization; base64url encoding round-trip |

### Integration Tests (@QuarkusTest)

| Test | What it validates |
|------|-------------------|
| `SignedAgentCardTest` | GET `/.well-known/agent.json` returns signed card when signing module active; unsigned when NoOp |
| `JwksEndpointTest` | GET `/.well-known/jwks.json` returns valid JWKS with correct key parameters |
| `AgentCardVerificationTest` | Mock HTTP server serves signed card → A2AOutboundBackend fetches and verifies → ExternalAgentBinding.verificationStatus updated |
| `VerificationTrustIntegrationTest` | Verified agent gets identity-verification dimension score; unverified agent does not; routing threshold affected |

### CDI-free test patterns

- `JwsAgentCardSigningService` tests generate ephemeral EC key pairs (no CredentialResolver mock needed for unit tests)
- `IdentityVerificationTrustSource` tests use a stub `TrustScoreSource` delegate and a stub `ExternalAgentBindingStore`
- `JcsCanonicalizer` tests are pure function tests — input JSON string → canonical output bytes

## Security Considerations

- **Key material in ledger:** Signing keys must never appear in `MessageDispatch.content` or any ledger-persisted field (protocol PP-20260612-bd6f8c). The `key-ref` config property points to CredentialResolver, which resolves the actual key material at runtime.
- **JWKS endpoint:** Public keys only. No private key material exposed. Rate limiting recommended in production.
- **Soft failure:** Unsigned or failed-verification cards are accepted but marked. The trust score penalty is the only consequence — verification is never an access gate. This is intentional for ecosystem compatibility.
- **Key compromise:** Platform-level key compromise affects all agent cards for that tenant. Rotation is manual via CredentialResolver (update key, re-deploy). Future DID work will add key rotation semantics.

## Future Work (Child Issues)

| Topic | Dependency | Notes |
|-------|------------|-------|
| DID anchoring (did:web) | This issue | Maps `/.well-known/` to `did:web:host` naturally; adds DID Document endpoint |
| DID anchoring (did:key) | This issue | Self-certifying IDs for ephemeral agents; key is the DID |
| Verifiable Credentials | DID anchoring | VC Data Model 2.0 credentials asserting agent capabilities |
| Per-agent signing keys | DID anchoring | Each Instance gets its own key pair, anchored to a DID |
| Key rotation automation | This issue | Scheduled key rotation with JWKS versioning |

## References

- [A2A v1.0 spec, sections 4.4.7, 8.4](https://google.github.io/A2A/) — agent card signing format
- [RFC 7515](https://tools.ietf.org/html/rfc7515) — JSON Web Signature
- [RFC 8785](https://tools.ietf.org/html/rfc8785) — JSON Canonicalization Scheme
- [RFC 7517](https://tools.ietf.org/html/rfc7517) — JSON Web Key
- [W3C DID v1.0](https://www.w3.org/TR/did-core/) — Decentralized Identifiers (future work)
- [W3C VC Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/) — Verifiable Credentials (future work)
- `runtime/src/main/java/io/casehub/qhorus/runtime/api/AgentCardResource.java` — existing resource
- `a2a-protocol/src/main/java/io/casehub/a2a/model/AgentCard.java` — Layer 0 agent card record
- `api/src/main/java/io/casehub/qhorus/api/spi/` — existing SPI pattern
- `compliance-report/.../ComplianceReportSigningService.java` — existing signing integration pattern
- `runtime/.../message/RoutingBridge.java` — trust score consumption point
- Protocol PP-20260612-bd6f8c — no credentials in ledger content
- casehubio/qhorus#403 — parent epic

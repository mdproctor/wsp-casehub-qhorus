## D1: Signing algorithm family

**Choice:** JWS (RFC 7515) with ES256 (ECDSA P-256), per A2A v1.0 spec
**Alternatives:**
- Ed25519 (JWS EdDSA) — faster/simpler keys but less aligned with A2A v1.0 spec
- Both ES256 + Ed25519 — maximum flexibility but doubles implementation surface
**Rationale:** A2A v1.0 sections 4.4.7 and 8.4 specify JWS with ES256/RS256. Aligning with the spec ensures interop with other A2A implementations. ES256 is well-supported by Nimbus JOSE+JWT.
**Trade-offs:** Ed25519 is preferred by the DID/VC ecosystem (did:key natively encodes it). When DID support is added later, may need to add EdDSA as a secondary algorithm.
**Sources:** A2A v1.0 spec (sections 4.4.7, 8.4), RFC 7515, issue #403
**Exploration:** quick
**Status:** captured

## D2: First-pass scope

**Choice:** Core signing + verification + trust score bonus. DID anchoring and verifiable credentials deferred to child issues.
**Alternatives:**
- Core + DID anchoring — moderate additional scope, meaningful but not load-bearing for signing
- Full scope (all 6 items) — too large for a single implementation pass
**Rationale:** Signing and verification are the foundation. DID and VCs build on top without requiring rework. Shipping the core first proves the signing pipeline and trust integration before adding identity layers.
**Trade-offs:** DID gives signing meaning beyond raw key management. Without DID, key discovery relies on HTTPS + JWKS endpoints (still secure, just centralized).
**Sources:** Issue #403 scope list, A2A v1.0 agent card spec
**Exploration:** quick
**Status:** captured

## D3: Key management model

**Choice:** Platform-level — one signing key per deployment/tenant, managed via platform CredentialResolver
**Alternatives:**
- Per-agent keys — each Instance gets its own key pair; stronger per-agent identity but complex key lifecycle
- Hierarchical (platform root, per-agent derived) — middle ground but adds cryptographic complexity
**Rationale:** Matches how DocumentSigningService works today. The platform signs all agent cards centrally. Simpler key lifecycle — one key to generate, rotate, and revoke per tenant. Per-agent keys are premature without DID infrastructure to anchor them.
**Trade-offs:** All agents share one signing identity — a compromised key compromises all agent cards for that tenant. Acceptable for now since the key is server-side and not exposed.
**Sources:** ComplianceReportSigningService pattern, CredentialResolver SPI, issue #403
**Exploration:** quick
**Status:** captured

## D4: Signing SPI location

**Choice:** New `AgentCardSigningService` SPI in `casehub-qhorus-api` with `@DefaultBean` NoOp in runtime
**Alternatives:**
- Extend platform SigningService — broader reuse but couples platform to JWS/JOSE, requires platform release
- Inline concrete service in qhorus runtime — simpler but not replaceable by consumers
**Rationale:** Clean separation. DocumentSigningService is document-focused (X.509/CAdES), this is identity-focused (JWS/JOSE). Follows established qhorus SPI pattern (CommitmentAttestationPolicy, ObligorTrustPolicy). @DefaultBean NoOp means unsigned cards by default — consumers opt in by providing an implementation.
**Trade-offs:** Consumers must bring a JWS implementation (e.g. Nimbus JOSE+JWT) to activate signing. The NoOp default means signing is off unless explicitly configured.
**Sources:** api/spi/ pattern, ComplianceReportSigningService, DocumentSigningService SPI
**Exploration:** quick
**Status:** captured

## D5: Inbound verification behavior

**Choice:** On-fetch with soft failure — verify signature when fetching remote agent cards; unsigned or bad-signature cards accepted but marked unverified
**Alternatives:**
- Hard failure — reject unsigned/bad-signature cards; breaks interop with unsigned ecosystem
- Lazy/cached verification — async background verification; more complex, zero fetch latency impact
**Rationale:** Backward compatibility is critical. Most A2A agents in the ecosystem don't sign cards yet. Soft failure lets the trust score bonus incentivize signing without breaking interop.
**Trade-offs:** Unverified agents can still participate fully — the only penalty is a lower trust score. This is by design: verification is a trust signal, not an access gate.
**Sources:** A2A v1.0 signatures field (optional), RoutingBridge trust threshold pattern
**Exploration:** quick
**Status:** captured

## D6: Trust score integration

**Choice:** New `identity-verification` dimension in TrustScoreSource — verified agents get a configurable floor score in that dimension
**Alternatives:**
- Routing-only bonus (threshold discount in RoutingBridge) — simpler but invisible to TrustScoreSource consumers
- Defer trust integration — ship signing without trust wiring; smallest scope
**Rationale:** Fits the existing multi-dimensional trust model. TrustScoreComputer already aggregates across dimensions. RoutingBridge and TrustGateService consume TrustScoreSource without changes. The bonus is visible in trust score queries, diagnostics, and routing decisions.
**Trade-offs:** Requires a new TrustScoreSource implementation or extension in qhorus — the existing ledger-based source computes from attestation history. Identity verification is a different signal (static vs behavioral). May need a composite TrustScoreSource pattern.
**Depends on:** D4 (SPI location determines where verification status is stored)
**Sources:** TrustScoreSource SPI, TrustGateService, TrustScoreComputer, RoutingBridge
**Exploration:** quick
**Status:** captured

## D7: Module structure for signing implementation

**Choice:** New `agent-card-signing` optional module — activates by classpath presence, contains `JwsAgentCardSigningService` (real impl using Nimbus JOSE+JWT), key loading from CredentialResolver
**Alternatives:**
- Inline in runtime — simpler but adds Nimbus JOSE+JWT as transitive dependency for all qhorus consumers whether they want signing or not
**Rationale:** Follows established optional module pattern (slack-channel, webhook-observer, a2a-outbound). @DefaultBean NoOp in runtime means unsigned cards by default; adding the optional module overrides it. Keeps runtime classpath lean.
**Trade-offs:** One more Maven module to maintain. Consumers must add an explicit dependency to enable signing.
**Depends on:** D4 (SPI in qhorus-api, NoOp default in runtime)
**Sources:** slack-channel/, webhook-observer/, a2a-outbound/ module patterns
**Exploration:** quick
**Status:** captured

## D8: AgentCard record changes

**Choice:** Additive nullable `List<AgentCardSignature>` field on `io.casehub.a2a.model.AgentCard` (Layer 0 shared library in casehub-a2a-protocol)
**Alternatives:**
- Separate `SignedAgentCard` wrapper record — avoids touching the shared record but complicates serialization and doubles the type surface
**Rationale:** A2A v1.0 defines `signatures` as an optional repeated field on the agent card itself. Adding it as nullable preserves backward compat — unsigned cards serialize with signatures omitted. `AgentCardSignature` record carries `protected` (base64url JWS header) and `signature` (base64url).
**Trade-offs:** Touches Layer 0 shared library — requires casehub-a2a-protocol release. But the change is purely additive (new nullable field + new record type).
**Sources:** A2A v1.0 spec (AgentCard.signatures), io.casehub.a2a.model.AgentCard
**Exploration:** quick
**Status:** captured

## D9: JWKS endpoint location

**Choice:** `/.well-known/jwks.json` served by `AgentCardResource` — co-located with agent cards, matches A2A `jku` convention
**Alternatives:**
- Separate JwksResource — unnecessary separation for a single GET endpoint
- No JWKS endpoint (key embedded in JWS header) — works but doesn't support key rotation; verifiers must re-fetch the full card to get the new key
**Rationale:** Co-locating with agent cards means one resource serves the complete A2A identity surface. The `jku` field in JWS protected headers points here. Key rotation = update JWKS, re-sign cards — verifiers re-fetch keys from the same well-known URL.
**Trade-offs:** JWKS endpoint is unauthenticated (public keys are public). Rate limiting may be needed in production.
**Sources:** RFC 7517 (JWK), A2A v1.0 jku convention, AgentCardResource.java
**Exploration:** quick
**Status:** captured

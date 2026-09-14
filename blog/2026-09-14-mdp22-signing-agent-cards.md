---
title: "Signing Agent Cards — Cryptographic Identity for Qhorus Agents"
date: 2026-09-14
author: mdp
entry_type: note
subtype: diary
series: issue-403-signed-agent-cards-did
projects: [casehubio/qhorus]
tags: [signing, jws, ed25519, agent-cards, trust, identity]
---

# Signing Agent Cards — Cryptographic Identity for Qhorus Agents

Agent identity in Qhorus has been configuration-based — you register an instance with an endpoint and an auth key, and the system takes your word for it. That worked when every agent was internal. It stops working the moment agents talk across trust boundaries, because "I am who I say I am" isn't verifiable without cryptography.

This session added JWS signing to agent cards. Every card served at `/.well-known/agent.json` can now carry an Ed25519 signature. Verifiers fetch the signing key from `/.well-known/jwks.json`, re-canonicalize the card via JCS (RFC 8785), and check the signature. Unsigned cards are still accepted — the only consequence of being unverified is a lower trust score.

## The Design Review Caught a Real Problem

The initial spec attributed agent card signing to A2A v1.0 sections 4.4.7 and 8.4, and specified ES256 as the signing algorithm. Claude flagged this during the design review: those sections don't exist. The A2A protocol doesn't define agent card signing at all — the research agent that surfaced those references had fabricated them.

That correction changed the design materially. With no A2A constraint on the algorithm, Ed25519 became the right choice — it matches the issue's original specification, aligns with the platform's `SigningProvider` SPI, and is natively supported by JCA since Java 15. We also dropped the plan to add a `signatures` field to the `AgentCard` record in `casehub-a2a-protocol`. Agent card signing is a platform extension, not a protocol feature. Contaminating the Layer 0 model with platform-specific types would have been an architectural mistake.

## JCA Over Nimbus for Crypto

The implementation hit a practical gotcha: Nimbus JOSE+JWT's `Ed25519Signer` and `Ed25519Verifier` silently depend on Google Tink at runtime. The dependency is optional in Maven — everything compiles fine — but `Ed25519Signer.<init>` throws `NoClassDefFoundError` the moment you try to use it.

The fix was to bypass Nimbus for the actual cryptography entirely. `java.security.Signature.getInstance("Ed25519")` handles signing and verification natively on Java 15+. Nimbus is still used for what it's good at — JWS header construction, Base64URL encoding, and JWK serialization — but the raw crypto is JCA-native. No Tink dependency, no optional-dependency surprises.

## Trust Score Integration

Verified agents get an additive boost in the trust scoring pipeline via a `@Decorator` on `TrustScoreSource`. The decorator adds an `identity-verification` dimension that contributes a configurable floor score (default 0.6) weighted at 0.15 of the global aggregate. With those defaults, a verified agent gets a +0.09 trust boost — enough to influence routing decisions in `RoutingBridge` without dominating the attestation-based dimensions.

The decorator reads verification status from `ExternalAgentBinding`, which gained three new fields: `verificationStatus` (VERIFIED/UNVERIFIED/FAILED), `verifiedAt`, and `verificationKeyId`. Verification runs asynchronously after binding creation — the PUT returns immediately with UNVERIFIED status, and a `@ObservesAsync` handler fetches and verifies the remote card in the background.

## What This Opens Up

The signing infrastructure is deliberately minimal for this first pass. Three things are needed before it's production-complete: a `JwksCache` for caching remote JWKS responses (right now each verification fetches the key fresh), per-agent signing keys anchored to DIDs (currently one key per deployment), and verifiable credentials for capability attestation. Each builds on this foundation without requiring rework — the SPI, the verification flow, and the trust dimension are stable.

The more interesting question is what happens when identity verification meets the existing trust machinery. A verified agent with a strong attestation history is qualitatively different from one with just a good track record. The combination of "I can prove who I am" and "my work has been independently validated" is stronger than either signal alone. That's where DID anchoring and verifiable credentials will earn their weight.

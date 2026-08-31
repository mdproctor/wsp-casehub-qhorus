---
layout: post
title: "Signing the Evidence"
date: 2026-08-31
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [eidas, digital-signatures, compliance, pades, cades, platform-signing]
---

# Signing the Evidence

The compliance report module produces tamper-evident records — attribution chains, obligation reports, trust histories, provenance graphs — all backed by Merkle-anchored ledger entries. But "tamper-evident" and "provably signed" are different things. A Merkle root proves no entries were modified after generation. A digital signature proves who produced the report and when. Regulators care about both.

With the PDF renderer landed, the next piece is cryptographic seals. eIDAS — the EU's electronic identification and trust services regulation — defines the framework. A qualified electronic seal gives a document the legal presumption of integrity. The architecture needed to support that without requiring qualification at the code level. Qualification is a deployment decision: which certificate authority, which hardware security module, which trust service provider. The code produces standards-compliant signature formats and stays agnostic about where the certificate came from.

## Two SPIs, Not One

The platform already has a `SigningProvider` — raw `sign(actorId, bytes)` returning a signature and public key. It works for ledger agent attestation where Ed25519 keypairs are the whole story. Document signing needs more: X.509 certificate chains, timestamp authority integration, PAdES and CAdES format packaging. Trying to bridge between the two would have meant an adapter that bypasses `SigningProvider` for certificates anyway, making the abstraction pointless.

The design settled on a self-contained `DocumentSigningService` SPI alongside the existing `SigningProvider`. They serve different domains — agent attestation signing versus organisational document sealing — and forcing them through a shared interface would couple things that evolve independently. A new `casehub-platform-signing` module provides the EU DSS implementation: PAdES for PDFs, CAdES for detached signatures on JSON and CSV, RFC 3161 timestamping. The `NoOp` default returns `Optional.empty()` when no signing backend is deployed, same pattern as `PdfGenerator`.

## Strict Profiles

PAdES defines four profile levels. B-B is the baseline — signature plus certificate chain, valid while the certificate is valid. B-T adds a timestamp from a trusted authority, proving when the signature was created. B-LT and B-LTA add revocation data and archival timestamps for indefinite validity.

The interesting design decision was what happens when a deployer configures B-T but the timestamp authority is unreachable. The instinct is to fall back to B-B — something signed is better than nothing signed. But for compliance infrastructure, that instinct is wrong. A deployer who configured B-T did so because their regulatory environment requires timestamp proof. Producing a B-B signature that looks compliant but lacks the timestamp is worse than an error. The system refuses to sign when the configured profile can't be satisfied. B-B is an explicit choice, not a degradation path.

## What Gets Signed

Scheduled reports — the regulatory submission path — are signed automatically when a signing backend is configured. On-demand REST queries return unsigned results by default because the timestamp authority round-trip adds hundreds of milliseconds. A `signed=true` query parameter opts in for the rare case where an on-demand report needs a signature.

The SPI types are implemented and installed. The EU DSS integration, SharedData binary support, pipeline wiring, and verification endpoint are next — probably two more sessions to get through the remaining three batches.

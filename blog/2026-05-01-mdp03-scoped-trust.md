---
layout: post
title: "Scoped Trust"
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [ledger, trust, maven, quarkus]
---

casehub-ledger's trust score infrastructure is built around capability-scoped
Beta distributions — a separate distribution per capability tag, not one global
confidence value for an agent. Every attestation Qhorus wrote had been landing
on `"*"`, which collapses all of that into a single global score. The fix was in
`LedgerWriteService`: extract the `"capability"` field from the originating
COMMAND's content JSON when writing the attestation, fall back to
`CapabilityTag.GLOBAL` if absent.

Two methods. The `extractCapabilityTag` helper parses the content JSON and
returns the field value if present; `writeAttestation` calls it and sets the
result on the attestation before persisting. Twenty minutes of code.

## The SNAPSHOT Tax

The compile failure came first, and it named our class rather than the
dependency that changed:

    [ERROR] MessageLedgerEntryRepository is not abstract and does not override
    abstract method findAttestationsByAttestorIdAndCapabilityTag(String, String)
    in LedgerEntryRepository

casehub-ledger 0.2-SNAPSHOT had added three abstract methods to the
`LedgerEntryRepository` interface. Maven pulled the update silently. Our
implementing classes were suddenly incomplete, but the error only pointed at our
code — nothing indicated that anything upstream had changed.

We needed the named queries before implementing. Source wasn't checked out
locally, so we extracted the entity class from the JAR and ran `javap -verbose`:

    jar xf casehub-ledger-0.2-SNAPSHOT.jar \
      io/casehub/ledger/runtime/model/LedgerAttestation.class
    javap -verbose LedgerAttestation.class | grep -A3 "NamedQuery"

The JVM constant pool stores annotation values verbatim. Every `@NamedQuery`
string came out clean — name and full JPQL body. GE-0047 in the garden covers
this technique for Quarkus config property names; the JPA named query variant
is there now too.

Three implementations in the blocking repo, three `UnsupportedOperationException`
stubs in the reactive mirror, two integration tests.

## The Tool That Was Already There

`get_obligation_activity` — the cross-channel correlation tool built during the
agent mesh documentation work — appeared in the open issues list. Reading the
code made it a formality: blocking and reactive implementations both complete,
repository method in place, tests passing. Nothing to add.

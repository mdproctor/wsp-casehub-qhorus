---
layout: post
title: "The import that broke everything"
date: 2026-09-29
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [ci, ledger, imports, cross-repo, maven]
---

The three tests in #459 all passed. That was the first surprise — the issue said they were failing, but they'd already been fixed by the runtime-core extraction in #458. The real CI blocker was hiding behind them: a compilation error in the compliance-report module, caused by a ledger SNAPSHOT that was two weeks stale.

The published `casehub-ledger-core` on GitHub Packages was from September 16. The local install had the September 25 version, with three new fields on `DecisionRecord` and two on `ComplianceReport`. The test code matched the local version. CI resolved the published one. Mismatch.

Fixing it meant fixing the ledger repo first. That turned into a chain: the `ComplianceReportServiceCore` constructor calls were stale, `MerkleVerificationBundleService` imported from a package that had moved, `InMemoryLedgerEntryRepository` had the same stale import, the REST API tests were wired to no-arg constructors that no longer existed, the `signing-spring` module depended on four backend modules that hadn't been created yet, the `graphql` module needed a generator plugin that hadn't been published, and the `ledger-spring` module used the same unpublished plugin. Seven commits to ledger before CI would pass and deploy the SNAPSHOT.

With the SNAPSHOT deployed, qhorus had its own layer of breakage. The ledger's JPA entity extraction from `io.casehub.ledger.runtime.model` to `io.casehub.ledger.jpa` had happened upstream but qhorus still referenced the old packages — `JpaLedgerEntry`, `LedgerAttestation`, `PlainLedgerEntry`, `LedgerPersistenceUnit`, `LedgerMerkleFrontierRepository`, `ActorTrustScoreRepository`. Twenty-nine files across runtime-core, runtime, and compliance-report. Plus every `application.properties` and test profile that declared Hibernate package scanning — eleven config files in total, including one hardcoded in a `@TestProfile` that was easy to miss.

The deeper issue was the ledger runtime pom. Its intra-project dependencies on `casehub-ledger-jpa-common` and `casehub-ledger-core` omitted `<version>` tags, relying on the parent's `<dependencyManagement>`. That works within the reactor but breaks for external consumers — Maven can't resolve the transitive dependency without an explicit version in the installed pom. The fix was explicit `<version>${project.version}</version>` on every intra-project dependency.

No qhorus logic changed. Every line in the diff is an import path, a pom dependency, or a Hibernate package string. The entire branch is a mechanical response to an upstream refactoring that shipped without updating its downstream consumers.

---
layout: post
title: "Platform-Wide Breaking Window"
date: 2026-04-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [maven, ci-cd, module-split, mcp, breaking-changes]
---

The last entry ended with all eight MCP consistency decisions made, the non-breaking ones shipped, and the breaking redesign deferred. Then casehub-engine finished a PR. That's a release window — a moment when downstream consumers are quiescent and won't be broken by anything we push upstream. I decided not to let it pass.

What started as "implement the breaking MCP changes" became: implement those, plus module splits for quarkus-ledger and casehub-qhorus, plus bug fixes in casehub-work, plus a naming audit across everything. Twenty-one tasks, five repos, one session.

## The MCP Surface

The changes landed exactly as decided:

- `share_data`/`get_shared_data`/`list_shared_data` → `share_artefact`/`get_artefact`/`list_artefacts`
- Chunked upload is now `begin_artefact` → `append_chunk` → `finalize_artefact`; `share_artefact` is single-shot only
- `send_message` with `artefact_refs` auto-claims for the sender; DONE/FAILURE/DECLINE auto-releases
- `register_observer` and `read_observer_events` are gone; observers are now `register(read_only=true)` instances and `check_messages(include_events=true)` replaces the separate event-reading tool
- `list_pending_waits` and `list_pending_approvals` merge into `list_pending_commitments(type_filter?)`
- `delete_channel` enforces `admin_instances` the same way `pause_channel` already did

## The Module Splits

The structural problem: any consumer of casehub-qhorus or quarkus-ledger pulled JPA entities onto the classpath, requiring a datasource even in pure in-memory tests.

The fix is the same pattern casehub-work already uses — a separate `-api` module containing no JPA: enums, SPI interfaces, value types. For quarkus-ledger this was clean. `LedgerEntry` became a `@MappedSuperclass` in the api module:

```java
// quarkus-ledger-api — no JPA engine, just annotations
@MappedSuperclass
public abstract class LedgerEntry {
    public Long id;
    public String actorId;
    // ...all the fields, @Column annotations, @PrePersist lifecycle
}

// quarkus-ledger runtime — adds @Entity and the table mapping
@Entity
@Table(name = "ledger_entry")
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class LedgerEntry
        extends io.quarkiverse.ledger.api.model.LedgerEntry { }
```

For casehub-qhorus the same approach hit Java's single-inheritance wall. Panache entities extend `PanacheEntityBase` — a class, not an interface. You can only have one. So the api module for qhorus contains enums, exception types, and SPI interfaces, but the store interfaces and domain entities stay in the runtime. Claudony and casehub-engine can now depend on `quarkus-ledger-api` for `LedgerTraceIdProvider` without pulling any JPA onto their classpath. That was the actual goal.

## The CI Timing Race

We pushed five repos in sequence. All five triggered CI simultaneously. Three failed with `Could not resolve casehub-ledger-api:0.2-SNAPSHOT`.

The upstream CI showed `conclusion: success` within seconds. But that reflects the workflow run state, not artifact availability. GitHub Packages hadn't published yet when the downstream resolvers ran. Re-triggering fixed it, but the diagnosis took time because `Could not resolve` looks like a misconfiguration.

A separate problem surfaced alongside it. One subagent added `findScore()` to `TrustGateService` locally, tested against the local `.m2` cache — which already had the method from the working-tree build — and reported success. The published artifact on GitHub Packages didn't have it. `git diff` caught the uncommitted file. The symptom in CI was `cannot find symbol: method findScore`, which also looks like a version mismatch rather than what it actually was.

And one more: GitHub Packages doesn't reliably resolve transitive SNAPSHOT dependencies. `casehub-qhorus-api` was a compile-scope dependency of `casehub-qhorus`, but claudony's CI couldn't find its types. Explicit declarations fixed it. Maven Central doesn't behave this way.

## The Naming Sweep

With everything pushed, we audited all five repos for stale naming. The READMEs in casehub-work and casehub-parent still said `quarkus-ledger | io.quarkiverse.ledger` in their dependency tables. The module directories in casehub/work were still named `quarkus-work-api/`, `quarkus-work-core/` — correct artifact IDs inside, wrong directory names outside. Thirteen `git mv`s, one pom.xml update. Design docs and ADRs referenced old Java package paths in example code. All fixed.

Eighty commits across five repos. All green.

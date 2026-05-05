---
layout: post
title: "Ledger Reconciliation and a Transaction Boundary Fix"
date: 2026-04-17
type: phase-update
entry_type: note
subtype: diary
projects: [quarkus-qhorus]
tags: [quarkus-ledger, transactions, flyway, schema]
---

This session was mostly reconciliation — catching Qhorus up to changes in quarkus-ledger that had shipped since Phase 12 was built.

The ledger had refactored `correlationId`. What had been a direct field on `LedgerEntry` was now stored via `ObservabilitySupplement` — a supplement object attached via `entry.attach(obs)`. The Qhorus side had to follow: `LedgerWriteService` switched from `entry.correlationId = value` to constructing an `ObservabilitySupplement` and attaching it; `QhorusMcpTools.toEventMap()` switched from reading the field directly to reading via `entry.observability().map(obs -> obs.correlationId).orElse(null)`.

The Flyway migration number changed too. V1002 was now reserved by quarkus-ledger's own supplement migration. Qhorus's `agent_message_ledger_entry` table moved to V1003. Then — mid-session — quarkus-ledger deleted all its SQL migration scripts entirely, deferring the decision. The right move for Qhorus tests was to stop using Flyway for test schema and switch to `hibernate-orm.database.generation=drop-and-create`. We removed the Flyway config from `runtime/src/test/resources/application.properties` and let Hibernate own the test schema. Simpler, and Flyway was never the right tool for a fast-changing test database anyway.

Phase 8 — Qhorus embedded in Claudony — happened this session on the Claudony side. We added `quarkus-qhorus` and `quarkus-jdbc-h2` to the Claudony pom. Claudony Claude subsequently removed the datasource config, breaking it. We wrote a detailed fix briefing and sent it back.

The more interesting work was identifying the next Qhorus fix before closing out. `LedgerWriteService.recordEvent` ran inside the same `@Transactional` boundary as `send_message`. That means a ledger write failure — malformed payload, ledger down, schema mismatch — rolls back the entire message send. An audit failure should never prevent message delivery. The fix: `@Transactional(REQUIRES_NEW)` on `recordEvent` to isolate the ledger transaction, plus a try/catch in the caller so a failed `REQUIRES_NEW` acquisition doesn't propagate up. Both changes landed in the next session.

561 tests. No new features. Sometimes the work is just keeping up with your own dependencies.

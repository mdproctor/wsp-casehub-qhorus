# Design Spec — Add classpath:db/ledger/migration to qhorus Flyway
**Issue:** casehubio/qhorus#179  
**Branch:** issue-179-add-ledger-migration-path  
**Date:** 2026-05-21

---

## Context

casehubio/ledger#95 moved ledger Flyway migrations from `classpath:db/migration` to
`classpath:db/ledger/migration`. `MessageLedgerEntry` extends `LedgerEntry` via JPA
JOINED inheritance — the qhorus named datasource must run ledger's base schema
(V1000–V1007) before the qhorus subclass join migration can execute.

Adding `classpath:db/ledger/migration` to `quarkus.flyway.qhorus.locations` is blocked
by a version conflict: qhorus has `V1003__agent_message_ledger_entry.sql`; ledger owns
`V1003__ledger_entry_archive.sql`. Flyway fails with `Found more than one migration with
version 1003` when both paths are scanned.

The fix requires two changes: renumber the qhorus migration, then add the ledger path.

---

## Changes

### 1. Rename migration file

```
db/qhorus/migration/V1003__agent_message_ledger_entry.sql
→
db/qhorus/migration/V1008__agent_message_ledger_entry.sql
```

**V1008** is the first available slot after ledger's base range (V1000–V1007). Platform
convention: consumer-owned ledger subclass join tables use V1008+.

SQL content (`CREATE TABLE agent_message_ledger_entry ...`) is unchanged. The comment
block inside the file is updated to reflect the new version number and rationale.

### 2. Update `quarkus.flyway.qhorus.locations`

`runtime/src/main/resources/application.properties`:

```properties
# Before
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration

# After
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration
```

`runtime/src/test/resources/application.properties` is untouched — qhorus tests use
`database.generation=drop-and-create` with `migrate-at-start=false`.

### 3. Improve `FlywayMigrationSchemaTest`

The existing test (`runtime/src/test/java/.../FlywayMigrationSchemaTest.java`) stubs
`ledger_entry` by hand in `@BeforeAll` before running Flyway on only
`classpath:db/qhorus/migration`. After this fix:

- Remove the manual `CREATE TABLE ledger_entry` stub
- Run Flyway with `classpath:db/qhorus/migration,classpath:db/ledger/migration`
- Ledger's V1000 migration creates the real `ledger_entry` table
- Qhorus V1008 migration creates `agent_message_ledger_entry` via the real FK

This mirrors production exactly. Existing assertions (`commitmentTableExists`,
`pendingReplyTableIsAbsent`) are unchanged. Add one new assertion:
`agentMessageLedgerEntryTableExists` — verifies the combined scan creates the subclass
table end-to-end.

### 4. Documentation updates (parent repo)

**`docs/PLATFORM.md`:**
- Persistence table: `V1004+ numbering` → `V1008+ numbering` for consumer subclass join tables
- Protocol bullet: `V1004+ = ledger subclass joins` → `V1008+ = ledger subclass joins`

**`docs/repos/casehub-qhorus.md`:**
- Migration table: V1003 → V1008 for `agent_message_ledger_entry`

---

## TDD Cycle

1. Update `FlywayMigrationSchemaTest` to scan both locations and add the new assertion
   → **RED** (V1003 still present, version conflict)
2. Rename `V1003` → `V1008` → **GREEN**
3. Update `application.properties` (runtime) → existing Quarkus integration tests validate
4. Run full `mvn clean test` → all green

---

## Out of scope

- Other consumers (aml, clinical, devtown): tracked by casehubio/aml#26

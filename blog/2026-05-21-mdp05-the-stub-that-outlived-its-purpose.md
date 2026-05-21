---
layout: post
title: "The stub that outlived its purpose"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
tags: [flyway, testing, ledger, migration]
---

`FlywayMigrationSchemaTest` had this at the top of `@BeforeAll`:

```java
// ledger_entry is owned by casehub-ledger and migrated before qhorus in production.
// Create the minimal schema here to satisfy the FK in V1003.
try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "")) {
    conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS ledger_entry (
                id UUID NOT NULL,
                CONSTRAINT pk_ledger_entry PRIMARY KEY (id)
            )
            """);
}
```

A hand-rolled table stub, four columns stripped away, just enough to satisfy the foreign key. Written at a point when qhorus couldn't scan `classpath:db/migration` to get the real ledger schema — it would have pulled in casehub-work's V1 through V27 alongside the ledger V1000 files, and Flyway would have failed with "Found more than one migration with version 1".

The stub worked. It also tested against a schema with no `dtype` discriminator, no `seq_num`, no `actor_id`. The FK resolved; the rest of the ledger structure was invisible.

After ledger#95 moved the base migrations to `classpath:db/ledger/migration`, the blocker was gone. We still couldn't add the path because qhorus had `V1003__agent_message_ledger_entry.sql` and ledger had `V1003__ledger_entry_archive.sql` — scanning both would still fail, different reason. Once qhorus renamed its migration to V2000, the path was clear.

With both fixes in place:

```java
@BeforeAll
static void migrate() {
    // Run both migration locations together — Flyway sorts by version number.
    // Ledger migrations (V1000+) create ledger_entry before qhorus V2000 runs.
    // No manual ledger_entry creation needed: mirrors the production config exactly.
    Flyway.configure()
            .dataSource(JDBC_URL, "sa", "")
            .locations("classpath:db/qhorus/migration", "classpath:db/ledger/migration")
            .load()
            .migrate();
}
```

The stub is gone. The `baselineOnMigrate(true)` and `baselineVersion("0")` that existed solely to let Flyway treat the pre-existing stub as an established baseline — also gone. 19 real migrations now run: ledger V1000–V1007 create the full `ledger_entry` schema, then qhorus V1–V10 build the channel and message tables, then V2000 adds `agent_message_ledger_entry` via the real foreign key.

The test now actually tests what production does. The stub was the right call at the time — you work with what you have. But a stub that outlasts the condition that required it becomes a liability: it hides schema mismatches, it makes the test less useful as a regression guard, and it carries a comment that eventually reads as archaeology rather than documentation.

The new assertion:

```java
void agentMessageLedgerEntryTableExists() throws Exception {
    try (Connection conn = DriverManager.getConnection(JDBC_URL, "sa", "");
         var rs = conn.getMetaData().getTables(null, null, "AGENT_MESSAGE_LEDGER_ENTRY", ...)) {
        assertTrue(rs.next(),
                "agent_message_ledger_entry must exist — created by qhorus V2000 after ledger_entry (V1000) is in place");
    }
}
```

If the FK migration fails — wrong version, wrong path, missing ledger dependency — this test will tell you exactly what broke.

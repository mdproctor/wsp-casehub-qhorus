---
layout: post
title: "The JAR disagreed with the protocol doc"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [qhorus]
---

The fix for issue #179 looked straightforward: add `classpath:db/ledger/migration`
to the qhorus Flyway locations and renumber the V1003 subclass join migration to
avoid the conflict with ledger's own V1003. The original issue suggested V1004 as
the destination, with V2000 as an alternative if we wanted to stay clear of the
ledger range entirely. We went with V2000.

Then AML's Claude flagged qhorus#180 asking for V1004 specifically, citing a
platform versioning protocol: V1000–V1003 = ledger base schema, V1004+ =
consumer-owned subclass join migrations.

I changed V2000 back to V1004 and ran the tests.

```
org.flywaydb.core.api.FlywayException:
Found more than one migration with version 1004
Offenders:
-> db/qhorus/migration/V1004__agent_message_ledger_entry.sql (SQL)
-> file:~/.m2/.../casehub-ledger-0.2-SNAPSHOT.jar!/db/ledger/migration/V1004__actor_identity.sql (SQL)
```

The platform versioning protocol was behind the JAR. Ledger 0.2-SNAPSHOT already
had `V1004__actor_identity.sql`. The protocol said V1004+ was consumer territory;
the actual artifact said otherwise. V2000 was right.

The only way to catch this was to run both locations together in a test — which
the original `FlywayMigrationSchemaTest` didn't do. It manually created a minimal
`ledger_entry` table in `@BeforeAll`, then ran Flyway on just `classpath:db/qhorus/migration`.
That approach works but it silently masks any version collision with the ledger's actual
migrations.

We updated the test to specify both locations directly:

```java
Flyway.configure()
        .dataSource(JDBC_URL, "sa", "")
        .locations("classpath:db/qhorus/migration", "classpath:db/ledger/migration")
        .load()
        .migrate();
```

No manual table bootstrap needed. Flyway sorts across locations by version number,
so ledger's V1000 runs before qhorus's V2000, and the FK dependency is satisfied
naturally. And now the test catches exactly the kind of version conflict that caught
us here. 1108 tests pass.

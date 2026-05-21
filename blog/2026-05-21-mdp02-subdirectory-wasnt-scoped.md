---
layout: post
title: "The subdirectory that wasn't scoped"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [flyway, quarkus, multi-datasource]
---

The platform protocol for Flyway migrations said: if your module has its own named datasource, place migrations in `db/migration/<module>/` and configure `quarkus.flyway.<datasource>.locations=classpath:db/migration/<module>`. This scopes which datasource *runs* those migrations.

What it doesn't scope is which datasource *finds* them.

Flyway scans classpath locations recursively. `classpath:db/migration` finds everything under `db/migration/` — including `db/migration/qhorus/`. So when casehub-aml tried to enable Flyway on its default datasource while depending on both casehub-work and casehub-qhorus, it got:

```
FlywayException: Found more than one migration with version 1
```

No indication that two datasource scanners were involved. No indication that the conflict came from a subdirectory of the scan root. Just a duplicate version, and a lot of head-scratching.

The fix is to move qhorus's migrations completely outside the `db/migration/` namespace:

```
db/qhorus/migration/V1__initial_schema.sql   (was: db/migration/qhorus/V1__initial_schema.sql)
```

And update `quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration`. Eleven SQL files renamed — no content changes — plus one line in `application.properties` and one line in `FlywayMigrationSchemaTest`. Build green, all tests pass.

The platform protocol also needed correcting. The "scoped directory" entry said `db/migration/<module>/` without noting this path is still under the default scan root. We updated Rule 4 to make the constraint explicit: the path must be outside `db/migration/` entirely. The corrected rule points to `db/<module>/migration/` as the pattern.

The error message gives you no useful signal. "Found more than one migration with version 1" looks like a duplicate file problem; it takes a while to realise the second scanner is the culprit. Worth knowing before you spend time looking for a duplicate SQL file that doesn't exist.

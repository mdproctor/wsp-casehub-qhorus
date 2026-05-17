# Flyway Scoped Migration Location — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate the Flyway V2 classpath conflict between casehub-qhorus and casehub-work by moving qhorus migrations into a scoped `db/migration/qhorus/` subdirectory.

**Architecture:** Move all 10 migration SQL files from `db/migration/` to `db/migration/qhorus/`, update the `quarkus.flyway.qhorus.locations` property to point at the scoped directory. No code changes — file moves and one config line only.

**Tech Stack:** Maven, Quarkus 3.32.2, Flyway (named `qhorus` datasource)

**Issue:** #142

---

## File Map

| Action | Path |
|--------|------|
| Create dir | `runtime/src/main/resources/db/migration/qhorus/` |
| Move (×10) | `runtime/src/main/resources/db/migration/V*.sql` → `runtime/src/main/resources/db/migration/qhorus/` |
| Modify | `runtime/src/main/resources/application.properties` |

---

## Task 1: Move migrations and update config

**Files:**
- Create dir: `runtime/src/main/resources/db/migration/qhorus/`
- Move: all 10 `V*.sql` files from `runtime/src/main/resources/db/migration/`
- Modify: `runtime/src/main/resources/application.properties`

- [ ] **Step 1.1: Verify baseline — full test suite passes before any changes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: all tests pass (same counts as before this session)

- [ ] **Step 1.2: Create the scoped directory and move all migration files**

```bash
mkdir -p /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V1__initial_schema.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V2__add_message_target.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V3__add_channel_paused.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V4__add_watchdog.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V5__add_channel_acl.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V6__add_channel_admin_instances.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V7__add_channel_rate_limits.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V8__add_instance_read_only.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V9__add_actor_type_to_message.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/

mv /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/V1003__agent_message_ledger_entry.sql \
   /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/
```

Verify 10 files moved and the old directory is now empty:
```bash
ls /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/
ls /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/resources/db/migration/qhorus/
```

Expected: old `db/migration/` dir is empty (or gone); `db/migration/qhorus/` has all 10 files.

- [ ] **Step 1.3: Update the Flyway location in `application.properties`**

File: `runtime/src/main/resources/application.properties`

Find the line:
```properties
quarkus.flyway.qhorus.locations=db/migration
```

Replace with:
```properties
quarkus.flyway.qhorus.locations=classpath:db/migration/qhorus
```

- [ ] **Step 1.4: Run the full test suite — must pass with the new location**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: all tests pass, same counts as Step 1.1. If Flyway cannot find the migrations, it will fail immediately on startup with `FlywayException: No migrations found` — this means the path is wrong; double-check the `classpath:` prefix and directory name.

- [ ] **Step 1.5: Run the full build to confirm examples modules also compile cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests
```

Expected: `BUILD SUCCESS`

- [ ] **Step 1.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add \
  runtime/src/main/resources/db/migration/qhorus/ \
  runtime/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "fix(#142): scope qhorus Flyway migrations to db/migration/qhorus/ — eliminates V2 classpath conflict"
```

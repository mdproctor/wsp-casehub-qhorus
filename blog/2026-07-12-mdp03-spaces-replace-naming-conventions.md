---
title: "Spaces Replace Naming Conventions"
date: 2026-07-12
entry_type: note
subtype: diary
author: mdp
sequence: 3
projects:
  - casehub-qhorus
tags:
  - qhorus
  - space
  - channel-hierarchy
  - data-model
---

## The Naming Convention That Wasn't Scaling

Qhorus grouped related channels by naming convention: `case-123/work`, `case-123/observe`, `case-123/oversight`. The normative channel layout creates these three channels for every case, and the `/` delimiter implies structure. But it's just a string — there's no queryable hierarchy behind it. "Give me all channels for case 123" requires a name-prefix scan. Nesting a project space above case spaces isn't expressible at all.

Matrix solved this years ago with MSC1772. Spaces are containers above channels. They nest recursively. The hierarchy is explicit and queryable, not inferred from naming patterns.

## What We Built

`Space(id, name, description, parentSpaceId, tenancyId, createdAt)` — a new first-class entity with adjacency-list nesting. Channels gain an optional `spaceId` foreign key. Null means top-level; set it and the channel belongs to a space. The existing naming conventions still work — spaces don't replace channel names, they replace the implicit grouping that channel names were carrying.

The interesting design choices:

**No metadata map.** The issue originally included `Map<String, String> metadata` on Space for consumer-specific data like `{caseId: "123"}`. We removed it. That mapping belongs in the engine layer, not in the communication mesh. Qhorus knows about organisational hierarchy; it doesn't know what a "case" is.

**Parent-scoped name uniqueness.** Two spaces named "Sprint 1" under different parents are valid — that's the whole point of hierarchy. A global unique constraint would force `project-alpha-sprint-1` naming, which is the convention-based approach we're trying to replace. PostgreSQL handles this cleanly with partial indexes; H2 doesn't support them, so the service layer validates on both platforms and the DB constraint catches races where it can.

**MAX_DEPTH = 10.** Unbounded recursion in an adjacency list means unbounded ancestor walks for cycle detection. The practical usage is 2-3 levels (organisation > project > case), but a hard limit prevents abuse and bounds the cycle check to O(10).

**Cycle detection with a documented TOCTOU race.** `moveSpace` walks the ancestor chain to check for cycles before committing. Under READ COMMITTED, two concurrent moves can each pass the check against stale snapshots. Rather than serialising all tree mutations with advisory locks — for a race window that requires two administrators to simultaneously re-parent spaces into each other — we documented the race, bounded its impact (MAX_DEPTH prevents infinite walks), and noted the fix path if it ever matters in production.

## The Dual-Identity Pattern

Channels already have `resolveChannel(String)` — UUID parse first, name lookup fallback. Spaces follow the same pattern with `resolveSpace(String)`. But there's a wrinkle: space names aren't globally unique (they're unique within a parent), so `findByName` returns a `List<Space>`. If the list has exactly one match, the service returns it. Multiple matches throw with "Ambiguous space name — use UUID." This keeps MCP tool ergonomics clean for the common case while handling hierarchy correctly.

## Nine MCP Tools

`create_space`, `get_space`, `list_spaces`, `list_space_channels`, `rename_space`, `update_space_description`, `move_space`, `move_channel_to_space`, `delete_space`. Plus `create_channel` gains an optional `space_id` parameter that validates via `resolveSpace` before creation.

The `list_channels` tool now includes `spaceName` alongside the existing `spaceId` in `ChannelDetail`, enriched via a batch pre-fetch that collects distinct space IDs from the result set and loads them in a single `findByIds` query.

The normative channel layout in the engine can now create a Space for each case and set `spaceId` on the work/observe/oversight channels. The UI renders spaces as expandable tree nodes. Qhorus provides the primitive; consumers decide what goes in it.

# Decisions — #456 ARC42STORIES.MD Migration

## D1: §9 retrospective scope

**Choice:** Full retrospective with casehub-work-level detail per chapter (~200-300 words, before/after narrative, accountability gaps, layer impact table)
**Alternatives:**
- Prospective only — empty chapter index, future work fills it. Keeps task at L scale but defers the most valuable content.
- Retrospective skeleton — journey overview + sparse index from major milestones. Lighter but inconsistent with the casehub-work template quality.
- Hybrid — full entries for major chapters, summary for incremental. Saves effort but creates inconsistent depth across chapters.
**Rationale:** Qhorus is a mature module with ~450 issues of history. A complete retrospective creates a comprehensive architectural record that matches the casehub-work template quality and makes the document immediately useful as a reference.
**Trade-offs:** Significantly increases task scale from L to XL. Requires extensive git/issue research across the full project history.
**Sources:** casehub-work/ARC42STORIES.MD (template), casehubio/qhorus#456 (issue)
**Exploration:** quick
**Status:** captured

## D2: DESIGN.md disposition

**Choice:** Delete docs/DESIGN.md after migration
**Alternatives:**
- Redirect stub — replace with a one-liner pointing to ARC42STORIES.MD. Low maintenance but an unnecessary indirection.
- Keep alongside — maintain both documents. Creates drift risk with no benefit.
**Rationale:** The arc42stories profile explicitly says "Those files are retired when the migration is complete." ARC42STORIES.MD fully supersedes DESIGN.md.
**Trade-offs:** Any external links to docs/DESIGN.md will break. Mitigated by git history preserving the content.
**Sources:** arc42stories-casehub-profile.md §Document Artifact Mapping
**Exploration:** quick
**Status:** captured

## D3: ARC42STORIES.MD location

**Choice:** Project repo root (`/qhorus/ARC42STORIES.MD`)
**Alternatives:**
- Workspace root — literal reading of the profile's artifact mapping table, but inconsistent with all other platform repos.
**Rationale:** Matches casehub-work and all other platform repos. The document is a permanent architecture record that belongs with the code.
**Trade-offs:** None significant — this is the established convention.
**Sources:** casehub-work/ARC42STORIES.MD location, arc42stories-casehub-profile.md
**Exploration:** quick
**Status:** captured

## D4: Migration approach

**Choice:** Phased writing with subagent parallelism — §1-§8 structural sections first, then §9 journey-by-journey via parallel subagent research
**Alternatives:**
- Sequential write — top-to-bottom single pass. Simpler coordination but slower, risk of context exhaustion before completing ~50 chapters.
- Split into two issues — §1-§8 now, §9 as follow-up. Gets file in place quickly but defers the most valuable content.
**Rationale:** Parallel research per journey is the most efficient way to reconstruct ~50 chapters. Structural sections (§1-§8) are straightforward from existing CLAUDE.md and DESIGN.md content. Subagent parallelism keeps total wall-clock time manageable.
**Trade-offs:** Requires assembly and consistency review across parallel outputs. Slightly more complex coordination than sequential.
**Sources:** CLAUDE.md project structure, DESIGN.md build roadmap
**Exploration:** quick
**Status:** captured

## D5: Source material for chapter reconstruction

**Choice:** All three sources — CLAUDE.md (primary feature documentation), git log (dates, ordering, issue numbers), and GitHub issues (descriptions, labels, cross-references)
**Alternatives:**
- CLAUDE.md + git log only — misses issue descriptions and cross-references that provide rationale.
- Git log primary — accurate timeline but slow and misses architectural rationale not captured in commits.
- GitHub issues primary — good descriptions but many closed issues may lack detail.
**Rationale:** Each source provides complementary information. CLAUDE.md has the richest technical detail. Git log provides the authoritative timeline. GitHub issues provide the "why" and cross-references between features.
**Trade-offs:** More research per chapter, but produces the most accurate and complete retrospective.
**Sources:** User preference
**Exploration:** quick
**Status:** captured

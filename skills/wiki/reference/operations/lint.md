# Operation: lint

> Reference doc for the `wiki` master skill. The agent reaches this file via the routing table in `wiki/SKILL.md` when the user says "lint the wiki", "health-check the knowledge base", "audit the tags", "what's broken in the index", or periodically as a maintenance pass.

Periodic health check for the tagged knowledge system. Surfaces drift, decay, and inconsistencies that accumulate as the system grows. Does **not** auto-fix non-trivial issues — produces a report and offers to route to the appropriate operation or skill for each fix.

## When to apply this operation

- "lint the wiki", "audit the knowledge base", "health-check tagging", "find orphans / dead links"
- After a long period without maintenance
- After bulk operations that may have left things inconsistent
- When the user notices weird behavior in retrieval and wants to find why

## Checks performed

For each check, the linter outputs: count, severity (`info` / `warn` / `error`), and a list of items.

### Vocabulary checks

| Check | Description | Severity |
|---|---|---|
| **Orphan tags** | Tags defined in `TAGS.md` that no doc uses | info |
| **Orphan entities** | Entities defined in `ENTITIES.md` that no doc uses | info |
| **Unknown tags** | Doc frontmatter uses a tag NOT in `TAGS.md` | error |
| **Unknown entities** | Doc frontmatter uses an entity NOT in `ENTITIES.md` | error |
| **Tag definition missing "Not for" clause** | Drift-prevention rule violated | warn |
| **Similar tag names** | Two tags differ only by hyphenation, plural, or near-synonym (heuristic) — possible drift candidates | warn |

### Index checks

| Check | Description | Severity |
|---|---|---|
| **Stale INDEX entry** | Entry points to a file that doesn't exist | error |
| **Missing INDEX entry** | File has full frontmatter but isn't in `INDEX.md` | error |
| **Drifted INDEX entry** | Entry's tags/summary/status disagrees with the file's frontmatter | warn |
| **Duplicate IDs** | Two files have the same `id` in their frontmatter | error |

### Source / summary checks

| Check | Description | Severity |
|---|---|---|
| **Broken `source_path`** | Summary frontmatter references a source file that doesn't exist | error |
| **Broken `source` reference** | Summary's `source:` id isn't in `INDEX.md` | warn |
| **Orphan summary** | Summary exists but its source doesn't (file missing AND not a URL) | warn |
| **Source without summary** | Source doc exists but has no corresponding summary — info only (user may not want one) | info |

### Document checks

| Check | Description | Severity |
|---|---|---|
| **Untagged candidate** | A `.md` file in the project (excluding `tagging/`, `wiki/`, `.claude/`, etc.) with no frontmatter — might be an oversight | info |
| **Partial frontmatter** | Has frontmatter but missing required fields (`id`, `type`, `tags`, `summary`) | warn |
| **Invalid type** | `type:` is not one of the allowed enum values | error |
| **Invalid status** | `status:` is not one of the allowed enum values | warn |

## Procedure

1. **Locate `tagging/`** and `wiki/` (if present). If `tagging/` is absent, nothing to lint.
2. **Run all checks above.** Build a categorized report.
3. **Print the report** as a summary table first (counts per severity), then expand by category. Keep it scannable — full lists only on user request for high-count categories.
4. **Offer fixes** per category. Each fix delegates to the right operation or skill:
   - Stale INDEX entry, missing INDEX entry, drifted entry → apply the `reindex` operation.
   - Orphan tag/entity, similar tag names → suggest the `tag-rename` operation (deprecate, rename, or merge).
   - Unknown tag/entity in doc → suggest the user fix the doc frontmatter (or apply `tag-rename` if it's a stale name).
   - Untagged candidate → suggest `tag-document` for one, the `bulk-tag` operation for many.
   - Broken `source_path` → surface for manual fix; can't auto-resolve.
   - Duplicate IDs → require manual resolution (which file should keep the ID?).
5. **Apply only trivial fixes automatically** if user says "fix what you can":
   - Trivial = adding orphan tags to a "deprecated" section in `TAGS.md` with the user's consent.
   - NOT trivial = anything modifying doc frontmatter, deleting entries, or applying the `reindex` operation. Always confirm.
6. **Append to `wiki/log.md`** (if present) — one entry per lint run, regardless of whether fixes were applied:
   ```
   ## [YYYY-MM-DD] lint | 12 issues found (3 error, 4 warn, 5 info)
   Categories: stale-index(2), unknown-tag(1), orphan-tag(3), untagged-candidate(5), similar-tags(1).
   ```

## Output format

Concise report. Example:

```
Wiki lint report — 2026-05-13

Summary:
  errors: 3
  warnings: 4
  info: 5

Errors:
  - Stale INDEX entry: notes/old-foo.md (file missing)
  - Stale INDEX entry: wiki/summaries/draft.md (file missing)
  - Unknown tag in doc: notes/bar.md uses `agents-old` (not in TAGS.md)

Warnings:
  - Similar tag names: `agents` vs `agent-frameworks` — possible drift candidates
  - Partial frontmatter: scratch/notes-from-tuesday.md (missing `summary`)
  - Drifted INDEX entry: wiki/summaries/acme.md (frontmatter has new tags, INDEX.md is older)
  - Tag missing "Not for": `prompt-engineering` (in TAGS.md)

Info:
  - 3 orphan tags: `temp-tag`, `unused-1`, `unused-2`
  - 5 untagged candidates: docs/* (5 files with no frontmatter)

Suggested actions:
  - Apply the reindex operation to fix the 3 INDEX issues.
  - Apply tag-rename to consolidate `agents` / `agent-frameworks`.
  - Apply bulk-tag on docs/* if those should be in the index.
  - Manually fix unknown tag in notes/bar.md.
```

## What this operation does NOT do

- **Auto-fix anything non-trivial.** All meaningful changes go through `tag-rename`, `reindex`, `tag-document`, or `bulk-tag` with user confirmation.
- **Modify document body content** under any circumstances.
- **Decide whether orphan tags should be removed.** Surfaces them; the user decides.
- **Run on a schedule.** Manual invocation only — there's no daemon.

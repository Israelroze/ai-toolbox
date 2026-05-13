# Operation: reindex

> Reference doc for the `wiki` master skill. The agent reaches this file via the routing table in `wiki/SKILL.md` when the user says "rebuild the index", "regenerate INDEX.md", "reindex", "the index is out of sync", or after manual frontmatter edits, file renames, or deletions.

Rebuild `tagging/INDEX.md` from the on-disk frontmatter of all tagged documents in the project. Use whenever the index has drifted — manual frontmatter edits, file renames, deletions, or doc moves can desync it.

This operation is **read-mostly**: it scans frontmatter, builds a fresh index in memory, shows the user a diff, and only writes `INDEX.md` after confirmation. It NEVER touches individual document frontmatter.

## When to apply this operation

- "rebuild the index", "regenerate INDEX.md", "reindex", "the index is out of date", "sync the index"
- The user has just done bulk manual edits to doc frontmatter and wants the index updated
- After a file rename / move / deletion done outside the skills
- After the `lint` operation flagged index drift

Do NOT apply when:
- The user is just adding a single new doc — `tag-document` updates `INDEX.md` directly
- The user wants to find untagged docs — that's the `lint` or `bulk-tag` operation

## Procedure

1. **Locate `tagging/`**. If absent, tell the user there's nothing to reindex; suggest `wiki` (master) or `tag-document` for setup.
2. **Read the current `INDEX.md`** into memory as the "before" state.
3. **Scan the project** for tagged docs:
   - Walk the project tree (respect `.gitignore`; skip `.git/`, `node_modules/`, `tagging/` itself).
   - For each `.md` file, read just the frontmatter. A doc is "tagged" if it has at minimum: `id`, `type`, and `tags` fields. Skip files without frontmatter.
   - Validate each doc's tags/entities exist in `TAGS.md` / `ENTITIES.md`. Collect any violations into a "vocabulary issues" list (don't fix here — surface to user).
4. **Build the "after" state** of `INDEX.md` from the scanned frontmatter, grouped by `status` (Active / Draft / Archived) and sorted within each group by `date` descending (newest first).
5. **Compute the diff** vs the current `INDEX.md`:
   - **Added** — docs found on disk but missing from current `INDEX.md`.
   - **Removed** — entries in current `INDEX.md` whose files no longer exist or no longer have frontmatter.
   - **Modified** — entries whose tags/entities/summary/title/status changed.
   - **Vocabulary issues** — docs using tags/entities not in `TAGS.md` / `ENTITIES.md`.
6. **Present the diff to the user** as a categorized report. Counts first, then specifics. Example:
   ```
   Reindex preview:
     + 3 added (paths: ...)
     - 1 removed (stale entry: notes/old-foo.md no longer exists)
     ~ 5 modified (mostly tag changes)
     ! 2 vocabulary issues (using tags not in TAGS.md: ...)
   ```
7. **Confirm with the user** before writing. *"Write the new index? (yes / no / show full diff)"*
8. **On approval, overwrite `INDEX.md`** with the rebuilt content. Append to `wiki/log.md` (if present):
   ```
   ## [YYYY-MM-DD] reindex | +3 -1 ~5
   Rebuilt INDEX.md from frontmatter scan. Vocabulary issues: 2 (see report).
   ```
9. **Vocabulary issues are NOT auto-fixed.** Surface them and suggest: edit the doc to use an approved tag, apply the `tag-rename` operation if the doc's tag is a stale name, or add the tag via `tag-document`'s approval flow.

## Edge cases

- **No tagged docs found.** Tell the user; nothing to write.
- **`INDEX.md` does not exist but tagged docs do.** Treat as full rebuild; the diff is "all added".
- **A doc has frontmatter but no `id`.** Skip and flag — it's not a fully tagged doc.
- **Duplicate `id`s across docs.** Surface as a vocabulary issue; do NOT write the index until the user resolves (could overwrite the wrong file's pointer otherwise).

## What this operation does NOT do

- **Modify document frontmatter.** Only `INDEX.md` is rewritten.
- **Add missing tags to docs.** Use `tag-document` or the `bulk-tag` operation for that.
- **Rename tags.** Use the `tag-rename` operation.
- **Health-check the whole system.** Use the `lint` operation (which may invoke this one).

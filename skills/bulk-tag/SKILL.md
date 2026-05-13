---
name: bulk-tag
description: Tag many existing documents in one batch operation, used to retrofit the tagging system onto a project that already has untagged markdown files. Invoke when the user says "tag everything in folder X", "bulk-tag the notes/ folder", "retrofit this project", "tag all untagged docs", or "add the existing files to the index". Batches new-tag proposals so the user doesn't get bombarded with approval prompts one-by-one — collects suggestions first, then asks for batch approval, then tags in chunks. Heavy operation, opt-in only.
---

# bulk-tag

Apply the `tag-document` procedure to many files in one batch, with smart batching of the approval flow so the user isn't asked about new tags one doc at a time. Used to retrofit an existing project (50, 200, 1000 docs) into the tagging system.

## When to invoke

- "bulk-tag everything in `notes/`"
- "tag all untagged markdown in this project"
- "retrofit this project into the wiki"
- "I have 80 files I need indexed — do them all"
- After `wiki-lint` flags many untagged candidates and the user wants to fix them all

Do NOT invoke when:
- The user has just one or a small handful of docs — that's `tag-document`.
- The user wants only to summarize many docs — there's no bulk-summarize skill yet (and it would be very heavy; build it only on explicit request).

## Procedure

### Phase 1 — scope

1. **Confirm scope with the user.** Required: which files/folders to process. Default candidate set: all `.md` files under the project root, excluding `tagging/`, `wiki/`, `.claude/`, `.git/`, `node_modules/`, and anything in `.gitignore`.
2. **Filter candidates:**
   - Skip files that already have a complete frontmatter (`id`, `type`, `tags`, `summary` all present) — they're already tagged.
   - Files with partial frontmatter: include in the batch but treat as "complete partial" (preserve existing fields, fill missing ones).
3. **Confirm the count and a sample.** *"Found 73 untagged candidates. Sample: notes/foo.md, docs/bar.md, scratch/baz.md. Proceed? (yes / show all / narrow scope)"* Don't proceed without explicit OK.

### Phase 2 — proposal collection (no writes yet)

4. **For each candidate, in a single sweep WITHOUT writing anything**:
   - Read the doc.
   - Extract candidate concepts (topics, entities, doc type).
   - Match concepts to existing `TAGS.md` / `ENTITIES.md` using semantic similarity.
   - For each doc, record: (a) the tags/entities it WOULD get from existing vocabulary, (b) any new tags/entities it WOULD propose.
5. **Aggregate the new-tag proposals across the whole batch.** Deduplicate — if 12 docs all need a new `agents` tag, that's one proposal, not 12.
6. **Present the consolidated proposal list to the user** in one batch:
   ```
   Batch-tagging 73 documents. Proposed NEW vocabulary entries (need your approval):

   New tags:
     1. `agents` — definition: ... — would be used by 12 docs (samples: notes/foo.md, ...)
     2. `prompt-injection` — definition: ... — would be used by 4 docs (...)
     3. ... (5 more)

   New entities:
     1. `Claude Code` — category: tools — would be used by 18 docs
     2. ... (3 more)

   Approve / edit / reject each, or `approve all`. After this batch is approved, I'll tag all 73 docs in one pass.
   ```
7. **Wait for batch approval.** User can: approve all, approve some, edit definitions, reject. Apply user's decisions to the proposals.

### Phase 3 — apply

8. **For each approved new tag/entity, append to `TAGS.md` / `ENTITIES.md`** with the agreed definitions.
9. **For docs whose proposed tags were rejected**, decide per-doc: either skip the doc (note in report), or re-match against existing vocabulary only (force-fit to closest existing tag — surface this as lossy).
10. **Process the batch in chunks** (e.g., 10 docs at a time). For each chunk:
    - Write frontmatter to each doc, preserving any pre-existing fields.
    - Update `INDEX.md` with the new entries (append; final reindex at end will sort/clean).
    - Show progress: *"Tagged 10/73. Continuing..."*
11. **After all chunks done, run `wiki-reindex`** to ensure `INDEX.md` is clean (sorted, grouped, no duplicates).
12. **Append to `wiki/log.md`** (if present):
    ```
    ## [YYYY-MM-DD] bulk-tag | 73 docs across notes/, docs/, scratch/
    Added vocabulary: 8 tags, 4 entities. Skipped: 0. Lossy force-fit: 0.
    ```
13. **Report the result** — count tagged, count skipped, vocabulary additions, any per-doc warnings.

## Batching rationale

`tag-document` asks the user about new tags one doc at a time. For 73 docs, that's 73 approval prompts and probably the same new tag proposed many times. Bulk-tag flips the order:

1. **Read everything first** (no writes, no prompts).
2. **Aggregate all proposed vocabulary additions** (deduplicate).
3. **One batch approval prompt** with the consolidated list.
4. **Write everything in one pass** after approval.

This converts O(N) prompts into O(1) — the user reviews the proposed vocabulary once, then the agent applies it everywhere.

## Edge cases

- **No candidates found.** Tell the user; nothing to do.
- **A candidate already has complete frontmatter.** Skip silently. Report at end: *"Skipped 12 docs that were already tagged."*
- **A candidate has partial frontmatter.** Preserve existing fields (especially `id` if present); fill in missing ones. Surface any conflicts (e.g., existing `type` is invalid).
- **The user rejects ALL proposed new tags.** Ask if they want to (a) skip docs that need new tags entirely, or (b) force-fit to existing tags (lossy). Default to skip — never silently lose precision.
- **Project has no `tagging/` folder yet.** Bootstrap it first (via `tag-document` or `wiki`), then proceed.

## Safety

- **One git commit per batch run.** After completion, suggest the user commit immediately. The cascade is large; recovery from a partial mistake is easier from a clean version.
- **Never overwrite existing frontmatter fields.** Merge only — fill in what's missing, leave the rest.
- **Show a sample, not the full list, in early prompts** (Phase 1). Full lists only on user request.

## What this skill does NOT do

- **Summarize docs.** Tagging only. Summarization is `wiki-ingest`, one source at a time, by explicit request.
- **Modify document body content.**
- **Move or rename files.**
- **Decide what's a "tagged candidate" beyond the heuristic** (any `.md` with no `id` in frontmatter, in non-excluded folders). The user can override the candidate set in Phase 1.

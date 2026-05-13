# Operation: tag-rename

> Reference doc for the `wiki` master skill. The agent reaches this file via the routing table in `wiki/SKILL.md` when the user says "rename tag X to Y", "merge tags A and B into C", "this tag should be called Z instead", or wants to consolidate vocabulary.

Rename or merge tags/entities across the project, cascading the change through the vocabulary file, every affected document's frontmatter, and `INDEX.md`. This is the one operation that bypasses the "tags are append-only" default in `tag-document`.

## When to apply this operation

- "rename tag `rag` to `retrieval-augmented-generation`"
- "merge tags `agents` and `agent-frameworks` into `agents`"
- "deprecate `temp-tag`"
- "this entity has the wrong name — fix it"
- "consolidate these similar tags"

Do NOT apply when:
- The user wants to ADD a tag — that's `tag-document`'s approval flow.
- The user wants to remove an unused tag with no docs using it — that's also fine here (counts as deprecation), but the `lint` operation will surface and offer the same.

## Operations supported

1. **Rename** — one old tag → one new tag. The new tag replaces the old everywhere.
2. **Merge** — multiple old tags → one new tag (which may or may not already exist).
3. **Deprecate** — remove a tag from `TAGS.md` (only allowed if no docs use it after the operation, OR if the user explicitly chooses to leave affected docs un-retagged, which is rare).

Works identically for entities (`ENTITIES.md`).

## Procedure

1. **Locate `tagging/`**. If absent, tell the user there's nothing to rename.
2. **Confirm the operation** with the user:
   - For rename: old name, new name. Is the new name already in `TAGS.md`? If yes, this is effectively a merge.
   - For merge: list of old names, target name. Target may or may not already exist.
   - For deprecate: name being deprecated.
3. **If the new/target name is not already in the vocabulary**, propose adding it via the standard `tag-document` approval flow first (definition, "Not for" clause). Do not proceed until vocabulary entry exists.
4. **Compute the change set** by scanning `INDEX.md`:
   - List every doc whose frontmatter contains the old tag(s).
   - For each: show the path, current tags, and tags-after-rename.
5. **Present the change set to the user**:
   ```
   Rename: rag → retrieval-augmented-generation
   
   Affects 7 documents:
     - notes/llm-overview.md: [rag, llm, hallucination] → [retrieval-augmented-generation, llm, hallucination]
     - wiki/summaries/karpathy-talk-summary.md: [rag, agents] → [retrieval-augmented-generation, agents]
     - ... (5 more)
   
   Will also: remove `rag` from TAGS.md, update INDEX.md.
   ```
6. **Confirm with the user** before writing. *"Apply? (yes / no / show full list)"*
7. **On approval, apply atomically** (sequence — if any step fails, roll back the earlier ones):
   1. Read each affected doc, rewrite its frontmatter with the new tag value. Preserve all other frontmatter fields exactly.
   2. Update `TAGS.md`: remove the old entry (rename / deprecate) or remove all old entries being merged.
   3. Update `INDEX.md` to reflect the new tags. (Or apply the `reindex` operation if many entries are affected — cleaner than ad-hoc editing.)
   4. Append to `wiki/log.md` (if present):
      ```
      ## [YYYY-MM-DD] tag-rename | rag → retrieval-augmented-generation
      Updated 7 documents. Removed `rag` from TAGS.md.
      ```
8. **Report the result** to the user — count of docs updated, vocabulary state after.

## Merge semantics

- If a doc has BOTH old tags (e.g., merging `agents` + `agent-frameworks` and a doc has both), the result has the target tag exactly once (no duplicates).
- If the merge target tag is also one of the old names (e.g., merging `agent-frameworks` INTO `agents`), `agents` is preserved as the survivor and `agent-frameworks` is removed.

## Edge cases

- **No docs use the old tag.** Allowed — just remove from `TAGS.md` and confirm with user.
- **The new tag and old tag are identical.** No-op; tell user.
- **A doc uses the old tag but is missing from `INDEX.md`.** Surface this — the index is desynced. Suggest applying the `reindex` operation before the rename.
- **The user wants to KEEP the old tag as an alias.** Add an "Aliases" note to the new tag's `TAGS.md` entry rather than removing the old one; mention this is unusual.

## Atomicity & safety

This operation touches many files. Two safety rules:

1. **Always show the full change set before writing.** Never apply without explicit confirmation, even for small operations.
2. **Prefer one git commit per rename.** After the operation completes, suggest the user commit immediately so the cascade is captured as a single atomic change in version control. Recovery from a partial mistake is then trivial.

## What this operation does NOT do

- **Add new tags** (other than the target of a merge that didn't exist yet — and even then, only via the standard approval flow).
- **Modify document content** outside the frontmatter `tags` / `entities` arrays.
- **Rename tag definitions in summary body text.** If a doc body mentions a tag by name in prose, this operation won't catch it. Surface this as a known limitation when the user runs the operation.

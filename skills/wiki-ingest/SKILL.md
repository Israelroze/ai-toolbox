---
name: wiki-ingest
description: Summarize a source document and file the summary into a project wiki, tagging both for later retrieval. ONLY invoke on EXPLICIT user request — phrases like "summarize this", "ingest this source", "add to wiki", "extract notes", "process this for the wiki". This is a heavy operation (full read + drafting + multiple file writes); NEVER trigger it as a side-effect of tagging, indexing, or general file handling. The default workflow for new documents is plain tagging via the tag-document skill; this skill activates only when the user wants the additional summary artifact. Bootstraps a wiki/ folder at the project root with summaries/ and log.md on first use. Composes on top of the tag-document skill — both the new summary and (optionally) the source get tagged via that skill so they appear in INDEX.md.
---

# wiki-ingest

Ingest a source document into a project's wiki: read it, write a summary, file the summary into `wiki/summaries/`, tag the summary (and optionally the source) using the `tag-document` skill, and append an entry to `wiki/log.md`. The pattern is adapted from Karpathy's LLM wiki concept — see `../wiki/reference/karpathy-wiki.md` for the full conceptual write-up.

This skill is the **wiki ingest operation**. Other Karpathy-pattern operations (lint, entity-page generation) are out of scope here. Retrieval is already handled by `tag-document`'s retrieval procedure.

## When to invoke

**This skill is opt-in and heavy.** It must NEVER activate automatically. The user has to explicitly ask for a summary. Drafting a summary involves a full read of the source and several file writes — do not pay that cost unless the user has clearly requested it.

Invoke ONLY when the user explicitly says (or clearly means) one of:
- "summarize this / ingest this / process this for the wiki"
- "add this to the wiki / file this into the wiki"
- "extract notes from this"
- "give me the gist and save it"

Do NOT invoke when:
- The user adds, opens, or tags a doc without asking for a summary — that's plain `tag-document` (lightweight).
- The user asks a retrieval question — `tag-document`'s retrieval procedure handles that.
- Another skill (or this skill's own logic) is processing a doc — never auto-chain into summarization.

If you're unsure whether the user wants a summary, ASK before doing the work. Example: *"You added this doc — want me to just tag it, or also generate a summary and file it in the wiki? (Summary is heavier — full read + drafting.)"*

### Confirmation gate (run before any summary work)

Before step 4 of the procedure (the actual summary drafting), briefly confirm with the user:
- The source you're about to summarize (path or title).
- The intended length (short ~200w / medium ~500w / long ~800w) — pick a default based on source size but let the user override.
- Optional focus (key claims, methodology, action items, open questions, balanced).

This confirmation is fast and prevents wasted work on the wrong source or the wrong scope.

## Storage layout (per host project)

Created at the host project root, alongside `tagging/`:

```
<host-project-root>/
├── tagging/                       # owned by tag-document skill
│   ├── TAGS.md
│   ├── ENTITIES.md
│   └── INDEX.md
├── sources/                       # raw source documents (this skill creates on first ingest)
│   ├── <source-id>.md
│   └── ...
└── wiki/                          # owned by this skill
    ├── log.md                     # append-only chronological log
    └── summaries/                 # one summary doc per ingested source
        ├── <source-id>-summary.md
        └── ...
```

`sources/` and `wiki/` both live at the project root (NOT under `.claude/`) so the user can review, edit, and version-control them directly. Future expansion within `wiki/` (not built yet): `wiki/entities/`, `wiki/concepts/`.

### Source storage rules

`sources/` is the canonical home for the raw materials this skill summarizes. Three cases on ingest:

1. **Source is already a file inside the project** (e.g., the user dropped `notes/acme-q3-call.md` into the repo and asks to ingest it). Leave the file where it is. The summary's `source_path` points to its existing path. Do NOT move it to `sources/`.
2. **Source is pasted text** (no file, the user pasted the article body into chat). Save it to `sources/<source-id>.md` with minimal frontmatter (just `id`, `title`, `date`, `type: source`), then summarize. The summary's `source_path` points to `sources/<source-id>.md`. This makes future retrieval and re-summarization possible.
3. **Source is a URL** (the user supplied a link and the agent has the fetched text). Default behavior: save the fetched content to `sources/<source-id>.md` (same as case 2) so the system has a stable copy that won't link-rot. Set `source_url` in the saved source's frontmatter so the original URL is preserved. If the user explicitly says "don't save it locally", set `source_path` to the URL itself and skip the save.

Sources in `sources/` are **immutable from the LLM's perspective** — the agent reads them but never edits them (the user may edit). Summaries and other generated artifacts go in `wiki/`, never in `sources/`.

Sources get tagged via the `tag-document` skill just like any other project doc — they appear in `INDEX.md` with `type: source`. This means a query can surface a source and its summary side by side.

## Bootstrap (first invocation in a project)

If `wiki/` does not exist:

1. Ask the user: *"I'll create `wiki/` (for summaries + log) and `sources/` (for raw source docs) at the project root. Defaults are `wiki/` and `sources/`. Accept, or supply different names?"*
2. Create the folders with:
   - `wiki/summaries/` (with a `.gitkeep`)
   - `wiki/log.md` using the template in **File templates** below
   - `sources/` (with a `.gitkeep`) — only create if it doesn't already exist; the user may have a different convention
3. If `tagging/` (owned by `tag-document`) does not exist either, bootstrap it first by running `tag-document`'s bootstrap procedure. Summaries are tagged via that skill, so its vocabulary files must exist before step 6 of the standing procedure can run.

If `wiki/` already exists, skip bootstrap and go straight to the standing procedure. Create `sources/` lazily if a pasted-text or URL source needs to be saved (case 2 or 3 above).

## Standing procedure (every invocation)

1. **Locate or bootstrap** `wiki/` and `tagging/` per above.
2. **Read the source document** in full.
3. **Confirmation gate** (mandatory — see the **Confirmation gate** subsection under **When to invoke**). Confirm: source identity, intended length, optional focus. Do not proceed to drafting until the user has confirmed (a one-liner "go ahead" is enough).
4. **Draft the summary** (see **Summary document format** below). Length scales with source:
   - Short article → ~200 words
   - Paper / long-form essay → ~500 words
   - Book chapter / long transcript → ~800 words
   Always include: key takeaways, supporting details, open questions / contradictions / things to verify.
5. **Write the summary** to `wiki/summaries/<source-id>-summary.md`. If a summary with that id already exists, confirm with the user before overwriting.
6. **Tag the summary** by following the `tag-document` skill procedure on the summary file. The `type` is always `summary`. Add `source:` and `source_path:` frontmatter fields linking back to the original.
7. **Tag the source** (only if it's a file stored inside the host project and not already in `INDEX.md`) by following the `tag-document` skill procedure on the source. Skip this step if the source is external (a URL, a one-off paste, a doc the user doesn't want stored in the project).
8. **Append to `wiki/log.md`** using the format in **File templates** below.

## Summary document format

```yaml
---
id: <source-id>-summary
title: Summary of <Source Title>
date: YYYY-MM-DD
type: summary
status: active
source: <source-id>                       # source's id in INDEX.md, if it's tagged
source_path: relative/path/to/source.md   # or an external URL if not stored in project
tags: [tag-from-vocabulary, ...]
entities: [...]
summary: One sentence describing what this summary covers.
---

# Summary of <Source Title>

## Key takeaways
- bullet 1
- bullet 2
- bullet 3

## Details
<2-4 paragraphs of supporting context, structured by sub-topic>

## Open questions / things to verify
- question or contradiction 1
- question or contradiction 2

## Related
- Cross-links to related summaries or sources in this project's wiki, if any
```

Rules:
- `id` follows the pattern `<source-id>-summary` so it's clearly derived from the source.
- `tags` and `entities` are populated by the `tag-document` procedure — only approved vocabulary values.
- The body sections (Key takeaways, Details, Open questions, Related) are conventional — adapt them to the source. A meeting transcript might use "Decisions / Action items" instead of "Key takeaways / Details", for example. Don't fight the source's shape.

## log.md format

`wiki/log.md` is append-only. Each entry starts with `## [YYYY-MM-DD] <op> | <title>` so it's parseable with `grep "^## \[" wiki/log.md | tail -N`.

```markdown
## [2026-05-13] ingest | <Source Title>
Summary at `wiki/summaries/<source-id>-summary.md`. Tags: [a, b, c]. Entities: [X, Y]. New vocabulary added (if any): tag `new-tag-1` (definition: ...).
```

Operations to use:
- `ingest` — this skill's operation
- `tag` — plain tagging via `tag-document` (without summary)
- `query` — retrieval (optional to log; log significant ones)
- `lint` — future skill

## Composition with tag-document

This skill DOES NOT duplicate tagging logic. When it's time to tag the summary (step 6) or the source (step 7), follow the `tag-document` skill's procedure exactly:
- Load `TAGS.md` / `ENTITIES.md` from `tagging/`.
- Match concepts to existing vocabulary using semantic similarity.
- Propose new tags via the approval flow if needed.
- Write frontmatter with only approved values.
- Update `INDEX.md`.

`tag-document` enforces vocabulary; this skill enforces the wiki structure. They cooperate.

## Retrieval (handled by wiki-query)

Don't re-implement retrieval here. When the user asks a question against the wiki or tagged docs, invoke the `wiki-query` skill — it owns the search + synthesis path (load `INDEX.md`, filter by tags/entities/type/status, read only candidates, synthesize with citations).

## File templates

Use these verbatim when bootstrapping the `wiki/` folder.

### `wiki/log.md`

```markdown
# Wiki Log

Append-only log of wiki operations (ingests, tags, queries, lints). Each entry starts with `## [YYYY-MM-DD] <op> | <title>` for grep-friendliness:

    grep "^## \[" wiki/log.md | tail -10

<!-- entries appended below this line -->
```

### `wiki/summaries/.gitkeep`

Empty file. Keeps the directory under version control even when no summaries exist yet.

## What this skill does NOT do

- **Fetch sources from arbitrary URLs.** If the source is a URL, the user (or another tool, e.g. a web-fetch capability) must surface its content first. This skill operates on text the agent can already read.
- **Auto-update summaries when sources change.** Summaries are written once per invocation. If a source is revised and a fresh summary is needed, re-invoke the skill — it will confirm before overwriting the existing summary file.
- **Lint or health-check the wiki.** Future skill (per Karpathy's lint operation).
- **Generate entity / concept pages.** Future skill.
- **Cross-summary synthesis.** That's a query/retrieval operation, handled by reading multiple summaries via `INDEX.md`.

## Reference

The conceptual basis for this skill — Karpathy's "LLM Wiki" gist — is bundled at `../wiki/reference/karpathy-wiki.md` (under the master `wiki` skill, since it informs the whole system, not just this skill). Read it when designing future wiki operations to stay aligned with the original pattern.

---
name: wiki-ingest
description: Summarize a source document and file the summary into a project wiki, tagging both for later retrieval. ONLY invoke on EXPLICIT user request — phrases like "summarize this", "ingest this source", "add to wiki", "extract notes", "process this for the wiki". This is a heavy operation (full read + drafting + multiple file writes); NEVER trigger it as a side-effect of tagging, indexing, or general file handling. The default workflow for new documents is plain tagging via the tag-document skill; this skill activates only when the user wants the additional summary artifact. Bootstraps a wiki/ folder at the project root with summaries/ and log.md on first use. Composes on top of the tag-document skill — both the new summary and (optionally) the source get tagged via that skill so they appear in INDEX.md.
---

# wiki-ingest

Ingest a source document into a project's wiki: read it, write a summary, file the summary into `wiki/summaries/`, tag the summary (and optionally the source) using the `tag-document` skill, and append an entry to `wiki/log.md`. The pattern is adapted from Karpathy's LLM wiki concept — see `reference/karpathy-wiki.md` for the full conceptual write-up.

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

Created at the host project root, parallel to `tagging/`:

```
<host-project-root>/
├── tagging/                       # owned by tag-document skill
│   ├── TAGS.md
│   ├── ENTITIES.md
│   └── INDEX.md
└── wiki/                          # owned by this skill
    ├── log.md                     # append-only chronological log
    └── summaries/                 # one summary doc per ingested source
        ├── <source-id>-summary.md
        └── ...
```

`wiki/` is at the project root (NOT under `.claude/`) so the user can review, edit, and version-control it directly.

Future expansion within `wiki/` (not built yet): `wiki/entities/` (per-entity pages), `wiki/concepts/` (per-concept pages).

## Bootstrap (first invocation in a project)

If `wiki/` does not exist:

1. Ask the user: *"I'll create a wiki folder at the project root for summaries and log. Default name: `wiki/`. Accept, or supply a different name?"*
2. Create the folder with:
   - `wiki/summaries/` (with a `.gitkeep` so it's version-controlled even when empty)
   - `wiki/log.md` using the template in **File templates** below
3. If `tagging/` (owned by `tag-document`) does not exist either, bootstrap it first by running `tag-document`'s bootstrap procedure. Summaries are tagged via that skill, so its vocabulary files must exist before step 6 of the standing procedure can run.

If `wiki/` already exists, skip bootstrap and go straight to the standing procedure.

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

## Retrieval (handled by tag-document)

Don't re-implement retrieval here. When the user asks a question against the wiki:
1. Load `tagging/INDEX.md` — it lists both source docs AND summaries (they're all tagged docs).
2. Filter by `type: summary` if the user wants only summaries, or include both source and summary entries for a richer answer.
3. Read the candidate docs in full and synthesize.

This is the same retrieval procedure documented in `tag-document/SKILL.md`.

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

The conceptual basis for this skill — Karpathy's "LLM Wiki" gist — is bundled at `reference/karpathy-wiki.md`. Read it when designing future wiki operations (lint, entity pages, alternative ingest workflows) to stay aligned with the original pattern.

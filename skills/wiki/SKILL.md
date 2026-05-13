---
name: wiki
description: Master skill for the project's tagged knowledge system. Invoke when the user wants an overview of "the wiki" / "the knowledge system" / "how this works", wants to set up / initialize the system from scratch ("set up the knowledge base", "init the wiki", "bootstrap tagging"), OR wants to apply any maintenance operation (rebuild index, rename/merge tags, lint/audit, bulk-tag many docs). This skill owns the architecture overview, the one-shot bootstrap, and four maintenance operations whose procedures live in reference/operations/. Read this first when the agent doesn't know which knowledge-system action to take.
---

# wiki (master)

This is the **orchestrator and overview** for a tagged knowledge system that lets a project accumulate knowledge over time: tagged documents, indexed and queryable, with optional LLM-generated summaries.

This skill does two kinds of things:
1. **Routes** routine requests to the right operational skill (`tag-document`, `wiki-ingest`, `wiki-query`).
2. **Owns** the system bootstrap and four maintenance operations whose procedures live in `reference/operations/`. When a maintenance request comes in, follow the procedure in the corresponding reference file.

Conceptual basis: Karpathy's "LLM Wiki" pattern — see `reference/karpathy-wiki.md`.

## The system at a glance

A project using this system has these folders at its root (created on demand, names confirmable on first use):

```
<host-project-root>/
├── tagging/                  # controlled vocabulary + master index
│   ├── TAGS.md
│   ├── ENTITIES.md
│   └── INDEX.md              # one-line-per-doc catalog of EVERY tagged doc
├── sources/                  # raw source documents (immutable from LLM's POV)
│   └── <source-id>.md
└── wiki/                     # LLM-generated content
    ├── log.md                # chronological log of operations (ingests, queries, etc.)
    ├── summaries/            # written by wiki-ingest — one summary per source
    │   └── <source-id>-summary.md
    └── answers/              # written by wiki-query (on user opt-in) — filed-back syntheses
        └── <answer-slug>.md
```

Everything tagged — sources, notes, prompts, configs, skills, summaries, **and filed-back answers** — appears as a row in `tagging/INDEX.md`. The index is the master catalog the agent consults first for any retrieval question.

The three derived-content types correspond to three first-class artifacts:
- `source` — raw, immutable
- `summary` — one source → one summary (via `wiki-ingest`)
- `answer` — many docs → one synthesis (via `wiki-query`'s file-back, on user accept)

## Routing (decision tree)

Two kinds of targets: **other skills** (separate, auto-discoverable) and **operations** (procedures in `reference/operations/` that this skill drives).

| User says / wants | Target | Kind |
|---|---|---|
| "tag this", "label this", "add this to the index", or adds a new doc | `tag-document` skill | skill |
| "summarize this", "ingest this source", "add to wiki", "extract notes" | `wiki-ingest` skill (heavy, opt-in) | skill |
| "what do I have on X?", "find docs about Y", any retrieval question | `wiki-query` skill | skill |
| "set up the wiki", "init the knowledge system", "bootstrap tagging" | bootstrap procedure below | **this skill** |
| "rebuild the index", "regenerate INDEX.md", "the index is out of sync" | `reference/operations/reindex.md` | operation |
| "rename tag X to Y", "merge tags A and B", "deprecate this tag" | `reference/operations/tag-rename.md` | operation |
| "health-check the wiki", "lint the knowledge base", "find orphans" | `reference/operations/lint.md` | operation |
| "tag all docs in folder X", "bulk-tag everything untagged", "retrofit" | `reference/operations/bulk-tag.md` | operation |

**For operations:** read the referenced file and follow its procedure exactly. The reference files are self-contained — they cover when to apply, the full procedure, edge cases, and safety rules.

**For skills:** the agent invokes the named skill directly; its own SKILL.md takes over.

When a user request is ambiguous, ask which they want before proceeding. Never silently chain heavy operations (`wiki-ingest`, `bulk-tag`, `lint`'s autofix mode).

## Bootstrap procedure (init the whole system in one shot)

Invoke when the user wants to set the system up from scratch in a project. Faster than letting each sub-skill bootstrap its own folder.

1. Detect what already exists. If `tagging/`, `sources/`, or `wiki/` is already present at the project root, do NOT overwrite — fold into the existing layout and skip the parts that exist.
2. Prompt the user for folder names in one shot: *"I'll set up the knowledge system at the project root with these folders. Defaults: `tagging/`, `sources/`, `wiki/`. Accept all, or supply different names?"*
3. Create the folders and template files:
   - `tagging/TAGS.md` — use the template from `tag-document/SKILL.md`
   - `tagging/ENTITIES.md` — use the template from `tag-document/SKILL.md`
   - `tagging/INDEX.md` — use the template from `tag-document/SKILL.md`
   - `sources/.gitkeep`
   - `wiki/summaries/.gitkeep`
   - `wiki/answers/.gitkeep` (filed-back syntheses from `wiki-query` go here on user accept)
   - `wiki/log.md` — use the template from `wiki-ingest/SKILL.md`
4. Append the first log entry to `wiki/log.md`:
   ```
   ## [YYYY-MM-DD] init | knowledge system bootstrapped
   Folders: tagging/, sources/, wiki/. Vocabulary starts empty.
   ```
5. Tell the user the system is ready and remind them: tagging is automatic on doc handling, but summarization (`wiki-ingest`) requires an explicit ask.

## Architecture invariants (worth reading once)

- **The index is authoritative for discovery.** Don't scan the project folder for docs; read `tagging/INDEX.md` first.
- **Vocabulary is closed.** Tags and entities must exist in `TAGS.md` / `ENTITIES.md` before they can be applied. New tags go through `tag-document`'s approval flow.
- **Sources are immutable from the LLM's POV.** `wiki/` is LLM-owned; `sources/` and the user's project files are not.
- **Summaries are first-class docs.** They have frontmatter and appear in `INDEX.md` like everything else.
- **Each project has its own scoped vocabulary.** No cross-project tag leakage.
- **Heavy operations are opt-in.** `wiki-ingest`, the `bulk-tag` operation, and the `lint` operation's autofix mode never trigger automatically.

## What this skill does NOT do

- **Tag, summarize, or query** individual docs. Those are separate skills (`tag-document`, `wiki-ingest`, `wiki-query`).
- **Modify existing state beyond the bootstrap** unless the user explicitly invoked a maintenance operation. If folders already exist, leave them alone.
- **Replace any sub-skill.** When a routine request matches a separate skill in the table above, that skill takes over directly — this skill stays out of the way.

---
name: wiki
description: Master skill for the project's tagged knowledge system. Invoke when the user asks for an overview of "the wiki", "the knowledge system", "how this works", or wants to set up / initialize the system from scratch in a fresh project ("set up the knowledge base", "init the wiki", "bootstrap tagging"). This skill describes the architecture, routes the agent to the correct sub-skill for each kind of request (tag, ingest, query, lint, rename, reindex, bulk-tag), and owns the one-shot project bootstrap. Read this first when the agent doesn't know which knowledge-system skill to invoke.
---

# wiki (master)

This is the **orchestrator and overview** for a tagged knowledge system that lets a project accumulate knowledge over time: tagged documents, indexed and queryable, with optional LLM-generated summaries. It does NOT do the work itself — it routes to the sub-skill that does.

Conceptual basis: Karpathy's "LLM Wiki" pattern (see `../wiki-ingest/reference/karpathy-wiki.md`).

## The system at a glance

A project using this system has three folders at its root (created on demand, names confirmable on first use):

```
<host-project-root>/
├── tagging/                  # controlled vocabulary + master index
│   ├── TAGS.md
│   ├── ENTITIES.md
│   └── INDEX.md              # one-line-per-doc catalog of EVERY tagged doc
├── sources/                  # raw source documents (immutable from LLM's POV)
│   └── <source-id>.md
└── wiki/                     # LLM-generated content
    ├── log.md                # chronological log of operations
    └── summaries/
        └── <source-id>-summary.md
```

Everything tagged — sources, notes, prompts, configs, skills, summaries — appears as a row in `tagging/INDEX.md`. The index is the master catalog the agent consults first for any retrieval question.

## Sub-skill routing (use this as a decision tree)

| User says / wants | Invoke |
|---|---|
| "tag this", "label this", "add this to the index", or adds a new doc to the project | `tag-document` |
| "summarize this", "ingest this source", "add to wiki", "extract notes from this" | `wiki-ingest` (heavy, opt-in only) |
| "what do I have on X?", "find docs about Y", "compare A vs B from my notes", any retrieval question | `wiki-query` |
| "rename tag X to Y", "merge tags A and B", "deprecate this tag" | `tag-rename` |
| "rebuild the index", "regenerate INDEX.md", "the index is out of sync" | `wiki-reindex` |
| "health-check the wiki", "lint the knowledge base", "find orphan tags / stale entries" | `wiki-lint` |
| "tag all the docs in folder X", "bulk-tag everything untagged", "retrofit this project" | `bulk-tag` |
| "set up the wiki", "init the knowledge system", "bootstrap tagging" | **this skill** — proceed below |

When a user request is ambiguous, ask which they want before invoking. Never silently chain heavy operations (`wiki-ingest`, `bulk-tag`, `wiki-lint`'s autofix mode).

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
- **Heavy operations are opt-in.** `wiki-ingest`, `bulk-tag`, and `wiki-lint`'s autofix never trigger automatically.

## What this skill does NOT do

- Tag, summarize, query, lint, rename, rebuild, or bulk-tag. Those are sub-skills. This skill only **routes** and **bootstraps**.
- Modify existing state beyond the bootstrap. If folders already exist, this skill leaves them alone.

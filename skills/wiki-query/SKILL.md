---
name: wiki-query
description: Search and synthesize answers from a project's tagged knowledge base. Invoke whenever the user asks a question that requires finding, comparing, summarizing, or synthesizing information from documents in this project — phrases like "what do I have on X?", "find docs about Y", "what does our wiki say about Z?", "compare A and B from my notes", "summarize what we know about W". This skill owns retrieval — it explains the system architecture to the agent (tagging/INDEX.md as primary lookup, wiki/summaries/ as compounding knowledge layer), filters candidates by tags/entities/type, reads only what's needed, and synthesizes with citations. Read this skill before answering any retrieval-style question if the project has a tagging/ folder.
---

# wiki-query

This skill is the agent's instruction manual for **using the tagged knowledge system** in a host project. Read it whenever the user asks something that might be answered from the project's existing documents.

The skill assumes the project already has the storage layout created by `tag-document` (and possibly `wiki-ingest`). If those folders don't exist, this skill has nothing to query — say so and suggest the user start tagging documents first.

## The system at a glance (what you're querying)

A host project using this knowledge system has two top-level folders:

```
<host-project-root>/
├── tagging/
│   ├── TAGS.md          # controlled tag vocabulary
│   ├── ENTITIES.md      # controlled entity vocabulary
│   └── INDEX.md         # one-line-per-doc catalog of EVERY tagged doc in the project
└── wiki/                # only present if wiki-ingest has been used
    ├── log.md           # chronological record of operations
    └── summaries/       # one summary doc per ingested source
```

Key invariants:
- **`INDEX.md` is the master catalog.** Every tagged document — raw notes, prompts, configs, skills, AND summaries from `wiki-ingest` — has a one-line entry there with its tags, entities, type, status, and a sentence-long summary.
- **Summaries are first-class.** A summary has `type: summary` and a `source:` / `source_path:` field. Querying surfaces both the source and its summary so you can choose which to read.
- **Vocabulary is closed.** Tags and entities used in `INDEX.md` all exist in `TAGS.md` / `ENTITIES.md`. To map a user's natural-language question onto the index, you must first map it onto the vocabulary.

## When to invoke

Invoke this skill whenever a user question implies retrieval from project docs:
- "what do I have on X?" / "do I have notes on Y?"
- "find / show me / list docs about Z"
- "what does my wiki say about W?"
- "compare what I have on A vs B"
- "summarize everything I have on T"
- "what did I ingest last week?" (use `wiki/log.md` for time-based queries)
- Any factual question where the answer plausibly lives in tagged project docs.

Do NOT invoke when:
- The user is asking a general-knowledge question unrelated to project docs.
- The user explicitly wants you to ignore the knowledge base.
- The project has no `tagging/` folder yet — say so, suggest tagging first via `tag-document`.

## Standing procedure

### 1. Load the catalog

Read `tagging/INDEX.md` first. It's a small file by design — the whole point is to scan it cheaply before opening any document.

If the user's question involves recency ("what did I do this week?", "recent ingests"), also read `wiki/log.md` (if present). Use `grep "^## \[" wiki/log.md | tail -N` if the log is long.

### 2. Parse the question into search criteria

Map the user's natural-language question onto the vocabulary:
- **Topics** → which tags from `TAGS.md` are semantically closest? Use semantic similarity, not just string match. "RAG" → existing tag `rag`. "agent design" might match `agents` or `agent-frameworks` depending on what exists.
- **Named things** → which entities from `ENTITIES.md` are mentioned? "Claude" → likely an entity like `Claude Code` or `Claude Opus 4.7`.
- **Time** → date filters; cross-reference `log.md` for ingest dates.
- **Type filter** — does the user want only summaries, only configs, only prompts? Default: all types.
- **Status filter** — usually `active`; include `draft`/`archived` only if explicitly asked.

If the question doesn't map onto any existing vocabulary, say so plainly — the user may need to ingest more sources, or the vocabulary may need a new tag (handled by `tag-document`'s approval flow, not by this skill).

### 3. Filter candidates

Scan `INDEX.md` and pull out entries that match the criteria. A candidate is a row whose tags/entities/type/status satisfy the question's filters.

Rank candidates with this preference order:
1. **Summaries first** for broad / "what do I know about X" questions — they're pre-synthesized.
2. **Sources first** for specific factual questions where details matter ("what was the exact figure?", "what's the API endpoint?").
3. **Most recent first** when recency matters (use `date` field).
4. **Most-tagged-matches first** when several criteria apply (a doc matching 3 of your 3 filter tags beats one matching 1 of 3).

Aim for 1-5 candidates. If you have >10 candidates, the filter is too loose — narrow it before reading.

### 4. Read the candidates

Read only the candidate docs in full. Do NOT scan the whole project folder — that's what the index exists to avoid.

If a candidate is a summary and you need more detail than the summary covers, also read the source it points to (`source_path` in the summary's frontmatter).

### 5. Synthesize the answer

- Default output: a markdown response with inline citations like `[acme-q3-summary](wiki/summaries/acme-q3-summary.md)` for each claim.
- For comparison questions: a markdown table.
- For timeline questions: a date-ordered list.
- For "what do I have on X" without a deep question attached: just list the matching docs with their one-line summaries from `INDEX.md` — no need to open them all if the user just wants to know what exists.

Cite every non-trivial claim. The citation IS the value — it lets the user verify against the source and find more.

If candidates disagree (one summary says X, another says Y), surface the contradiction explicitly rather than picking one silently. Disagreements are signal.

### 6. Optional: log significant queries

If the query is non-trivial (you read 3+ docs, the answer is novel, the user expressed it as a research question), append to `wiki/log.md` using the format from `wiki-ingest`:

```markdown
## [2026-05-13] query | <short question>
Read: [doc-a, doc-b, doc-c]. Answer: <one-line gist>.
```

Skip the log for casual lookups. Don't pollute the log with every "what's in this index?" question.

### 7. Optional: offer to file the answer back into the wiki

If the synthesis is substantive (comparison, novel connection, deep analysis) and likely to be useful again, offer: *"Want me to file this answer into the wiki as a new doc? It'll be tagged and queryable like anything else."*

Filing back is NOT automatic — only on user request. Use the `wiki-ingest` skill's procedure if accepted, with `type: summary` and `source: query-<date>-<slug>` (since the "source" is the conversation, not an external doc). Per Karpathy's pattern, this is how the wiki compounds — your explorations accumulate.

## Output format conventions

- **Citations**: relative markdown links to the actual doc paths, e.g. `[title](wiki/summaries/foo.md)`.
- **Quotes**: use blockquotes for verbatim source text; otherwise paraphrase + cite.
- **Confidence**: if the index has only one match on a question that probably needs more sources, say so. Don't over-confidently synthesize from a single doc.

## Edge cases

- **Empty index.** Tell the user there's nothing tagged yet; suggest `tag-document`.
- **No matches.** Don't fabricate. Say no matches, suggest related tags/entities the user might have meant, or suggest ingesting new sources.
- **Stale index.** If you notice `INDEX.md` references files that no longer exist, flag it — the index is out of sync and needs a maintenance pass (future `wiki-lint` skill).
- **Vocabulary gap.** If the user's question implies a concept that has no tag, don't silently force-fit. Say "I don't see a tag covering this — closest existing tags are X, Y" and let the user decide whether to broaden the search or add a tag via `tag-document`.

## What this skill does NOT do

- **Tag new documents.** That's `tag-document`.
- **Generate summaries of new sources.** That's `wiki-ingest`.
- **Web search or external lookups.** This skill operates purely on the local project's tagged content.
- **Modify the vocabulary.** New tags go through `tag-document`'s approval flow.
- **Health-check / lint the index.** Future `wiki-lint` skill.

## Cross-skill summary

| Skill | Role | Triggered by |
|---|---|---|
| `tag-document` | Tag/index a single doc into the vocabulary system. Lightweight. | "tag this", "label this", "add to index" — or implicit when adding new docs |
| `wiki-ingest` | Summarize a source and file it into `wiki/`. Heavy, opt-in. | "summarize this", "ingest", "add to wiki" — explicit only |
| `wiki-query` | Search + synthesize an answer from tagged docs. | Any retrieval-style question against project docs |

---
description: "Query the Claw KB wiki and get a cited answer"
disable-model-invocation: true
---

# /kb-query — Ask the Knowledge Base

## User Prompt: `$ARGUMENTS`

## Step 1 — Run the search

```bash
python3 ~/skill-backends/knowledgebase/kb-query.py "$ARGUMENTS" --snippets --limit 8
```

## Step 2 — Read top hits

Read the top 3–5 hits in full with the Read tool. Ignore hits with very low scores or snippets that don't match the intent of the question.

## Step 3 — Synthesize

Answer the user's question in 1–3 paragraphs, citing each fact with `[[wikilink]]` to the source wiki page. Do NOT fabricate — if the KB doesn't cover it, say so.

Format:

```
**Answer:**
<synthesis with [[citations]]>

**Sources:**
- [[wiki/path/to/page-a]]
- [[wiki/path/to/page-b]]

**Gaps:**
<what the KB doesn't cover that would strengthen the answer, if any>
```

## Step 4 — Optionally save the answer

If the answer is reusable (well-cited, captures a non-trivial synthesis), offer to save it to `output/queries/`:

```bash
# Example — write via the Write tool
# path: ~/.openclaw/vaults/Claw/output/queries/<slug>.md
# include frontmatter: type: query, created, sources: [...], tags: [...]
```

Skip for trivial lookups.

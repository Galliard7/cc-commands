---
description: "Health-check the Claw KB and fix or flag issues"
disable-model-invocation: true
---

# /kb-lint — Knowledge Base Health Check

## Step 1 — Run the linter

```bash
python3 ~/skill-backends/knowledgebase/kb-lint.py
```

## Step 2 — Triage the findings

Group the report into:

1. **Uncovered raw** — sources never compiled. These are the highest-value targets: feed them through `/kb-compile --source <path>`.
2. **Orphan pages** — wiki pages with no source citations. Either find the source they came from (check `log.md`, `source-map.json`), add inline citations, and update the `sources:` frontmatter; or delete the page if it's not load-bearing.
3. **Drift** — files on disk missing from their section `_index.md`. Run `python3 ~/skill-backends/knowledgebase/kb-index.py --section <section>` to rebuild, then add a one-line description for each new entry.
4. **Stale** — pages not updated in >90 days. Skim, refresh if still relevant, or mark with `status: stale` in frontmatter.

## Step 3 — Act on top issues

Fix the top 3–5 issues directly (not a blanket sweep). For each fix, follow the ingest protocol in `~/skill-backends/knowledgebase/SKILL.md` if you're creating/updating wiki pages.

## Step 4 — Report

Print a short summary of what was fixed and what remains, with counts per category.

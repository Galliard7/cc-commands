---
description: "Compile raw KB sources into Karpathy-style wiki pages"
disable-model-invocation: true
---

# /kb-compile — Ingest Raw Sources into the KB Wiki

Compile raw sources under `~/.openclaw/vaults/Claw/raw/` into cross-linked wiki pages under `wiki/`. You own the wiki — the scripts are bookkeepers. Follow the ingest protocol in `~/skill-backends/knowledgebase/SKILL.md` exactly.

## User Prompt: `$ARGUMENTS`

`$ARGUMENTS` can be:
- Empty → compile the most recently captured pending source (or ask if multiple)
- `--recent` → compile all pending sources
- `--all-new` → compile every raw file not yet in `source-map.json`
- `--source <path>` → compile a specific raw source

## Step 1 — Resolve scope

Run `python3 ~/skill-backends/knowledgebase/kb-compile.py --list-pending` to see pending sources. For `--all-new`, also run `--list-uncompiled`.

If multiple sources match and the user didn't specify, ask which to compile first.

## Step 2 — Read schema and protocol

Read `~/.openclaw/vaults/Claw/schema.md` if not already loaded this session, and the **Ingest Protocol** section of `~/skill-backends/knowledgebase/SKILL.md`.

## Step 3 — Drive mode

For each source to compile:

```bash
python3 ~/skill-backends/knowledgebase/kb-compile.py --source <rel-path>
```

This prints the source path and the compile protocol reminder. It does not modify the wiki.

## Step 4 — Do the work (LLM turn)

Follow the Ingest Protocol (Steps 1–6 in SKILL.md):

1. **Read** the raw source in full with the Read tool.
2. **Classify** — what is it, what topics/entities/concepts, which project(s)?
3. **Search first** to avoid duplicates:
   ```bash
   python3 ~/skill-backends/knowledgebase/kb-query.py "<candidate page name>"
   ```
4. **Write/update wiki pages** under the right section using the Write/Edit tools. Every page gets frontmatter (type, title, created, updated, sources, tags). Kebab-case filenames. Inline citations.
5. **Cross-link** — wikilink entities, concepts, projects, related pages.
6. **Update section `_index.md`** with a one-line description if a new page was added (only the human-readable part; `kb-index.py` rebuilds the auto block).

## Step 5 — Record

```bash
python3 ~/skill-backends/knowledgebase/kb-compile.py \
  --source <rel-path> \
  --record <page1> <page2> ...
```

This:
- writes the source→pages mapping into `source-map.json`
- removes the source from `pending.json`
- rebuilds `_index.md` for each touched section (via `kb-index.py`)
- appends a log entry to `log.md`

## Step 6 — Report

For each compiled source, print:

```
Compiled: <source>
  Pages created: N (<list>)
  Pages updated: M (<list>)
  Cross-links added: K
  Sections touched: <list>
```

End with a one-line summary of total compiles and suggest `/kb-lint` if several new pages were created.

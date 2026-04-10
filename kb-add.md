---
description: "Capture a URL or file into the Claw KB inbox (raw/)"
disable-model-invocation: true
---

# /kb-add — Capture Raw Source to KB

Capture the URL or file given in `$ARGUMENTS` into `~/.openclaw/vaults/Claw/raw/` via the `knowledgebase` skill.

## User Prompt: `$ARGUMENTS`

## Step 1 — Detect input type

- Starts with `http://` or `https://` → URL
- Starts with `/` or `~/` or looks like a file path → file
- Otherwise → ask the user to clarify

## Step 2 — Run `kb-add.py`

### For URLs
```bash
python3 ~/skill-backends/knowledgebase/kb-add.py --url "<URL>"
```

### For files
```bash
python3 ~/skill-backends/knowledgebase/kb-add.py --file "<PATH>"
```

(Optional `--category` flag: `articles | papers | clips | transcripts | sessions`. If unsure, omit it — the script infers from extension.)

## Step 3 — Report

Print the captured path and remind the user that the source is in `pending.json` until compiled. Offer to run `/kb-compile --source <path>` next.

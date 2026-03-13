---
description: "Scan workspace for stray files and propose filing them into the correct project/reference directories"
disable-model-invocation: true
---

# /organize — Workspace Cleanup

You are organizing the workspace by finding stray files and filing them into the correct locations per the workspace conventions in CLAUDE.md.

All paths below are relative to `~/.openclaw/` unless otherwise noted.

## Step 1: Scan for Stray Files

Scan these three locations:

### 1a. OpenClaw root (`~/.openclaw/`)
Look for `*.md` files at the root level. **Exclude** these (they belong there):
- `CLAUDE.md`
- Any non-`.md` files
- Any dotfiles or config files (`openclaw.json`, etc.)

### 1b. Workspace root (`~/.openclaw/workspace/`)
Look for files at the root level that are NOT in this allowlist:

**Allowlisted files** (canonical agent docs — leave in place):
- `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`
- `MEMORY.md`, `TOOLS.md`, `HEARTBEAT.md`
- `.gitignore`

**Allowlisted directories** (expected structure — don't scan inside):
- `projects/`, `skills/`, `memory/`, `reference/`
- `noteflow/`, `cc-remote/`, `mission-control/`

Anything at workspace root not in the allowlist is a stray.

### 1c. Obsidian vault (`~/.openclaw/vaults/Claw/`)
First, list all subdirectories in the vault root to discover the available folders. Then look for any files sitting at the vault root level (not inside a subfolder). Any file at the vault root is a stray — it should be filed into whichever existing subfolder best fits its content. If no subfolder is a good fit, propose creating a new one or ask the user.

### Never touch
- `.claude/`, `.git/`, `agents/`, `logs/`, `credentials/`, `telegram/`
- `openclaw.json` or any config/dotfiles
- Data stores (`noteflow/store.json`, `cc-remote/state.json`)
- Infrastructure directories
- `.obsidian/` folder inside the vault (Obsidian config)

## Step 2: Read and Classify Each Stray

For each stray file found, read its contents and classify it:

| Pattern | Destination |
|---|---|
| Design brief / brainstorm for a project | `workspace/projects/{name}/DESIGN-PROMPT.md` |
| Spec document for a project | `workspace/projects/{name}/SPEC.md` |
| Reference material / external docs | `workspace/reference/{filename}` |
| Scratch notes / session artifacts | `workspace/memory/{filename}` or propose deletion if stale |
| Config / dotfiles at openclaw root | Leave in place (infrastructure) |
| Vault stray file | `vaults/Claw/{best-matching-subfolder}/{filename}` — pick the subfolder whose purpose best matches the file's content. If none fits, propose a new subfolder or ask the user. |
| Unknown / ambiguous | Ask the user |

When classifying to a project, try to match an existing project under `workspace/projects/`. If no match exists, propose creating a new project directory.

## Step 3: Propose Move Plan

Display a table:

```
| # | File | Destination | Reason |
|---|------|-------------|--------|
| 1 | ClawxCCode.md | workspace/projects/cc-remote/DESIGN-PROMPT.md | CC-Remote design brief |
```

If no stray files are found, report: **"Workspace is clean — nothing to file."** and stop.

## Step 4: Confirm

Ask the user to approve the plan. They can:
- **Approve all** — proceed with all moves
- **Skip specific items** — by number
- **Change destination** — for any item
- **Cancel** — abort entirely

Do NOT proceed without explicit approval.

## Step 5: Execute

For each approved move:
1. Create the destination directory if it doesn't exist (`mkdir -p`)
2. Move the file (preserving content exactly)
3. Delete the original

## Step 6: Report

Print a summary:

```
Organized:
- {source} → {destination}

Skipped:
- {file} — {reason}
```

If nothing was moved: `No changes made.`

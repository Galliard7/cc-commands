---
description: "Checkpoint: capture all work since last checkpoint across all sessions"
disable-model-invocation: true
---

# /checkpoint — Cross-Session Checkpoint

You are performing a checkpoint. This captures **all work since the last checkpoint**, not just the current session. Work may have happened across multiple Claude Code sessions, Telegram, Discord, or direct file edits.

All script paths use the skill backend at `~/skill-backends/noteflow/`.

## Step 1: Gather All Changes Since Last Checkpoint

### 1a. Find the last checkpoint

```bash
cd ~/.openclaw/workspace && git log --oneline --grep="checkpoint:" -1
```

Note the commit hash. If no checkpoint commit exists, use the initial commit.

### 1b. Diff the workspace repo

```bash
cd ~/.openclaw/workspace
git diff <last-checkpoint-hash>..HEAD --stat
git diff <last-checkpoint-hash>..HEAD
git log <last-checkpoint-hash>..HEAD --oneline
```

Also check for uncommitted changes:
```bash
git status
git diff
```

### 1c. Diff all code repos

Scan every code repo — skill backends AND standalone repos:
```bash
for repo in ~/skill-backends/*/ ~/repos/*/; do
  [ -d "$repo/.git" ] || continue
  echo "=== $(basename $repo) ==="
  cd "$repo"
  git log --since="$(git -C ~/.openclaw/workspace log --grep='checkpoint:' -1 --format='%ai' 2>/dev/null || echo '1 week ago')" --oneline 2>/dev/null
  git diff --stat 2>/dev/null
  cd -
done
```

### 1d. Synthesize

From all the diffs and logs, identify:
- Which projects were worked on (across all sessions)
- What was accomplished (features built, bugs fixed, specs written, decisions made)
- New blockers or open questions
- Key technical decisions or implementation notes
- Any new projects that were started

Use the **current conversation history** as supplementary context — it adds color and reasoning that git diffs alone can't capture, but it is NOT the primary source.

If **nothing changed** since the last checkpoint (no diffs, no commits, no uncommitted changes), say so and skip to Step 5.

## Step 2: Board Sync (via /msync)

**Delegate board reconciliation to the `/msync` procedure.** Read `.claude/commands/msync.md` and execute its Steps 1 through 10. This handles:

- Board state reading with dynamic status detection
- Git scan of workspace + all skill backends (msync's own Step 2)
- NoteFlow → MC task sync (auto-creates cards for unlinked tasks)
- Card reconciliation using git evidence, conversation context, and codebase checks
- Project phase sync
- Cross-session sync and new work detection
- Project assignment for unassigned cards
- Orphan detection (plan files without cards, cards with missing plan files)
- Stale activity cleanup
- CC auto-memory Active Project Summary update

**Checkpoint provides extended range:** msync's own git scan uses a 7-day rolling window. Checkpoint's Step 1 provides the precise range since the last checkpoint. Both are available — use whichever gives the most complete picture. If the last checkpoint was more than 7 days ago, Step 1's diffs are the authoritative source for older changes.

**Skip msync's Step 11 (Report)** — checkpoint's own report (Step 13) incorporates msync's findings.

## Step 3: Log and Decisions

Based on the full diff from Step 1 and msync's card reconciliation:
- Add new completion entries to the `log` array in board.json for significant milestones
- Add new decisions to the `decisions` array

Skip if nothing noteworthy.

## Step 4: Clear Activity

Clear ALL CC session activity (supersedes msync's stale-only cleanup):
```bash
python3 ~/skill-backends/noteflow/mc-activity.py --clear-all
```

## Step 5: Supplement CC Auto-Memory

msync already updated the **Active Project Summary**. Now review your auto-memory MEMORY.md for broader updates:

- If a major convention or pattern was established, add it to the appropriate section
- If a new key path was created, add it to **Key Paths**
- If technical findings, implementation patterns, or lessons learned are worth remembering across sessions, capture them

### Rules:
- Keep the file concise — it's loaded into every conversation context
- Do NOT duplicate information that's already in board.json
- Focus on what Claude Code needs to know to resume effectively
- Do NOT re-update the Active Project Summary — msync handled that

## Step 6: Write Daily Note

**File:** `workspace/memory/YYYY-MM-DD.md` (using today's date)

This note should capture **all work since the last checkpoint**, not just the current session. Use the diffs and logs from Step 1 as the primary source.

If the file doesn't exist, create it with this format:

```markdown
# YYYY-MM-DD

## Checkpoint: HH:MM CT

### What Happened
- Summary of all work since last checkpoint, across all sessions/channels
- Group by project or theme, not by individual session

### Decisions
- [Decision]: [reasoning behind it]

### Technical Findings
- Discoveries, API behaviors, tool quirks worth remembering

### Mistakes & Lessons
- What went wrong, what to do differently next time

### Open Threads
- Things to pick up next
```

Sections can be omitted if empty.

If the file already exists (previous checkpoint today), **append** a new `## Checkpoint` block.

To get the current time, run: `TZ=America/Chicago date '+%H:%M'`

## Step 7: Distill to Long-Term Memory

**File:** `workspace/MEMORY.md`

Read `workspace/MEMORY.md` and the daily notes from the past 7 days (`workspace/memory/`).

Identify items worth promoting to long-term memory:
- **Patterns** that recurred across sessions or are broadly applicable
- **Technical findings** that will matter beyond this week
- **Mistakes** that should be permanently remembered to avoid repetition
- **User preferences** or working patterns newly discovered
- **Decisions** with lasting implications (not project-specific tactical choices)

Update MEMORY.md:
- Add new entries to the appropriate section
- Update entries that have evolved (e.g., a technical note that's now more nuanced)
- Remove entries that are outdated, superseded, or no longer relevant
- Keep total length under ~100 lines

Do NOT add:
- Session-specific details (daily notes handle that)
- Project status updates (board.json handles that)
- Information already captured in MEMORY.md
- Tactical/temporary decisions that won't matter in 2 weeks

If nothing is worth promoting, skip this step and note "No new long-term memories" in the checkpoint report.

## Step 8: Write Obsidian Vault Note

### 8a. Migrate flat daily files

Check `~/.openclaw/vaults/Claw/Daily/` for any flat `YYYY-MM-DD.md` files. For each one found, move it into a date folder:

```bash
cd ~/.openclaw/vaults/Claw/Daily
for f in ????-??-??.md; do
  [ -f "$f" ] || continue
  dir="${f%.md}"
  mkdir -p "$dir"
  mv "$f" "$dir/summary.md"
  echo "Migrated $f → $dir/summary.md"
done
```

This is a no-op after the first run (no flat files left to move).

### 8b. Read session files

Check if `~/.openclaw/workspace/sessions/` exists and contains `.md` files. If yes, read all session files — these provide per-task context that should inform the daily summary narrative.

```bash
ls ~/.openclaw/workspace/sessions/*.md 2>/dev/null
```

If session files exist, read each one. Use their content as **primary input** alongside the git diffs from Step 1 when writing the summary below.

### 8c. Write summary

**File:** `~/.openclaw/vaults/Claw/Daily/YYYY-MM-DD/summary.md` (using today's date)

Ensure the date folder exists:
```bash
mkdir -p ~/.openclaw/vaults/Claw/Daily/YYYY-MM-DD
```

Write a **human-readable journal** covering all work since the last checkpoint. This is a narrative briefing, not a mechanical status dump. Use session files (if any) to add detail and specificity.

If the file doesn't exist, create it. If it already exists (previous checkpoint today), **append** a new block.

### Format:

```markdown
# YYYY-MM-DD

## Checkpoint: HH:MM CT

### What We Worked On
Narrative summary of all work since the last checkpoint — what was built, discussed, and solved across all sessions. Write in first-person plural ("we built...", "we decided..."). Include enough context that reading this months later still makes sense.

### Project Status
Brief status of each active project. Where things stand right now.

### Ideas & Threads
Open ideas, future directions discussed, things to explore later. Capture the creative/strategic thinking, not just the task list.

### Decisions
Key decisions made and the reasoning behind them.

### Next Up
What's queued next.
```

### Rules:
- Write for the **future reader** — assume no recent context
- Include the "why" behind decisions, not just the "what"
- Capture ideas and threads that might not make it into board.json
- Keep it concise but complete — aim for a 2-minute read
- Use natural language, not bullet-point soup
- **Incorporate session file details** — they capture real-time work entries that git diffs miss

## Step 9: Sweep Session Files

Move session files into today's Obsidian date folder so they're archived alongside the summary.

1. Check if `~/.openclaw/workspace/sessions/` exists and has `.md` files
2. If yes:
   ```bash
   mkdir -p ~/.openclaw/vaults/Claw/Daily/YYYY-MM-DD
   mv ~/.openclaw/workspace/sessions/*.md ~/.openclaw/vaults/Claw/Daily/YYYY-MM-DD/
   rmdir ~/.openclaw/workspace/sessions/
   ```
3. If no session files exist, skip silently

The swept files live alongside `summary.md` in the date folder, providing per-task detail that complements the narrative summary.

## Step 10: Update Reference Docs

Check whether anything since the last checkpoint changed the OpenClaw setup itself (not just project work). If it did, update the relevant reference doc.

### `reference/setup-manifest.md`
Update if any of these changed:
- Channels added, removed, or reconfigured (Telegram, Discord, etc.)
- Model or provider changes
- Agent defaults (sandbox, concurrency, compaction)
- Gateway settings (port, bind, auth, denied commands)
- Plugins enabled/disabled
- Skills added or removed
- Code backends added
- Secrets locations changed

### `reference/security-hardening.md`
Update the **Changelog** section if any security-relevant change was made:
- Sandbox mode changes
- denyCommands changes
- Auth or pairing policy changes
- Log redaction changes
- New channels with security implications

### Rules:
- Only update if something actually changed — don't touch these for normal project work
- Use targeted edits, don't rewrite
- Add changelog entries with today's date

## Step 11: Trim Memory Files

Check these files and trim if they exceed the stated limits:

| File | Max Lines | Trim Strategy |
|---|---|---|
| CC auto-memory MEMORY.md | ~50 lines | Merge related bullets, remove outdated info |
| `workspace/MEMORY.md` | ~100 lines | Archive old entries to daily notes |

Only trim if actually over the limit. Use your judgment about what to cut.

## Step 12: Commit and Push All Repos

Commit and push **every repo with changes** — workspace, skill backends, and standalone repos.

### 12a. Workspace repo
```bash
cd ~/.openclaw/workspace
git add -A
git status --short
```
If there are staged changes:
```bash
git commit -m "checkpoint: YYYY-MM-DD — [1-line summary of key changes]"
git push
```

### 12b. All code repos
For each repo, check for uncommitted changes and push:
```bash
for repo in ~/skill-backends/*/ ~/repos/*/; do
  [ -d "$repo/.git" ] || continue
  cd "$repo"
  if [ -n "$(git status --porcelain)" ]; then
    echo "=== Committing $(basename $repo) ==="
    git add -A
    git commit -m "checkpoint: YYYY-MM-DD — [1-line summary of changes in this repo]"
  fi
  # Push if there are local commits not on remote
  if [ -n "$(git log @{u}..HEAD --oneline 2>/dev/null)" ]; then
    echo "=== Pushing $(basename $repo) ==="
    git push
  fi
  cd -
done
```

Skip repos with no changes and no unpushed commits.

## Step 13: Report Summary

Print a confirmation in this format:

```
Checkpoint complete. (Changes since: <last checkpoint date or "initial">)

Sources:
- Workspace: X commits + Y uncommitted changes
- Backends: [list repos with changes]
- Current session: [brief note]

Board sync (via msync):
- [card-slug]: [old-status] → [new-status] (reason)
- [card-slug]: commented (summary)
- [N] cards unchanged
- Phases: [changes or "no changes"]
- New cards: [list or "none"]
- Orphans: [list or "none"]

Updated:
- board.json — [log/decisions additions, if any]
- Activity — cleared
- CC memory — [what changed beyond Active Project Summary]
- Daily note — [created/appended] workspace/memory/YYYY-MM-DD.md
- Long-term memory — [what was added/updated/removed, or "no changes"]
- Vault note — [created/appended] vaults/Claw/Daily/YYYY-MM-DD/summary.md
- Sessions — swept N file(s) to vaults/Claw/Daily/YYYY-MM-DD/ [or "no session files"]
- Flat file migration — migrated N file(s) [or "none needed"]
- Reference docs — [which ones, if any]
- Git — workspace: [committed and pushed / no changes]; backends: [list repos pushed, or "no changes"]

No changes:
- [list any files that didn't need updating and why]
```

If nothing changed since last checkpoint: `Checkpoint complete. No changes detected since last checkpoint — nothing to update.`

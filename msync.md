---
description: "Mission Control board sync — reconcile card states using git evidence, conversation context, and codebase checks"
disable-model-invocation: true
---

# /msync — Mission Control Board Sync

You are performing a board sync. This reconciles `board.json` with reality using **git history, conversation context, and codebase checks**. Work happens across multiple sessions, channels, and tools — msync must see all of it, not just the current conversation.

All script paths use the skill backend at `~/skill-backends/noteflow/`.

## Step 1: Read Board State

Read `workspace/mission-control/board.json` and note:
- The board's `statuses` array — these are the user-defined column names (they can be renamed at any time, so never assume specific names)
- Total cards grouped by their `status` field
- Identify which status represents "completed/finished" work vs "not started" vs "in progress" — infer from the label semantics, not from hardcoded names
- Cards in any non-terminal status are candidates for reconciliation

## Step 2: Scan Recent Changes

Scan git repos to capture cross-session work. This is critical — most work happens outside the current conversation.

### 2a. Workspace repo
```bash
cd ~/.openclaw/workspace
git log --oneline --since="7 days ago"
git diff --stat
git status --short
```

### 2b. All code repos (skill backends + standalone repos)
```bash
for repo in ~/skill-backends/*/ ~/repos/*/; do
  [ -d "$repo/.git" ] || continue
  echo "=== $(basename $repo) ==="
  cd "$repo"
  git log --oneline --since="7 days ago" 2>/dev/null
  git diff --stat 2>/dev/null
  cd -
done
```

### 2c. Synthesize

From all the diffs and logs, identify:
- Which projects had codebase changes (map repos to projects: noteflow → noteflow, cc-remote → cc-remote, content-extractor → contentflow, dataflow-toolkit → dataflow, cc-commands → claw, workspace changes → claw/missioncontrol)
- What was built, modified, or fixed — read commit messages and diff stats
- Any uncommitted work in progress

This evidence feeds into card reconciliation (Step 4) and cross-session sync (Step 6). **Git evidence is equally authoritative to conversation context** — git doesn't lie about what changed.

**When called from /checkpoint:** Checkpoint's Step 1 already gathered more precise diffs (since last checkpoint hash). Use both — checkpoint's range is exact, msync's 7-day rolling window is the fallback.

## Step 3: NoteFlow → MC Task Sync

Run the NoteFlow-to-MC sync script to ensure every open NoteFlow task has a corresponding MC card:

```bash
python3 ~/skill-backends/noteflow/nf-mc-sync.py
```

This script:
- Finds NoteFlow items of type `task` that are `open` and have no `linked_cards`
- Creates an MC card in the board's initial status for each (title + body carried over, no plan file, no project)
- Links the NoteFlow item back to the new card via `linked_cards`
- Syncs done status bidirectionally: NoteFlow done → MC card done, MC card done → NoteFlow done

After running, note any newly created cards — they'll appear in the report (Step 11) and are candidates for project assignment and reconciliation in subsequent steps.

## Step 4: Reconcile Card Statuses

**Determine terminal vs non-terminal statuses** by reading the board's `statuses` array. The status whose label most clearly means "finished" or "completed" is the terminal status. All others are non-terminal (to be reconciled).

### For each card in a status that implies active/in-progress work:
1. **Identify success criteria** — read the plan file (`plan_file` field). If `plan_file` is null, use the card's `title` and `description` as the criteria instead. Every card has enough context to verify — never skip a card just because it lacks a plan file.
2. Check the actual codebase for evidence of completion (read relevant files, check if deliverables exist)
3. **Check git evidence from Step 2** — do the diffs, commits, or uncommitted changes show work related to this card?
4. **Check completed cards for overlap** — scan cards in terminal status for plan files and comments to see if this card's scope was already delivered as part of another card's work.
5. Scan the full conversation for signals — user statements about this card's topic are **authoritative**, including mentions of work happening in other sessions/channels. "We're working on X" or "I've been doing Y" = confirmed progress.
6. Classify and act:

| Verdict | Action |
|---|---|
| **All criteria met** | Move card to the terminal (completed) status. Archive plan file to `~/.openclaw/vaults/Claw/OpenClaw/`. Add comment via `mc-comment.py`. |
| **Partially done** | Leave in current status. Add comment noting which criteria passed/didn't (only if meaningful progress since last comment). |
| **Progress made (no status change)** | Leave in current status. Add comment capturing what progress was made — e.g., discussion, decisions, partial implementation, blockers identified. |
| **No change** | Leave as-is, no comment needed. |

### For each card in a status that implies blocked:
1. **Identify the blocker** — read the card's most recent comment or description for what's blocking progress
2. **Check if the blocker is resolved** — scan git evidence from Step 2, codebase state, and conversation context for signals that the blocking condition has changed
3. **Check completed cards for overlap** — the blocking dependency may have been delivered as part of another card's work
4. Classify and act:

| Verdict | Action |
|---|---|
| **Blocker resolved, work complete** | Move card to the terminal (completed) status. Archive plan file. Add comment. |
| **Blocker resolved, work remains** | Move card to the in-progress status. Add comment noting what unblocked it. |
| **Still blocked** | Leave as-is. Add comment only if new information about the blocker exists. |

### For each card in a status that implies not-yet-started or queued:
1. **Identify criteria** — read the plan file. If `plan_file` is null, use the card's `title` and `description` as criteria. Never skip a card because it lacks a plan file.
2. Check the codebase for evidence that work has begun or is complete
3. **Check git evidence from Step 2** — do the diffs or commits show work related to this card?
4. **Check completed cards for overlap** — scan cards in terminal status to see if this card's scope was already delivered as part of another card's work. Cards created from the dashboard UI often describe work that gets implemented under a different, more detailed card.
5. Scan the full conversation for signals — user mentions of this card's topic (including cross-session work) are authoritative evidence of progress
6. Classify and act:

| Verdict | Action |
|---|---|
| **Fully implemented** | Move card to the terminal (completed) status. Archive plan file. Add comment. |
| **Work started** | Move card to the in-progress status. Add comment noting what evidence triggered the move. |
| **Progress made (no codebase changes)** | Leave in current status. Add comment capturing relevant progress — e.g., planning discussion, design decisions, requirements clarified, blockers identified. |
| **Not started** | Leave as-is — no action. |

### Comment format
```bash
python3 ~/skill-backends/noteflow/mc-comment.py --plan "<card-slug>" --comment "<1-2 sentence update>"
```

### Rules
- **Four evidence sources, in priority order:**
  1. **User statements** — what the user tells you (including about other sessions) is the most authoritative signal. "We're working on X" = progress, period.
  2. **Git evidence** — commits, diffs, and uncommitted changes from Step 2. This is the primary cross-session signal. A commit touching `~/skill-backends/noteflow/` IS evidence of NoteFlow work, even if nobody mentioned it.
  3. **Codebase evidence** — reading actual files to verify deliverables exist and work as expected
  4. **Plan file criteria** — success criteria from the plan
- "Work started" for status changes means actual codebase changes. But **progress comments don't require codebase changes** — user-reported cross-session work, design decisions, requirements refined, blockers identified, approaches discussed all qualify.
- Don't add redundant comments — skip if the last comment already captures the current state
- Compare against the card's most recent comment to determine if new information exists worth recording
- **Scan the full conversation including the triggering message** — the user's `/msync` invocation often comes with context ("we're working on X", "just finished Y") that is itself evidence of progress
- **Never assume specific status names** — always read the board's `statuses` array and match by semantic meaning (completed, in-progress, not-started, blocked, etc.)

### Obsidian enrichment for archived plan files

When archiving a plan file to `~/.openclaw/vaults/Claw/OpenClaw/`, enrich it before (or after) copying:

1. **Prepend YAML frontmatter** (if not already present):
   ```yaml
   ---
   type: plan
   project: <project-id from card>
   card: <card-slug>
   date: <today's date, YYYY-MM-DD>
   status: done
   ---
   ```

2. **Add a "Related" section** at the bottom of the file linking to other plans that are dependencies, prerequisites, or were superseded — but only if the relationship is clear from board.json (same project, comments referencing the other card, etc.). Use `[[wikilinks]]`:
   ```markdown
   ## Related
   - Supersedes [[older-plan-name]]
   - Depends on [[prerequisite-plan]]
   - See also [[related-plan]]
   ```
   Skip this section if no clear relationships exist — don't force links.

3. **Add wikilinks inline** to any project hub references in the plan body (e.g., wrap "NoteFlow" with `[[NoteFlow]]`) — only for 1-3 key project references, don't rewrite the whole file.

## Step 5: Sync Project Phases

When a card moves to the terminal (completed) status, check if it maps to a project phase. If so, update the phase:
```bash
python3 ~/skill-backends/noteflow/mc-phase.py --project "<project-id>" --phase "<phase-name>" --status done
```

## Step 5b: Re-sync NoteFlow ↔ MC Done Status

Step 3 ran nf-mc-sync.py **before** card reconciliation, so any cards moved to done in Step 4 haven't propagated to their linked NoteFlow items yet. Re-run the sync to catch them:

```bash
python3 ~/skill-backends/noteflow/nf-mc-sync.py
```

This ensures the calendar (which reads NoteFlow done timestamps) reflects all completions from this msync run, and that NoteFlow items linked to newly-done MC cards are marked done.

Skip this step if no cards were moved to a terminal status in Step 4.

## Step 6: Cross-Session Sync & New Work

### 6a. Cross-session updates

Scan **both the conversation AND git evidence from Step 2** for work that should be recorded on existing cards:

**From conversation:** User mentions of work happening in other sessions, channels, or outside this conversation.

**From git:** Commits and changes in skill backends or workspace that relate to existing cards but weren't captured yet. For example, if `~/skill-backends/cc-remote/` has 500+ lines of changes and mc-004 (handoff mode) has no recent comment about it, that's a gap — add a comment.

For each match: find the relevant existing card and add a progress comment via `mc-comment.py`.

### 6b. Detect new work

Review the current conversation and git evidence for scoped, actionable work that should have a card but doesn't:
- Create a new card and plan file via `mc_lib` or direct board.json edit
- Plan files go in `workspace/mission-control/plans/`
- Use kebab-case for plan file names

Only create cards for scoped, actionable work — not exploratory discussion or one-off fixes.

## Step 7: Assign Projects to Unassigned Cards

After all card-creating steps are complete (NoteFlow sync, new work detection), assign projects to any cards still missing one.

For each card where `project` is `null`:

1. Read the card's `title` and `description`
2. Read the board's `projects` array — note each project's `id`, `name`, and `summary`
3. Use existing project assignments as reference patterns — cards already assigned to projects demonstrate what kind of work belongs where:
   - **claw** — OpenClaw infrastructure, config, optimization, cross-cutting tooling
   - **missioncontrol** — Dashboard UI, board features, MC improvements
   - **noteflow** — NoteFlow features, task/note management, reminders
   - **contentflow** — Content ingestion, extraction, creation, video/media
   - **cc-remote** — Remote access, Telegram bridge, handoff mode
   - **dataflow** — Data analysis, Kaggle, profiling, EDA
   - **career** — Career development, portfolio, publishing, professional growth
4. Infer the best project match based on keyword alignment, similarity to already-assigned cards, and the project's `summary` field
5. If a confident match exists, set the card's `project` field to the project `id` in board.json
6. If no clear match (personal tasks, vague cards, cross-cutting work that doesn't fit one project), leave `project` as `null`

### Rules
- **Never change a card that already has a project assigned**
- Prefer leaving `null` over a forced fit — accuracy matters more than completeness
- Update board.json directly (no comment needed for routine project assignment)

## Step 8: Detect Orphans

Check for inconsistencies:
- **Plan files without cards:** Scan `workspace/mission-control/plans/*.md` — any file not referenced by a card's `plan_file` field is an orphan. Report it.
- **Cards with missing plan files:** Any card whose `plan_file` points to a non-existent file. Report it.

Report orphans but don't auto-fix — let the user decide.

## Step 9: Clean Stale Activity

Clear activity entries older than 120 minutes:
```bash
python3 ~/skill-backends/noteflow/mc-activity.py --clear-stale
```

If `--clear-stale` isn't supported, skip this step silently — activity auto-stale handles it.

## Step 10: Update CC Auto-Memory

Update the **Active Project Summary** section in your auto-memory MEMORY.md to reflect current board state. Each project gets exactly three fields:

```
- **ProjectName**: [current state — one line: phase, what's working, overall posture]
  - Last: [most recent meaningful action — what happened, not just "worked on X"]
  - Next: [specific next card slug or decision — e.g., "mc-116 (profile context layer)" or "decide on storage backend"]
```

### Rules:
- **Last** = the most recent action that moved the project forward (a completed card, a key decision, a shipped feature). Not "worked on" — what specifically changed.
- **Next** = the single most likely next step. Use the card slug if one exists. If multiple cards are pending, pick the one most likely to be worked on next.
- Update status lines for any project whose cards changed
- Add new projects if any were created
- Remove references to projects that no longer exist
- The goal is **zero-read resume** — a new session should be able to answer "what's next for X?" from this section alone, without reading board.json

## Step 11: Report

Print a concise summary:

```
msync complete.

Git scan:
- Workspace: [N commits, M uncommitted changes]
- Backends: [list repos with changes]

NoteFlow → MC sync:
- Created [N] card(s): [nf-id] → [mc-slug], ...
- Synced [N] done status(es)
(or: All NoteFlow tasks already linked)

Project assignments:
- [card-slug] → [project-name]
- [N] cards left unassigned (no confident match)
(or: No unassigned cards)

Cards:
- [card-slug]: [old-status] → [new-status] (reason)
- [card-slug]: commented (summary of progress)
- [N] cards unchanged

Phases:
- [project]/[phase]: → done
(or: No phase changes)

New cards:
- [card-slug]: [title]
(or: None)

Orphans:
- [list any orphaned plan files or cards]
(or: None)

Activity: cleaned stale entries
(or: Activity: no stale entries)
```

If nothing changed at all: `msync complete. Board is current — no changes needed.`

## What msync Updates (Calendar Impact)

msync is the **one-stop command** for keeping Mission Control current. For the calendar specifically:
- **MC card completions** — cards moved to done in Step 4 appear directly on the calendar via board.json
- **NoteFlow done sync** — Step 5b propagates done status to linked NoteFlow items, ensuring task completion timestamps are written
- **Activity cleanup** — Step 9 clears stale activity entries

The calendar also shows **vault checkpoint notes** (written by `/checkpoint`) and **cron job occurrences** (managed by OpenClaw's cron engine) — these are outside msync's scope.

## What This Command Does NOT Do

- Daily notes (`workspace/memory/`)
- Obsidian vault journal (`vaults/Claw/Daily/`) — this is `/checkpoint`'s job
- Long-term memory distillation (`workspace/MEMORY.md`)
- Reference doc updates (`workspace/reference/`)
- Git commit/push
- Memory file trimming

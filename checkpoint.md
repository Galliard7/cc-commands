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

msync already updated the **Active Project Summary** (using the structured Current/Last/Next format). Now review your auto-memory MEMORY.md for broader updates:

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

All files written to the vault in this step must follow the **Obsidian linking conventions** below. These conventions build a connected graph in Obsidian and prepare the vault for future vector DB + graph-aware retrieval.

### Obsidian Linking Conventions

**Frontmatter:** Every vault file gets YAML frontmatter:

```yaml
---
type: summary | session | plan          # what kind of file this is
project: noteflow                        # primary project (if applicable)
card: mc-016                             # MC card slug (if applicable)
date: 2026-03-12                         # date of the work
tags: [noteflow, cc-remote]              # all projects touched (for summaries)
---
```

- `type`, `date` are always required
- `project` and `card` — include when the file is about a specific card/project
- `tags` — for summaries, list all projects mentioned; for session/plan files, list the primary project

**Wikilinks:** Use `[[double brackets]]` to link to other vault files. Rules:

1. **Link plan names** when referencing work done on a card: `[[stack-task-pool-sidebar]]`, `[[contentflow-ingestion-skill]]` — use the plan filename without `.md`
2. **Link project hub notes** when referencing a project: `[[NoteFlow]]`, `[[CC-Remote]]`, `[[MissionControl]]` — these hubs may not exist yet, and that's fine (Obsidian shows them as unresolved links, ready to be created later)
3. **Use typed context** around links — the surrounding words give the link meaning:
   - "Finished [[stack-task-pool-sidebar]]" — completion relationship
   - "Started [[noteflow-update-system]]" — began work
   - "Blocked by [[heartbeat-v2]]" — dependency
   - "Supersedes [[token-optimization-v1]]" — evolution
   - "Related: [[skill-architecture-standardization]]" — lateral connection
4. **Don't over-link** — link cards/plans that had meaningful status changes or decisions, not every passing mention. For summaries, aim for 3-8 links per checkpoint block. For session files, 1-3 links.
5. **Link between plan files** — when archiving a plan to `OpenClaw/`, add a "Related" section at the bottom linking to dependency/prerequisite/superseded plans if the relationship is clear from board.json

### 8a. Migrate flat daily files (legacy — no-op on KB-restructured vault)

As of 2026-04-05, the vault was restructured for the LLM knowledge base (see `workspace/projects/claw/plans/llm-knowledge-base.md`). Session files now live in `~/.openclaw/vaults/Claw/raw/sessions/YYYY-MM-DD/`. The old `Daily/` tree no longer exists. If `Daily/` reappears (e.g., from a restore), migrate it into `raw/sessions/`:

```bash
if [ -d ~/.openclaw/vaults/Claw/Daily ]; then
  mkdir -p ~/.openclaw/vaults/Claw/raw/sessions
  mv ~/.openclaw/vaults/Claw/Daily/* ~/.openclaw/vaults/Claw/raw/sessions/ 2>/dev/null
  rmdir ~/.openclaw/vaults/Claw/Daily 2>/dev/null
fi
```

### 8b. Read session files

Check if `~/.openclaw/workspace/sessions/` exists and contains `.md` files. If yes, read all session files — these provide per-task context that should inform the daily summary narrative.

```bash
ls ~/.openclaw/workspace/sessions/*.md 2>/dev/null
```

If session files exist, read each one. Use their content as **primary input** alongside the git diffs from Step 1 when writing the summary below.

### 8c. Write summary

**File:** `~/.openclaw/vaults/Claw/raw/sessions/YYYY-MM-DD/summary-YYYY-MM-DD.md` (using today's date)

Ensure the date folder exists:
```bash
mkdir -p ~/.openclaw/vaults/Claw/raw/sessions/YYYY-MM-DD
```

This file becomes a raw-source feeder for the KB wiki. `kb-compile` can later ingest it into `wiki/projects/{slug}/` pages.

Write a **human-readable journal** covering all work since the last checkpoint. This is a narrative briefing, not a mechanical status dump. Use session files (if any) to add detail and specificity.

If the file doesn't exist, create it. If it already exists (previous checkpoint today), **append** a new block.

### Format:

```markdown
---
type: summary
date: YYYY-MM-DD
tags: [project1, project2, ...]
---

# YYYY-MM-DD

## Checkpoint: HH:MM CT

### What We Worked On
Narrative summary using [[wikilinks]] to reference plans worked on. Example: "We finished [[stack-task-pool-sidebar]] and started [[noteflow-update-system]]." Write in first-person plural. Include enough context that reading this months later still makes sense.

### Project Status
Brief status of each active project, linking to project hubs: "**[[NoteFlow]]**: Update system complete."

### Ideas & Threads
Open ideas, future directions discussed, things to explore later. Link to relevant plans or projects where applicable.

### Decisions
Key decisions made and the reasoning behind them. Link to the plan the decision affects.

### Next Up
What's queued next, with links to the relevant plans.
```

Frontmatter goes at the very top of the file (only on initial creation, not when appending a second checkpoint block to an existing file). The `tags` array lists all project names mentioned in this checkpoint.

### Rules:
- Write for the **future reader** — assume no recent context
- Include the "why" behind decisions, not just the "what"
- Capture ideas and threads that might not make it into board.json
- Keep it concise but complete — aim for a 2-minute read
- Use natural language, not bullet-point soup
- **Incorporate session file details** — they capture real-time work entries that git diffs miss
- **Use wikilinks per the Obsidian Linking Conventions above** — link plans that had meaningful progress, not every passing mention

## Step 9: Enrich and Sweep Session Files

Enrich session files with frontmatter and wikilinks, then move them into the vault.

1. Check if `~/.openclaw/workspace/sessions/` exists and has `.md` files
2. If yes, **read each session file** and prepend frontmatter + add wikilinks:
   - Add YAML frontmatter: `type: session`, `date`, `project` (inferred from content or card reference), `card` (if the file references a specific MC card slug like `mc-016`)
   - Add wikilinks within the existing content where plan names or project names appear naturally — e.g., wrap existing references like "cc-daemon.py" in context linking to `[[cc-remote-handoff-mode]]` if that's the plan being worked on
   - Don't rewrite the session file — just prepend frontmatter and add links to 1-3 key plan/project references inline
3. Move enriched files to the vault:
   ```bash
   mkdir -p ~/.openclaw/vaults/Claw/raw/sessions/YYYY-MM-DD
   mv ~/.openclaw/workspace/sessions/*.md ~/.openclaw/vaults/Claw/raw/sessions/YYYY-MM-DD/
   rmdir ~/.openclaw/workspace/sessions/
   ```
4. If no session files exist, skip silently

The swept files live alongside `summary-YYYY-MM-DD.md` in the date folder, providing per-task detail that complements the narrative summary. They are raw-source input for the KB wiki.

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

## Step 12: Knowledge Base Compile

Compile any raw sources that have landed in `~/.openclaw/vaults/Claw/raw/` since the last KB compile — **not just the session files swept in Step 9**, but anything new in `raw/articles/`, `raw/papers/`, `raw/clips/`, `raw/transcripts/`, or `raw/sessions/` (Web Clipper saves, `/kb-add` captures, Telegram shares, direct drops). This is what makes the wiki compound over time.

The knowledge base is the LLM-compiled Obsidian wiki at `~/.openclaw/vaults/Claw/`. See `workspace/projects/claw/plans/llm-knowledge-base.md` for the full design and `~/skill-backends/knowledgebase/SKILL.md` for the authoritative ingest protocol.

### 12a. Find uncompiled sources

```bash
python3 ~/skill-backends/knowledgebase/kb-compile.py --list-uncompiled
```

This scans `raw/**` and lists every file not yet recorded in `source-map.json`.

### 12b. Decide batch size

Ingest is expensive (one LLM read + multiple wiki writes per source). Apply these guardrails:

| Uncompiled count | Action |
|---|---|
| 0 | Skip. Report "KB: nothing to compile." |
| 1–8 | Compile **all** inline in this checkpoint. |
| 9–20 | Compile the **top 10** by recency. Prioritize: (1) today's session files, (2) yesterday's session files, (3) articles/papers/clips/transcripts captured in the last 3 days, (4) older items. Log the deferred remainder to the report. |
| 21+ | Compile **only today's `raw/sessions/YYYY-MM-DD/*.md`**. Warn the user in the report: "KB backlog is N sources — run `/kb-compile --all-new` in a dedicated session to catch up." |

Sort by file modification time (`stat -f %m` on macOS) when picking the newest.

### 12c. Read schema + protocol (once per checkpoint)

Before the first compile in this checkpoint, read:
1. `~/.openclaw/vaults/Claw/schema.md` — wiki conventions (frontmatter, naming, cross-linking)
2. The **Ingest Protocol** section of `~/skill-backends/knowledgebase/SKILL.md` — Steps 1–7

Skip if already loaded earlier in this session.

### 12d. For each source in the batch

Follow the ingest protocol:

1. **Drive** — `python3 ~/skill-backends/knowledgebase/kb-compile.py --source <rel-path>` prints the source path and protocol reminder
2. **Read** — use the Read tool on the raw source in full
3. **Search first** — `python3 ~/skill-backends/knowledgebase/kb-query.py "<candidate page name>"` to avoid duplicates
4. **Classify** — what is it, what entities/concepts, which project(s)? Use `kb_lib.PROJECTS` for the canonical list
5. **Write pages** — create or update under `wiki/concepts/`, `wiki/entities/`, `wiki/projects/<slug>/`, `wiki/research/<topic>/`, or `wiki/synthesis/` as appropriate. Every page gets proper frontmatter (`type`, `title`, `created`, `updated`, `sources`, `tags`) and inline `(Source: [[raw/...]])` citations. Kebab-case filenames.
6. **Cross-link** — wikilink entities, concepts, projects, related pages
7. **Record** — `python3 ~/skill-backends/knowledgebase/kb-compile.py --source <rel-path> --record <page1> <page2> ...`

Recording automatically rebuilds the `_index.md` for touched sections and appends to `log.md`.

### 12e. Session-file priority rules

Session files (`raw/sessions/YYYY-MM-DD/*.md`) are the highest-volume feeder and compile differently from articles:

- **Summary files (`summary-YYYY-MM-DD.md`)** — extract key decisions, new concepts introduced, project status changes, blockers resolved. Update `wiki/projects/<slug>/_index.md` "Recent activity" with a dated bullet. Create new concept/entity pages only for *new* ideas (not re-mentions).
- **Task files (`task-<slug>.md` or `<card-slug>.md`)** — append a dated entry under the matching `wiki/projects/<slug>/plans/<card-slug>.md` in a "Session notes" section (create the section if absent). Do **not** create new concept pages from task files unless they introduce something genuinely new — task files are granular and noisy.

### 12f. Batch report

After the batch is done, run:
```bash
python3 ~/skill-backends/knowledgebase/kb-index.py
python3 ~/skill-backends/knowledgebase/kb-log.py --op compile --subject "checkpoint-batch" --details "<N sources, M pages>"
```

Track counts to include in Step 14's report:
- Sources compiled (by category: sessions/articles/papers/clips/transcripts)
- Pages created
- Pages updated
- Sources deferred (if any)

### 12g. Skip conditions

Skip Step 12 entirely if:
- `--list-uncompiled` returns nothing
- The checkpoint itself is a "nothing changed" checkpoint (Step 1 found no diffs)
- The user explicitly passed `--no-kb` in `$ARGUMENTS`

## Step 13: Commit and Push All Repos

Commit and push **every repo with changes** — workspace, skill backends, and standalone repos.

### 13a. Workspace repo
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

### 13b. All code repos
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

## Step 14: Report Summary

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
- Vault note — [created/appended] vaults/Claw/Daily/YYYY-MM-DD/summary-YYYY-MM-DD.md
- Sessions — swept N file(s) to vaults/Claw/raw/sessions/YYYY-MM-DD/ [or "no session files"]
- Flat file migration — migrated N file(s) [or "none needed"]
- Reference docs — [which ones, if any]
- KB compile — compiled N source(s) → M page(s) created, K updated; deferred: X [or "nothing to compile" or "skipped (--no-kb)"]
- Git — workspace: [committed and pushed / no changes]; backends: [list repos pushed, or "no changes"]

No changes:
- [list any files that didn't need updating and why]
```

If nothing changed since last checkpoint: `Checkpoint complete. No changes detected since last checkpoint — nothing to update.`

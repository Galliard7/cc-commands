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

### 1d. Scan chronicle files

Scan all project directories for CHRONICLE.md entries since the last checkpoint:
```bash
for proj in ~/.openclaw/workspace/projects/*/; do
  [ -f "$proj/CHRONICLE.md" ] && echo "$(basename $proj): $proj/CHRONICLE.md"
done
```

Read each CHRONICLE.md found. Extract entries dated **after** the last checkpoint date. Note which projects have new entries and what they cover. If no entries are new, note it and move on.

Also check for chronicles in standalone project dirs:
```bash
for proj in ~/projects/*/; do
  [ -f "$proj/CHRONICLE.md" ] && echo "$(basename $proj): $proj/CHRONICLE.md"
done
```

### 1e. Scan changed plan files

Find plan files modified since the last checkpoint:
```bash
cd ~/.openclaw/workspace
git diff <last-checkpoint-hash>..HEAD --name-only -- 'projects/*/plans/*.md'
git diff --name-only -- 'projects/*/plans/*.md'  # uncommitted changes
```

Read any changed plan files — these represent evolving project strategy worth capturing in the KB.

### 1f. Synthesize

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
- CC auto-memory update (if warranted)

**Checkpoint provides extended range:** msync's own git scan uses a 7-day rolling window. Checkpoint's Step 1 provides the precise range since the last checkpoint. Both are available — use whichever gives the most complete picture. If the last checkpoint was more than 7 days ago, Step 1's diffs are the authoritative source for older changes.

**Skip msync's Step 11 (Report)** — checkpoint's own report (Step 15) incorporates msync's findings.

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

Review whether anything learned since the last checkpoint warrants a new or updated auto-memory entry.

CC auto-memory uses individual files at `~/.claude/projects/-Users-galliard7--openclaw/memory/` with a `MEMORY.md` index. Each memory file has frontmatter (`name`, `description`, `type`) and content. The index has one-line pointers.

### What to save (memory types):
- **user** — user role, preferences, knowledge (rarely changes)
- **feedback** — corrections or confirmed approaches from the user
- **project** — ongoing work context not derivable from code/git (convert relative dates to absolute)
- **reference** — pointers to external systems/resources

### What NOT to save:
- Project status, active cards, what's next — board.json is the source of truth
- Code patterns, file paths, architecture — derivable from the codebase
- Anything already in CLAUDE.md or existing memory files

### Process:
1. Check if any new memory is warranted (new convention, user correction, external reference learned)
2. If yes: check existing memory files for one to update before creating a new one
3. Write/update the individual memory file with proper frontmatter
4. Update the MEMORY.md index (one-line pointer per entry, under ~150 chars each)
5. If nothing new, skip silently

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

## Step 10: Deposit KB Raw Sources

Deposit chronicle digests, changed plan files, and git diff summaries into the KB vault's `raw/` directory. These become additional raw source material for kb-compile (Step 13).

### 10a. Chronicle digests

For each project that had new chronicle entries (from Step 1e), write a digest file:

```bash
mkdir -p ~/.openclaw/vaults/Claw/raw/chronicles
```

**File:** `~/.openclaw/vaults/Claw/raw/chronicles/YYYY-MM-DD-{project-slug}.md`

```markdown
---
type: chronicle-digest
project: {slug}
date: YYYY-MM-DD
source: workspace/projects/{slug}/CHRONICLE.md
---

# Chronicle Digest: {Project Name} — YYYY-MM-DD

{Summarize the new chronicle entries since last checkpoint. Preserve the "why" — decisions, pivots, dead ends, lessons learned. Keep the narrative voice from the original entries. 1-3 paragraphs per entry.}

## Entries Covered
- YYYY-MM-DD — {entry title} `#type-tag`
- YYYY-MM-DD — {entry title} `#type-tag`
```

Skip if no new chronicle entries for any project.

### 10b. Changed plan files

For each plan file that changed since last checkpoint (from Step 1f), snapshot it into KB raw:

```bash
mkdir -p ~/.openclaw/vaults/Claw/raw/plans
```

**File:** `~/.openclaw/vaults/Claw/raw/plans/YYYY-MM-DD-{plan-filename}`

Copy the changed plan file with a date-prefixed name. Prepend frontmatter if the plan doesn't already have it:

```yaml
---
type: plan-snapshot
project: {slug}
date: YYYY-MM-DD
source: workspace/projects/{slug}/plans/{filename}
---
```

If the plan already has frontmatter, add `snapshot_date: YYYY-MM-DD` to it instead. Skip if no plan files changed.

### 10c. Git diff summaries

Write a condensed diff summary covering all repos that had changes:

```bash
mkdir -p ~/.openclaw/vaults/Claw/raw/diffs
```

**File:** `~/.openclaw/vaults/Claw/raw/diffs/YYYY-MM-DD-diff-summary.md`

```markdown
---
type: diff-summary
date: YYYY-MM-DD
repos: [workspace, noteflow, knowledgebase, ...]
---

# Git Diff Summary — YYYY-MM-DD

## workspace
{Condensed summary: what files changed, what the changes accomplish. Not raw diff output — a human-readable 2-4 sentence summary.}

## {backend-name}
{Same format per repo with changes.}
```

This is a **human-readable narrative**, not a paste of `git diff --stat`. Focus on what changed and why, not line counts. Skip if nothing changed.

### Rules:
- All files in `raw/` are **verbatim captures** — don't edit them after creation
- Use date-prefixed filenames to avoid collisions across checkpoints
- Frontmatter is required for kb-compile to classify sources correctly
- These files feed into Step 13 (KB compile) on the next run

## Step 11: Update Reference Docs

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

## Step 12: Trim Memory Files

Check these files and trim if they exceed the stated limits:

| File | Max Lines | Trim Strategy |
|---|---|---|
| CC auto-memory MEMORY.md (index) | ~200 lines | Remove stale entries, merge related pointers. Individual memory files have no line limit but should be concise. |
| `workspace/MEMORY.md` | ~100 lines | Archive old entries to daily notes |

Only trim if actually over the limit. Use your judgment about what to cut.

## Step 13: Knowledge Base Compile (delegated to Sonnet subagent)

Compile any raw sources that have landed in `~/.openclaw/vaults/Claw/raw/` since the last KB compile — **not just the session files swept in Step 9**, but anything new in `raw/articles/`, `raw/papers/`, `raw/clips/`, `raw/transcripts/`, `raw/sessions/`, `raw/chronicles/`, `raw/plans/`, or `raw/diffs/` (Web Clipper saves, `/kb-add` captures, Telegram shares, checkpoint deposits, direct drops). This is what makes the wiki compound over time.

The knowledge base is the LLM-compiled Obsidian wiki at `~/.openclaw/vaults/Claw/`. See `workspace/projects/claw/plans/llm-knowledge-base.md` for the full design and `~/skill-backends/knowledgebase/SKILL.md` for the authoritative ingest protocol.

**This step ALWAYS delegates to a Sonnet subagent via the Agent tool** — never compile inline on the main checkpoint thread. Reasons: (1) keeps the main thread's context clean for the rest of the checkpoint, (2) Sonnet is faster and cheaper than Opus for this structured protocol-driven work, (3) consistent behavior regardless of backlog size, (4) the `source-map.json` persistence means even a failed subagent run is resumable on the next checkpoint.

### 13a. Find uncompiled sources

```bash
python3 ~/skill-backends/knowledgebase/kb-compile.py --list-uncompiled
```

This scans `raw/**` and lists every file not yet recorded in `source-map.json`.

### 13b. Skip conditions

Skip Step 13 entirely (no subagent spawn) if ANY of these are true:
- `--list-uncompiled` returns 0 sources → report "KB: nothing to compile."
- The checkpoint itself is a "nothing changed" checkpoint (Step 1 found no diffs) **AND** backlog is 0
- The user explicitly passed `--no-kb` in `$ARGUMENTS` → report "KB: skipped (--no-kb)"

Note: do NOT skip just because Step 1 found no git diffs. Non-git sources (Web Clipper saves, `/kb-add` captures, Telegram shares) land in `raw/` without touching any git repo, and the checkpoint is the only thing that picks them up. Only skip on "no diffs" if the backlog is also empty.

### 13c. Soft warning at high volume

If backlog is >50 sources, include a note in the subagent spawn message and the final Step 15 report: "KB compile processing N sources — this may take several minutes." This is a UX courtesy, not a gate. The subagent still runs the full batch.

There is **no hard size limit**. The subagent processes everything. Historical note: an earlier version of this command had a 21+ defer rule that caused permanent backlog accumulation — removed 2026-04-11 in favor of always-delegate.

### 13d. Spawn the Sonnet compile worker

Use the Agent tool with `subagent_type: general-purpose`, `model: sonnet`, and the prompt template below. Do not omit any section of the prompt — the subagent runs with a fresh context and needs the full protocol inline.

**Agent tool call:**
- `subagent_type`: `general-purpose`
- `model`: `sonnet`
- `description`: `KB compile — <N> uncompiled sources`
- `prompt`: (use the template below, substituting `<N>` with the actual backlog count)

**Prompt template:**

```
You are running a knowledge-base compile for the Claw Obsidian wiki. There are <N> uncompiled raw sources and you need to process ALL of them per the ingest protocol. Work to completion — don't stop early, don't ask questions, don't leave things half-done. If you hit an error on a specific source, log it, skip that source, and continue.

## Your environment
- Vault root: `~/.openclaw/vaults/Claw/`
- Raw sources: `~/.openclaw/vaults/Claw/raw/` (sessions/, articles/, papers/, clips/, transcripts/, chronicles/, plans/, diffs/)
- Compiled wiki: `~/.openclaw/vaults/Claw/wiki/` (concepts/, entities/, projects/, research/, synthesis/, _master-index.md)
- Schema: `~/.openclaw/vaults/Claw/schema.md` — READ THIS FIRST, it defines frontmatter, naming, cross-linking conventions
- Ingest protocol: `~/skill-backends/knowledgebase/SKILL.md` — READ STEPS 1-7 SECOND, authoritative ingest process
- KB scripts: `~/skill-backends/knowledgebase/kb-*.py` — use these, don't bypass them
- Mission Control board for project context: `~/.openclaw/workspace/mission-control/board.json`
- Project slug list: `kb_lib.PROJECTS` (import from `~/skill-backends/knowledgebase/kb_lib.py`)

## Step 0: One-time setup (do once, not per source)
1. Read `~/.openclaw/vaults/Claw/schema.md` in full
2. Read `~/skill-backends/knowledgebase/SKILL.md` — specifically the Ingest Protocol steps 1-7
3. Run `python3 ~/skill-backends/knowledgebase/kb-compile.py --list-uncompiled` to get the full list
4. Glance at `~/.openclaw/vaults/Claw/wiki/_master-index.md` and one existing page in each category (`wiki/concepts/`, `wiki/entities/`, `wiki/projects/*/`) to understand existing conventions before writing

## Per-source ingest loop (for each uncompiled source)

1. **Drive** — `python3 ~/skill-backends/knowledgebase/kb-compile.py --source <rel-path>` (prints source + reminder)
2. **Read** — Use the Read tool on the raw source file in full
3. **Search first** — `python3 ~/skill-backends/knowledgebase/kb-query.py "<candidate page name>"` to avoid duplicates. Do this BEFORE writing any new page.
4. **Classify** — what kind of source is it? what entities/concepts/decisions does it contain? which project(s) does it belong to (common slugs: claw, noteflow, missioncontrol, cc-remote, contentflow, dataflow, career, tradeflow, trader, flow, gemma4good, tutor)?
5. **Write pages** — Create or update pages under the appropriate directory:
   - `wiki/concepts/` — general concepts, patterns, techniques
   - `wiki/entities/` — specific named things (tools, libraries, people, systems)
   - `wiki/projects/<slug>/` — project-specific pages, with `plans/` subdir for plan notes
   - `wiki/research/<topic>/` — research topic deep-dives
   - `wiki/synthesis/` — cross-cutting syntheses that tie multiple sources together

   Every page MUST have proper frontmatter:
   ```yaml
   ---
   type: concept | entity | project | plan | research | synthesis
   title: <Page Title>
   created: <today>
   updated: <today>
   sources:
     - raw/sessions/YYYY-MM-DD/foo.md
   tags: [project-slug, topic]
   ---
   ```
   Use kebab-case for filenames. Add inline `(Source: [[raw/sessions/YYYY-MM-DD/foo]])` citations when making factual claims.
6. **Cross-link** — wikilink entities, concepts, projects, and related pages using `[[double-brackets]]`. Pages shouldn't be islands.
7. **Record** — `python3 ~/skill-backends/knowledgebase/kb-compile.py --source <rel-path> --record <page1> <page2> ...` — this automatically rebuilds the touched section's `_index.md` and appends to `log.md`. Pass relative paths from vault root.

## CRITICAL: Session-file special rules

Session files (`raw/sessions/YYYY-MM-DD/*.md`) are the highest-volume feeder and compile DIFFERENTLY from articles:

- **Summary files (`summary-YYYY-MM-DD.md`)**: Extract KEY decisions, NEW concepts introduced, project status changes, blockers resolved. Update `wiki/projects/<slug>/_index.md` "Recent activity" section with a dated bullet (create the section if it doesn't exist). ONLY create new concept/entity pages for *genuinely new* ideas (not re-mentions of existing concepts). Summary files typically touch 3-8 pages.

- **Task files (`task-<slug>.md` or named after a card slug like `neetcode-150-mission-control-tab.md`)**: APPEND a dated entry under the matching `wiki/projects/<slug>/plans/<card-slug>.md` file in a "Session notes" section (create the section if absent, create the plan page if it doesn't exist — use the card's title as page title and look up the card in board.json for context). Do NOT create new concept/entity pages from task files unless they introduce something genuinely new. Task files are granular and noisy — resist the urge to over-compile. A task file typically records to 1-2 pages.

- **Chronicle digests (`raw/chronicles/YYYY-MM-DD-{slug}.md`)**: These capture the "why" behind project decisions, pivots, and dead ends. Update the matching `wiki/projects/<slug>/_index.md` with decision/pivot entries. If a chronicle mentions a genuinely new concept or pattern, create a concept page. Chronicle digests are higher-signal than session files — they represent curated narrative, not raw work logs.

- **Plan snapshots (`raw/plans/YYYY-MM-DD-{plan-name}`)**: Update the matching `wiki/projects/<slug>/plans/<plan-name>.md` page with the latest plan state. If the plan page doesn't exist, create it. Plans define project direction — treat them as authoritative for the project's current architecture and goals.

- **Diff summaries (`raw/diffs/YYYY-MM-DD-diff-summary.md`)**: These are context for what changed, not standalone pages. Use them to enrich existing project pages with "Recent changes" context. Don't create new pages from diffs alone — they supplement other sources.

- **Non-session sources** (articles, papers, clips, transcripts): Follow the full protocol — create concept/entity pages freely, add to research/ for deep-dives, synthesize.

## Batch strategy

Process sources in the order `--list-uncompiled` returns them (chronological by date folder). Newer summary files often reference earlier work, so earlier files should exist in the wiki before the summaries that reference them.

Report progress every 20 sources with a one-line summary: "Processed N/<total> — X pages created, Y updated, Z sources skipped."

If you hit an error on a specific source, log the error, skip that source, and continue. Don't stop the whole batch for one bad source.

## After the batch is complete

1. Run `python3 ~/skill-backends/knowledgebase/kb-index.py` — rebuilds all indexes
2. Run `python3 ~/skill-backends/knowledgebase/kb-stats.py` — prints final stats
3. Run `python3 ~/skill-backends/knowledgebase/kb-log.py --op compile --subject "checkpoint-<YYYY-MM-DD>" --details "<N sources compiled, M pages created, K pages updated, X skipped>"`
4. Run `python3 ~/skill-backends/knowledgebase/kb-lint.py` — report any issues but don't try to fix them all

## Final report format (return this to the caller)

```
KB compile complete.

Sources processed: N/<total>
- Sessions: X (summary) + Y (task)
- Chronicles: Z
- Plans: Z
- Diffs: Z
- Articles: Z
- Papers: Z
- Clips: Z
- Transcripts: Z
- Skipped/errored: W (with brief reason per source)

Pages created: M
- concepts/: ...
- entities/: ...
- projects/<slug>/: ...
- research/: ...
- synthesis/: ...

Pages updated: K

Notable new concepts/entities: [list 5-10 most important, or "none notable" for routine batches]

Lint results: [summary from kb-lint]
```

## Rules

- **Work to completion.** All sources processed before you stop.
- **Use the scripts.** Always `kb-compile --record` to register sources. Don't write directly to source-map.json or log.md.
- **Trust existing pages.** If `kb-query` returns a hit, update the existing page (add new citation, new content) rather than creating a duplicate.
- **Don't over-compile session files.** They're noisy by design. Extract the signal, skip the noise.
- **Preserve user's work.** Never delete existing wiki pages. Only create or update.
- **No conversational fluff.** Pages should read like reference material, not chat logs.

Begin now. No confirmation needed. Work to completion.
```

### 13e. Incorporate subagent report

After the Sonnet subagent returns, capture its report summary and roll it into Step 15's KB compile line. The subagent handles all the heavy lifting (schema reading, protocol execution, session-file priority rules, kb-index rebuild, kb-lint); the main checkpoint thread just records the outcome.

If the subagent fails or errors out, log the failure but don't fail the whole checkpoint — the next checkpoint will resume from `source-map.json` automatically. Note the failure in Step 15's report with enough detail for the user to debug.

## Step 14: Commit and Push All Repos

Commit and push **every repo with changes** — workspace, skill backends, and standalone repos.

### 14a. Workspace repo
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

### 14b. KB vault repo
```bash
cd ~/.openclaw/vaults/Claw
git add -A
git status --short
```
If there are staged changes:
```bash
git commit -m "checkpoint: YYYY-MM-DD — [1-line summary: wiki pages updated, raw sources added, etc.]"
git push
```

### 14c. All code repos
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

## Step 15: Report Summary

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
- CC memory — [new/updated memory files, or "no changes"]
- Daily note — [created/appended] workspace/memory/YYYY-MM-DD.md
- Long-term memory — [what was added/updated/removed, or "no changes"]
- Vault note — [created/appended] vaults/Claw/raw/sessions/YYYY-MM-DD/summary-YYYY-MM-DD.md
- Sessions — swept N file(s) to vaults/Claw/raw/sessions/YYYY-MM-DD/ [or "no session files"]
- Flat file migration — migrated N file(s) [or "none needed"]
- KB raw deposits — chronicles: N project(s), plans: N file(s), diffs: [yes/no] [or "nothing to deposit"]
- Reference docs — [which ones, if any]
- KB compile — Sonnet subagent processed N source(s) → M page(s) created, K updated [or "nothing to compile" or "skipped (--no-kb)" or "subagent failed: <reason>, next checkpoint will resume"]
- Git — workspace: [committed and pushed / no changes]; backends: [list repos pushed, or "no changes"]

No changes:
- [list any files that didn't need updating and why]
```

If nothing changed since last checkpoint: `Checkpoint complete. No changes detected since last checkpoint — nothing to update.`

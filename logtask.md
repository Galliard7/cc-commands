---
description: "Log current session work as an active card on the Mission Control board"
disable-model-invocation: true
---

# /logtask — Log Current Work to Mission Control

You are logging the current session's work to the Mission Control board. **The primary goal is to avoid redundancy** — always prefer updating an existing card over creating a new one.

## User Prompt: `$ARGUMENTS`

## Step 1: Analyze What to Log

**If the user provided a prompt** (i.e., `$ARGUMENTS` is not empty):
- The user's prompt is the **primary signal**. Interpret it to understand what they want tracked.
- Use the conversation history as **supporting context** — it can fill in details the prompt doesn't mention, but the prompt defines the scope and intent.
- If the prompt describes something not discussed in the session (e.g., a future task, a side idea), that's fine — use it as-is.

**If no prompt was provided** (i.e., `$ARGUMENTS` is empty):
- Infer the task entirely from the conversation history.

Either way, identify:
- **What the work is** — the primary goal or task
- **Key details** — technologies, files, features, bugs involved
- **Scope** — is this a bug fix, feature, refactor, research, infra, etc.?

Distill this into a concise summary of the work.

## Step 2: Read the Board & Find a Match

Read `workspace/mission-control/board.json`. Examine **all cards** (including `done`).

For each card, compare its `title`, `description`, `project`, and `comments` against the current session's work. Look for:
- **Direct match** — the card describes the same work being done now (even if worded differently)
- **Superset match** — the card covers a broader scope that includes this work
- **Subset match** — the card covers part of what's being done

**Matching is generous** — if the session work reasonably falls under an existing card's umbrella, that's a match. Prefer false positives (updating an existing card) over false negatives (creating a duplicate).

**Priority:** If multiple cards match, prefer non-done cards over done cards. Only match a `done` card when no live (active/pending/backlog/blocked) card covers the work.

### If a match is found → go to Step 3 (Update Existing Card)
### If no match → go to Step 4 (Create New Card)

## Step 3: Update Existing Card

### 3a. Move to active (if not already — skip for done cards)

If the matched card is `done`, **do not reopen it** — skip directly to Step 3b.

If the matched card is in `pending`, `backlog`, or `blocked` status, move it to `active`:

```bash
python3 -c "
import sys
sys.path.insert(0, '$HOME/skill-backends/noteflow')
from mc_lib import load_board, move_card

board = load_board()
card, err = move_card(board, '<MATCHED-SLUG>', 'active')
if err:
    print(f'Error: {err}')
else:
    print(f'Moved {card[\"id\"]} to active')
"
```

If already `active`, skip this.

### 3b. Update description (if needed)

If the current work adds meaningful context that the existing description lacks, update it. **Don't rewrite** — enrich. If the description is already adequate, skip this.

```bash
python3 -c "
import sys
sys.path.insert(0, '$HOME/skill-backends/noteflow')
from mc_lib import load_board, update_card

board = load_board()
card, err = update_card(board, '<MATCHED-SLUG>', {'description': '<UPDATED-DESCRIPTION>'})
if err:
    print(f'Error: {err}')
else:
    print(f'Updated description for {card[\"id\"]}')
"
```

### 3c. Add a progress comment

Add a comment summarizing what's being done in this session. Keep it to 1-2 sentences — capture the specific work, not a restatement of the card's purpose.

For `done` cards, prefix the comment with "Follow-up:" to distinguish post-completion updates from in-progress work.

```bash
python3 ~/skill-backends/noteflow/mc-comment.py \
  --plan "<MATCHED-SLUG>" --comment "<1-2 sentence update on current session work>"
```

### 3d. Assign project (if missing)

If the matched card has `project: null` and you can confidently match it to a project (see project heuristics below), update it:

```bash
python3 -c "
import sys
sys.path.insert(0, '$HOME/skill-backends/noteflow')
from mc_lib import load_board, update_card

board = load_board()
card, err = update_card(board, '<MATCHED-SLUG>', {'project': '<PROJECT-ID>'})
"
```

→ Skip to Step 5 (Log Activity + Session Markdown).

## Step 4: Create New Card

Only reach this step if no existing card matches.

### 4a. Determine project

Match the current work to a project:
- **claw** — OpenClaw infrastructure, config, optimization, cross-cutting tooling, new commands/skills
- **missioncontrol** — Dashboard UI, board features, MC improvements
- **noteflow** — NoteFlow features, task/note management, reminders
- **contentflow** — Content ingestion, extraction, creation, video/media
- **cc-remote** — Remote access, Telegram bridge, handoff mode
- **dataflow** — Data analysis, Kaggle, profiling, EDA
- **career** — Career-related tasks, pitch decks, presentations

If no project fits cleanly, use `None`.

### 4b. Create the card

- **Title**: Short, imperative, descriptive. Max ~60 chars.
- **Description**: 2-4 sentences capturing what's being done and why.
- **Status**: `active` (this is current work).

```bash
python3 -c "
import sys
sys.path.insert(0, '$HOME/skill-backends/noteflow')
from mc_lib import load_board, add_card

board = load_board()
card = add_card(
    board,
    title='<TITLE>',
    description='<DESCRIPTION>',
    status='active',
    project=<PROJECT>,  # string or None
)
print(f'Created {card[\"id\"]} ({card[\"slug\"]}): {card[\"title\"]}')
print(f'Status: active | Project: {card[\"project\"] or \"unassigned\"}')
"
```

## Step 5: Log Activity

```bash
python3 ~/skill-backends/noteflow/mc-activity.py \
  --session "Claude Code" --text "<CARD-TITLE>" --label "CC" --color "#3b82f6"
```

## Step 6: Write Session Markdown

Create a local session file that accumulates work entries during the session. These files are swept into the Obsidian vault by `/checkpoint`.

### 6a. Ensure directory exists

```bash
mkdir -p ~/.openclaw/workspace/sessions
```

### 6b. Determine filename

Use the board card slug: `~/.openclaw/workspace/sessions/<CARD-SLUG>.md`

### 6c. Get current time

```bash
TZ=America/Chicago date '+%H:%M'
```

### 6d. Write the entry

**IMPORTANT: Always use bash `cat >>` to write session files. Never use Write/Edit tools — Write overwrites, and Edit can fail on nonexistent files. The bash command below handles both create and append correctly.**

```bash
SESSION_FILE=~/.openclaw/workspace/sessions/<CARD-SLUG>.md

# If file doesn't exist, write header first
if [ ! -f "$SESSION_FILE" ]; then
  cat > "$SESSION_FILE" << 'HEADER'
# <Card Title>
**Card:** <card-id> | **Project:** <project or unassigned>
HEADER
fi

# Always append the timestamped entry
cat >> "$SESSION_FILE" << 'ENTRY'

## HH:MM CT
<2-3 sentence summary of current work>
ENTRY
```

### Rules:
- **Always use the bash command above** — no exceptions, no tool substitution
- Summary should be short — what was done/is being done, not a restatement of the card description
- Use the conversation context to write a meaningful summary
- Each re-log appends, never overwrites

## Step 7: Push to Stack

**If the matched card is `done`, skip this step entirely** — done cards auto-remove from stack and should not be re-added.

Push a stack card into the Stack tab (via dashboard API at `http://127.0.0.1:8765`) **if one does not already exist** for this board card.

First, check if a stack item with the matching `boardSlug` already exists:

```bash
curl -s http://127.0.0.1:8765/api/stack | python3 -c "
import sys, json
items = json.load(sys.stdin).get('items', [])
match = [i for i in items if i.get('boardSlug') == '<CARD-SLUG>']
if match:
    print(f'EXISTS: {match[0][\"id\"]}')
else:
    print('NOT_FOUND')
"
```

**If NOT_FOUND**, create a stack item linked to the board card:

```bash
curl -s -X POST http://127.0.0.1:8765/api/stack \
  -H 'Content-Type: application/json' \
  -d '{"title": "<CARD-TITLE>", "source": "board", "boardSlug": "<CARD-SLUG>"}'
```

**If EXISTS**, skip — the card is already on the stack.

## Step 8: Report

**If an existing card was updated (non-done):**
```
Logged to existing card:
- Card: <id> — <title>
- Project: <project or "unassigned">
- Status: <status>
- Updated: <what changed — e.g., "moved to active + comment added" or "comment added">
- Session: <created / appended to> workspace/sessions/<slug>.md
- Stack: <pushed / already on stack>
```

**If a done card was matched (follow-up logged):**
```
Follow-up logged to completed card:
- Card: <id> — <title>
- Project: <project or "unassigned">
- Status: done (unchanged)
- Updated: follow-up comment added
- Session: <created / appended to> workspace/sessions/<slug>.md
- Stack: skipped (done)
```

**If a new card was created:**
```
New card created:
- Card: <id> — <title>
- Project: <project or "unassigned">
- Status: active
- Session: created workspace/sessions/<slug>.md
- Stack: pushed
```

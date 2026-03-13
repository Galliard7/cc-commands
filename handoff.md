---
description: "Toggle CC-Remote handoff mode (on/off/status)"
---

# /handoff — CC-Remote Handoff Mode

Toggle handoff mode for the CC-Remote skill. Handoff mode forwards Claude Code's approval requests and notifications to Telegram for remote supervision.

## Arguments: `$ARGUMENTS`

## Instructions

If `$ARGUMENTS` is empty or missing, print this usage help and stop:

```
Usage: /handoff <on|off|status>

  on      Activate handoff mode (Telegram gets approval requests)
  off     Deactivate handoff mode
  status  Show current handoff state
```

Otherwise, run:

```bash
python3 ~/skill-backends/cc-remote/cc-handoff.py $ARGUMENTS
```

Print the script's output as your response. Do not add extra commentary.

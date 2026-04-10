---
description: "Print KB size, coverage, and recent activity"
disable-model-invocation: true
---

# /kb-stats — Knowledge Base Stats

```bash
python3 ~/skill-backends/knowledgebase/kb-stats.py
```

Interpret the output: call out anything unusual (zero-page sections, large pending backlog, many uncovered sources). Suggest `/kb-compile --all-new` if pending backlog is >10, or `/kb-lint` if it's been more than a week since the last lint.

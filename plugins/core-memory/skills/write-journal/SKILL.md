---
name: write-journal
description: Reflect on session, write journal entries via MCP and create task/journal.md. Use at end of each session.
trigger: "TRIGGER when: session is ending, Stop hook blocks for missing journal.md, user says 'save learnings' or 'write journal', task completed with significant decisions or outcomes"
tools:
  - journal_recent
  - journal_write
  - journal_search
---

Reflect on what happened in this session and record significant events.

## Process

1. Call `journal_recent` MCP tool to review your recent memory
2. Identify significant events from this session:
   - `decision` — you made a choice between alternatives
   - `outcome` — a task/experiment produced a result
   - `observation` — you noticed a pattern, insight, or issue
   - `feedback` — you received a correction from a human
3. For each significant event, call `journal_write` MCP tool:
   - `type`: one of the four types above
   - `content`: what happened, with reasoning
   - `context`: which task/project this relates to
   - `tags`: relevant keywords (array)
4. Create `task/journal.md` summarizing what you wrote

## task/journal.md format

If entries were written:

```markdown
## Session Journal — {task_id}

- [decision] Brief description of what was decided
- [outcome] Brief description of the result
```

If nothing significant happened:

```markdown
## Session Journal — {task_id}

No significant entries this session.
```

## Rules

- Not every session produces journal entries — that's normal
- But you MUST always create task/journal.md (even if empty)
- Quality over quantity — one good entry beats five trivial ones
- Include reasoning in decisions, not just the choice

---
name: update-state
description: Update state.md with current session results. Use at end of each session.
---

Update the agent state file that persists between sessions.

## Process

1. Read current `state.md` in the repo root (if it exists)
2. Read `task/context.json` to get the current task identifier
3. Update relevant sections based on what happened this session:
   - **Текущий фокус** — if priorities shifted
   - **Активные задачи/инициативы** — if status changed
   - **Известные проблемы** — if discovered or resolved
   - **Ключевые проекты** — if progress was made (executive only)
   - **Ограничения** — if new blockers found (executive only)
   - **Открытые решения** — if decisions needed (executive only)
4. Set `last_task` in the first line to the current task identifier
5. Update the date in the heading

## Format

```markdown
<!-- last_task: SNAP-123 -->
# {Worker Name} State — {YYYY-MM-DD}

## Текущий фокус
...sections appropriate to role...
```

## Rules

- If state.md doesn't exist, create it from scratch
- Keep it compact: 20-50 lines. This is loaded into context every session.
- Every line must earn its place — remove stale information
- Update the date to today
- The `<!-- last_task: -->` comment MUST be the first line — the stop hook checks it

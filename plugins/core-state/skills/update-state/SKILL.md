---
name: update-state
description: Update state.md with current session results. Use at end of each session.
---

Update the agent state file that persists between sessions.

## Process

1. Read current `state.md` in the repo root (if it exists)
2. Read `task/context.json` to get the current task identifier
3. Update relevant sections based on what happened this session:
   - **Current focus** — if priorities shifted
   - **Active work / status** — if task or initiative status changed
   - **Known issues / blockers** — if discovered or resolved
   - Add role-specific sections as relevant (e.g. key projects, constraints,
     open decisions) — only keep sections that earn their place.
4. Set `last_task` in the first line to the current task identifier
5. Update the date in the heading

## Format

```markdown
<!-- last_task: SNAP-123 -->
# {Worker Name} State — {YYYY-MM-DD}

## Current focus
...sections appropriate to this worker's role...
```

## Rules

- If state.md doesn't exist, create it from scratch
- Keep it compact: 20-50 lines. This is loaded into context every session.
- Every line must earn its place — remove stale information
- Update the date to today
- The `<!-- last_task: -->` comment MUST be the first line — the stop hook checks it

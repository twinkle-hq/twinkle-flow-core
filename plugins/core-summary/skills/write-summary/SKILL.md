# Execution Summary

Before ending the session, you MUST create `task/summary.yaml` with a structured summary of your work. The Stop hook will validate this file — if it's missing or malformed, you will be blocked from ending the session.

## Format

### Regular Tasks

```yaml
type: regular
files_changed: 3
key_decisions: 1

summary: |
  One paragraph describing what was accomplished.

changes:
  - file: src/example.ts
    description: Added new validation endpoint
  - file: tests/example.test.ts
    description: Added tests for validation
  - file: README.md
    description: Updated API documentation

decisions:
  - choice: Used zod for schema validation instead of joi
    reason: Already used elsewhere in the codebase, avoids adding a new dependency
```

### Flow Tasks (multi-phase)

For tasks with phases defined in the task YAML, include a `phases` section:

```yaml
type: flow
phases_completed: 2
phases_total: 3
files_changed: 5
key_decisions: 2

summary: |
  Overall result of the flow task.

phases:
  - name: Research
    status: done
    summary: Analyzed the codebase and identified 3 integration points.
  - name: Implementation
    status: done
    summary: Added adapter layer with full test coverage.
  - name: Documentation
    status: skipped
    summary: Skipped — README already covers the new API.

changes:
  - file: src/adapter.ts
    description: New adapter implementation
  - file: tests/adapter.test.ts
    description: Adapter tests

decisions:
  - choice: Used adapter pattern instead of direct integration
    reason: Allows swapping backends without changing consumers
  - choice: Skipped documentation phase
    reason: Existing README already documents this API surface
```

## Required Fields

- `type` — `regular` or `flow` (use `flow` if the task has phases)
- `summary` — one paragraph, what was accomplished overall
- `changes` — list of files changed, each with `file` and `description`
- `decisions` — list of decisions made, each with `choice` and `reason`
- `files_changed` — must equal the number of entries in `changes`
- `key_decisions` — must equal the number of entries in `decisions`

### Additional fields for flow tasks:
- `phases` — list of phases, each with `name`, `status` (done/skipped/failed), and `summary`
- `phases_total` — must equal the number of entries in `phases`
- `phases_completed` — must equal the count of phases with `status: done`

## Rules

1. Write `task/summary.yaml` BEFORE ending the session
2. Every file you changed must appear in `changes`
3. Every non-trivial decision must appear in `decisions`
4. Phase names should match the phase names from the task definition
5. Phase statuses: `done` (completed), `skipped` (intentionally skipped with reason), `failed` (attempted but failed)

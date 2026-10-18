# core-state

`tier: core` · force-attached, **toggleable** (default-on, a workspace can turn
it off).

Gives a worker **cross-session state**: a compact `state.md` it carries between
sessions — current focus, active work, known blockers. Unlike `task/` (assembled
fresh per task and thrown away), `state.md` lives at the **repo root**, is
committed to the worker git repo, and is re-injected at the start of the next
session. It contributes:

- the **`update-state`** skill (the recipe, available as `/update-state`);
- a **SessionStart** hook that re-injects the current `state.md`;
- a blocking **`Stop`** gate that requires an updated `state.md`.

## `state.md` — location & lifecycle

`state.md` sits in the **repo root** (not under `task/`). That placement is
deliberate: the worker repo is committed by the system tier's autocommit /
`git_guard`, so `state.md` persists across sessions and across the ephemeral
worker container. The lifecycle each session:

1. **SessionStart** — if `state.md` exists, the hook emits `## Agent State`
   followed by the file body, so the agent starts the session already aware of its
   prior state.
2. The agent works the task.
3. **`update-state` skill** — at session end the agent rewrites the relevant
   sections, sets `last_task` to the current task id, and updates the date.
4. The autocommit / `git_guard` commits the changed `state.md` to the repo.

## The `<!-- last_task: ... -->` first-line contract

The `Stop` gate enforces more than mere presence. A blocking `Stop` hook (POSIX
`/bin/sh`):

1. Reads the Stop payload on stdin; **exits 0 immediately when `stop_hook_active`
   is true** (one-round blocking cap).
2. **Exits 2 (blocks)** if `state.md` is missing, **or** if its **first line**
   does not carry a `<!-- last_task: ... -->` marker. The message points at
   `/update-state`.

Requiring the first-line marker makes the `update-state` skill's contract real:
the skill claims the marker is mandatory, and the gate now enforces it (a stale,
untouched `state.md` without an updated `last_task` no longer trivially passes).
The marker can't *prove* the file was touched this session, but requiring it is
the proportionate gate — it pushes the agent through the update recipe.

## The `update-state` skill (role-agnostic)

The skill is **role-agnostic** — neutral English section guidance ("Current
focus", "Active work / status", "Known issues / blockers", plus "add role-specific
sections as relevant"). The earlier version hard-coded Russian section names and
an "(executive only)" block; that was dropped so the skill fits any worker. A
richer, role-specific state layout can be layered by a higher tier (e.g. the
company / executive plugins). The skill also instructs: keep it compact (20–50
lines, it loads into context every session), make the `<!-- last_task: -->`
comment the **first line**, and update the date.

## Dependencies & runtime

- `jq` (parsing the Stop stdin), `grep`, `head` — present in the worker image.
- cwd `/work`; POSIX `/bin/sh` (dash); blocking is `exit 2`.
- `stop_hook_active` caps blocking at one round.
- No service dependency — `state.md` has no service consumer; it persists only via
  the worker git repo commit.

## Testing

Tier-1 deterministic suite (run under dash):
`services/controllers/runner/tests/core-state.bats`. It asserts:

- Stop — no `state.md` → exit 2; `state.md` whose first line lacks
  `<!-- last_task:` → exit 2 (locks the first-line contract); valid → exit 0;
  `stop_hook_active: true` → exit 0;
- SessionStart — injects `## Agent State` + body when `state.md` is present;
  emits nothing when absent.

See the repo [README](../../README.md) for the tier/toggle model.

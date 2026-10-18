# core-summary

`tier: core` · force-attached, **toggleable** (default-on, a workspace can turn
it off).

Requires the agent to write a **structured execution summary** before the session
ends. This is the **only** core artifact the runner actually consumes: it becomes
the worker's `execution-summary` typed report in history-service. It contributes:

- the **`write-summary`** skill (the format reference, available as
  `/write-summary`);
- a blocking **`Stop`** gate that requires `task/summary.yaml`.

## `summary.yaml` → `execution-summary` report

`task/summary.yaml` is the one core artifact that crosses the worker → runner →
service boundary. The runner reads it, `cat`s it **verbatim** into the
history-service `execution-summary` typed report, and history-service indexes the
content in Qdrant for semantic search. The runner does **no field validation** —
it ships whatever the agent wrote.

## What the gate checks vs. what the conventions are

A blocking `Stop` hook (POSIX `/bin/sh`):

1. Reads the Stop payload on stdin; **exits 0 immediately when `stop_hook_active`
   is true** (one-round blocking cap).
2. **Exits 2 (blocks)** unless `task/summary.yaml` exists **and** declares
   top-level `^type:` **and** `^summary:` keys.

That presence check is the **entire** gate. Everything else the `write-summary`
skill documents — `changes` / `decisions` lists, the `files_changed` ==
len(changes) and `key_decisions` == len(decisions) count invariants, the flow
`phases` block and `phases_total` / `phases_completed` invariants — are
**conventions** the reviewer and history-service rely on, **not** gate-enforced.
The skill is honest about this: it states the gate checks `type:` + `summary:`
presence and the rest are conventions (no false "validate / malformed" claim).

Two `type` shapes:

- `type: regular` — a one-task summary (`summary`, `changes`, `decisions`,
  `files_changed`, `key_decisions`).
- `type: flow` — a multi-phase task summary (adds a `phases` list +
  `phases_total` / `phases_completed`).

## `stop_hook_active` and `blocking: true`

- The gate **honors `stop_hook_active`** — it reads stdin and exits 0 when the
  flag is true, so a re-prompted Stop never blocks again (this was the lone
  contract-violating one-liner among the six force-attach gates; it now reads
  stdin and caps at one round like its siblings).
- The `plugin.yaml` entry **keeps `blocking: true`**. It is read by
  flows-core's `graph.js` to render the Stop node in the UI DAG — dropping it
  would remove the node from the graph. The `message:` field is runtime-dead
  (Claude-native hooks surface the command's stderr, not this field) but is left
  in place for parity with the sibling system gates.

## Dependencies & runtime

- `jq` (parsing the Stop stdin), `grep`, `test` — present in the worker image.
- cwd `/work`; POSIX `/bin/sh` (dash); blocking is `exit 2`.
- `stop_hook_active` caps blocking at one round.

## Testing

Tier-1 deterministic suite (run under dash):
`services/controllers/runner/tests/core-summary.bats`. It asserts:

- Stop — missing `summary.yaml` → exit 2; present but no `^type:` → exit 2;
  present but no `^summary:` → exit 2; valid (`type:` + `summary:`) → exit 0;
  `stop_hook_active: true` → exit 0 (locks the one-round cap).

See the repo [README](../../README.md) for the tier/toggle model and the
artifact-consumption reality.

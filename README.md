# twinkle-flow-core

The **core / optional-effectiveness** tier of the Orchestra plugin catalog.
These three plugins make a worker session *more effective* — they give it
persistent memory, cross-session state, and a structured execution summary — but
the platform runs without them. They are the middle tier between
[`twinkle-flow-system`](https://github.com/artme-dev/twinkle-flow-system) (the
locked life-support layer) and the per-workspace company libraries.

```toml
# flows.toml
[package]
name = "artme-dev/twinkle-flow-core"
version = "v0.1.0"

[exports]
plugins = "*"
```

## Tier & toggle model

This repo is **force-attached globally but toggleable per plugin**. Unlike
`twinkle-flow-system` (force-attached *and locked* — non-removable, every plugin
locked-on), the core force-attach preset is **default-on but a workspace admin
can toggle any of its three plugins off**. The intent: these plugins ship on by
default for every worker, but because they are "optional effectiveness" rather
than life-support, a workspace can opt out of any of them without breaking the
platform.

| Plugin | Job | Mechanism |
|---|---|---|
| [`core-memory`](plugins/core-memory/README.md) | Persistent working memory — recall past context, save learnings | `session.mcp:[journal-mcp]` + the `recall-memory` capability + the `write-journal` skill + a SessionStart priming hook + a blocking `Stop` gate (`task/journal.md`, **degrading** when journal-mcp is unwired) |
| [`core-state`](plugins/core-state/README.md) | Cross-session agent state in `state.md` (repo root) | the `update-state` skill + a SessionStart hook that re-injects `state.md` + a blocking `Stop` gate (`state.md` present **and** carrying the `<!-- last_task: ... -->` first-line marker) |
| [`core-summary`](plugins/core-summary/README.md) | Structured execution summary for history-service | the `write-summary` skill + a blocking `Stop` gate (`task/summary.yaml` declares `type:` + `summary:`) |

## The journal-mcp dependency (core-memory)

`core-memory` is the only plugin in this tier with a service dependency: its
memory tools (`journal_write` / `journal_search` / `journal_recent`) reach the
agent only through the **journal-mcp** MCP service, written into the worker's
`.mcp.json` by the runner's `dispatch_mcp_integrations`. That dispatcher resolves
provider names against `GET mcp-registry/integrations/<ws>/resolve`.

Historically journal-mcp was silently absent on any workspace that had no
explicit, enabled `journal-mcp` integration row (seeding populated only the
`providers` table, never an `integrations` row), so the memory tools never
appeared and the journal.md gate demanded a reflection artifact with no backing
tools. The platform fix (Wave 2, §1a) makes **builtin `connection_type: internal`
providers resolve enabled-by-default**: `resolveIntegrations` now returns the
union of (explicit enabled integrations) and (builtin-internal providers **not**
suppressed by an explicit `enabled=0` integration row). journal-mcp is
`connection_type: internal`, so it is now available on every workspace — existing
and future — with no seeding migration, and a workspace can still opt out with an
explicit `enabled=0` row. External / custom providers continue to require an
explicit enabled row.

Because the tier is optional-effectiveness, `core-memory`'s `Stop` gate also
**degrades to a no-op when `.mcp.json` has no journal-mcp server** — it never
blocks a session on a reflection artifact whose backing tools are absent.

## Artifact-consumption reality

Only one core artifact crosses the worker → runner → service boundary:

- **`task/summary.yaml`** is read by the runner. It is `cat`'d **verbatim** into
  the history-service `execution-summary` typed report (indexed in Qdrant for
  semantic search). The runner does **no field validation** — the rich fields and
  count invariants in the `write-summary` skill are conventions the reviewer and
  history-service rely on, not gate-enforced rules.
- **`task/journal.md`** has **zero service consumers**. It is a throwaway
  in-session reflection marker; the real memory persistence is the `journal_write`
  MCP tool, which writes into journal-mcp's vector store. journal.md exists only so
  the `Stop` gate has an observable "did you reflect?" signal.
- **`state.md`** (repo root) has zero service consumers either, but it **persists
  across sessions**: it is committed to the worker git repo by the system-tier
  `git_guard` / autocommit and re-injected at the next SessionStart by core-state.

## Testing

The two-tier test template is inherited from the
[`twinkle-flow-system`](https://github.com/artme-dev/twinkle-flow-system) repo
(the gold-standard pattern).

### Tier 1 — deterministic hook-command harness (CI-runnable)

Each plugin's hook `command` is lifted straight out of `plugin.yaml` and run
under **dash** (matching the worker image's `/bin/sh`) against crafted temp
`task/` fixtures, asserting exit code (0 = pass-through, 2 = blocking) +
stderr/stdout. Suites live in the twinkle monorepo:

- `services/controllers/runner/tests/core-memory.bats`
- `services/controllers/runner/tests/core-state.bats`
- `services/controllers/runner/tests/core-summary.bats`

They reuse `plugin-hook-harness.bash`, resolving these plugin sources via the
`SYSTEM_PLUGINS_DIR` knob repointed at this repo's `plugins/`.

The cross-plugin aggregation **capstone** —
`services/controllers/runner/tests/stop-aggregation.bats` — is a docker-based
Tier-1 test that pins the claude-code multi-Stop aggregation behavior against
worker-image bumps (see the
[stop-hook contract](https://github.com/artme-dev/twinkle-flow-system/blob/main/docs/stop-hook-contract.md)).

### Tier 2 — live sandbox lane (integration confidence)

`tests/scenarios/harness/core-plugins-live.test.mjs` (cloned from
`system-plugins-live.test.mjs`): the headline check is the journal
**write → recall round-trip** (Task 1 calls `journal_write` with a unique marker;
Task 2 calls `journal_search` for it and the marker round-trips, gateway-
observable), plus the 6-gate coexistence assertion (mid-run `.hooks.Stop` ≥ 6,
all gates satisfiable in one clean session).

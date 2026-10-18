# core-memory

`tier: core` · force-attached, **toggleable** (default-on, a workspace can turn
it off).

Gives a worker **persistent working memory**: it can recall decisions, outcomes,
feedback, and observations from past sessions, and save new learnings at session
end. The memory itself lives in the **journal-mcp** vector store, scoped per
worker — it survives across sessions, unlike anything in the worker's `task/`
directory. The plugin contributes:

- `session.mcp: [journal-mcp]` — declares the journal-mcp dependency so the runner
  wires its tools into the worker's `.mcp.json`;
- the **`recall-memory`** capability (the recall path, compiled into SKILL.md
  and available as `/recall-memory`);
- the **`write-journal`** skill (the save path, available as `/write-journal`);
- a **SessionStart** priming hook (static, curl-free);
- a blocking **`Stop`** gate that requires `task/journal.md` — **degrading** to a
  no-op when journal-mcp is not wired.

## MCP tools & scope

The three memory tools come from journal-mcp:

| Tool | Use |
|---|---|
| `journal_write` | Persist a learning. Requires `type` ∈ `{decision, outcome, feedback, observation}`, `content`, `context`, and `tags`. |
| `journal_search` | Semantic search over past entries (optionally filtered by type / tags / date). |
| `journal_recent` | Review the most recent entries. |

Entries are scoped by **role = `workspace:worker`** — each worker gets its own
private memory, isolated from other workers in the workspace.

## Recall — the `recall-memory` capability

Recall is **agent-driven**, not automatic. The earlier design auto-recalled at
SessionStart by `curl`ing journal-mcp directly; that hook was **deleted** — it
hit a route journal-mcp does not expose (`POST /api/search` → 404, silently
swallowed by `|| exit 0`), used the wrong response shape and scope key, and
relied on env vars the worker container never had. In its place:

- The **`recall-memory`** capability instructs the agent to call `journal_search`
  (with at least one query, broadening if sparse) and apply the retrieved context
  to the current task. Trigger it via `/recall-memory`, or it activates when the
  agent recognizes it needs prior context.
- A **static SessionStart hook** echoes a curl-free priming block —
  `## Working Memory` plus a note to use `/recall-memory` and `/write-journal` —
  so the agent knows the capability exists.

## Save — the `write-journal` skill

At session end the agent reflects: it reviews recent memory (`journal_recent`),
identifies significant `decision` / `outcome` / `observation` / `feedback`
events, calls `journal_write` for each, and then writes a short
`task/journal.md` summarizing what it recorded. Not every session produces
entries — that is normal — but the skill always writes `task/journal.md` (even an
"no significant entries" note).

If the `journal_*` tools are **unavailable** in a session, the skill still writes
`task/journal.md`, noting memory was unavailable, and skips the tool calls.

## The journal.md Stop gate (and its degradation)

A blocking `Stop` hook (POSIX `/bin/sh`):

1. Reads the Stop payload on stdin; **exits 0 immediately when `stop_hook_active`
   is true** (one-round blocking cap).
2. **`grep -q 'journal-mcp' .mcp.json || exit 0`** — if journal-mcp is **not**
   wired into `.mcp.json`, the gate **no-ops**. This is the optional-effectiveness
   degradation: the gate never demands a reflection artifact whose backing tools
   are absent.
3. Otherwise, **exits 2 (blocks)** if `task/journal.md` is missing, pointing the
   agent at `/write-journal`.

`task/journal.md` is an **in-session reflection marker only** — it has no service
consumer. The real persistence is the `journal_write` MCP tool; journal.md just
gives the gate something observable to enforce.

## Dependency — journal-mcp must be resolvable

`journal_*` tools reach the agent only via `.mcp.json`, written by the runner's
`dispatch_mcp_integrations` from `GET mcp-registry/integrations/<ws>/resolve`.
journal-mcp is a builtin **`connection_type: internal`** provider, and as of the
Wave 2 resolve-default fix (§1a), `resolveIntegrations` returns builtin-internal
providers **enabled-by-default** — the union of explicit enabled integrations and
builtin-internal providers not suppressed by an explicit `enabled=0` row. So
journal-mcp is now available on every workspace, existing and future, with no
seeding migration; a workspace opts out with an explicit `enabled=0` integration
row. When it is unwired (opted-out), the gate degrades per above and the
write/recall skills note tools are unavailable.

## Dependencies & runtime

- `jq` (parsing the Stop stdin), `grep` — present in the worker image.
- journal-mcp (for the tools; the gate degrades gracefully without it).
- cwd `/work`; POSIX `/bin/sh` (dash); blocking is `exit 2`.
- `stop_hook_active` caps blocking at one round.

## Testing

Tier-1 deterministic suite (run under dash):
`services/controllers/runner/tests/core-memory.bats`. It asserts:

- `journal.md` present + journal-mcp in `.mcp.json` → exit 0;
- `journal.md` absent + journal-mcp in `.mcp.json` → exit 2;
- `journal.md` absent + **no** journal-mcp in `.mcp.json` → exit 0 (the
  degradation);
- `stop_hook_active: true` → exit 0;
- the dead `/api/search` SessionStart curl is **gone**.

Tier-2 live (`tests/scenarios/harness/core-plugins-live.test.mjs`): the journal
**write → recall round-trip** — a unique marker written via `journal_write` in
one task is found via `journal_search` in the next.

See the repo [README](../../README.md) for the tier/toggle and journal-mcp model.

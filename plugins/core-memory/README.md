# core-memory

`tier: core` · force-attached, **toggleable** (default-on, a workspace can turn
it off).

Gives a worker **persistent working memory**: it can recall decisions, outcomes,
feedback, and observations from past sessions, and save new learnings at session
end. The memory itself lives in the **journal-mcp** vector store, scoped per
worker — it survives across sessions, unlike anything in the worker's `task/`
directory. The plugin contributes:

- `session.mcp: [journal-mcp]` — declares the journal-mcp dependency so the runner registers it for the session (`--mcp-config` on a runner-native-config runner, the repository's `.mcp.json` on an older one);
- the **`recall-memory`** capability (the recall path, compiled into SKILL.md,
  `/core-memory:recall-memory`);
- the **`write-journal`** skill (the save path, `/core-memory:write-journal`);
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
  to the current task. Trigger it via `/core-memory:recall-memory`, or it activates when the
  agent recognizes it needs prior context.
- A **static SessionStart hook** echoes a curl-free priming block —
  `## Working Memory` plus a note naming both skills —
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

1. Reads the Stop payload from stdin with `jq` directly; **exits 0
   immediately when `stop_hook_active` is true** (one-round blocking cap).
2. **Is journal-mcp wired?** On a runner-native-config runner the answer is
   `TWINKLE_MCP_SERVERS` (a comma list the runner always sets, possibly
   empty): the gate arms only when `journal-mcp` is one of its items. A
   repository's own `.mcp.json` is never consulted there. On an older runner
   (`TWINKLE_MCP_SERVERS` unset) it is
   `grep -q 'journal-mcp' "$CLAUDE_PROJECT_DIR/.mcp.json"`. Either way an
   unwired session **no-ops** — the gate never demands a reflection artifact
   whose backing tools are absent.
3. Otherwise, **exits 2 (blocks)** if `journal.md` is missing from the task
   directory, naming its absolute path and `/core-memory:write-journal`.

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
- Paths: the task directory is `${TWINKLE_TASK_DIR:-$CLAUDE_PROJECT_DIR/task}`; nothing depends on the cwd. POSIX `/bin/sh` (dash); blocking is `exit 2`.
- `stop_hook_active` caps blocking at one round.

## Testing

Tier-1 deterministic suite (run under dash):
`services/controllers/runner/tests/core-memory.bats`. It asserts:

- `journal.md` present + journal-mcp in `.mcp.json` → exit 0;
- `journal.md` absent + journal-mcp in `.mcp.json` → exit 2;
- `journal.md` absent + **no** journal-mcp in `.mcp.json` → exit 0 (the
  degradation);
- `stop_hook_active: true` → exit 0;
- the dead `/api/search` SessionStart curl is **gone**;
- runner-native-config: `TWINKLE_MCP_SERVERS` containing `journal-mcp` arms
  the gate; without it — empty included — the gate is a no-op even when the
  repository's `.mcp.json` names journal-mcp; a name that merely contains
  `journal-mcp` does not arm it;
- an older runner: `.mcp.json` is read at `$CLAUDE_PROJECT_DIR`, not the cwd;
- a Stop payload carrying a JSON escape keeps `stop_hook_active`.

Tier-2 live (`tests/scenarios/harness/core-plugins-live.test.mjs`): the journal
**write → recall round-trip** — a unique marker written via `journal_write` in
one task is found via `journal_search` in the next.

See the repo [README](../../README.md) for the tier/toggle and journal-mcp model.

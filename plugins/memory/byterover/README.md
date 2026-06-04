# ByteRover memory plugin (mono)

Persistent memory for Hermes via [byterover-mono](https://github.com/campfirein/byterover-mono).
Replaces the cli-binary backend (v1.x) — no more `brv` binary, no more
`brv curate` session protocol; this build calls mono's bundled `.mjs`
scripts (`recall.mjs`, `record.mjs`, `brv.mjs`) as one-shot subprocesses.

## How it integrates

| Hermes hook | What this plugin does |
|---|---|
| `system_prompt_block()` | Returns the full curate guidance (~8.5KB) every turn — IRON LAW + 19-element `<bv-*>` vocabulary + structural rules + the `brv_record` tool contract |
| `prefetch(query)` | Runs `node recall.mjs "<query>" --cwd $HERMES_HOME/byterover/ --limit 5`. Returns a `<byterover-context>` block with the matched topics' rendered HTML, or empty if nothing relevant. |
| `brv_record` tool | Agent calls this with `{path, html, overwrite?}`. The plugin shells `node record.mjs <path> --html '<bv-topic …>…</bv-topic>'`. record.mjs is one-shot — no kickoff/continuation session. |

**Storage location.** Hermes runs every subprocess from `$HERMES_HOME/byterover/`.
Mono's centralized layout resolves that cwd to
`~/.brv/projects/<flat-of-cwd>/context-tree/`. So memory persists at
e.g. `~/.brv/projects/-Users-you-.hermes-byterover/context-tree/`, NOT
inside the Hermes profile dir.

## Install

You need:

1. **Node.js** on `PATH` (any modern version).
2. **The byterover-mono `scripts/` directory** somewhere on disk. The
   plugin auto-discovers it in this order:

   1. `$BYTEROVER_MONO_SCRIPTS_DIR` — explicit override.
   2. `~/.openclaw/skills/byterover/scripts/` — if you have the byterover
      skill installed for OpenClaw, this is the standard location.
   3. `~/workspaces/byterover-mono/skills/byterover/scripts/` — dev
      fallback for byterover-mono checkouts.

If you have neither openclaw nor a byterover-mono checkout:

```bash
git clone https://github.com/campfirein/byterover-mono.git ~/workspaces/byterover-mono
cd ~/workspaces/byterover-mono
pnpm install && pnpm build:skill
```

That produces `~/workspaces/byterover-mono/skills/byterover/scripts/`,
which the plugin finds at the dev-fallback path.

To use a custom location:

```bash
export BYTEROVER_MONO_SCRIPTS_DIR=/path/to/your/scripts
```

## What the agent sees

Every turn, Hermes injects:

1. **`system_prompt_block`** — the curate guidance. Tells the agent when
   to curate, the full `<bv-*>` vocabulary, the topic structure rules,
   and the `brv_record` tool contract.
2. **`prefetch` output** (when there's a query) — a `<byterover-context>`
   block with directive instructions and the retrieved topics as raw
   `<bv-topic>` HTML.

The agent decides whether to call `brv_record` after producing its
answer. There's no auto-curate, no `on_pre_compress` flush, no
`sync_turn` background job — Hermes' previous auto-curate hooks were
guessing what to save from raw message text; the agent picks better.

## Tool: `brv_record`

| Arg | Required | Description |
|---|---|---|
| `path` | yes | Slash-separated snake_case, no `.html` (e.g. `security/auth`) |
| `html` | yes | Bare `<bv-topic>...</bv-topic>` HTML document |
| `overwrite` | no | Replace existing topic. Default `false`. Use ONLY after reading + merging prior facts. |

Returns the JSON envelope from `record.mjs` verbatim:

```json
{ "ok": true, "data": { "created": true, "filePath": "...", "warnings": [] } }
```

On `ok: false`, the `error` field carries a human-readable message the
agent should surface.

## Migration from the cli-binary version

| v1.x (cli) | v2.0.0-mono.0 (this) |
|---|---|
| `brv` binary on PATH | `node` + mono `scripts/` dir |
| `brv_query` agent tool | recall is automatic via `prefetch()`; tool removed |
| `brv_curate` agent tool (natural-language content) | `brv_record` agent tool (structured `<bv-topic>` HTML) |
| `brv_status` tool | removed (no analog) |
| Per-turn auto-curate via `sync_turn` / `on_pre_compress` | removed; agent invokes `brv_record` directly |
| `BRV_API_KEY` env var | removed (no remote auth) |

If you have an existing tree at `$HERMES_HOME/byterover/.brv/context-tree/`
(cli layout), the mono build will NOT read it — mono's storage lives at
`~/.brv/projects/<flat>/context-tree/`. Migrate by re-recording the topics
into the new location, or stay on the v1.x cli plugin until you've migrated.

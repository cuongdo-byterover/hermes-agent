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

**Storage workspace.** Hermes runs every mono subprocess from
`$HERMES_HOME/byterover/`. ByteRover resolves the context tree from that cwd
through its space registry; the actual tree lives under the ByteRover data dir,
not under the Hermes plugin directory. If no space is bound yet, create one in
the ByteRover desktop app, then bind `$HERMES_HOME/byterover/` with
`space.mjs bind`.

## Requirements

1. **Node.js** on `PATH` (any modern version).
2. **The assembled ByteRover skill** at `$HERMES_HOME/skills/byterover/`.
   Hermes expects this final layout:

```text
$HERMES_HOME/skills/byterover/
├── SKILL.md
└── scripts/
    ├── recall.mjs
    ├── record.mjs
    └── brv.mjs
```

## Install the skill

The Hermes plugin does not consume `apps/skill` directly. In
`byterover-mono`, `apps/skill` is the private source app: hand-authored docs
live in `apps/skill/skill/`, runtime entry points live in
`apps/skill/src/entries/`, and `pnpm build:skill` assembles the generated,
installable artifact at `skills/byterover/`.

Released builds are published to the public `campfirein/skills` repo for
`skills.sh` consumers:

```bash
# latest released skill
npx skills add campfirein/skills

# pinned release
npx skills add campfirein/skills@skill-vX.Y.Z
```

Whichever install path you use, the final directory must be available to
Hermes as `$HERMES_HOME/skills/byterover/`. For a source checkout:

```bash
git clone https://github.com/campfirein/byterover-mono.git ~/workspaces/byterover-mono
cd ~/workspaces/byterover-mono
pnpm install && pnpm build:skill

mkdir -p "${HERMES_HOME:-$HOME/.hermes}/skills"
ln -s "$PWD/skills/byterover" \
  "${HERMES_HOME:-$HOME/.hermes}/skills/byterover"
```

Use a copy instead of a symlink if you want a fixed snapshot. The source app
can also publish the same generated artifact with `pnpm publish:skill`, which
mirrors `skills/byterover/` into `campfirein/skills` and tags it as
`skill-vX.Y.Z`.

## Setup

```bash
hermes memory setup    # select "byterover"
```

Or manually:

```bash
hermes config set memory.provider byterover
```

If ByteRover reports that no context tree is bound for the Hermes workspace,
bind the cwd used by this plugin:

```bash
mkdir -p "${HERMES_HOME:-$HOME/.hermes}/byterover"
cd "${HERMES_HOME:-$HOME/.hermes}/byterover"
node "${HERMES_HOME:-$HOME/.hermes}/skills/byterover/scripts/space.mjs" bind "Hermes"
```

Spaces are provisioned in the ByteRover desktop app; `space.mjs bind` links
this Hermes workspace to one of those spaces.

## Config

| Env Var | Required | Description |
|---------|----------|-------------|
| `BYTEROVER_MONO_SCRIPTS_DIR` | No | Override the scripts directory. Use only when the skill is not installed at `$HERMES_HOME/skills/byterover/scripts`. |
| `BRV_DATA_DIR` | No | Override ByteRover's data directory; normally leave unset and let the bundled scripts resolve it. |

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
(cli layout), the mono build will NOT read it automatically. Mono resolves the
active tree through the workspace's bound ByteRover space under the ByteRover
data dir. Migrate by re-recording the topics into the bound space, or stay on
the v1.x cli plugin until you've migrated.

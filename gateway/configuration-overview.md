---
domain: gateway
topic: "Gateway Configuration Overview: openclaw.json Editing and Common Tasks"
type: procedure
keywords:
  - openclaw.json
  - configuration
  - config set
  - config get
  - gateway config
  - JSON5
  - openclaw configure
related:
  - gateway/configuration-reference
  - gateway/configuration-examples
  - gateway/secrets
source: gateway/configuration.md
---

OpenClaw reads an optional JSON5 config from `~/.openclaw/openclaw.json`. If missing, OpenClaw uses safe defaults. Point `OPENCLAW_CONFIG_PATH` at a different path if needed (symlinked configs are not supported for atomic writes).

## Minimal Configuration

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Editing Config

**Interactive wizard:**
```bash
openclaw onboard       # full onboarding flow
openclaw configure     # config wizard
```

**CLI one-liners:**
```bash
openclaw config get agents.defaults.workspace
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
```

**Control UI:** Open http://127.0.0.1:18789 and use the **Config** tab.

**Direct file edit:** Open `~/.openclaw/openclaw.json` in any editor (JSON5: supports comments, trailing commas).

## Common Configuration Tasks

- **Set provider/model**: `agents.defaults.model`
- **Restrict channel access**: `channels.<name>.allowFrom`  
- **Set workspace path**: `agents.defaults.workspace`
- **Enable heartbeat**: `agents.defaults.heartbeat.every`
- **Enable cron**: `cron.jobs`
- **Set sandbox mode**: `agents.defaults.sandbox`
- **Configure MCP servers**: `mcp.servers`

OpenClaw reads an optional **JSON5** config from `~/.openclaw/openclaw.json`.
The active config path must be a regular file. Symlinked `openclaw.json`
layouts are unsupported for OpenClaw-owned writes; an atomic write may replace
the path instead of preserving the symlink. If you keep config outside the
default state directory, point `OPENCLAW_CONFIG_PATH` directly at the real file.

If the file is missing, OpenClaw uses safe defaults. Common reasons to add a config:

- Connect channels and control who can message the bot
- Set models, tools, sandboxing, or automation (cron, hooks)
- Tune sessions, media, networking, or UI

See the [full reference](/gateway/configuration-reference) for every available field.

Agents and automation should use `config.schema.lookup` for exact field-level
docs before editing config. Use this page for task-oriented guidance and
[Configuration reference](/gateway/configuration-reference) for the broader
field map and defaults.

**New to configuration?** Start with `openclaw onboard` for interactive setup, or check out the [Configuration Examples](/gateway/configuration-examples) guide for complete copy-paste configs.

## Minimal config

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Editing config

    ```bash
    openclaw onboard       # full onboarding flow
    openclaw configure     # config wizard
    ```

    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```

    Open [http://127.0.0.1:18789](http://127.0.0.1:18789) and use the **Config** tab.
    The Control UI renders a form from the live config schema, including field
    `title` / `description` docs metadata plus plugin and channel schemas when
    available, with a **Raw JSON** editor as an escape hatch. For drill-down
    UIs and other tooling, the gateway also exposes `config.schema.lookup` to
    fetch one path-scoped schema node plus immediate child summaries.

    Edit `~/.openclaw/openclaw.json` directly. The Gateway watches the file and applies changes automatically (see [hot reload](#config-hot-reload)).

## Strict validation

OpenClaw only accepts configurations that fully match the schema. Unknown keys, malformed types, or invalid values cause the Gateway to **refuse to start**. The only root-level exception is `$schema` (string), so editors can attach JSON Schema metadata.

`openclaw config schema` prints the canonical JSON Schema used by Control UI
and validation. `config.schema.lookup` fetches a single path-scoped node plus
child summaries for drill-down tooling. Field `title`/`description` docs metadata
carries through nested objects, wildcard (`*`), array-item (`[]`), and `anyOf`/
`oneOf`/`allOf` branches. Runtime plugin and channel schemas merge in when the
manifest registry is loaded.

When validation fails:

- The Gateway does not boot
- Only diagnostic commands work (`openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`)
- Run `openclaw doctor` to see exact issues
- Run `openclaw doctor --fix` (or `--yes`) to apply repairs

The Gateway keeps a trusted last-known-good copy after each successful startup.
If `openclaw.json` later fails validation (or drops `gateway.mode`, shrinks
sharply, or has a stray log line prepended), OpenClaw preserves the broken file
as `.clobbered.*`, restores the last-known-good copy, and logs the recovery
reason. The next agent turn also receives a system-event warning so the main
agent does not blindly rewrite the restored config. Promotion to last-known-good
is skipped when a candidate contains redacted secret placeholders such as `***`.
When every validation issue is scoped to `plugins.entries.<id>...`, OpenClaw
does not perform whole-file recovery. It keeps the current config active and
surfaces the plu

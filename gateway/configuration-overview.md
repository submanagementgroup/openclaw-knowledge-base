---
domain: gateway
topic: "Gateway Configuration Overview"
type: concept
keywords:
  - openclaw.json
  - config
  - JSON5
  - openclaw configure
  - openclaw onboard
  - hot reload
  - config schema
  - OPENCLAW_CONFIG_PATH
source: gateway/configuration.md
related:
  - gateway/configuration-reference
  - gateway/configuration-examples
---

OpenClaw reads an optional JSON5 config from `~/.openclaw/openclaw.json`. If missing, safe defaults apply. Set `OPENCLAW_CONFIG_PATH` to use a different path.

## Minimal Config

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Editing Config

Interactive: `openclaw onboard` or `openclaw configure`

CLI one-liners:
```bash
openclaw config get agents.defaults.workspace
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
```

Direct edit: `~/.openclaw/openclaw.json` — the Gateway watches the file and hot-reloads changes.

Control UI: `http://127.0.0.1:18789` — Config tab renders a form from live config schema with Raw JSON escape hatch.

## Strict Validation

OpenClaw only accepts configurations that fully match the schema. Unknown keys, malformed types, or invalid values prevent Gateway startup. The only root-level exception is `$schema`.

When validation fails, only diagnostic commands work: `openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`. Run `openclaw doctor --fix` to apply repairs.

## Config Hot Reload

The Gateway watches `openclaw.json` and applies changes automatically. A trusted last-known-good copy is kept after each successful startup, but hot reload does not restore it automatically if a new reload fails.

## Top-Level Config Sections

| Section | Purpose |
|---------|---------|
| `agents` | Agent defaults, workspace, model, tools |
| `channels` | Messaging integrations (WhatsApp, Telegram, etc.) |
| `models` | Provider definitions and model registry |
| `plugins` | Plugin entries, slots, hooks |
| `gateway` | Network, port, auth, TLS |
| `memory` | Memory backend selection |
| `automation` | Cron jobs and triggers |

See [Configuration Reference](/gateway/configuration-reference) for every field with defaults.

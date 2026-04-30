---
domain: cli
topic: "openclaw configure: Interactive Configuration Wizard for Credentials and Settings"
type: procedure
keywords:
  - openclaw configure
  - interactive configuration
  - configure credentials
  - configure web search
  - configure gateway
  - configure channels
  - configure model
  - configure plugins
  - configure skills
  - configure daemon
source: cli/configure.md
related:
  - cli/cli-overview
  - cli/config-cli
  - gateway/configuration-overview
---

`openclaw configure` runs an interactive prompt to set up credentials, model preferences, devices, and agent defaults. It is equivalent to `openclaw config` with no subcommand.

## Usage

```bash
openclaw configure
openclaw configure --section web
openclaw configure --section model --section channels
openclaw configure --section gateway --section daemon
```

## Available Sections

| Section | What It Configures |
|---------|-------------------|
| `workspace` | Agent workspace directory |
| `model` | Model provider auth, allowlist, and default model |
| `web` | Web search provider and credentials |
| `gateway` | Gateway port, bind address, and auth mode |
| `daemon` | Background service installation |
| `channels` | Chat channel setup (Slack, Discord, WhatsApp, etc.) |
| `plugins` | Plugin installation and config |
| `skills` | Skills setup and API keys |
| `health` | Health check configuration |

## Model Section Notes

The **Model** section includes a multi-select for the `agents.defaults.models` allowlist. Provider-scoped setup choices merge selected models into the existing allowlist. Re-running provider auth preserves existing `agents.defaults.model.primary` — use `openclaw models set <model>` or `openclaw models auth login --set-default` to intentionally change the default.

For paired providers such as Volcengine and BytePlus, the same preference also matches their coding-plan variants (`volcengine-plan/*`, `byteplus-plan/*`).

## Web Search Section Notes

`openclaw configure --section web` lets you choose a search provider and configure credentials. Some providers prompt for additional options:

- **Grok**: can offer optional `x_search` setup with the same `XAI_API_KEY`.
- **Kimi**: can ask for Moonshot API region (`api.moonshot.ai` vs `api.moonshot.cn`) and default Kimi web-search model.

## Daemon Install Notes

If you run the daemon install step with token auth and `gateway.auth.token` is SecretRef-managed, configure validates the SecretRef but does not persist resolved plaintext values into supervisor environment metadata. If both `gateway.auth.token` and `gateway.auth.password` are configured with `gateway.auth.mode` unset, configure blocks daemon install until mode is set.

## Related

- [Config CLI](/cli/config-cli)
- [Gateway configuration](/gateway/configuration-overview)

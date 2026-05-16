---
domain: plugins
topic: "Codex Harness Plugin"
type: concept
keywords:
  - Codex
  - codex harness
  - Codex plugin
  - OpenAI Codex
  - codex-cli
  - agent harness
source: plugins/codex-harness.md
---

The bundled `codex` plugin lets OpenClaw run embedded OpenAI agent turns
through Codex app-server instead of the built-in PI harness.

Use the Codex harness when you want Codex to own the low-level agent session:
native thread resume, native tool continuation, native compaction, and
app-server execution. OpenClaw still owns chat channels, session files, model
selection, OpenClaw dynamic tools, approvals, media delivery, and the visible
transcript mirror.

The normal setup uses canonical OpenAI model refs such as `openai/gpt-5.5`.
Do not configure `openai-codex/gpt-*` model refs. Put OpenAI agent auth order
under `auth.order.openai`; older `openai-codex:*` profiles and
`auth.order.openai-codex` entries remain supported for existing installs.

OpenClaw starts Codex app-server threads with Codex native code mode and
code-mode-only enabled. That keeps deferred/searchable OpenClaw dynamic tools
inside Codex's own code execution and tool-search surface instead of adding a
PI-style tool-search wrapper on top of Codex.

For the broader model/provider/runtime split, start with
[Agent runtimes](/concepts/agent-runtimes). The short version is:
`openai/gpt-5.5` is the model ref, `codex` is the runtime, and Telegram,
Discord, Slack, or another channel remains the communication surface.

## Requirements

- OpenClaw with the bundled `codex` plugin available.
- If your config uses `plugins.allow`, include `codex`.
- Codex app-server `0.125.0` or newer. The bundled plugin manages a compatible
  Codex app-server binary by default, so local `codex` commands on `PATH` do not
  affect normal harness startup.
- Codex auth available through `openclaw models auth login --provider openai-codex`,
  an app-server account in the agent's Codex home, or an explicit Codex API-key
  auth profile.

For auth precedence, environment isolation, custom app-server commands, model
discovery, and all config fields, see
[Codex harness reference](/plugins/codex-harness-reference).

## Quickstart

Most users who want Codex in OpenClaw want this path: sign in with a
ChatGPT/Codex subscription, enable the bundled `codex` plugin, and use a
canonical `openai/gpt-*` model ref.

Sign in with Codex OAuth:

```bash
openclaw models auth login --provider openai-codex
```

Enable the bundled `codex` plugin and select an OpenAI agent model:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.5",
    },
  },
}
```

If your config uses `plugins.allow`, add `codex` there too:

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Restart the gateway after changing plugin config. If an existing chat already
has a session, use `/new` or `/reset` before testing runtime changes so the next
turn resolves the harness from current config.

## Configuration

The quickstart config is the minimum viable Codex harness config. Set Codex
harness options in OpenClaw config, and use the CLI only for Codex auth:

| Need                                   | Set                                                                              | Where                              |
| -------------------------------------- | -------------------------------------------------------------------------------- | ---------------------------------- |
| Enable the harness                     | `plugins.entries.codex.enabled: true`                                            | OpenClaw config                    |
| Keep an allowlisted plugin install     | Include `codex` in `plugins.allow`                                               | OpenClaw config                    |
| Route OpenAI agent turns through Codex | `agents.defaults.model` or `agents.list[].model` as `openai/gpt-*`               | OpenClaw agent config              |
| Sign in with Codex OAuth               | `openclaw models auth login --provider openai-codex`                             | CLI auth profile                   |
| Add API-key backup for Codex runs      | `openai:*` API-key profile listed after subscription auth in `auth.order.openai` | CLI auth profile + OpenClaw config |
| Fail closed when Codex is unavailable  | Provider or model `agentRuntime.id: "codex"`                                     | OpenClaw model/provider config     |
| Use direct OpenAI API traffic          | Provider or model `agentRuntime.id: "pi"` with normal OpenAI auth                | OpenClaw model/provider config     |
| Tune app-server behavior               | `plugins.entries.codex.config.appServer.*`                                       | Codex plugin config                |
| Enable native Codex plugin apps        | `plugins.entries.codex.config.codexPlugins.*`                                    | Codex plugin config                |
| Enable Codex Computer Use              | `plugins.entries.codex.config.computerUse.*`                                     | Codex plugin config                |

Use `openai/gpt-*` model refs for Codex-backed OpenAI agent turns. Prefer
`auth.order.openai` for subscription-first/API-key-backup ordering. Existing
`openai-codex:*` auth profiles and `auth.order.openai-codex` remain valid, but
do not write new `openai-codex/gpt-*` model refs.

```json5
{
  auth: {
    order: {
      openai: ["openai-codex:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

In that shape, both profiles still run through Codex for `openai/gpt-*` agent
turns. The API key is only an auth fallback, not a request to switch to PI or
plain OpenAI Responses.

The rest of this page covers common variants users must choose between:
deployment shape, fail-closed routing, guardian approval policy, native Codex
plugins, and Computer Use. For full option lists, defaults, enums, discovery,
environment isolation, timeouts, and app-server transport fields, see
[Codex harness reference](/plugins/codex-harness-reference).

## Verify Codex runtime

Use `/status` in the chat where you expect Codex. A Codex-backed OpenAI agent
turn shows:

```text
Runtime: OpenAI Codex
```

Then check Codex app-server state:

```text
/codex status
/codex models
```

`/codex status` reports app-server connectivity, account, rate limits, MCP
servers, and skills. `/codex models` lists the live Codex app-server catalog for
the harness and account. If `/status` is surprising, see
[Troubleshooting](#troubleshooting).

## Routing and model selection

Keep provider refs and runtime policy separate:

- Use `openai/gpt-*` for OpenAI agent turns through Codex.
- Do not use `openai-codex/gpt-*` in config. Run `openclaw doctor --fix` to
  repair legacy refs and stale session route pins.
- `agentRuntime.id: "codex"` is optional for normal OpenAI auto mode, but useful
  when a deployment should fail closed if Codex is unavailable.
- `agentRuntime.id: "pi"` opts a provider or model into direct PI behavior when
  that is intentional.
- `/codex ...` controls native Codex app-server conversations from chat.
- ACP/acpx is a separate external harness path. Use it only when the user asks
  for ACP/acpx or an external harness adapter.

Common command routing:

| User intent                     | Use                                     |
| ------------------------------- | --------------------------------------- |
| Attach the current chat         | `/codex bind [--cwd <path>]`            |
| Resume an existing Codex thread | `/codex resume <thread-id>`             |
| List or filter Codex threads    | `/codex threads [filter]`               |
| Send Codex feedback only        | `/codex diagnostics [note]`             |
| Start an ACP/acpx task          | ACP/acpx session commands, not `/codex` |

| Use case                                             | Configure                                                        | Verify                                  | Notes                              |
| ---------------------------------------------------- | ---------------------------------------------------------------- | --------------------------------------- | ---------------------------------- |
| ChatGPT/Codex subscription with native Codex runtime | `openai/gpt-*` plus enabled `codex` plugin                       | `/status` shows `Runtime: OpenAI Codex` | Recommended path                   |
| Fail closed if Codex is unavailable                  | Provider or model `agentRuntime.id: "codex"`                     | Turn fails instead of PI fallback       | Use for Codex-only deployments     |
| Direct OpenAI API-key traffic through PI             | Provider or model `agentRuntime.id: "pi"` and normal OpenAI auth | `/status` shows PI runtime              | Use only when PI is intentional    |
| Legacy config                                        | `openai-codex/gpt-*`                                             | `openclaw doctor --fix` rewrites it     | Do not write new config this way   |
| ACP/acpx Codex adapter                               | ACP `sessions_spawn({ runtime: "acp" })`                         | ACP task/session status                 | Separate from native Codex harness |

`agents.defaults.imageModel` follows the same prefix split. Use `openai/gpt-*`
for the normal OpenAI route and `codex/gpt-*` only when image understanding
should run through a bounded Codex app-server turn. Do not use
`openai-codex/gpt-*`; doctor rewrites that legacy prefix to `openai/gpt-*`.

## Deployment patterns

### Basic Codex deployment

Use the quickstart config when all OpenAI agent turns should use Codex by
default.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.5",
    },
  },
}
```

### Mixed provider deployment

This shape keeps Claude as the default agent and adds a named Codex agent:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.5",
      },
    ],
  },
}
```

With this config, the `main` agent uses its normal provider path and the
`codex` agent uses Codex app-server.

### Fail-closed Codex deployment

For OpenAI agent turns, `openai/gpt-*` already resolves to Codex when the
bundled plugin is available. Add explicit runtime policy when you want a written
fail-closed rule:

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: {
          id: "codex",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.5",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

With Codex forced, OpenClaw fails early if the Codex plugin is disabled, the
app-server is too old, or the app-server cannot start.

## App-server policy

By default, the plugin starts OpenClaw's managed Codex binary locally with stdio
transport. Set `appServer.command` only when you intentionally want to run a
different executable. Use WebSocket transport only when an app-server is already
running elsewhere:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
          },
        },
      },
    },
  },
}
```

Local stdio app-server sessions default to the trusted local operator posture:
`approvalPolicy: "never"`, `approvalsReviewer: "user"`, and
`sandbox: "danger-full-access"`. If local Codex requirements disallow that
implicit YOLO posture, OpenClaw selects allowed guardian permissions instead.
When an OpenClaw sandbox is active for the session, OpenClaw narrows Codex
`danger-full-access` to Codex `workspace-write` so native Codex code-mode turns
stay inside the sandboxed workspace.

Use guardian mode when you want Codex native auto-review before sandbox escapes
or extra permissions:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Guardian mode expands to Codex app-server approvals, usually
`approvalPolicy: "on-request"`, `approvalsReviewer: "auto_review"`, and
`sandbox: "workspace-write"` when the local requirements allow those values.

For every app-server field, auth order, environment isolation, discovery, and
timeout behavior, see [Codex harness reference](/plugins/codex-harness-reference).

## Commands and diagnostics

The bundled plugin registers `/codex` as a slash command on any channel that
supports OpenClaw text commands.

Common forms:

- `/codex status` checks app-server connectivity, models, account, rate limits,
  MCP servers, and skills.
- `/codex models` lists live Codex app-server models.
- `/codex threads [filter]` lists recent Codex app-server threads.
- `/codex resume <thread-id>` attaches the current OpenClaw session to an
  existing Codex thread.
- `/codex compact` asks Codex app-server to compact the attached thread.
- `/codex review` starts Codex native review for the attached thread.
- `/codex diagnostics [note]` asks before sending Codex feedback for the
  attached thread.
- `/codex account` shows account and rate-limit status.
- `/codex mcp` lists Codex app-server MCP server status.
- `/codex skills` lists Codex app-server skills.

For most support reports, start with `/diagnostics [note]` in the conversation
where the bug happened. It creates one Gateway diagnostics report and, for Codex
harness sessions, asks for approval to send the relevant Codex feedback bundle.
See [Diagnostics export](/gateway/diagnostics) for the privacy mode
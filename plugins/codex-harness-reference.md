---
domain: plugins
topic: "Codex Harness Reference"
type: reference
keywords:
  - Codex harness reference
  - codex harness config
  - codex config
  - codex settings
source: plugins/codex-harness-reference.md
---

This reference covers the detailed configuration for the bundled `codex`
plugin. For setup and routing decisions, start with
[Codex harness](/plugins/codex-harness).

## Plugin config surface

All Codex harness settings live under `plugins.entries.codex.config`.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

Supported top-level fields:

| Field                      | Default                  | Meaning                                                                                                                                   |
| -------------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`                | enabled                  | Model discovery settings for Codex app-server `model/list`.                                                                               |
| `appServer`                | managed stdio app-server | Transport, command, auth, approval, sandbox, and timeout settings.                                                                        |
| `codexDynamicToolsLoading` | `"searchable"`           | Use `"direct"` to put OpenClaw dynamic tools directly in the initial Codex tool context.                                                  |
| `codexDynamicToolsExclude` | `[]`                     | Additional OpenClaw dynamic tool names to omit from Codex app-server turns.                                                               |
| `codexPlugins`             | disabled                 | Native Codex plugin/app support for migrated source-installed curated plugins. See [Native Codex plugins](/plugins/codex-native-plugins). |
| `computerUse`              | disabled                 | Codex Computer Use setup. See [Codex Computer Use](/plugins/codex-computer-use).                                                          |

## App-server transport

By default, OpenClaw starts the managed Codex binary shipped with the bundled
plugin:

```bash
codex app-server --listen stdio://
```

This keeps the app-server version tied to the bundled `codex` plugin instead of
whichever separate Codex CLI happens to be installed locally. Set
`appServer.command` only when you intentionally want to run a different
executable.

For an already-running app-server, use WebSocket transport:

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
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Supported `appServer` fields:

| Field                         | Default                                                | Meaning                                                                                                                                                                                         |
| ----------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                   | `"stdio"`                                              | `"stdio"` spawns Codex; `"websocket"` connects to `url`.                                                                                                                                        |
| `command`                     | managed Codex binary                                   | Executable for stdio transport. Leave unset to use the managed binary.                                                                                                                          |
| `args`                        | `["app-server", "--listen", "stdio://"]`               | Arguments for stdio transport.                                                                                                                                                                  |
| `url`                         | unset                                                  | WebSocket app-server URL.                                                                                                                                                                       |
| `authToken`                   | unset                                                  | Bearer token for WebSocket transport.                                                                                                                                                           |
| `headers`                     | `{}`                                                   | Extra WebSocket headers.                                                                                                                                                                        |
| `clearEnv`                    | `[]`                                                   | Extra environment variable names removed from the spawned stdio app-server process after OpenClaw builds its inherited environment.                                                             |
| `requestTimeoutMs`            | `60000`                                                | Timeout for app-server control-plane calls.                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs` | `60000`                                                | Quiet window after a turn-scoped app-server request while OpenClaw waits for `turn/completed`.                                                                                                  |
| `mode`                        | `"yolo"` unless local Codex requirements disallow YOLO | Preset for YOLO or guardian-reviewed execution.                                                                                                                                                 |
| `approvalPolicy`              | `"never"` or an allowed guardian approval policy       | Native Codex approval policy sent to thread start, resume, and turn.                                                                                                                            |
| `sandbox`                     | `"danger-full-access"` or an allowed guardian sandbox  | Native Codex sandbox mode sent to thread start and resume.                                                                                                                                      |
| `approvalsReviewer`           | `"user"` or an allowed guardian reviewer               | Use `"auto_review"` to let Codex review native approval prompts when allowed.                                                                                                                   |
| `defaultWorkspaceDir`         | current process directory                              | Workspace used by `/codex bind` when `--cwd` is omitted.                                                                                                                                        |
| `serviceTier`                 | unset                                                  | Optional Codex app-server service tier. `"priority"` enables fast-mode routing, `"flex"` requests flex processing, and `null` clears the override. Legacy `"fast"` is accepted as `"priority"`. |

The plugin blocks older or unversioned app-server handshakes. Codex app-server
must report stable version `0.125.0` or newer.

## Approval and sandbox modes

Local stdio app-server sessions default to YOLO mode:
`approvalPolicy: "never"`, `approvalsReviewer: "user"`, and
`sandbox: "danger-full-access"`. This trusted local operator posture lets
unattended OpenClaw turns and heartbeats make progress without native approval
prompts that nobody is around to answer.

If Codex's local system requirements file disallows implicit YOLO approval,
reviewer, or sandbox values, OpenClaw treats the implicit default as guardian
instead and selects allowed guardian permissions. Hostname-matching
`[[remote_sandbox_config]]` entries in the same requirements file are honored
for the sandbox default decision.

Set `appServer.mode: "guardian"` for Codex guardian-reviewed approvals:

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

The `guardian` preset expands to `approvalPolicy: "on-request"`,
`approvalsReviewer: "auto_review"`, and `sandbox: "workspace-write"` when those
values are allowed. Individual policy fields override `mode`. The older
`guardian_subagent` reviewer value is still accepted as a compatibility alias,
but new configs should use `auto_review`.

## Auth and environment isolation

Auth is selected in this order:

1. An explicit OpenClaw Codex auth profile for the agent.
2. The app-server's existing account in that agent's Codex home.
3. For local stdio app-server launches only, `CODEX_API_KEY`, then
   `OPENAI_API_KEY`, when no app-server account is present and OpenAI auth is
   still required.

When OpenClaw sees a ChatGPT subscription-style Codex auth profile, it removes
`CODEX_API_KEY` and `OPENAI_API_KEY` from the spawned Codex child process. That
keeps Gateway-level API keys available for embeddings or direct OpenAI models
without making native Codex app-server turns bill through the API by accident.

Explicit Codex API-key profiles and local stdio env-key fallback use app-server
login instead of inherited child-process env. WebSocket app-server connections
do not receive Gateway env API-key fallback; use an explicit auth profile or the
remote app-server's own account.

Stdio app-server launches inherit OpenClaw's process environment by default, but
OpenClaw owns the Codex app-server account bridge and sets both `CODEX_HOME` and
`HOME` to per-agent directories under that agent's OpenClaw state. Codex's own
skill loader reads `$CODEX_HOME/skills` and `$HOME/.agents/skills`, so both
values are isolated for local app-server launches. That keeps Codex-native
skills, plugins, config, accounts, and thread state scoped to the OpenClaw agent
instead of leaking in from the operator's personal Codex CLI home.

OpenClaw plugins and OpenClaw skill snapshots still flow through OpenClaw's own
plugin registry and skill loader. Personal Codex CLI assets do not. If you have
useful Codex CLI skills or plugins that should become part of an OpenClaw agent,
inventory them explicitly:

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

If a deployment needs additional environment isolation, add those variables to
`appServer.clearEnv`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` only affects the spawned Codex app-server child process.
`CODEX_HOME` and `HOME` remain reserved for OpenClaw's per-agent Codex
isolation on local launches.

## Dynamic tools

Codex dynamic tools default to `searchable` loading. OpenClaw does not expose
dynamic tools that duplicate Codex-native workspace operations:

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`

Remaining OpenClaw integration tools, such as messaging, sessions, media, cron,
browser, nodes, gateway, `heartbeat_resp
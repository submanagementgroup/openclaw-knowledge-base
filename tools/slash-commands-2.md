---
domain: tools
topic: "Slash Commands Extended Reference"
type: reference
keywords:
  - slash commands
  - commands extended
  - /skills
  - /memory
  - /agent
  - /tools
source: tools/slash-commands.md
---

 more slash commands. Current bundled commands in this repo:

- `/dreaming [on|off|status|help]` toggles memory dreaming. See [Dreaming](/concepts/dreaming).
- `/pair [qr|status|pending|approve|cleanup|notify]` manages device pairing/setup flow. See [Pairing](/channels/pairing).
- `/phone status|arm <camera|screen|writes|all> [duration]|disarm` temporarily arms high-risk phone node commands.
- `/voice status|list [limit]|set <voiceId|name>` manages Talk voice config. On Discord, the native command name is `/talkvoice`.
- `/card ...` sends LINE rich card presets. See [LINE](/channels/line).
- `/codex status|models|threads|resume|compact|review|diagnostics|account|mcp|skills` inspects and controls the bundled Codex app-server harness. See [Codex harness](/plugins/codex-harness).
- QQBot-only commands:
  - `/bot-ping`
  - `/bot-version`
  - `/bot-help`
  - `/bot-upgrade`
  - `/bot-logs`

### Dynamic skill commands

User-invocable skills are also exposed as slash commands:

- `/skill <name> [input]` always works as the generic entrypoint.
- skills may also appear as direct commands like `/prose` when the skill/plugin registers them.
- native skill-command registration is controlled by `commands.nativeSkills` and `channels.<provider>.commands.nativeSkills`.
- command specs can provide `descriptionLocalizations` for native surfaces that support localized descriptions, including Discord.

### Argument and parser notes

- Commands accept an optional `:` between the command and args (e.g. `/think: high`, `/send: on`, `/help:`).
    - `/new <model>` accepts a model alias, `provider/model`, or a provider name (fuzzy match); if no match, the text is treated as the message body.
    - For full provider usage breakdown, use `openclaw status --usage`.
    - `/allowlist add|remove` requires `commands.config=true` and honors channel `configWrites`.
    - In multi-account channels, config-targeted `/allowlist --account <id>` and `/config set channels.<provider>.accounts.<id>...` also honor the target account's `configWrites`.
    - `/usage` controls the per-response usage footer; `/usage cost` prints a local cost summary from OpenClaw session logs.
    - `/restart` is enabled by default; set `commands.restart: false` to disable it.
    - `/plugins install <spec>` accepts the same plugin specs as `openclaw plugins install`: local path/archive, npm package, `git:<repo>`, or `clawhub:<pkg>`, then requests a Gateway restart because plugin source modules changed.
    - `/plugins enable|disable` updates plugin config and triggers Gateway plugin reload for new agent turns.

  ### Channel-specific behavior

- Discord-only native command: `/vc join|leave|status` controls voice channels (not available as text). `join` requires a guild and selected voice/stage channel. Requires `channels.discord.voice` and native commands.
    - Discord thread-binding commands (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`) require effective thread bindings to be enabled (`session.threadBindings.enabled` and/or `channels.discord.threadBindings.enabled`).
    - ACP command reference and runtime behavior: [ACP agents](/tools/acp-agents).

  ### Verbose / trace / fast / reasoning safety

- `/verbose` is meant for debugging and extra visibility; keep it **off** in normal use.
    - `/trace` is narrower than `/verbose`: it only reveals plugin-owned trace/debug lines and keeps normal verbose tool chatter off.
    - `/fast on|off` persists a session override. Use the Sessions UI `inherit` option to clear it and fall back to config defaults.
    - `/fast` is provider-specific: OpenAI/OpenAI Codex map it to `service_tier=priority` on native Responses endpoints, while direct public Anthropic requests, including OAuth-authenticated traffic sent to `api.anthropic.com`, map it to `service_tier=auto` or `standard_only`. See [OpenAI](/providers/openai) and [Anthropic](/providers/anthropic).
    - Tool failure summaries are still shown when relevant, but detailed failure text is only included when `/verbose` is `on` or `full`.
    - `/reasoning`, `/verbose`, and `/trace` are risky in group settings: they may reveal internal reasoning, tool output, or plugin diagnostics you did not intend to expose. Prefer leaving them off, especially in group chats.

  ### Model switching

- `/model` persists the new session model immediately.
    - If the agent is idle, the next run uses it right away.
    - If a run is already active, OpenClaw marks a live switch as pending and only restarts into the new model at a clean retry point.
    - If tool activity or reply output has already started, the pending switch can stay queued until a later retry opportunity or the next user turn.
    - In the local TUI, `/crestodian [request]` returns from the normal agent TUI to Crestodian. This is separate from message-channel rescue mode and does not grant remote config authority.

  ### Fast path and inline shortcuts

- **Fast path:** command-only messages from allowlisted senders are handled immediately (bypass queue + model).
    - **Group mention gating:** command-only messages from allowlisted senders bypass mention requirements.
    - **Inline shortcuts (allowlisted senders only):** certain commands also work when embedded in a normal message and are stripped before the model sees the remaining text.
      - Example: `hey /status` triggers a status reply, and the remaining text continues through the normal flow.
    - Currently: `/help`, `/commands`, `/status`, `/whoami` (`/id`).
    - Unauthorized command-only messages are silently ignored, and inline `/...` tokens are treated as plain text.

  ### Skill commands and native arguments

- **Skill commands:** `user-invocable` skills are exposed as slash commands. Names are sanitized to `a-z0-9_` (max 32 chars); collisions get numeric suffixes (e.g. `_2`).
      - `/skill <name> [input]` runs a skill by name (useful when native command limits prevent per-skill commands).
      - By default, skill commands are forwarded to the model as a normal request.
      - Skills may optionally declare `command-dispatch: tool` to route the command directly to a tool (deterministic, no model).
      - Example: `/prose` (OpenProse plugin) — see [OpenProse](/prose).
    - **Native command arguments:** Discord uses autocomplete for dynamic options (and button menus when you omit required args). Telegram and Slack show a button menu when a command supports choices and you omit the arg. Dynamic choices are resolved against the target session model, so model-specific options such as `/think` levels follow that session's `/model` override.

  ## `/tools`

`/tools` answers a runtime question, not a config question: **what this agent can use right now in this conversation**.

- Default `/tools` is compact and optimized for quick scanning.
- `/tools verbose` adds short descriptions.
- Native-command surfaces that support arguments expose the same mode switch as `compact|verbose`.
- Results are session-scoped, so changing agent, channel, thread, sender authorization, or model can change the output.
- `/tools` includes tools that are actually reachable at runtime, including core tools, connected plugin tools, and channel-owned tools.

For profile and override editing, use the Control UI Tools panel or config/catalog surfaces instead of treating `/tools` as a static catalog.

## Usage surfaces (what shows where)

- **Provider usage/quota** (example: "Claude 80% left") shows up in `/status` for the current model provider when usage tracking is enabled. OpenClaw normalizes provider windows to `% left`; for MiniMax, remaining-only percent fields are inverted before display, and `model_remains` responses prefer the chat-model entry plus a model-tagged plan label.
- **Token/cache lines** in `/status` can fall back to the latest transcript usage entry when the live session snapshot is sparse. Existing nonzero live values still win, and transcript fallback can also recover the active runtime model label plus a larger prompt-oriented total when stored totals are missing or smaller.
- **Execution vs runtime:** `/status` reports `Execution` for the effective sandbox path and `Runtime` for who is actually running the session: `OpenClaw Pi Default`, `OpenAI Codex`, a CLI backend, or an ACP backend.
- **Per-response tokens/cost** is controlled by `/usage off|tokens|full` (appended to normal replies).
- `/model status` is about **models/auth/endpoints**, not usage.

## Model selection (`/model`)

`/model` is implemented as a directive.

Examples:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

Notes:

- `/model` and `/model list` show a compact, numbered picker (model family + available providers).
- On Discord, `/model` and `/models` open an interactive picker with provider and model dropdowns plus a Submit step. The picker respects `agents.defaults.models`, including `provider/*` entries, so provider-scoped discovery can keep the picker below Discord's 25-option component limit.
- `/model <#>` selects from that picker (and prefers the current provider when possible).
- `/model status` shows the detailed view, including configured provider endpoint (`baseUrl`) and API mode (`api`) when available.

## Debug overrides

`/debug` lets you set **runtime-only** config overrides (memory, not disk). Owner-only. Disabled by default; enable with `commands.debug: true`.

Examples:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

> **Note:** Overrides apply immediately to new config reads, but do **not** write to `openclaw.json`. Use `/debug reset` to clear all overrides and return to the on-disk config.


## Plugin trace output

`/trace` lets you toggle **session-scoped plugin trace/debug lines** without turning on full verbose mode.

Examples:

```text
/trace
/trace on
/trace off
```

Notes:

- `/trace` with no argument shows the current session trace state.
- `/trace on` enables plugin trace lines for the current session.
- `/trace off` disables them again.
- Plugin trace lines can appear in `/status` and as a follow-up diagnostic message after the normal assistant reply.
- `/trace` does not replace `/debug`; `/debug` still manages runtime-only config overrides.
- `/trace` does not replace `/verbose`; normal verbose tool/status output still belongs to `/verbose`.

## Config updates

`/config` writes to your on-disk config (`openclaw.json`). Owner-only. Disabled by default; enable with `commands.config: true`.

Examples:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

> **Note:** Config is validated before write; invalid changes are rejected. `/config` updates persist across restarts.


## MCP updates

`/mcp` writes OpenClaw-managed MCP server definitions under `mcp.servers`. Owner-only. Disabled by default; enable with `commands.mcp: true`.

Examples:

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

> **Note:** `/mcp` stores config in OpenClaw config, not Pi-owned project settings. Runtime adapters decide which transports are actually executable.


## Plugin updates

`/plugins` lets operators inspect discovered plugins and toggle enablement in config. Read-only flows can use `/plugin` as an alias. Disabled by default; enable with `commands.plugins: true`.

Examples:

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

> **Note:** - `/plugins list` and `/plugins show` use real plugin discovery against the current workspace plus on-disk config.
- `/plugins install` installs from ClawHub, npm, git, local directories, and archives.
- `/plugins enable|disable` updates plugin config only; it does not install or uninstall plugins.
- Enable and disable changes hot-reload Gateway plugin runtime surfaces for new agent turns; install requests a Gateway restart because plugin source modules changed.



## Surface notes

### Sessions per surface

- **Text commands** run in the normal chat session (DMs share `main`, groups have their own session).
    - **Native commands** use isolated sessions:
      - Discord: `agent:<agentId>:discord:slash:<userId>`
      - Slack: `agent:<agentId>:slack:slash:<userId>` (prefix configurable via `channels.slack.slashCommand.sessionPrefix`)
      - Telegram: `telegram:slash:<userId>` (targets the chat session via `CommandTargetSessionKey`)
    - **`/stop`** targets the active chat session so it can abort the current run.

  ### Slack specifics

`channels.slack.slashCommand` is still supported for a single `/openclaw`-style command. If you enable `commands.native`, you must create one Slack slash command per built-in command (same names as `/help`). Command argument menus for Slack are delivered as ephemeral Block Kit buttons.

    Slack native exception: register `/agentstatus` (not `/status`) because Slack reserves `/status`. Text `/status` still works in Slack messages.

  ## BTW side questions

`/btw` is a quick **side question** about the current session. `/side` is an alias.

Unlike normal chat:

- it uses the current session as background context,
- in Codex harness sessions, it runs as an ephemeral Codex side thread with the
  current Codex permissions and native tool surface,
- in non-Codex sessions, it keeps the older direct one-shot side-call behavior,
- it does not change future session context,
- it is not written to transcript history,
- it is delivered as a live side result instead of a normal assistant message.

That makes `/btw` useful when you want a temporary clarification while the main task keeps going.

Example:

```text
/btw what are we doing right now?
/side what changed while the main run continued?
```

Se
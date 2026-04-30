---
domain: tools
topic: "Slash Commands Advanced: Plugin Commands, Custom Commands, and Channel Behavior"
type: reference
keywords:
  - custom slash commands
  - plugin commands
  - slash command plugins
  - command aliases
related:
  - tools/slash-commands
source: tools/slash-commands.md
---

Slash commands advanced: plugin commands, channel-specific behavior, and adding custom commands.

 shows what the current agent can use right now.
    - `/status` shows execution/runtime status, including `Execution`/`Runtime` labels and provider usage/quota when available.
    - `/diagnostics [note]` is the owner-only support-report flow for Gateway bugs and Codex harness runs. It asks for explicit exec approval every time before running `openclaw gateway diagnostics export --json`; do not approve diagnostics with an allow-all rule. After approval, it sends a pasteable report with the local bundle path, manifest summary, privacy notes, and relevant session ids. In group chats, the approval prompt and report go to the owner privately. When the active session uses the OpenAI Codex harness, the same approval also sends relevant Codex feedback to OpenAI servers and the completed reply lists the OpenClaw session ids, Codex thread ids, and `codex resume <thread-id>` commands. See [Diagnostics Export](/gateway/diagnostics).
    - `/crestodian <request>` runs the Crestodian setup and repair helper from an owner DM.
    - `/tasks` lists active/recent background tasks for the current session.
    - `/context [list|detail|json]` explains how context is assembled.
    - `/whoami` shows your sender id. Alias: `/id`.
    - `/usage off|tokens|full|cost` controls the per-response usage footer or prints a local cost summary.

    - `/skill <name> [input]` runs a skill by name.
    - `/allowlist [list|add|remove] ...` manages allowlist entries. Text-only.
    - `/approve <id> <decision>` resolves exec approval prompts.
    - `/btw <question>` asks a side question without changing future session context. See [BTW](/tools/btw).

    - `/subagents list|kill|log|info|send|steer|spawn` manages sub-agent runs for the current session.
    - `/acp spawn|cancel|steer|close|sessions|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|help` manages ACP sessions and runtime options.
    - `/focus <target>` binds the current Discord thread or Telegram topic/conversation to a session target.
    - `/unfocus` removes the current binding.
    - `/agents` lists thread-bound agents for the current session.
    - `/kill <id|#|all>` aborts one or all running sub-agents.
    - `/steer <id|#> <message>` sends steering to a running sub-agent. Alias: `/tell`.

    - `/config show|get|set|unset` reads or writes `openclaw.json`. Owner-only. Requires `commands.config: true`.
    - `/mcp show|get|set|unset` reads or writes OpenClaw-managed MCP server config under `mcp.servers`. Owner-only. Requires `commands.mcp: true`.
    - `/plugins list|inspect|show|get|install|enable|disable` inspects or mutates plugin state. `/plugin` is an alias. Owner-only for writes. Requires `commands.plugins: true`.
    - `/debug show|set|unset|reset` manages runtime-only config overrides. Owner-only. Requires `commands.debug: true`.
    - `/restart` restarts OpenClaw when enabled. Default: enabled; set `commands.restart: false` to disable it.
    - `/send on|off|inherit` sets send policy. Owner-only.

    - `/tts on|off|status|chat|latest|provider|limit|summary|audio|help` controls TTS. See [TTS](/tools/tts).
    - `/activation mention|always` sets group activation mode.
    - `/bash <command>` runs a host shell command. Text-only. Alias: `! <command>`. Requires `commands.bash: true` plus `tools.elevated` allowlists.
    - `!poll [sessionId]` checks a background bash job.
    - `!stop [sessionId]` stops a background bash job.

### Generated dock commands

Dock commands switch the current session's reply route to another linked
channel. See [Channel docking](/concepts/channel-docking) for setup,
examples, and troubleshooting.

Dock commands are generated from channel plugins with native-command support. Current bundled set:

- `/dock-discord` (alias: `/dock_discord`)
- `/dock-mattermost` (alias: `/dock_mattermost`)
- `/dock-slack` (alias: `/dock_slack`)
- `/dock-telegram` (alias: `/dock_telegram`)

Use dock commands from a direct chat to switch the current session's reply route to another linked channel. The agent keeps the same session context, but future replies for that session are delivered to the selected channel peer.

Dock commands require `session.identityLinks`. The source sender and target peer must be in the same identity group, for example `["telegram:123", "discord:456"]`. If a Telegram user with id `123` sends `/dock_discord`, OpenClaw stores `lastChannel: "discord"` and `lastTo: "456"` on the active session. If the sender is not linked to a Discord peer, the command replies with a setup hint instead of falling through to normal chat.

Docking changes the active session route only. It does not create channel accounts, grant access, bypass channel allowlists, or move transcript history to another session. Use `/dock-telegram`, `/dock-slack`, `/dock-mattermost`, or another generated dock command to switch the route again.

### Bundled plugin commands

Bundled plugins can add more slash commands. Current bundled commands in this repo:

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

    - Commands accept an optional `:` between the command and args (e.g. `/think: high`, `/send: on`, `/help:`).
    - `/new <model>` accepts a model alias, `provider/model`, or a provider name (fuzzy match); if no match, the text is treated as the message body.
    - For full provider usage breakdown, use `openclaw status --usage`.
    - `/allowlist add|remove` requires `commands.config=true` and honors channel `configWrites`.
    - In multi-account channels, config-targeted `/allowlist --account <id>` and `/config set channels.<provider>.accounts.<id>...` also honor the target account's `configWrites`.
    - `/usage` controls the per-response usage footer; `/usage cost` prints a local cost summary from OpenClaw session logs.
    - `/restart` is enabled by default; set `co

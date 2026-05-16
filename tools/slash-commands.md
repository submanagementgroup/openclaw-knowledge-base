---
domain: tools
topic: "Slash Commands Reference"
type: reference
keywords:
  - slash commands
  - /commands
  - chat commands
  - command reference
  - /model
  - /verbose
  - /trace
source: tools/slash-commands.md
---

Commands are handled by the Gateway. Most commands must be sent as a **standalone** message that starts with `/`. The host-only bash chat command uses `! <cmd>` (with `/bash <cmd>` as an alias).

When a conversation or thread is bound to an ACP session, normal follow-up text routes to that ACP harness. Gateway management commands still stay local: `/acp ...` always reaches the OpenClaw ACP command handler, and `/status` plus `/unfocus` stay local whenever command handling is enabled for the surface.

There are two related systems:

### Commands

Standalone `/...` messages.
  ### Directives

`/think`, `/fast`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.

    - Directives are stripped from the message before the model sees it.
    - In normal chat messages (not directive-only), they are treated as "inline hints" and do **not** persist session settings.
    - In directive-only messages (the message contains only directives), they persist to the session and reply with an acknowledgement.
    - Directives are only applied for **authorized senders**. If `commands.allowFrom` is set, it is the only allowlist used; otherwise authorization comes from channel allowlists/pairing plus `commands.useAccessGroups`. Unauthorized senders see directives treated as plain text.

  ### Inline shortcuts

Allowlisted/authorized senders only: `/help`, `/commands`, `/status`, `/whoami` (`/id`).

    They run immediately, are stripped before the model sees the message, and the remaining text continues through the normal flow.

  ## Config

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: true,
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw",
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```


  Enables parsing `/...` in chat messages. On surfaces without native commands (WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams), text commands still work even if you set this to `false`.


  Registers native commands. Auto: on for Discord/Telegram; off for Slack (until you add slash commands); ignored for providers without native support. Set `channels.discord.commands.native`, `channels.telegram.commands.native`, or `channels.slack.commands.native` to override per provider (bool or `"auto"`). On Discord, `false` skips slash-command registration and cleanup during startup; previously registered commands may remain visible until you remove them from the Discord app. Slack commands are managed in the Slack app and are not removed automatically.

On Discord, native command specs may include `descriptionLocalizations`, which OpenClaw publishes as Discord `description_localizations` and includes in reconcile comparisons.

  Registers **skill** commands natively when supported. Auto: on for Discord/Telegram; off for Slack (Slack requires creating a slash command per skill). Set `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills`, or `channels.slack.commands.nativeSkills` to override per provider (bool or `"auto"`).


  Enables `! <cmd>` to run host shell commands (`/bash <cmd>` is an alias; requires `tools.elevated` allowlists).


  Controls how long bash waits before switching to background mode (`0` backgrounds immediately).


  Enables `/config` (reads/writes `openclaw.json`).


  Enables `/mcp` (reads/writes OpenClaw-managed MCP config under `mcp.servers`).


  Enables `/plugins` (plugin discovery/status plus install + enable/disable controls).


  Enables `/debug` (runtime-only overrides).


  Enables `/restart` plus gateway restart tool actions.


  Sets the explicit owner allowlist for owner-only command/tool surfaces. This is the human operator account that can approve dangerous actions and run commands such as `/diagnostics`, `/export-trajectory`, and `/config`. It is separate from `commands.allowFrom` and from DM pairing access.

.commands.enforceOwnerForCommands" type="boolean" default="false">
  Per-channel: makes owner-only commands require **owner identity** to run on that surface. When `true`, the sender must either match a resolved owner candidate (for example an entry in `commands.ownerAllowFrom` or provider-native owner metadata) or hold internal `operator.admin` scope on an internal message channel. A wildcard entry in channel `allowFrom`, or an empty/unresolved owner-candidate list, is **not** sufficient — owner-only commands fail closed on that channel. Leave this off if you want owner-only commands gated only by `ownerAllowFrom` and the standard command allowlists.


  Controls how owner ids appear in the system prompt.


  Optionally sets the HMAC secret used when `commands.ownerDisplay="hash"`.


  Per-provider allowlist for command authorization. When configured, it is the only authorization source for commands and directives (channel allowlists/pairing and `commands.useAccessGroups` are ignored). Use `"*"` for a global default; provider-specific keys override it.


  Enforces allowlists/policies for commands when `commands.allowFrom` is not set.


## Command list

Current source-of-truth:

- core built-ins come from `src/auto-reply/commands-registry.shared.ts`
- generated dock commands come from `src/auto-reply/commands-registry.data.ts`
- plugin commands come from plugin `registerCommand()` calls
- actual availability on your gateway still depends on config flags, channel surface, and installed/enabled plugins

### Core built-in commands

### Sessions and runs

- `/new [model]` starts a new session; `/reset` is the reset alias.
    - Control UI intercepts typed `/new` to create and switch to a fresh dashboard session, except when `session.dmScope: "main"` is configured and the current parent is the agent's main session; in that case `/new` resets the main session in place. Typed `/reset` still runs the Gateway's in-place reset.
    - `/reset soft [message]` keeps the current transcript, drops reused CLI backend session ids, and reruns startup/system-prompt loading in-place.
    - `/compact [instructions]` compacts the session context. See [Compaction](/concepts/compaction).
    - `/stop` aborts the current run.
    - `/session idle <duration|off>` and `/session max-age <duration|off>` manage thread-binding expiry.
    - `/export-session [path]` exports the current session to HTML. Alias: `/export`.
    - `/export-trajectory [path]` asks for exec approval, then exports a JSONL [trajectory bundle](/tools/trajectory) for the current session. Use it when you need the prompt, tool, and transcript timeline for one OpenClaw session. In group chats, the approval prompt and export result go to the owner privately. Alias: `/trajectory`.

  ### Model and run controls

- `/think <level|default>` sets the thinking level or clears the session override. Options come from the active model's provider profile; common levels are `off`, `minimal`, `low`, `medium`, and `high`, with custom levels such as `xhigh`, `adaptive`, `max`, or binary `on` only where supported. Aliases: `/thinking`, `/t`.
    - `/verbose on|off|full` toggles verbose output. Alias: `/v`.
    - `/trace on|off` toggles plugin trace output for the current session.
    - `/fast [status|on|off|default]` shows, sets, or clears fast mode.
    - `/reasoning [on|off|stream]` toggles reasoning visibility. Alias: `/reason`.
    - `/elevated [on|off|ask|full]` toggles elevated mode. Alias: `/elev`.
    - `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` shows or sets exec defaults.
    - `/model [name|#|status]` shows or sets the model.
    - `/models [provider] [page] [limit=<n>|size=<n>|all]` lists configured/auth-available providers or models for a provider; add `all` to browse that provider's full catalog. `provider/*` entries in `agents.defaults.models` make `/model` and `/models` show discovered models only for those providers.
    - `/queue <mode>` manages queue behavior (`steer`, legacy `queue`, `followup`, `collect`, `steer-backlog`, `interrupt`) plus options like `debounce:0.5s cap:25 drop:summarize`; `/queue default` or `/queue reset` clears the session override. See [Command queue](/concepts/queue) and [Steering queue](/concepts/queue-steering).
    - `/steer <message>` injects guidance into the active run for the current session, independent of `/queue` mode. It does not start a new run when the session is idle. Alias: `/tell`. See [Steer](/tools/steer).

  ### Discovery and status

- `/help` shows the short help summary.
    - `/commands` shows the generated command catalog.
    - `/tools [compact|verbose]` shows what the current agent can use right now.
    - `/status` shows execution/runtime status, Gateway and system uptime, plus provider usage/quota when available.
    - `/diagnostics [note]` is the owner-only support-report flow for Gateway bugs and Codex harness runs. It asks for explicit exec approval every time before running `openclaw gateway diagnostics export --json`; do not approve diagnostics with an allow-all rule. After approval, it sends a pasteable report with the local bundle path, manifest summary, privacy notes, and relevant session ids. In group chats, the approval prompt and report go to the owner privately. When the active session uses the OpenAI Codex harness, the same approval also sends relevant Codex feedback to OpenAI servers and the completed reply lists the OpenClaw session ids, Codex thread ids, and `codex resume <thread-id>` commands. See [Diagnostics Export](/gateway/diagnostics).
    - `/crestodian <request>` runs the Crestodian setup and repair helper from an owner DM.
    - `/tasks` lists active/recent background tasks for the current session.
    - `/context [list|detail|map|json]` explains how context is assembled. `map` sends a treemap image of the current session context.
    - `/whoami` shows your sender id. Alias: `/id`.
    - `/usage off|tokens|full|cost` controls the per-response usage footer or prints a local cost summary.

  ### Skills, allowlists, approvals

- `/skill <name> [input]` runs a skill by name.
    - `/allowlist [list|add|remove] ...` manages allowlist entries. Text-only.
    - `/approve <id> <decision>` resolves exec approval prompts.
    - `/btw <question>` asks a side question without changing future session context. Alias: `/side`. See [BTW](/tools/btw).

  ### Subagents and ACP

- `/subagents list|kill|log|info|send|steer|spawn` manages sub-agent runs for the current session.
    - `/acp spawn|cancel|steer|close|sessions|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|help` manages ACP sessions and runtime options.
    - `/focus <target>` binds the current Discord thread or Telegram topic/conversation to a session target.
    - `/unfocus` removes the current binding.
    - `/agents` lists thread-bound agents for the current session.
    - `/kill <id|#|all>` aborts one or all running sub-agents.
    - `/subagents steer <id|#> <message>` sends steering to a running sub-agent. See [Steer](/tools/steer).

  ### Owner-only writes and admin

- `/config show|get|set|unset` reads or writes `openclaw.json`. Owner-only. Requires `commands.config: true`.
    - `/mcp show|get|set|unset` reads or writes OpenClaw-managed MCP server config under `mcp.servers`. Owner-only. Requires `commands.mcp: true`.
    - `/plugins list|inspect|show|get|install|enable|disable` inspects or mutates plugin state. `/plugin` is an alias. Owner-only for writes. Requires `commands.plugins: true`.
    - `/debug show|set|unset|reset` manages runtime-only config overrides. Owner-only. Requires `commands.debug: true`.
    - `/restart` restarts OpenClaw when enabled. Default: enabled; set `commands.restart: false` to disable it.
    - `/send on|off|inherit` sets send policy. Owner-only.

  ### Voice, TTS, channel control

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

Bundled plugins can add
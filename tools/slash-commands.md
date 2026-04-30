---
domain: tools
topic: "Slash Commands: /new, /reset, /stop, /help, and All Built-In Commands"
type: reference
keywords:
  - slash commands
  - /new
  - /reset
  - /stop
  - /help
  - chat commands
  - built-in commands
related:
  - tools/tools-overview
  - automation/hooks
source: tools/slash-commands.md
---

Slash commands are text commands prefixed with `/` that users can type in any channel. Built-in commands include /new, /reset, /stop, /help, and many more.

Commands are handled by the Gateway. Most commands must be sent as a **standalone** message that starts with `/`. The host-only bash chat command uses `! <cmd>` (with `/bash <cmd>` as an alias).

When a conversation or thread is bound to an ACP session, normal follow-up text routes to that ACP harness. Gateway management commands still stay local: `/acp ...` always reaches the OpenClaw ACP command handler, and `/status` plus `/unfocus` stay local whenever command handling is enabled for the surface.

There are two related systems:

    Standalone `/...` messages.

    `/think`, `/fast`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.

    - Directives are stripped from the message before the model sees it.
    - In normal chat messages (not directive-only), they are treated as "inline hints" and do **not** persist session settings.
    - In directive-only messages (the message contains only directives), they persist to the session and reply with an acknowledgement.
    - Directives are only applied for **authorized senders**. If `commands.allowFrom` is set, it is the only allowlist used; otherwise authorization comes from channel allowlists/pairing plus `commands.useAccessGroups`. Unauthorized senders see directives treated as plain text.

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

<ParamField path="commands.text" type="boolean" default="true">
  Enables parsing `/...` in chat messages. On surfaces without native commands (WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams), text commands still work even if you set this to `false`.
</ParamField>
<ParamField path="commands.native" type='boolean | "auto"' default='"auto"'>
  Registers native commands. Auto: on for Discord/Telegram; off for Slack (until you add slash commands); ignored for providers without native support. Set `channels.discord.commands.native`, `channels.telegram.commands.native`, or `channels.slack.commands.native` to override per provider (bool or `"auto"`). `false` clears previously registered commands on Discord/Telegram at startup. Slack commands are managed in the Slack app and are not removed automatically.
</ParamField>
<ParamField path="commands.nativeSkills" type='boolean | "auto"' default='"auto"'>
  Registers **skill** commands natively when supported. Auto: on for Discord/Telegram; off for Slack (Slack requires creating a slash command per skill). Set `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills`, or `channels.slack.commands.nativeSkills` to override per provider (bool or `"auto"`).
</ParamField>
<ParamField path="commands.bash" type="boolean" default="false">
  Enables `! <cmd>` to run host shell commands (`/bash <cmd>` is an alias; requires `tools.elevated` allowlists).
</ParamField>
<ParamField path="commands.bashForegroundMs" type="number" default="2000">
  Controls how long bash waits before switching to background mode (`0` backgrounds immediately).
</ParamField>
<ParamField path="commands.config" type="boolean" default="false">
  Enables `/config` (reads/writes `openclaw.json`).
</ParamField>
<ParamField path="commands.mcp" type="boolean" default="false">
  Enables `/mcp` (reads/writes OpenClaw-managed MCP config under `mcp.servers`).
</ParamField>
<ParamField path="commands.plugins" type="boolean" default="false">
  Enables `/plugins` (plugin discovery/status plus install + enable/disable controls).
</ParamField>
<ParamField path="commands.debug" type="boolean" default="false">
  Enables `/debug` (runtime-only overrides).
</ParamField>
<ParamField path="commands.restart" type="boolean" default="true">
  Enables `/restart` plus gateway restart tool actions.
</ParamField>
<ParamField path="commands.ownerAllowFrom" type="string[]">
  Sets the explicit owner allowlist for owner-only command/tool surfaces. This is the human operator account that can approve dangerous actions and run commands such as `/diagnostics`, `/export-trajectory`, and `/config`. It is separate from `commands.allowFrom` and from DM pairing access.
</ParamField>
<ParamField path="channels.<channel>.commands.enforceOwnerForCommands" type="boolean" default="false">
  Per-channel: makes owner-only commands require **owner identity** to run on that surface. When `true`, the sender must either match a resolved owner candidate (for example an entry in `commands.ownerAllowFrom` or provider-native owner metadata) or hold internal `operator.admin` scope on an internal message channel. A wildcard entry in channel `allowFrom`, or an empty/unresolved owner-candidate list, is **not** sufficient — owner-only commands fail closed on that channel. Leave this off if you want owner-only commands gated only by `ownerAllowFrom` and the standard command allowlists.
</ParamField>
<ParamField path="commands.ownerDisplay" type='"raw" | "hash"'>
  Controls how owner ids appear in the system prompt.
</ParamField>
<ParamField path="commands.ownerDisplaySecret" type="string">
  Optionally sets the HMAC secret used when `commands.ownerDisplay="hash"`.
</ParamField>
<ParamField path="commands.allowFrom" type="object">
  Per-provider allowlist for command authorization. When configured, it is the only authorization source for commands and directives (channel allowlists/pairing and `commands.useAccessGroups` are ignored). Use `"*"` for a global default; provider-specific keys override it.
</ParamField>
<ParamField path="commands.useAccessGroups" type="boolean" default="true">
  Enforces allowlists/policies for commands when `commands.allowFrom` is not set.
</ParamField>

## Command list

Current source-of-truth:

- core built-ins come from `src/auto-reply/commands-registry.shared.ts`
- generated dock commands come from `src/auto-reply/commands-registry.data.ts`
- plugin commands come from plugin `registerCommand()` calls
- actual availability on your gateway still depends on config flags, channel surface, and installed/enabled plugins

### Core built-in commands

    - `/new [model]` starts a new session; `/reset` is the reset alias.
    - `/reset soft [message]` keeps the current transcript, drops reused CLI backend session ids, and reruns startup/system-prompt loading in-place.
    - `/compact [instructions]` compacts the session context. See [Compaction](/concepts/compaction).
    - `/stop` aborts the current run.
    - `/session idle <duration|off>` and `/session max-age <duration|off>` manage thread-binding expiry.
    - `/export-session [path]` exports the current session to HTML. Alias: `/export`.
    - `/export-trajectory [path]` asks for exec approval, then exports a JSONL [trajectory bundle](/tools/trajectory) for the current session. Use it when you need the prompt, tool, and transcript timeline for one OpenClaw session. In group chats, the approval prompt and export result go to the owner privately. Alias: `/trajectory`.

    - `/think <level>` sets the thinking level. Options come from the active model's provider profile; common levels are `off`, `minimal`, `low`, `medium`, and `high`, with custom levels such as `xhigh`, `adaptive`, `max`, or binary `on` only where supported. Aliases: `/thinking`, `/t`.
    - `/verbose on|off|full` toggles verbose output. Alias: `/v`.
    - `/trace on|off` toggles plugin trace output for the current session.
    - `/fast [status|on|off]` shows or sets fast mode.
    - `/reasoning [on|off|stream]` toggles reasoning visibility. Alias: `/reason`.
    - `/elevated [on|off|ask|full]` toggles elevated mode. Alias: `/elev`.
    - `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` shows or sets exec defaults.
    - `/model [name|#|status]` shows or sets the model.
    - `/models [provider] [page] [limit=<n>|size=<n>|all]` lists configured/auth-available providers or models for a provider; add `all` to browse that provider's full catalog.
    - `/queue <mode>` manages queue behavior (`steer`, legacy `queue`, `followup`, `collect`, `steer-backlog`, `interrupt`) plus options like `debounce:0.5s cap:25 drop:summarize`; `/queue default` or `/queue reset` clears the session override. See [Command queue](/concepts/queue) and [Steering queue](/concepts/queue-steering).

    - `/help` shows the short help summary.
    - `/commands` shows the generated command catalog.
    - `/tools [compact|verbose]`

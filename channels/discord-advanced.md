---
domain: channels
topic: "Discord Advanced Configuration and Features"
type: reference
keywords:
  - Discord
  - discord advanced
  - discord permissions
  - discord slash commands
  - discord voice
source: channels/discord.md
---

it should match the attachment reference

Modal forms:

- Add `components.modal` with up to 5 fields
- Field types: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw adds a trigger button automatically

Example:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optional fallback text",
  components: {
    reusable: true,
    text: "Choose a path",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Approve",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Decline", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Pick an option",
          options: [
            { label: "Option A", value: "a" },
            { label: "Option B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Open form",
      fields: [
        { type: "text", label: "Requester" },
        {
          type: "select",
          label: "Priority",
          options: [
            { label: "Low", value: "low" },
            { label: "High", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Access control and routing

**DM policy:**

`channels.discord.dmPolicy` controls DM access. `channels.discord.allowFrom` is the canonical DM allowlist.

    - `pairing` (default)
    - `allowlist`
    - `open` (requires `channels.discord.allowFrom` to include `"*"`)
    - `disabled`

    If DM policy is not open, unknown users are blocked (or prompted for pairing in `pairing` mode).

    Multi-account precedence:

    - `channels.discord.accounts.default.allowFrom` applies only to the `default` account.
    - For one account, `allowFrom` takes precedence over legacy `dm.allowFrom`.
    - Named accounts inherit `channels.discord.allowFrom` when their own `allowFrom` and legacy `dm.allowFrom` are unset.
    - Named accounts do not inherit `channels.discord.accounts.default.allowFrom`.

    Legacy `channels.discord.dm.policy` and `channels.discord.dm.allowFrom` still read for compatibility. `openclaw doctor --fix` migrates them to `dmPolicy` and `allowFrom` when it can do so without changing access.

    DM target format for delivery:

    - `user:<id>`
    - `<@id>` mention

    Bare numeric IDs normally resolve as channel IDs when a channel default is active, but IDs listed in the account's effective DM `allowFrom` are treated as user DM targets for compatibility.

  
**Access groups:**

Discord DMs and text command authorization can use dynamic `accessGroup:<name>` entries in `channels.discord.allowFrom`.

    Access group names are shared across message channels. Use `type: "message.senders"` for a static group whose members are expressed in each channel's normal `allowFrom` syntax, or `type: "discord.channelAudience"` when a Discord channel's current `ViewChannel` audience should define membership dynamically. Shared access-group behavior is documented here: [Access groups](/channels/access-groups).

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

    A Discord text channel has no separate member list. `type: "discord.channelAudience"` models membership as: the DM sender is a member of the configured guild and currently has effective `ViewChannel` permission on the configured channel after role and channel overwrites are applied.

    Example: allow anyone who can see `#maintainers` to DM the bot, while keeping DMs closed to everyone else.

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

    You can mix dynamic and static entries:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers", "discord:123456789012345678"],
    },
  },
}
```

    Lookups fail closed. If Discord returns `Missing Access`, the member lookup fails, or the channel belongs to a different guild, the DM sender is treated as unauthorized.

    Enable the Discord Developer Portal **Server Members Intent** for the bot when using channel-audience access groups. DMs do not include guild member state, so OpenClaw resolves the member through Discord REST at authorization time.

  
**Guild policy:**

Guild handling is controlled by `channels.discord.groupPolicy`:

    - `open`
    - `allowlist`
    - `disabled`

    Secure baseline when `channels.discord` exists is `allowlist`.

    `allowlist` behavior:

    - guild must match `channels.discord.guilds` (`id` preferred, slug accepted)
    - optional sender allowlists: `users` (stable IDs recommended) and `roles` (role IDs only); if either is configured, senders are allowed when they match `users` OR `roles`
    - direct name/tag matching is disabled by default; enable `channels.discord.dangerouslyAllowNameMatching: true` only as break-glass compatibility mode
    - names/tags are supported for `users`, but IDs are safer; `openclaw security audit` warns when name/tag entries are used
    - if a guild has `channels` configured, non-listed channels are denied
    - if a guild has no `channels` block, all channels in that allowlisted guild are allowed

    Example:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    If you only set `DISCORD_BOT_TOKEN` and do not create a `channels.discord` block, runtime fallback is `groupPolicy="allowlist"` (with a warning in logs), even if `channels.defaults.groupPolicy` is `open`.

  
**Mentions and group DMs:**

Guild messages are mention-gated by default.

    Mention detection includes:

    - explicit bot mention
    - configured mention patterns (`agents.list[].groupChat.mentionPatterns`, fallback `messages.groupChat.mentionPatterns`)
    - implicit reply-to-bot behavior in supported cases

    When writing outbound Discord messages, use canonical mention syntax: `<@USER_ID>` for users, `<#CHANNEL_ID>` for channels, and `<@&ROLE_ID>` for roles. Do not use the legacy `<@!USER_ID>` nickname mention form.

    `requireMention` is configured per guild/channel (`channels.discord.guilds...`).
    `ignoreOtherMentions` optionally drops messages that mention another user/role but not the bot (excluding @everyone/@here).

    Group DMs:

    - default: ignored (`dm.groupEnabled=false`)
    - optional allowlist via `dm.groupChannels` (channel IDs or slugs)

  
### Role-based agent routing

Use `bindings[].match.roles` to route Discord guild members to different agents by role ID. Role-based bindings accept role IDs only and are evaluated after peer or parent-peer bindings and before guild-only bindings. If a binding also sets other match fields (for example `peer` + `guildId` + `roles`), all configured fields must match.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Native commands and command auth

- `commands.native` defaults to `"auto"` and is enabled for Discord.
- Per-channel override: `channels.discord.commands.native`.
- `commands.native=false` skips Discord slash-command registration and cleanup during startup. Previously registered commands may remain visible in Discord until you remove them from the Discord app.
- Native command auth uses the same Discord allowlists/policies as normal message handling.
- Commands may still be visible in Discord UI for users who are not authorized; execution still enforces OpenClaw auth and returns "not authorized".

See [Slash commands](/tools/slash-commands) for command catalog and behavior.

Default slash command settings:

- `ephemeral: true`

## Feature details

### Reply tags and native replies

Discord supports reply tags in agent output:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    Controlled by `channels.discord.replyToMode`:

    - `off` (default)
    - `first`
    - `all`
    - `batched`

    Note: `off` disables implicit reply threading. Explicit `[[reply_to_*]]` tags are still honored.
    `first` always attaches the implicit native reply reference to the first outbound Discord message for the turn.
    `batched` only attaches Discord's implicit native reply reference when the
    inbound turn was a debounced batch of multiple messages. This is useful
    when you want native replies mainly for ambiguous bursty chats, not every
    single-message turn.

    Message IDs are surfaced in context/history so agents can target specific messages.

  ### Live stream preview

OpenClaw can stream draft replies by sending a temporary message and editing it as text arrives. `channels.discord.streaming` takes `off` | `partial` | `block` | `progress` (default). `progress` keeps one editable status draft and updates it with tool progress until final delivery; the shared starter label is a rolling line, so it scrolls away like the rest once enough work appears. `streamMode` is a legacy runtime alias. Run `openclaw doctor --fix` to rewrite persisted config to the canonical key.

    Set `channels.discord.streaming.mode` to `off` to disable Discord preview edits. If Discord block streaming is explicitly enabled, OpenClaw skips the preview stream to avoid double-streaming.

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto",
          maxLines: 8,
          toolProgress: true,
        },
      },
    },
  },
}
```

    - `partial` edits a single preview message as tokens arrive.
    - `block` emits draft-sized chunks (use `draftChunk` to tune size and breakpoints, clamped to `textChunkLimit`).
    - Media, error, and explicit-reply finals cancel pending preview edits.
    - `streaming.preview.toolProgress` (default `true`) controls whether tool/progress updates reuse the preview message.
    - Tool/progress rows render as compact emoji + title + detail when available, for example `🛠️ Bash: run tests` or `🔎 Web Search: for "query"`.
    - `streaming.preview.commandText` / `streaming.progress.commandText` controls command/exec detail in compact progress lines: `raw` (default) or `status` (tool label only).

    Hide raw command/exec text while keeping compact progress lines:

    ```json
    {
      "channels": {
        "discord": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    Preview streaming is text-only; media replies fall back to normal delivery. When `block` streaming is explicitly enabled, OpenClaw skips the preview stream to avoid double-streaming.

  ### History, context, and thread behavior

Guild history context:

    - `channels.discord.historyLimit` default `20`
    - fallback: `messages.groupChat.historyLimit`
    - `0` disables

    DM history controls:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    Thread behavior:

    - Discord threads route as channel sessions and inherit parent channel config unless overridden.
    - Thread sessions inherit the parent channel's session-level `/model` selection as a model-only fallback; thread-local `/model` selections still take precedence and parent transcript history is not copied unless transcript inheritance is enabled.
    - `channels.discord.thread.inheritParent` (default `false`) opts new auto-threads into seeding from the parent transcript. Per-account overrides live under `channels.discord.accounts.<id>.thread.inheritParent`.
    - Message-tool reactions can resolve `user:<id>` DM targets.
    - `guilds.<guild>.channels.<channel>.requireMention: false` is preserved during reply-stage activation fallback.

    Channel topics are injected as **untrusted** context. Allowlists gate who can trigger the agent, not a full supplemental-context redaction boundary.

  ### Thread-bound sessions for subagents

Discord can bind a thread to a session target so follow-up messages in that thread keep routing to the same session (including subagent sessions).

    Commands:

    - `/focus <target>` bind current/new thread to a subagent/session target
    - `/unfocus` remove current thread binding
    - `/agents` show active runs and binding state
    - `/session idle <duration|off>` inspect/update inactivity auto-unfocus for focused bindings
    - `/session max-age <duration|off>` inspect/update hard max age for focused bindings

    Config:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {

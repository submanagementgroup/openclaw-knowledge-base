---
domain: channels
topic: "Feishu (Lark) Channel: App Setup, Access Control, and Groups"
type: procedure
keywords:
  - Feishu
  - Lark
  - feishu channel
  - Chinese messaging
  - Feishu bot
related:
  - channels/wechat
  - channels/qqbot
source: channels/feishu.md
---

Feishu (Lark) channel for OpenClaw. Supports DMs, groups, and Feishu-specific features.

# Feishu / Lark

Feishu/Lark is an all-in-one collaboration platform where teams chat, share documents, manage calendars, and get work done together.

**Status:** production-ready for bot DMs + group chats. WebSocket is the default mode; webhook mode is optional.

---

## Quick start

Requires OpenClaw 2026.4.25 or above. Run `openclaw --version` to check. Upgrade with `openclaw update`.

  ```bash
  openclaw channels login --channel feishu
  ```
  Scan the QR code with your Feishu/Lark mobile app to create a Feishu/Lark bot automatically.

  ```bash
  openclaw gateway restart
  ```

---

## Access control

### Direct messages

Configure `dmPolicy` to control who can DM the bot:

- `"pairing"` — unknown users receive a pairing code; approve via CLI
- `"allowlist"` — only users listed in `allowFrom` can chat (default: bot owner only)
- `"open"` — allow public DMs only when `allowFrom` includes `"*"`; with restrictive entries, only matching users can chat
- `"disabled"` — disable all DMs

**Approve a pairing request:**

```bash
openclaw pairing list feishu
openclaw pairing approve feishu <CODE>
```

### Group chats

**Group policy** (`channels.feishu.groupPolicy`):

| Value         | Behavior                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `"open"`      | Respond to all messages in groups                                                            |
| `"allowlist"` | Only respond to groups in `groupAllowFrom` or explicitly configured under `groups.<chat_id>` |
| `"disabled"`  | Disable all group messages; explicit `groups.<chat_id>` entries do not override this         |

Default: `allowlist`

**Mention requirement** (`channels.feishu.requireMention`):

- `true` — require @mention (default)
- `false` — respond without @mention
- Per-group override: `channels.feishu.groups.<chat_id>.requireMention`
- Broadcast-only `@all` and `@_all` are not treated as bot mentions. A message that mentions both `@all` and the bot directly still counts as a bot mention.

---

## Group configuration examples

### Allow all groups, no @mention required

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
    },
  },
}
```

### Allow all groups, still require @mention

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true,
    },
  },
}
```

### Allow specific groups only

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      // Group IDs look like: oc_xxx
      groupAllowFrom: ["oc_xxx", "oc_yyy"],
    },
  },
}
```

In `allowlist` mode, you can also admit a group by adding an explicit `groups.<chat_id>` entry. Explicit entries do not override `groupPolicy: "disabled"`. Wildcard defaults under `groups.*` configure matching groups, but they do not admit groups by themselves.

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groups: {
        oc_xxx: {
          requireMention: false,
        },
      },
    },
  },
}
```

### Restrict senders within a group

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          // User open_ids look like: ou_xxx
          allowFrom: ["ou_user1", "ou_user2"],
        },
      },
    },
  },
}
```

---

<a id="get-groupuser-ids"></a>

## Get group/user IDs

### Group IDs (`chat_id`, format: `oc_xxx`)

Open the group in Feishu/Lark, click the menu icon in the top-right corner, and go to **Settings**. The group ID (`chat_id`) is listed on the settings page.

![Get Group ID](/images/feishu-get-group-id.png)

### User IDs (`open_id`, format: `ou_xxx`)

Start the gateway, send a DM to the bot, then check the logs:

```bash
openclaw logs --follow
```

Look for `open_id` in the log output. You can also check pending pairing requests:

```bash
openclaw pairing list feishu
```

---

## Common commands

| Command   | Description                 |
| --------- | --------------------------- |
| `/status` | Show bot status             |
| `/reset`  | Reset the current session   |
| `/model`  | Show or switch the AI model |

Feishu/Lark does not support native slash-command menus, so send these as plain text messages.

---

## Troubleshooting

### Bot does not respond in group chats

1. Ensure the bot is added to the group
2. Ensure you @mention the bot (required by default)
3. Verify `groupPolicy` is not `"disabled"`
4. Check logs: `openclaw logs --follow`

### Bot does not receive messages

1. Ensure the bot is published and approved in Feishu Open Platform / Lark Developer
2. Ensure event subscription includes `im.message.receive_v1`
3. Ensure **persistent connection** (WebSocket) is selected
4. Ensure all required permission scopes are granted
5. Ensure the gateway is running: `openclaw gateway status`
6. Check logs: `openclaw logs --follow`

### App Secret leaked

1. Reset the App Secret in Feishu Open Platform / Lark Developer
2. Update the value in your config
3. Restart the gateway: `openclaw gateway restart`

---

## Advanced configuration

### Multiple accounts

```json5
{
  channels: {
    feishu: {
      defaultAccount: "main",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "Primary bot",
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
        backup: {
          appId: "cli_yyy",
          appSecret: "yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` controls which account is used when outbound APIs do not specify an `accountId`.
`accounts.<id>.tts` uses the same shape as `messages.tts` and deep-merges over
global TTS config, so multi-bot Feishu setups can keep shared provider
credentials globally while overriding only voice, model, persona, or auto mode
per account.

### Message limits

- `textChunkLimit` — outbound text chunk size (default: `2000` chars)
- `mediaMaxMb` — media upload/download limit (default: `30` MB)

### Streaming

Feishu/Lark supports streaming replies via interactive cards. When enabled, the bot updates the card in real time as it generates text.

```json5
{
  channels: {
    feishu: {
      streaming: true, // enable streaming card output (default: true)
      blockStreaming: true, // enable block-level streaming (default: true)
    },
  },
}
```

Set `streaming: false` to send the complete reply in one message.

### Quota optimization

Reduce the number of Feishu/Lark API calls with two optional flags:

- `typingIndicator` (default `true`): set `false` to skip typing reaction calls
- `resolveSenderNames` (default `true`): set `false` to skip sender profile lookups

```json5
{
  channels: {
    feishu: {
      typingIndicator: false,
      resolveSenderNames: false,
    },
  },
}
```

### ACP sessions

Feishu/Lark supports ACP for DMs and group thread messages. Feishu/Lark ACP is text-command driven — there are no native slash-command menus, so use `/acp ...` messages directly in the conversation.

#### Persistent ACP binding

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "direct", id: "ou_1234567890" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "group", id: "oc_group_chat:topic:om_topic_root" },
      },
      acp: { label: "codex-feishu-topic" },
    },
  ],

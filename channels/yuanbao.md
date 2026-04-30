---
domain: channels
topic: "Yuanbao Channel: Setup, Access Control, and Configuration"
type: procedure
keywords:
  - Yuanbao
  - Tencent AI
  - channel setup
related:
  - channels/channel-routing
  - gateway/configuration-overview
source: channels/yuanbao.md
---

Yuanbao channel setup and configuration for OpenClaw.

# Yuanbao

Tencent Yuanbao is Tencent's AI assistant platform. The OpenClaw channel plugin
connects Yuanbao bots to OpenClaw over WebSocket so they can interact with users
through direct messages and group chats.

**Status:** production-ready for bot DMs + group chats. WebSocket is the only supported connection mode.

---

## Quick start

> **Requires OpenClaw 2026.4.10 or above.** Run `openclaw --version` to check. Upgrade with `openclaw update`.

  ```bash
  openclaw channels add --channel yuanbao --token "appKey:appSecret"
  ```
  The `--token` value uses colon-separated `appKey:appSecret` format. You can obtain these from the Yuanbao app by creating a robot in your application settings.

  ```bash
  openclaw gateway restart
  ```

### Interactive setup (alternative)

You can also use the interactive wizard:

```bash
openclaw channels login --channel yuanbao
```

Follow the prompts to enter your App ID and App Secret.

---

## Access control

### Direct messages

Configure `dmPolicy` to control who can DM the bot:

- `"pairing"` — unknown users receive a pairing code; approve via CLI
- `"allowlist"` — only users listed in `allowFrom` can chat
- `"open"` — allow all users (default)
- `"disabled"` — disable all DMs

**Approve a pairing request:**

```bash
openclaw pairing list yuanbao
openclaw pairing approve yuanbao <CODE>
```

### Group chats

**Mention requirement** (`channels.yuanbao.requireMention`):

- `true` — require @mention (default)
- `false` — respond without @mention

Replying to the bot's message in a group chat is treated as an implicit mention.

---

## Configuration examples

### Basic setup with open DM policy

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "open",
      },
    },
  },
}
```

### Restrict DMs to specific users

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "allowlist",
        allowFrom: ["user_id_1", "user_id_2"],
      },
    },
  },
}
```

### Disable @mention requirement in groups

```json5
{
  channels: {
    yuanbao: {
      requireMention: false,
    },
  },
}
```

### Optimize outbound message delivery

```json5
{
  channels: {
    yuanbao: {
      // Send each chunk immediately without buffering
      outboundQueueStrategy: "immediate",
    },
  },
}
```

### Tune merge-text strategy

```json5
{
  channels: {
    yuanbao: {
      outboundQueueStrategy: "merge-text",
      minChars: 2800, // buffer until this many chars
      maxChars: 3000, // force split above this limit
      idleMs: 5000, // auto-flush after idle timeout (ms)
    },
  },
}
```

---

## Common commands

| Command    | Description                 |
| ---------- | --------------------------- |
| `/help`    | Show available commands     |
| `/status`  | Show bot status             |
| `/new`     | Start a new session         |
| `/stop`    | Stop the current run        |
| `/restart` | Restart OpenClaw            |
| `/compact` | Compact the session context |

> Yuanbao supports native slash-command menus. Commands are synced to the platform automatically when the gateway starts.

---

## Troubleshooting

### Bot does not respond in group chats

1. Ensure the bot is added to the group
2. Ensure you @mention the bot (required by default)
3. Check logs: `openclaw logs --follow`

### Bot does not receive messages

1. Ensure the bot is created and approved in the Yuanbao app
2. Ensure `appKey` and `appSecret` are correctly configured
3. Ensure the gateway is running: `openclaw gateway status`
4. Check logs: `openclaw logs --follow`

### Bot sends empty or fallback replies

1. Check if the AI model is returning valid content
2. The default fallback reply is: "暂时无法解答，你可以换个问题问问我哦"
3. Customize it via `channels.yuanbao.fallbackReply`

### App Secret leaked

1. Reset the App Secret in YuanBao APP
2. Update the value in your config
3. Restart the gateway: `openclaw gateway restart`

---

## Advanced configuration

### Multiple accounts

```json5
{
  channels: {
    yuanbao: {
      defaultAccount: "main",
      accounts: {
        main: {
          appKey: "key_xxx",
          appSecret: "secret_xxx",
          name: "Primary bot",
        },
        backup: {
          appKey: "key_yyy",
          appSecret: "secret_yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` controls which account is used when outbound APIs do not specify an `accountId`.

### Message limits

- `maxChars` — single message max character count (default: `3000` chars)
- `mediaMaxMb` — media upload/download limit (default: `20` MB)
- `overflowPolicy` — behavior when message exceeds limit: `"split"` (default) or `"stop"`

### Streaming

Yuanbao supports block-level streaming output. When enabled, the bot sends text in chunks as it generates.

```json5
{
  channels: {
    yuanbao: {
      disableBlockStreaming: false, // block streaming enabled (default)
    },
  },
}
```

Set `disableBlockStreaming: true` to send the complete reply in one message.

### Group chat history context

Control how many historical messages are included in the AI context for group chats:

```json5
{
  channels: {
    yuanbao: {
      historyLimit: 100, // default: 100, set 0 to disable
    },
  },
}
```

### Reply-to mode

Control how the bot quotes messages when replying in group chats:

```json5
{
  channels: {
    yuanbao: {
      replyToMode: "first", // "off" | "first" | "all" (default: "first")
    },
  },
}
```

| Value     | Behavior                                                 |
| --------- | -------------------------------------------------------- |
| `"off"`   | No quote reply                                           |
| `"first"` | Quote only the first reply per inbound message (default) |
| `"all"`   | Quote every reply                                        |

### Markdown hint injection

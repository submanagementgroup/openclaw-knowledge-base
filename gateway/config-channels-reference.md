---
domain: gateway
topic: "Channel Configuration Reference: allowFrom, botToken, groups, requireMention"
type: reference
keywords:
  - channels config
  - allowFrom
  - botToken
  - requireMention
  - groups config
  - channel access control
related:
  - gateway/configuration-overview
  - gateway/config-agents-reference
  - channels/telegram
source: gateway/config-channels.md
---

Channel configuration lives under the `channels` top-level key. Each channel name maps to a config object. Common fields across most channels: `allowFrom`, `token`/`botToken`, `groups`, and `requireMention`.

## Common Channel Config Pattern

```json5
{
  channels: {
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
      allowFrom: ["@username", "123456789"],  // user IDs or usernames
      groups: { "*": { requireMention: true } }
    },
    whatsapp: {
      allowFrom: ["+15555550123"],
    },
    discord: {
      token: "YOUR_DISCORD_BOT_TOKEN",
      guildId: "YOUR_GUILD_ID",
    },
    slack: {
      appToken: "xapp-...",
      botToken: "xoxb-...",
    }
  }
}
```

## Access Control

`allowFrom` accepts:
- Phone numbers: `"+15555550123"`
- Telegram usernames: `"@alice"`
- Telegram user IDs: `"123456789"`
- Discord user IDs or role IDs
- `"*"` to allow all (open/public mode — use with caution)

## Group and Mention Rules

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: true },    // all groups: require mention
        "-1001234567890": { requireMention: false }  // specific group: allow all
      }
    }
  },
  messages: {
    groupChat: {
      mentionPatterns: ["@openclaw", "@bot"]
    }
  }
}
```

The gateway hot-reloads `messages` config after the file is saved. Restart only when file watching or config reload is disabled in the deployment.

Per-channel configuration keys under `channels.*`. Covers DM and group access,
multi-account setups, mention gating, and per-channel keys for Slack, Discord,
Telegram, WhatsApp, Matrix, iMessage, and the other bundled channel plugins.

For agents, tools, gateway runtime, and other top-level keys, see
[Configuration reference](/gateway/configuration-reference).

## Channels

Each channel starts automatically when its config section exists (unless `enabled: false`).

### DM and group access

All channels support DM policies and group policies:

| DM policy           | Behavior                                                        |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (default) | Unknown senders get a one-time pairing code; owner must approve |
| `allowlist`         | Only senders in `allowFrom` (or paired allow store)             |
| `open`              | Allow all inbound DMs (requires `allowFrom: ["*"]`)             |
| `disabled`          | Ignore all inbound DMs                                          |

| Group policy          | Behavior                                               |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (default) | Only groups matching the configured allowlist          |
| `open`                | Bypass group allowlists (mention-gating still applies) |
| `disabled`            | Block all group/room messages                          |

`channels.defaults.groupPolicy` sets the default when a provider's `groupPolicy` is unset.
Pairing codes expire after 1 hour. Pending DM pairing requests are capped at **3 per channel**.
If a provider block is missing entirely (`channels.<provider>` absent), runtime group policy falls back to `allowlist` (fail-closed) with a startup warning.

### Channel model overrides

Use `channels.modelByChannel` to pin specific channel IDs to a model. Values accept `provider/model` or configured model aliases. The channel mapping applies when a session does not already have a model override (for example, set via `/model`).

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-4.1",
      },
      telegram: {
        "-1001234567890": "openai/gpt-4.1-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

### Channel defaults and heartbeat

Use `channels.defaults` for shared group-policy and heartbeat behavior across providers:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: fallback group policy when a provider-level `groupPolicy` is unset.
- `channels.defaults.contextVisibility`: default supplemental context visibility mode for all channels. Values: `all` (default, include all quoted/thread/history context), `allowlist` (only include context from allowlisted senders), `allowlist_quote` (same as allowlist but keep explicit quote/reply context). Per-channel override: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: include healthy channel statuses in heartbeat output.
- `channels.defaults.heartbeat.showAlerts`: include degraded/error statuses in heartbeat output.
- `channels.defaults.heartbeat.useIndicator`: render compact indicator-style heartbeat output.

### WhatsApp

WhatsApp runs through the gateway's web channel (Baileys Web). It starts automatically when a linked session exists.

```json5
{
  web: {
    whatsapp: {
      keepAliveIntervalMs: 25000,
      connectTimeoutMs: 60000,
      defaultQueryTimeoutMs: 60000,
    },
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // blue ticks (false in self-chat mode)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
  web: {
    enabled: true,
    heartbeatSeconds: 60,
    reconnect: {
      initialMs: 2000,
      maxMs: 120000,
      factor: 1.4,
      jitter: 0.2,
      maxAttempts: 0,
    },
  },
}
```

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- Outbound commands default to account `default` if present; otherwise the first configured account id (sorted).
- Optional `channels.whatsapp.defaultAccount` overrides that f

---
domain: channels
topic: "QQ Bot Channel: Setup, Access Control, and Configuration"
type: procedure
keywords:
  - QQ
  - QQBot
  - Tencent QQ
  - channel setup
related:
  - channels/channel-routing
  - gateway/configuration-overview
source: channels/qqbot.md
---

QQ Bot channel setup and configuration for OpenClaw.

QQ Bot connects to OpenClaw via the official QQ Bot API (WebSocket gateway). The
plugin supports C2C private chat, group @messages, and guild channel messages with
rich media (images, voice, video, files).

Status: bundled plugin. Direct messages, group chats, guild channels, and
media are supported. Reactions and threads are not supported.

## Bundled plugin

Current OpenClaw releases bundle QQ Bot, so normal packaged builds do not need
a separate `openclaw plugins install` step.

## Setup

1. Go to the [QQ Open Platform](https://q.qq.com/) and scan the QR code with your
   phone QQ to register / log in.
2. Click **Create Bot** to create a new QQ bot.
3. Find **AppID** and **AppSecret** on the bot's settings page and copy them.

> AppSecret is not stored in plaintext — if you leave the page without saving it,
> you'll have to regenerate a new one.

4. Add the channel:

```bash
openclaw channels add --channel qqbot --token "AppID:AppSecret"
```

5. Restart the Gateway.

Interactive setup paths:

```bash
openclaw channels add
openclaw configure --section channels
```

## Configure

Minimal config:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: "YOUR_APP_SECRET",
    },
  },
}
```

Default-account env vars:

- `QQBOT_APP_ID`
- `QQBOT_CLIENT_SECRET`

File-backed AppSecret:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecretFile: "/path/to/qqbot-secret.txt",
    },
  },
}
```

Notes:

- Env fallback applies to the default QQ Bot account only.
- `openclaw channels add --channel qqbot --token-file ...` provides the
  AppSecret only; the AppID must already be set in config or `QQBOT_APP_ID`.
- `clientSecret` also accepts SecretRef input, not just a plaintext string.

### Multi-account setup

Run multiple QQ bots under a single OpenClaw instance:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "111111111",
      clientSecret: "secret-of-bot-1",
      accounts: {
        bot2: {
          enabled: true,
          appId: "222222222",
          clientSecret: "secret-of-bot-2",
        },
      },
    },
  },
}
```

Each account launches its own WebSocket connection and maintains an independent
token cache (isolated by `appId`).

Add a second bot via CLI:

```bash
openclaw channels add --channel qqbot --account bot2 --token "222222222:secret-of-bot-2"
```

### Group chats

QQ Bot group chat support uses QQ group OpenIDs, not display names. Add the bot
to a group, then mention it or configure the group to run without a mention.

```json5
{
  channels: {
    qqbot: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["member_openid"],
      groups: {
        "*": {
          requireMention: true,
          historyLimit: 50,
          toolPolicy: "restricted",
        },
        GROUP_OPENID: {
          name: "Release room",
          requireMention: false,
          ignoreOtherMentions: true,
          historyLimit: 20,
          prompt: "Keep replies short and operational.",
        },
      },
    },
  },
}
```

`groups["*"]` sets defaults for every group, and a concrete
`groups.GROUP_OPENID` entry overrides those defaults for one group. Group
settings include:

- `requireMention`: require an @mention before the bot replies. Default: `true`.
- `ignoreOtherMentions`: drop messages that mention someone else but not the bot.
- `historyLimit`: keep recent non-mention group messages as context for the next mentioned turn. Set `0` to disable.
- `toolPolicy`: `full`, `restricted`, or `none` for group-scoped tools.
- `name`: friendly label used in logs and group context.
- `prompt`: per-group behavior prompt appended to the agent context.

Activation modes are `mention` and `always`. `requireMention: true` maps to
`mention`; `requireMention: false` maps to `always`. A session-level activation
override, when present, wins over config.

The inbound queue is per peer. Group peers get a larger queue cap, keep human
messages ahead of bot-authored chatter when full, and merge bursts of normal
group messages into one attributed turn. Slash commands still run one by one.

### Voice (STT / TTS)

STT and TTS support two-level configuration with priority fallback:

| Setting | Plugin-specific                                          | Framework fallback            |
| ------- | -------------------------------------------------------- | ----------------------------- |
| STT     | `channels.qqbot.stt`                                     | `tools.media.audio.models[0]` |
| TTS     | `channels.qqbot.tts`, `channels.qqbot.accounts.<id>.tts` | `messages.tts`                |

```json5
{
  channels: {
    qqbot: {
      stt: {
        provider: "your-provider",
        model: "your-stt-model",
      },
      tts: {
        provider: "your-provider",
        model: "your-tts-model",
        voice: "your-voice",
      },
      accounts: {
        qq-main: {
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

Set `enabled: false` on either to disable.
Account-level TTS overrides use the same shape as `messages.tts` and deep-merge
over the channel/global TTS config.

Inbound QQ voice attachments are exposed to agents as audio media metadata while
keeping raw voice files out of generic `MediaPaths`. `[[audio_as_voice]]` plain
text replies synthesize TTS and send a native QQ voice message when TTS is
configured.

Outbound audio upload/transcode behavior can also be tuned with
`channels.qqbot.audioFormatPolicy`:

- `sttDirectFormats`
- `uploadDirectFormats`
- `transcodeEnabled`

## Target formats

| Format                     | Description        |
| -------------------------- | ------------------ |
| `qqbot:c2c:OPENID`         | Private chat (C2C) |
| `qqbot:group:GROUP_OPENID` | Group chat         |
| `qqbot:channel:CHANNEL_ID` | Guild channel      |

> Each bot has its own set of user O

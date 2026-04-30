---
domain: channels
topic: "Twitch Channel: Setup, Access Control, and Configuration"
type: procedure
keywords:
  - Twitch
  - Twitch chat
  - Twitch bot
  - channel setup
related:
  - channels/channel-routing
  - gateway/configuration-overview
source: channels/twitch.md
---

Twitch channel setup and configuration for OpenClaw.

Twitch chat support via IRC connection. OpenClaw connects as a Twitch user (bot account) to receive and send messages in channels.

## Bundled plugin

Twitch ships as a bundled plugin in current OpenClaw releases, so normal packaged builds do not need a separate install.

If you are on an older build or a custom install that excludes Twitch, install a current npm package when one is published:

    ```bash
    openclaw plugins install @openclaw/twitch
    ```

    ```bash
    openclaw plugins install ./path/to/local/twitch-plugin
    ```

If npm reports the OpenClaw-owned package as deprecated, use a current packaged
OpenClaw build or the local checkout path until a newer npm package is
published.

Details: [Plugins](/tools/plugin)

## Quick setup (beginner)

    Current packaged OpenClaw releases already bundle it. Older/custom installs can add it manually with the commands above.

    Create a dedicated Twitch account for the bot (or use an existing account).

    Use [Twitch Token Generator](https://twitchtokengenerator.com/):

    - Select **Bot Token**
    - Verify scopes `chat:read` and `chat:write` are selected
    - Copy the **Client ID** and **Access Token**

    Use [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) to convert a username to a Twitch user ID.

    - Env: `OPENCLAW_TWITCH_ACCESS_TOKEN=...` (default account only)
    - Or config: `channels.twitch.accessToken`

    If both are set, config takes precedence (env fallback is default-account only).

    Start the gateway with the configured channel.

Add access control (`allowFrom` or `allowedRoles`) to prevent unauthorized users from triggering the bot. `requireMention` defaults to `true`.

Minimal config:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // Bot's Twitch account
      accessToken: "oauth:abc123...", // OAuth Access Token (or use OPENCLAW_TWITCH_ACCESS_TOKEN env var)
      clientId: "xyz789...", // Client ID from Token Generator
      channel: "vevisk", // Which Twitch channel's chat to join (required)
      allowFrom: ["123456789"], // (recommended) Your Twitch user ID only - get it from https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/
    },
  },
}
```

## What it is

- A Twitch channel owned by the Gateway.
- Deterministic routing: replies always go back to Twitch.
- Each account maps to an isolated session key `agent:<agentId>:twitch:<accountName>`.
- `username` is the bot's account (who authenticates), `channel` is which chat room to join.

## Setup (detailed)

### Generate credentials

Use [Twitch Token Generator](https://twitchtokengenerator.com/):

- Select **Bot Token**
- Verify scopes `chat:read` and `chat:write` are selected
- Copy the **Client ID** and **Access Token**

No manual app registration needed. Tokens expire after several hours.

### Configure the bot

    ```bash
    OPENCLAW_TWITCH_ACCESS_TOKEN=oauth:abc123...
    ```

    ```json5
    {
      channels: {
        twitch: {
          enabled: true,
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "vevisk",
        },
      },
    }
    ```

If both env and config are set, config takes precedence.

### Access control (recommended)

```json5
{
  channels: {
    twitch: {
      allowFrom: ["123456789"], // (recommended) Your Twitch user ID only
    },
  },
}
```

Prefer `allowFrom` for a hard allowlist. Use `allowedRoles` instead if you want role-based access.

**Available roles:** `"moderator"`, `"owner"`, `"vip"`, `"subscriber"`, `"all"`.

**Why user IDs?** Usernames can change, allowing impersonation. User IDs are permanent.

Find your Twitch user ID: [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) (Convert your Twitch username to ID)

## Token refresh (optional)

Tokens from [Twitch Token Generator](https://twitchtokengenerator.com/) cannot be automatically refreshed - regenerate when expired.

For automatic token refresh, create your own Twitch application at [Twitch Developer Console](https://dev.twitch.tv/console) and add to config:

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

The bot automatically refreshes tokens before expiration and logs refresh events.

## Multi-account support

Use `channels.twitch.accounts` with per-account tokens. See [Configuration](/gateway/configuration) for the shared pattern.

Example (one bot account in two channels):

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "vevisk",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

Each account needs its own token (one token per channel).

## Access control

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowFrom: ["123456789", "987654321"],
            },
          },
        },
      },
    }
    ```

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowedRoles: ["moderator", "vip"],
            },
          },
        },
      },
    }
    ```

    `allowFrom` is a hard allowlist. When set, only those user IDs are allowed. If you want role-based access, leave `allowFrom` unset and configure `allowedRoles` instead.

    By default, `requireMention` is `true`. To disable and respond to all messages:

    ```json5
    {
      channels: {
        twitch: {
          accounts: {

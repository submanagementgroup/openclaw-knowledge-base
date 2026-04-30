---
domain: channels
topic: "Tlon Landscape Channel: Setup, Access Control, and Configuration"
type: procedure
keywords:
  - Tlon
  - Urbit
  - Landscape
  - channel setup
related:
  - channels/channel-routing
  - gateway/configuration-overview
source: channels/tlon.md
---

Tlon Landscape channel setup and configuration for OpenClaw.

Tlon is a decentralized messenger built on Urbit. OpenClaw connects to your Urbit ship and can
respond to DMs and group chat messages. Group replies require an @ mention by default and can
be further restricted via allowlists.

Status: bundled plugin. DMs, group mentions, thread replies, rich text formatting, and
image uploads are supported. Reactions and polls are not yet supported.

## Bundled plugin

Tlon ships as a bundled plugin in current OpenClaw releases, so normal packaged
builds do not need a separate install.

If you are on an older build or a custom install that excludes Tlon, install a
current npm package when one is published:

Install via CLI (npm registry, when a current package exists):

```bash
openclaw plugins install @openclaw/tlon
```

If npm reports the OpenClaw-owned package as deprecated, use a current packaged
OpenClaw build or the local checkout path until a newer npm package is
published.

Local checkout (when running from a git repo):

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

Details: [Plugins](/tools/plugin)

## Setup

1. Ensure the Tlon plugin is available.
   - Current packaged OpenClaw releases already bundle it.
   - Older/custom installs can add it manually with the commands above.
2. Gather your ship URL and login code.
3. Configure `channels.tlon`.
4. Restart the gateway.
5. DM the bot or mention it in a group channel.

Minimal config (single account):

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // recommended: your ship, always allowed
    },
  },
}
```

## Private/LAN ships

By default, OpenClaw blocks private/internal hostnames and IP ranges for SSRF protection.
If your ship is running on a private network (localhost, LAN IP, or internal hostname),
you must explicitly opt in:

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      allowPrivateNetwork: true,
    },
  },
}
```

This applies to URLs like:

- `http://localhost:8080`
- `http://192.168.x.x:8080`
- `http://my-ship.local:8080`

⚠️ Only enable this if you trust your local network. This setting disables SSRF protections
for requests to your ship URL.

## Group channels

Auto-discovery is enabled by default. You can also pin channels manually:

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
    },
  },
}
```

Disable auto-discovery:

```json5
{
  channels: {
    tlon: {
      autoDiscoverChannels: false,
    },
  },
}
```

## Access control

DM allowlist (empty = no DMs allowed, use `ownerShip` for approval flow):

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

Group authorization (restricted by default):

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

## Owner and approval system

Set an owner ship to receive approval requests when unauthorized users try to interact:

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

The owner ship is **automatically authorized everywhere** — DM invites are auto-accepted and
channel messages are always allowed. You don't need to add the owner to `dmAllowlist` or
`defaultAuthorizedShips`.

When set, the owner receives DM notifications for:

- DM requests from ships not in the allowlist
- Mentions in channels without authorization
- Group invite requests

## Auto-accept settings

Auto-accept DM invites (for ships in dmAllowlist):

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

Auto-accept group invites:

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
    },
  },
}
```

## Delivery targets (CLI/cron)

Use these with `openclaw message send` or cron delivery:

- DM: `~sampel-palnet` or `dm/~sampel-palnet`
- Group: `chat/~host-ship/channel` or `group:~host-ship/channel`

## Bundled skill

The Tlon plugin includes a bundled skill ([`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill))
that provides CLI access to Tlon operations:

- **Contacts**: get/update profiles, list contacts
- **Channels**: list, create, post messages, fetch history
- **Groups**: list, create, manage members
- **DMs**: send messages, react to messages
- **Reactions**: add/remove emoji reactions to posts and DMs
- **Settings**: manage plugin permissions via slash commands

The skill is automatically available when the plugin is installed.

## Capabilities

| Feature         | Status                                  |
| --------------- | --------------------------------------- |
| Direct messages | ✅ Supported                            |
| Groups/channels | ✅ Supported (mention-gated by default) |
| Threads         | ✅ Supported (auto-replies in thread)   |
| Rich text       | ✅ Markdown converted to Tlon format    |
| Images          | ✅ Uploaded to Tlon storage             |
| Reactions       | ✅ Via [bundled skill](#bundled-skill)  |
| Polls           | ❌ Not yet supported                    |
| Native commands | ✅ Supported (owner-only by default)    |

## Troubleshooting

Run this ladder first:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

Common failures:

- **DMs ignored**: sender not in `dmAllowlist` and no `ownerShip` configured for approval flow.
- **Group messages ignored**: channel not discovered or sender not authorized.
- **Connection errors**: check ship URL is reachable; enable `allowPrivateNetwork` for local ships.
- **Auth errors**: verify login code is current (codes rotate

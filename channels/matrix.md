---
domain: channels
topic: "Matrix Channel: Homeserver Setup, E2E Encryption, Threads, Push Rules, and ACP Bindings"
type: procedure
keywords:
  - Matrix
  - Matrix channel
  - Matrix bot
  - homeserver
  - Matrix encryption
  - E2E encryption
  - Matrix threads
related:
  - channels/mattermost
  - channels/slack
  - gateway/configuration-overview
source: channels/matrix.md
---

Matrix channel for OpenClaw. Connects to a Matrix homeserver as a bot account. Supports E2E encryption, threads, push rules, and ACP conversation bindings.

## Setup

```bash
openclaw channels setup matrix   # interactive setup
```

Or manually:
```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.org",
      userId: "@mybot:matrix.org",
      accessToken: "YOUR_ACCESS_TOKEN",
      allowFrom: ["@user:matrix.org"],
    }
  }
}
```

## Features

Matrix is a bundled channel plugin for OpenClaw.
It uses the official `matrix-js-sdk` and supports DMs, rooms, threads, media, reactions, polls, location, and E2EE.

## Bundled plugin

Current packaged OpenClaw releases ship the Matrix plugin in the box. You do not need to install anything; configuring `channels.matrix.*` (see [Setup](#setup)) is what activates it.

For older builds or custom installs that exclude Matrix, install a current npm
package when one is published:

```bash
openclaw plugins install @openclaw/matrix
```

If npm reports the OpenClaw-owned package as deprecated, use a current packaged
OpenClaw build or a local checkout until a newer npm package is published.

From a local checkout:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

`plugins install` registers and enables the plugin, so no separate `openclaw plugins enable matrix` step is needed. The plugin still does nothing until you configure the channel below. See [Plugins](/tools/plugin) for general plugin behavior and install rules.

## Setup

1. Create a Matrix account on your homeserver.
2. Configure `channels.matrix` with either `homeserver` + `accessToken`, or `homeserver` + `userId` + `password`.
3. Restart the gateway.
4. Start a DM with the bot, or invite it to a room (see [auto-join](#auto-join) — fresh invites only land when `autoJoin` allows them).

### Interactive setup

```bash
openclaw channels add
openclaw configure --section channels
```

The wizard asks for: homeserver URL, auth method (access token or password), user ID (password auth only), optional device name, whether to enable E2EE, and whether to configure room access and auto-join.

If matching `MATRIX_*` env vars already exist and the selected account has no saved auth, the wizard offers an env-var shortcut. To resolve room names before saving an allowlist, run `openclaw channels resolve --channel matrix "Project Room"`. When E2EE is enabled, the wizard writes the config and runs the same bootstrap as [`openclaw matrix encryption setup`](#encryption-and-verification).

### Minimal config

Token-based:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Password-based (the token is cached after first login):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

### Auto-join

`channels.matrix.autoJoin` defaults to `off`. With the default, the bot will not appear in new rooms or DMs from fresh invites until you join manually.

OpenClaw cannot tell at invite time whether an invited room is a DM or a group, so all invites — including DM-style invites — go through `autoJoin` first. `dm.policy` only applies later, after the bot has joined and the room has been classified.

Set `autoJoin: "allowlist"` plus `autoJoinAllowlist` to restrict which invites the bot accepts, or `autoJoin: "always"` to accept every invite.

`autoJoinAllowlist` only accepts stable targets: `!roomId:server`, `#alias:server`, or `*`. Plain room names are rejected; alias entries are resolved against the homeserver, not against state claimed by the invited room.

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": { requireMention: true },
      },
    },
  },
}
```

To accept every invite, use `autoJoin: "always"`.

### Allowlist target formats

DM and room allowlists are best populated with stable IDs:

- DMs (`dm.allowFrom`, `groupAllowFrom`, `groups.<room>.users`): use `@user:server`. Display names only resolve when the homeserver directory returns exactly one match.
- Rooms (`groups`, `autoJoinAllowlist`): use `!room:server` or `#alias:server`. Names are resolved best-effort against joined rooms; unresolved entries are ignored at runtime.

### Account ID normalization

The wizard converts a friendly name into a normalized account ID. For example, `Ops Bot` becomes `ops-bot`. Punctuation is escaped in scoped env-var names so that two accounts cannot collide: `-` → `_X2D_`, so `ops-prod` maps to `MATRIX_OPS_X2D_PROD_*`.

### Cached credentials

Matrix stores cached credentials under `~/.openclaw/credentials/matrix/`:

- default account: `credentials.json`
- named accounts: `credentials-<account>.json`

When cached credentials exist there, OpenClaw treats Matrix as configured even if the access token is not in the config file — that covers setup, `openclaw doctor`, and channel-status probes.

### Environment variables

Used when the equivalent config key is not set. The default account uses unprefixed names; named accounts use the account ID inserted before the suffix.

| Default account       | Named accoun

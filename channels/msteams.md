---
domain: channels
topic: "Microsoft Teams Channel: Azure Bot Setup, App Manifest, Federated Auth, and Local Dev"
type: procedure
keywords:
  - Microsoft Teams
  - Teams
  - Azure Bot
  - msteams
  - Teams channel
  - Teams manifest
  - Bot Framework
related:
  - channels/slack
  - channels/discord
  - gateway/configuration-overview
source: channels/msteams.md
---

Microsoft Teams channel for OpenClaw. Requires an Azure Bot, Teams manifest, and a public HTTPS endpoint.

## Overview

Teams requires:
1. An Azure Bot (registered in Azure Portal)
2. A messaging endpoint (public HTTPS URL)
3. A Teams app manifest
4. OpenClaw configured with App ID and password

## Setup Steps

Status: text + DM attachments are supported; channel/group file sending requires `sharePointSiteId` + Graph permissions (see [Sending files in group chats](#sending-files-in-group-chats)). Polls are sent via Adaptive Cards. Message actions expose explicit `upload-file` for file-first sends.

## Bundled plugin

Microsoft Teams ships as a bundled plugin in current OpenClaw releases, so no
separate install is required in the normal packaged build.

If you are on an older build or a custom install that excludes bundled Teams,
install a current npm package when one is published:

```bash
openclaw plugins install @openclaw/msteams
```

If npm reports the OpenClaw-owned package as deprecated, use a current packaged
OpenClaw build or the local checkout path until a newer npm package is
published.

Local checkout (when running from a git repo):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

Details: [Plugins](/tools/plugin)

## Quick setup

The [`@microsoft/teams.cli`](https://www.npmjs.com/package/@microsoft/teams.cli) handles bot registration, manifest creation, and credential generation in a single command.

**1. Install and log in**

```bash
npm install -g @microsoft/teams.cli@preview
teams login
teams status   # verify you're logged in and see your tenant info
```

The Teams CLI is currently in preview. Commands and flags may change between releases.

**2. Start a tunnel** (Teams can't reach localhost)

Install and authenticate the devtunnel CLI if you haven't already ([getting started guide](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started)).

```bash
# One-time setup (persistent URL across sessions):
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# Each dev session:
devtunnel host my-openclaw-bot
# Your endpoint: https://<tunnel-id>.devtunnels.ms/api/messages
```

`--allow-anonymous` is required because Teams cannot authenticate with devtunnels. Each incoming bot request is still validated by the Teams SDK automatically.

Alternatives: `ngrok http 3978` or `tailscale funnel 3978` (but these may change URLs each session).

**3. Create the app**

```bash
teams app create \
  --name "OpenClaw" \
  --endpoint "https://<your-tunnel-url>/api/messages"
```

This single command:

- Creates an Entra ID (Azure AD) application
- Generates a client secret
- Builds and uploads a Teams app manifest (with icons)
- Registers the bot (Teams-managed by default — no Azure subscription needed)

The output will show `CLIENT_ID`, `CLIENT_SECRET`, `TENANT_ID`, and a **Teams App ID** — note these for the next steps. It also offers to install the app in Teams directly.

**4. Configure OpenClaw** using the credentials from the output:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<CLIENT_ID>",
      appPassword: "<CLIENT_SECRET>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Or use environment variables directly: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

**5. Install the app in Teams**

`teams app create` will prompt you to install the app — select "Install in Teams". If you skipped it, you can get the link later:

```bash
teams app get <teamsAppId> --install-link
```

**6. Verify everything works**

```bash
teams app doctor <teamsAppId>
```

This runs diagnostics across bot registration, AAD app config, manifest validity, and SSO setup.

For production deployments, consider using [federated authentication](/channels/msteams#federated-authentication-certificate-plus-managed-identity) (certificate or managed identity) instead of client secrets.

Group chats are blocked by default (`channels.msteams.groupPolicy: "allowlist"`). To allow group replies, set `channels.msteams.groupAllowFrom`, or use `groupPolicy: "open"` to allow any member (mention-gated).

## Goals

- Talk to OpenClaw via Teams DMs, group chats, or channels.
- Keep routing deterministic: replies always go back to the channel they arrived on.
- Default to safe channel behavior (mentions required unless configured otherwise).

## Config writes

By default, Microsoft Teams is allowed to write config updates triggered by `/config set|unset` (requires `commands.config: true`).

Disable with:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## Access control (DMs + groups)

**DM access**

- Default: `channels.msteams.dmPolicy = "pairing"`. Unknown senders are ignored until approved.
- `channels.msteams.allowFrom` should use stable AAD object IDs.
- Do not rely on UPN/display-name matching for allowlists — they can change. OpenClaw disables direct name matching by default; opt in explicitly with `channels.msteams.dangerouslyAllowNameMatching: true`.
- The wizard can resolve names to IDs via Microsoft Graph when credentials allow.

**Group access**

- Default: `channels.msteams.groupPolicy = "allowlist"` (blocked unless you add `groupAllowFrom`). Use `channels.defaults.groupPolicy` to override the default when unset.
- `channels.msteams.groupAllowFrom` controls which senders can trigger in group chats/channels (falls back to `channels.msteams.allowFrom`).
- Set `groupPolicy: "open"` to allow any member (still mention‑gated by default).
- To allow **no channels**, set `channels.msteams.groupPolicy: "disabled"`.

Example:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["user@org.com"],
    },
  },
}
```

**Teams + channel allowlist**

- Scope group/channel replies by listing teams and channels under `channels.msteams.teams`.
- Keys should use stable Teams conversation IDs from Teams links, not mutable display names.
- When `groupPolicy="allowlist"` and a teams allowlist is present, only listed teams/channels are accepted (mention‑gated).
- The configure wizard accepts `Team/Channel` entries and stores them for you.
- On startup, OpenClaw resolves team/channel and user allowlist names to IDs (when Graph permissions allow)
  and logs the mapping; unresolved team/channel names are kept as typed but ignored for routing by default unless `channels.msteams.dangerouslyAllowNameMatching: true` is enabled.

Example:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

<details>
<summary><strong>Manual setup (without the Teams CLI)</strong></summary>

If you can't use the Teams CLI, you can set up the bot manually through the Azure Portal.

### How it works

1. Ensure the Microsoft Teams plugin is available (bundled in current releases).
2. Create an **Azure Bot** (App ID + secret + tenant ID).
3. Build a **Teams app package** that references the bot and includes the RSC permissions below.
4. Upload/install the Teams app into a team (or personal scope for DMs).
5. Configure `msteams` in `~/

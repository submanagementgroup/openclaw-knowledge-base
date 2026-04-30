---
domain: channels
topic: "Zalo Channel: Setup, Access Control, and Configuration"
type: procedure
keywords:
  - Zalo
  - Vietnamese messaging
  - channel setup
related:
  - channels/channel-routing
  - gateway/configuration-overview
source: channels/zalo.md
---

Zalo channel setup and configuration for OpenClaw.

Status: experimental. DMs are supported. The [Capabilities](#capabilities) section below reflects current Marketplace-bot behavior.

## Bundled plugin

Zalo ships as a bundled plugin in current OpenClaw releases, so normal packaged
builds do not need a separate install.

If you are on an older build or a custom install that excludes Zalo, install a
current npm package when one is published:

- Install via CLI: `openclaw plugins install @openclaw/zalo`
- Or from a source checkout: `openclaw plugins install ./path/to/local/zalo-plugin`
- Details: [Plugins](/tools/plugin)

If npm reports the OpenClaw-owned package as deprecated, use a current packaged
OpenClaw build or the local checkout path until a newer npm package is
published.

## Quick setup (beginner)

1. Ensure the Zalo plugin is available.
   - Current packaged OpenClaw releases already bundle it.
   - Older/custom installs can add it manually with the commands above.
2. Set the token:
   - Env: `ZALO_BOT_TOKEN=...`
   - Or config: `channels.zalo.accounts.default.botToken: "..."`.
3. Restart the gateway (or finish setup).
4. DM access is pairing by default; approve the pairing code on first contact.

Minimal config:

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "12345689:abc-xyz",
          dmPolicy: "pairing",
        },
      },
    },
  },
}
```

## What it is

Zalo is a Vietnam-focused messaging app; its Bot API lets the Gateway run a bot for 1:1 conversations.
It is a good fit for support or notifications where you want deterministic routing back to Zalo.

This page reflects current OpenClaw behavior for **Zalo Bot Creator / Marketplace bots**.
**Zalo Official Account (OA) bots** are a different Zalo product surface and may behave differently.

- A Zalo Bot API channel owned by the Gateway.
- Deterministic routing: replies go back to Zalo; the model never chooses channels.
- DMs share the agent's main session.
- The [Capabilities](#capabilities) section below shows current Marketplace-bot support.

## Setup (fast path)

### 1) Create a bot token (Zalo Bot Platform)

1. Go to [https://bot.zaloplatforms.com](https://bot.zaloplatforms.com) and sign in.
2. Create a new bot and configure its settings.
3. Copy the full bot token (typically `numeric_id:secret`). For Marketplace bots, the usable runtime token may appear in the bot's welcome message after creation.

### 2) Configure the token (env or config)

Example:

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "12345689:abc-xyz",
          dmPolicy: "pairing",
        },
      },
    },
  },
}
```

If you later move to a Zalo bot surface where groups are available, you can add group-specific config such as `groupPolicy` and `groupAllowFrom` explicitly. For current Marketplace-bot behavior, see [Capabilities](#capabilities).

Env option: `ZALO_BOT_TOKEN=...` (works for the default account only).

Multi-account support: use `channels.zalo.accounts` with per-account tokens and optional `name`.

3. Restart the gateway. Zalo starts when a token is resolved (env or config).
4. DM access defaults to pairing. Approve the code when the bot is first contacted.

## How it works (behavior)

- Inbound messages are normalized into the shared channel envelope with media placeholders.
- Replies always route back to the same Zalo chat.
- Long-polling by default; webhook mode available with `channels.zalo.webhookUrl`.

## Limits

- Outbound text is chunked to 2000 characters (Zalo API limit).
- Media downloads/uploads are capped by `channels.zalo.mediaMaxMb` (default 5).
- Streaming is blocked by default due to the 2000 char limit making streaming less useful.

## Access control (DMs)

### DM access

- Default: `channels.zalo.dmPolicy = "pairing"`. Unknown senders receive a pairing code; messages are ignored until approved (codes expire after 1 hour).
- Approve via:
  - `openclaw pairing list zalo`
  - `openclaw pairing approve zalo <CODE>`
- Pairing is the default token exchange. Details: [Pairing](/channels/pairing)
- `channels.zalo.allowFrom` accepts numeric user IDs (no username lookup available).

## Access control (Groups)

For **Zalo Bot Creator / Marketplace bots**, group support was not available in practice because the bot could not be added to a group at all.

That means the group-related config keys below exist in the schema, but were not usable for Marketplace bots:

- `channels.zalo.groupPolicy` controls group inbound handling: `open | allowlist | disabled`.
- `channels.zalo.groupAllowFrom` restricts which sender IDs can trigger the bot in groups.
- If `groupAllowFrom` is unset, Zalo falls back to `allowFrom` for sender checks.
- Runtime note: if `channels.zalo` is missing entirely, runtime still falls back to `groupPolicy="allowlist"` for safety.

The group policy values (when group access is available on your bot surface) are:

- `groupPolicy: "disabled"` — blocks all group messages.
- `groupPolicy: "open"` — allows any group member (mention-gated).
- `groupPolicy: "allowlist"` — fail-closed default; only allowed senders are accepted.

If you are using a different Zalo bot product surface and have verified working group behavior, document that separately rather than assuming it matches the Marketplace-bot flow.

## Long-polling vs webhook

- Default: long-polling (no public URL required).
- Webhook mode: set `channels.zalo.webhookUrl` and `channels.zalo.webhookSecret`.
  - The webhook secret must be 8-256 characters.
  - Webhook URL must use HTTPS.
  - Zalo sends events with `X-Bot-Api-Secret-Token` header for verification.
  - Gateway HTTP handles webhook requests at `channels.zalo.webhookPath` (defaults to the webhook URL path).
  - Requests must use `Content-Type: application/json` (or `+json` media types).
  - Duplicate events (`event_name + message_id`) are ignored for a short replay window.
  - Burst traffic is rate-limited per path/source a

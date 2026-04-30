---
domain: channels
topic: "Telegram Channel Setup: Bot Creation, Access Control, Groups, and Feature Reference"
type: procedure
keywords:
  - Telegram
  - telegram bot
  - BotFather
  - botToken
  - Telegram channel
  - allowFrom
  - Telegram groups
related:
  - channels/whatsapp
  - channels/discord
  - gateway/configuration-overview
source: channels/telegram.md
---

Telegram is the fastest channel to set up in OpenClaw. Create a bot via @BotFather, add the token to config, and you're ready.

## Quick Setup

1. Message [@BotFather](https://t.me/botfather) → `/newbot` → copy the token
2. Add to `openclaw.json`:
   ```json5
   {
     channels: {
       telegram: {
         botToken: "YOUR_BOT_TOKEN",
         allowFrom: ["@yourusername"],
       }
     }
   }
   ```
3. Start/restart the Gateway: `openclaw gateway --force`
4. Message your bot on Telegram

## Access Control

- `allowFrom`: array of Telegram usernames (`@alice`) or user IDs (`"123456789"`)
- Groups: set `groups."*".requireMention: true` to require mentions in group chats

## Full Configuration and Features

Production-ready for bot DMs and groups via grammY. Long polling is the default mode; webhook mode is optional.

    Default DM policy for Telegram is pairing.

    Cross-channel diagnostics and repair playbooks.

    Full channel config patterns and examples.

## Quick setup

    Open Telegram and chat with **@BotFather** (confirm the handle is exactly `@BotFather`).

    Run `/newbot`, follow prompts, and save the token.

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    Env fallback: `TELEGRAM_BOT_TOKEN=...` (default account only).
    Telegram does **not** use `openclaw channels login telegram`; configure token in config/env, then start gateway.

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    Pairing codes expire after 1 hour.

    Add the bot to your group, then set `channels.telegram.groups` and `groupPolicy` to match your access model.

Token resolution order is account-aware. In practice, config values win over env fallback, and `TELEGRAM_BOT_TOKEN` only applies to the default account.

## Telegram side settings

    Telegram bots default to **Privacy Mode**, which limits what group messages they receive.

    If the bot must see all group messages, either:

    - disable privacy mode via `/setprivacy`, or
    - make the bot a group admin.

    When toggling privacy mode, remove + re-add the bot in each group so Telegram applies the change.

    Admin status is controlled in Telegram group settings.

    Admin bots receive all group messages, which is useful for always-on group behavior.

    - `/setjoingroups` to allow/deny group adds
    - `/setprivacy` for group visibility behavior

## Access control and activation

    `channels.telegram.dmPolicy` controls direct message access:

    - `pairing` (default)
    - `allowlist` (requires at least one sender ID in `allowFrom`)
    - `open` (requires `allowFrom` to include `"*"`)
    - `disabled`

    `dmPolicy: "open"` with `allowFrom: ["*"]` lets any Telegram account that finds or guesses the bot username command the bot. Use it only for intentionally public bots with tightly restricted tools; one-owner bots should use `allowlist` with numeric user IDs.

    `channels.telegram.allowFrom` accepts numeric Telegram user IDs. `telegram:` / `tg:` prefixes are accepted and normalized.
    In multi-account configs, a restrictive top-level `channels.telegram.allowFrom` is treated as a safety boundary: account-level `allowFrom: ["*"]` entries do not make that account public unless the effective account allowlist still contains an explicit wildcard after merging.
    `dmPolicy: "allowlist"` with empty `allowFrom` blocks all DMs and is rejected by config validation.
    Setup asks for numeric user IDs only.
    If you upgraded and your config contains `@username` allowlist entries, run `openclaw doctor --fix` to resolve them (best-effort; requires a Telegram bot token).
    If you previously relied on pairing-store allowlist files, `openclaw doctor --fix` can recover entries into `channels.telegram.allowFrom` in allowlist flows (for example when `dmPolicy: "allowlist"` has no explicit IDs yet).

    For one-owner bots, prefer `dmPolicy: "allowlist"` with explicit numeric `allowFrom` IDs to keep access policy durable in config (instead of depending on previous pairing approvals).

    Common confusion: DM pairing approval does not mean "this sender is authorized everywhere".
    Pairing grants DM access. If no command owner exists yet, the first approved pairing also sets `commands.ownerAllowFrom` so owner-only commands and exec approvals have an explicit operator account.
    Group sender authorization still comes from explicit config allowlists.
    If you want "I am authorized once and both DMs and group commands work", put your numeric Telegram user ID in `channels.telegram.allowFrom`; for owner-only commands, make sure `commands.ownerAllowFrom` contains `telegram:<your user id>`.

    ### Finding your Telegram user ID

    Safer (no third-party bot):

    1. DM your bot.
    2. Run `openclaw logs --follow`.
    3. Read `from.id`.

    Official Bot API method:

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    Third-party method (less private): `@userinfobot` or `@getidsbot`.

    Two controls apply together:

    1. **Which groups are allowed** (`channels.telegram.groups`)
       - no `groups` config:
         - with `groupPolicy: "open"`: any group can pass group-ID checks
         - with `groupPolicy: "allowlist"` (default): groups are blocked until you add `groups` entries (or `"*"`)
       - `groups` configured: acts as allowlist (explicit IDs or `"*"`)

    2. **Which senders are allowed in groups** (`channels.telegram.groupPolicy`)
       - `open`
       - `allowlist` (default)
       - `disabled`

    `groupAllowFrom` is used for group sender filtering. If not set, Telegram falls back to `allowFrom`.
    `groupAllowFrom` entries should be numeric Telegram user IDs (`telegram:` / `tg:` prefixes are normalized).
    Do not put Telegram group or supergroup chat IDs in `groupAllowFrom`. Negative chat IDs belong under `channels.telegram.groups`.
    Non-numeric entries are ignored for sender authorization.
    Security boundary (`2026.2.25+`): group sender auth does **not** inherit DM pairing-store approvals.
    Pairing stays DM-only. For groups, set `groupAllowFrom` or per-group/per-topic `allowFrom`.
    If `groupAllowFrom` is unset, Telegram falls back to config `allowFrom`, not the pairing store.
    Practical pattern for one-owner bots: set your user ID in `channels.telegram.allowFrom`, leave `groupAllowFrom` unset, and allow the target groups under `channels.telegram.groups`.
    Runtime note: if `channels.telegram` is completely missing, runtime defaults to fail-closed `groupPolicy="allowlist"` unless `channels.defaults.groupPolicy` is explicitly set.

    Example: allow any member in one specific group:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

    Example: allow only specific users inside one specific group:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

      Common mistake: `groupAllowFrom` is not a Telegram group allowlist.

      - Put negative Telegram group or supergroup chat IDs like `-1001234567890` under `channels.telegram.groups`.
      - Put Telegram user IDs like `8734062810` under `groupAllowFrom` when you want to limit which people inside an allowed group can trigger the bot.
      - Use `groupAllowFrom: ["*"]` only when you want any member of an allowed group to be able to

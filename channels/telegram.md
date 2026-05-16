---
domain: channels
topic: "Telegram Channel Setup and Configuration"
type: procedure
keywords:
  - Telegram
  - telegram bot
  - telegram setup
  - TELEGRAM_BOT_TOKEN
  - BotFather
  - telegram config
source: channels/telegram.md
---

Production-ready for bot DMs and groups via grammY. Long polling is the default mode; webhook mode is optional.

**Pairing:** Default DM policy for Telegram is pairing.
  

**Channel troubleshooting:** Cross-channel diagnostics and repair playbooks.
  

**Gateway configuration:** Full channel config patterns and examples.
  

## Quick setup

**Create the bot token in BotFather**

Open Telegram and chat with **@BotFather** (confirm the handle is exactly `@BotFather`).

    Run `/newbot`, follow prompts, and save the token.

  
**Configure token and DM policy**

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

  
**Start gateway and approve first DM**

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram 
```

    Pairing codes expire after 1 hour.

  
**Add the bot to a group**

Add the bot to your group, then get both IDs that group access needs:

    - your Telegram user ID, used in `allowFrom` / `groupAllowFrom`
    - the Telegram group chat ID, used as the key under `channels.telegram.groups`

    For first-time setup, get the group chat ID from `openclaw logs --follow`, a forwarded-ID bot, or Bot API `getUpdates`. After the group is allowed, `/whoami@<bot_username>` can confirm the user and group IDs.

    Negative Telegram supergroup IDs that start with `-100` are group chat IDs. Put them under `channels.telegram.groups`, not under `groupAllowFrom`.

  
> **Note:** Token resolution order is account-aware. In practice, config values win over env fallback, and `TELEGRAM_BOT_TOKEN` only applies to the default account.


## Telegram side settings

### Privacy mode and group visibility

Telegram bots default to **Privacy Mode**, which limits what group messages they receive.

    If the bot must see all group messages, either:

    - disable privacy mode via `/setprivacy`, or
    - make the bot a group admin.

    When toggling privacy mode, remove + re-add the bot in each group so Telegram applies the change.

  ### Group permissions

Admin status is controlled in Telegram group settings.

    Admin bots receive all group messages, which is useful for always-on group behavior.

  ### Helpful BotFather toggles

- `/setjoingroups` to allow/deny group adds
    - `/setprivacy` for group visibility behavior

  ## Access control and activation

**DM policy:**

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

  
**Group policy and allowlists:**

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

    Owner-only group setup:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: [""],
      groupPolicy: "allowlist",
      groups: {
        "": {
          requireMention: true,
        },
      },
    },
  },
}
```

    Test it from the group with `@<bot_username> ping`. Plain group messages do not trigger the bot while `requireMention: true`.

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

    > **Note:** Common mistake: `groupAllowFrom` is not a Telegram group allowlist.

      - Put negative Telegram group or supergroup chat IDs like `-1001234567890` under `channels.telegram.groups`.
      - Put Telegram user IDs like `8734062810` under `groupAllowFrom` when you want to limit which people inside an allowed group can trigger the bot.
      - Use `groupAllowFrom: ["*"]` only when you want any member of an allowed group to be able to talk to the bot.

    

  
**Mention behavior:**

Group replies require mention by default.

    Mention can come from:

    - native `@botusername` mention, or
    - mention patterns in:
      - `agents.list[].groupChat.mentionPatterns`
      - `messages.groupChat.mentionPatterns`

    Session-level command toggles:

    - `/activation always`
    - `/activation mention`

    These update session state only. Use config for persistence.

    Persistent config example:

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    Getting the group chat ID:

    - forward a group message to `@userinfobot` / `@getidsbot`
    - or read `chat.id` from `openclaw logs --follow`
    - or inspect Bot API `getUpdates`
    - after the group is allowed, run `/whoami@<bot_username>` if native commands are enabled

  
## Runtime behavior

- Telegram is owned by the gateway process.
- Routing is deterministic: Telegram inbound replies back to Telegram (the model does not pick channels).
- Inbound messages normalize into the shared channel envelope with reply metadata, media placeholders, and persisted reply-chain context for Telegram replies the gateway has observed.
- Group sessions are isolated by group ID. Forum topics append `:topic:<threadId>` to keep topics isolated.
- DM messages can carry `message_thread_id`; OpenClaw preserves the thread ID for replies but keeps DMs on the flat session by default. Configure `channels.telegram.dm.threadReplies: "inbound"`, `channels.telegram.direct.<chatId>.threadReplies: "inbound"`, `requireTopic: true`, or a matching topic config when you intentionally want DM topic session isolation.
- Long polling uses grammY runner with per-chat/per-thread sequencing. Overall runner sink concurrency uses `agents.defaults.maxConcurrent`.
- Long polling is guarded inside each gateway process so only one active poller can use a bot token at a time. If you still see `getUpdates` 409 conflicts, another OpenClaw gateway, script, or external poller is likely using the same token.
- Long-polling watchdog restarts trigger after 120 seconds without completed `getUpdates` liveness by default. Increase `channels.telegram.pollingStallThresholdMs` only if your deployment still sees false polling-stall restarts during long-running work. The value is in milliseconds and is allowed from `30000` to `600000`; per-account overrides are supported.
- Telegram Bot API has no read-receipt support (`sendReadReceipts` does not apply).

## Feature reference

### Live stream preview (message edits)

OpenClaw can stream partial replies in real time:

    - direct chats: preview message + `editMessageText`
    - groups/topics: preview message + `editMessageText`

    Requirement:

    - `channels.telegram.streaming` is `off | partial | block | progress` (default: `partial`)
    - `progress` keeps one editable status draft for tool progress, clears it at completion, and sends the final answer as a normal message
    - `streaming.preview.toolProgress` controls whether tool/progress updates reuse the same edited preview message (default: `true` when preview streaming is active)
    - `streaming.preview.commandText` controls command/exec detail inside those tool-progress lines: `raw` (default, preserves released behavior) or `status` (tool label only)
    - legacy `channels.telegram.streamMode` and boolean `streaming` values are detected; run `openclaw doctor --fix` to migrate them to `channels.telegram.streaming.mode`

    Tool-progress preview updates are the short status lines shown while tools run, for example command execution, file reads, planning updates, or patch summaries. Telegram keeps these enabled by default to match released OpenClaw behavior from `v2026.4.22` and later. To keep the edited preview for answer text but hide tool-progress lines, set:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": {
              "toolProgress": false
            }
          }
        }
      }
    }
    ```

    To keep tool-progress visible but hide command/exec text, set:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "partial",
            "preview": {
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    Use `progress` mode when you want visible tool progress without editing the final answer into that same message. Put the command-text policy under `streaming.progress`:

    ```json
    {
      "channels": {
        "telegram": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    Use `streaming.mode: "off"` only when you want final-only delivery: Telegram preview edits are disabled and generic tool/progress chatter is suppressed instead of being sent as standalone status messages. Approval prompts, media payloads, and errors still route through normal final delivery. Use `streaming.preview.toolProgress: false` when you only want to keep answer preview edits while hiding the tool-progress status lines.

    > **Note:** Telegram selected quote replies are the exception. When `replyToMode` is `"first"`, `"all"`, or `"batched"` and the inbound message includes selected quote text, OpenClaw sends the final answer through Telegram's native quote-reply path instead of editing the answer preview, so `streaming.preview.toolProgress` cannot show the short status lines for that turn. Current-message replies without selected quote text still keep prev
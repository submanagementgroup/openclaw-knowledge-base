---
domain: channels
topic: "Telegram Advanced Features and Troubleshooting"
type: reference
keywords:
  - Telegram
  - telegram advanced
  - telegram groups
  - telegram inline
  - telegram webhook
source: channels/telegram.md
---

iew streaming. Set `replyToMode: "off"` when tool-progress visibility matters more than native quote replies, or set `streaming.preview.toolProgress: false` to acknowledge the trade-off.
    

    For text-only replies:

    - short DM/group/topic previews: OpenClaw keeps the same preview message and performs the final edit in place
    - long text finals that split into multiple Telegram messages reuse the existing preview as the first final chunk when possible, then send only the remaining chunks
    - progress-mode finals clear the status draft and use normal final delivery instead of editing the draft into the answer
    - if the final edit fails before the completed text is confirmed, OpenClaw uses normal final delivery and cleans up the stale preview

    For complex replies (for example media payloads), OpenClaw falls back to normal final delivery and then cleans up the preview message.

    Preview streaming is separate from block streaming. When block streaming is explicitly enabled for Telegram, OpenClaw skips the preview stream to avoid double-streaming.

    Telegram-only reasoning stream:

    - `/reasoning stream` sends reasoning to the live preview while generating
    - the reasoning preview is deleted after final delivery; use `/reasoning on` when reasoning should remain visible
    - final answer is sent without reasoning text

  ### Formatting and HTML fallback

Outbound text uses Telegram `parse_mode: "HTML"`.

    - Markdown-ish text is rendered to Telegram-safe HTML.
    - Supported Telegram HTML tags are preserved; unsupported HTML is escaped.
    - If Telegram rejects parsed HTML, OpenClaw retries as plain text.

    Link previews are enabled by default and can be disabled with `channels.telegram.linkPreview: false`.

  ### Native commands and custom commands

Telegram command menu registration is handled at startup with `setMyCommands`.

    Native command defaults:

    - `commands.native: "auto"` enables native commands for Telegram

    Add custom command menu entries:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
    },
  },
}
```

    Rules:

    - names are normalized (strip leading `/`, lowercase)
    - valid pattern: `a-z`, `0-9`, `_`, length `1..32`
    - custom commands cannot override native commands
    - conflicts/duplicates are skipped and logged

    Notes:

    - custom commands are menu entries only; they do not auto-implement behavior
    - plugin/skill commands can still work when typed even if not shown in Telegram menu

    If native commands are disabled, built-ins are removed. Custom/plugin commands may still register if configured.

    Common setup failures:

    - `setMyCommands failed` with `BOT_COMMANDS_TOO_MUCH` means the Telegram menu still overflowed after trimming; reduce plugin/skill/custom commands or disable `channels.telegram.commands.native`.
    - `deleteWebhook`, `deleteMyCommands`, or `setMyCommands` failing with `404: Not Found` while direct Bot API curl commands work can mean `channels.telegram.apiRoot` was set to the full `/bot` endpoint. `apiRoot` must be only the Bot API root, and `openclaw doctor --fix` removes an accidental trailing `/bot`.
    - `getMe returned 401` means Telegram rejected the configured bot token. Update `botToken`, `tokenFile`, or `TELEGRAM_BOT_TOKEN` with the current BotFather token; OpenClaw stops before polling so this is not reported as a webhook cleanup failure.
    - `setMyCommands failed` with network/fetch errors usually means outbound DNS/HTTPS to `api.telegram.org` is blocked.

    ### Device pairing commands (`device-pair` plugin)

    When the `device-pair` plugin is installed:

    1. `/pair` generates setup code
    2. paste code in iOS app
    3. `/pair pending` lists pending requests (including role/scopes)
    4. approve the request:
       - `/pair approve <requestId>` for explicit approval
       - `/pair approve` when there is only one pending request
       - `/pair approve latest` for most recent

    The setup code carries a short-lived bootstrap token. Built-in bootstrap handoff keeps the primary node token at `scopes: []`; any handed-off operator token stays bounded to `operator.approvals`, `operator.read`, `operator.talk.secrets`, and `operator.write`. Bootstrap scope checks are role-prefixed, so that operator allowlist only satisfies operator requests; non-operator roles still need scopes under their own role prefix.

    If a device retries with changed auth details (for example role/scopes/public key), the previous pending request is superseded and the new request uses a different `requestId`. Re-run `/pair pending` before approving.

    More details: [Pairing](/channels/pairing#pair-via-telegram-recommended-for-ios).

  ### Inline buttons

Configure inline keyboard scope:

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    Per-account override:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    Scopes:

    - `off`
    - `dm`
    - `group`
    - `all`
    - `allowlist` (default)

    Legacy `capabilities: ["inlineButtons"]` maps to `inlineButtons: "all"`.

    Message action example:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Choose an option:",
  buttons: [
    [
      { text: "Yes", callback_data: "yes" },
      { text: "No", callback_data: "no" },
    ],
    [{ text: "Cancel", callback_data: "cancel" }],
  ],
}
```

    Callback clicks are passed to the agent as text:
    `callback_data: <value>`

  ### Telegram message actions for agents and automation

Telegram tool actions include:

    - `sendMessage` (`to`, `content`, optional `mediaUrl`, `replyToMessageId`, `messageThreadId`)
    - `react` (`chatId`, `messageId`, `emoji`)
    - `deleteMessage` (`chatId`, `messageId`)
    - `editMessage` (`chatId`, `messageId`, `content`)
    - `createForumTopic` (`chatId`, `name`, optional `iconColor`, `iconCustomEmojiId`)

    Channel message actions expose ergonomic aliases (`send`, `react`, `delete`, `edit`, `sticker`, `sticker-search`, `topic-create`).

    Gating controls:

    - `channels.telegram.actions.sendMessage`
    - `channels.telegram.actions.deleteMessage`
    - `channels.telegram.actions.reactions`
    - `channels.telegram.actions.sticker` (default: disabled)

    Note: `edit` and `topic-create` are currently enabled by default and do not have separate `channels.telegram.actions.*` toggles.
    Runtime sends use the active config/secrets snapshot (startup/reload), so action paths do not perform ad-hoc SecretRef re-resolution per send.

    Reaction removal semantics: [/tools/reactions](/tools/reactions)

  ### Reply threading tags

Telegram supports explicit reply threading tags in generated output:

    - `[[reply_to_current]]` replies to the triggering message
    - `[[reply_to:<id>]]` replies to a specific Telegram message ID

    `channels.telegram.replyToMode` controls handling:

    - `off` (default)
    - `first`
    - `all`

    When reply threading is enabled and the original Telegram text or caption is available, OpenClaw includes a native Telegram quote excerpt automatically. Telegram caps native quote text at 1024 UTF-16 code units, so longer messages are quoted from the start and fall back to a plain reply if Telegram rejects the quote.

    Note: `off` disables implicit reply threading. Explicit `[[reply_to_*]]` tags are still honored.

  ### Forum topics and thread behavior

Forum supergroups:

    - topic session keys append `:topic:<threadId>`
    - replies and typing target the topic thread
    - topic config path:
      `channels.telegram.groups.<chatId>.topics.<threadId>`

    General topic (`threadId=1`) special-case:

    - message sends omit `message_thread_id` (Telegram rejects `sendMessage(...thread_id=1)`)
    - typing actions still include `message_thread_id`

    Topic inheritance: topic entries inherit group settings unless overridden (`requireMention`, `allowFrom`, `skills`, `systemPrompt`, `enabled`, `groupPolicy`).
    `agentId` is topic-only and does not inherit from group defaults.

    **Per-topic agent routing**: Each topic can route to a different agent by setting `agentId` in the topic config. This gives each topic its own isolated workspace, memory, and session. Example:

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // General topic → main agent
                "3": { agentId: "zu" },        // Dev topic → zu agent
                "5": { agentId: "coder" }      // Code review → coder agent
              }
            }
          }
        }
      }
    }
    ```

    Each topic then has its own session key: `agent:zu:telegram:group:-1001234567890:topic:3`

    **Persistent ACP topic binding**: Forum topics can pin ACP harness sessions through top-level typed ACP bindings (`bindings[]` with `type: "acp"` and `match.channel: "telegram"`, `peer.kind: "group"`, and a topic-qualified id like `-1001234567890:topic:42`). Currently scoped to forum topics in groups/supergroups. See [ACP Agents](/tools/acp-agents).

    **Thread-bound ACP spawn from chat**: `/acp spawn <agent> --thread here|auto` binds the current topic to a new ACP session; follow-ups route there directly. OpenClaw pins the spawn confirmation in-topic. Requires `channels.telegram.threadBindings.spawnSessions` to remain enabled (default: `true`).

    Template context exposes `MessageThreadId` and `IsForum`. DM chats with `message_thread_id` keep DM routing and reply metadata on flat sessions by default; they only use thread-aware session keys when configured with `threadReplies: "inbound"`, `threadReplies: "always"`, `requireTopic: true`, or a matching topic config. Use top-level `channels.telegram.dm.threadReplies` for the account default, or `direct.<chatId>.threadReplies` for one DM.

  ### Audio, video, and stickers

### Audio messages

    Telegram distinguishes voice notes vs audio files.

    - default: audio file behavior
    - tag `[[audio_as_voice]]` in agent reply to force voice-note send
    - inbound voice-note transcripts are framed as machine-generated,
      untrusted text in the agent context; mention detection still uses the raw
      transcript so mention-gated voice messages continue to work.

    Message action example:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### Video messages

    Telegram distinguishes video files vs video notes.

    Message action example:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    Video notes do not support captions; provided message text is sent separately.

    ### Stickers

    Inbound sticker handling:

    - static WEBP: downloaded and processed (placeholder `<media:sticker>`)
    - animated TGS: skipped
    - video WEBM: skipped

    Sticker context fields:

    - `Sticker.emoji`
    - `Sticker.setName`
    - `Sticker.fileId`
    - `Sticker.fileUniqueId`
    - `Sticker.cachedDescription`

    Sticker cache file:

    - `~/.openclaw/telegram/sticker-cache.json`

    Stickers are described once (when possible) and cached to reduce repeated vision calls.

    Enable sticker actions:

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    Send sticker action:

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    Search cached stickers:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

  ### Reaction notifications

Telegram reactions arrive as `message_reaction` updates (separate from message payloads).

    When enabled, OpenClaw enqueues system events like:

    - `Telegram reaction added: 👍 by Alice (@alice) on msg 42`

    Config:

    - `channels.telegram.reactionNotifications`: `off | own | all` (default: `own`)
    - `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` (default: `minimal`)

    Notes:

    - `own` means user reactions to bot-sent messages only (best-effort via sent-message cache).
    - Reaction events still respect Telegram access controls (`dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`); unauthorized senders are dropped.
    - Telegram does not provide thread IDs in reaction updates.
      - non-forum groups route to group chat session
      - forum groups route to the group general-topic session (`:topic:1`), not the exact originating topic

    `allowed_updates` for polling/webhook include `message_reaction` automatically.

  ### Ack reactions

`ackReaction` sends an acknowledgement emoji while OpenClaw is processing an inbound message.

    Resolution order:

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - agent identity emoji fallback (`agents.list[].identity.emoji`, else "👀")

    Notes:

    - Telegram expects unicode emoji (for example "👀").
    - Use `""` to disable the reaction for a channel or account.

  ### Config writes from Telegram events and commands

Channel config writes are enabled by default (`configWrites !== false`).

    Telegram-triggered writes include:

    - group migration events (`migrate_to_chat_id`) to update `channels.telegram.groups`
    - `/config set` and `/config unset` (requires command enablement)

    Disable:

```json5
{
  channels: {
    telegram: {
      configWrites: 
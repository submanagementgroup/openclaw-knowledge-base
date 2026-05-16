---
domain: channels
topic: "WhatsApp Channel Setup"
type: procedure
keywords:
  - WhatsApp
  - whatsapp bot
  - WHATSAPP_PHONE_NUMBER
  - WhatsApp channel
  - whatsapp setup
source: channels/whatsapp.md
---

Status: production-ready via WhatsApp Web (Baileys). Gateway owns linked session(s).

## Install (on demand)

- Onboarding (`openclaw onboard`) and `openclaw channels add --channel whatsapp`
  prompt to install the WhatsApp plugin the first time you select it.
- `openclaw channels login --channel whatsapp` also offers the install flow when
  the plugin is not present yet.
- Dev channel + git checkout: defaults to the local plugin path.
- Stable/Beta: installs the official `@openclaw/whatsapp` plugin from ClawHub
  first, with npm as the fallback.
- The WhatsApp runtime is distributed outside the core OpenClaw npm package so
  WhatsApp-specific runtime dependencies stay with the external plugin.

Manual install stays available:

```bash
openclaw plugins install clawhub:@openclaw/whatsapp
```

Use the bare npm package (`@openclaw/whatsapp`) only when you need the registry
fallback. Pin an exact version only when you need a reproducible install.

**Pairing:** Default DM policy is pairing for unknown senders.
  

**Channel troubleshooting:** Cross-channel diagnostics and repair playbooks.
  

**Gateway configuration:** Full channel config patterns and examples.
  

## Quick setup

**Configure WhatsApp access policy**

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  
**Link WhatsApp (QR)**

```bash
openclaw channels login --channel whatsapp
```

    For a specific account:

```bash
openclaw channels login --channel whatsapp --account work
```

    To attach an existing/custom WhatsApp Web auth directory before login:

```bash
openclaw channels add --channel whatsapp --account work --auth-dir /path/to/wa-auth
openclaw channels login --channel whatsapp --account work
```

  
**Start the gateway**

```bash
openclaw gateway
```

  
**Approve first pairing request (if using pairing mode)**

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp 
```

    Pairing requests expire after 1 hour. Pending requests are capped at 3 per channel.

  
> **Note:** OpenClaw recommends running WhatsApp on a separate number when possible. (The channel metadata and setup flow are optimized for that setup, but personal-number setups are also supported.)


## Deployment patterns

### Dedicated number (recommended)

This is the cleanest operational mode:

    - separate WhatsApp identity for OpenClaw
    - clearer DM allowlists and routing boundaries
    - lower chance of self-chat confusion

    Minimal policy pattern:

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  ### Personal-number fallback

Onboarding supports personal-number mode and writes a self-chat-friendly baseline:

    - `dmPolicy: "allowlist"`
    - `allowFrom` includes your personal number
    - `selfChatMode: true`

    In runtime, self-chat protections key off the linked self number and `allowFrom`.

  ### WhatsApp Web-only channel scope

The messaging platform channel is WhatsApp Web-based (`Baileys`) in current OpenClaw channel architecture.

    There is no separate Twilio WhatsApp messaging channel in the built-in chat-channel registry.

  ## Runtime model

- Gateway owns the WhatsApp socket and reconnect loop.
- The reconnect watchdog uses WhatsApp Web transport activity, not only inbound app-message volume, so a quiet linked-device session is not restarted solely because nobody has sent a message recently. A longer application-silence cap still forces a reconnect if transport frames keep arriving but no application messages are handled for the watchdog window; after a transient reconnect for a recently active session, that application-silence check uses the normal message timeout for the first recovery window.
- Baileys socket timings are explicit under `web.whatsapp.*`: `keepAliveIntervalMs` controls WhatsApp Web application pings, `connectTimeoutMs` controls the opening handshake timeout, and `defaultQueryTimeoutMs` controls Baileys query timeouts.
- Outbound sends require an active WhatsApp listener for the target account.
- Group sends attach native mention metadata for `@+<digits>` and `@<digits>` tokens in text and media captions when the token matches current WhatsApp participant metadata, including LID-backed groups.
- Status and broadcast chats are ignored (`@status`, `@broadcast`).
- The reconnect watchdog follows WhatsApp Web transport activity, not only inbound app-message volume: quiet linked-device sessions stay up while transport frames continue, but a transport stall forces reconnect well before the later remote disconnect path.
- Direct chats use DM session rules (`session.dmScope`; default `main` collapses DMs to the agent main session).
- Group sessions are isolated (`agent:<agentId>:whatsapp:group:<jid>`).
- WhatsApp Channels/Newsletters can be explicit outbound targets with their native `@newsletter` JID. Outbound newsletter sends use channel session metadata (`agent:<agentId>:whatsapp:channel:<jid>`) rather than DM session semantics.
- WhatsApp Web transport honors standard proxy environment variables on the gateway host (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY` / lowercase variants). Prefer host-level proxy config over channel-specific WhatsApp proxy settings.
- When `messages.removeAckAfterReply` is enabled, OpenClaw clears the WhatsApp ack reaction after a visible reply is delivered.

## Plugin hooks and privacy

WhatsApp inbound messages can contain personal message content, phone numbers,
group identifiers, sender names, and session correlation fields. For that reason,
WhatsApp does not broadcast inbound `message_received` hook payloads to plugins
unless you explicitly opt in:

```json5
{
  channels: {
    whatsapp: {
      pluginHooks: {
        messageReceived: true,
      },
    },
  },
}
```

You can scope the opt-in to one account:

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        work: {
          pluginHooks: {
            messageReceived: true,
          },
        },
      },
    },
  },
}
```

Only enable this for plugins you trust to receive inbound WhatsApp message
content and identifiers.

## Access control and activation

**DM policy:**

`channels.whatsapp.dmPolicy` controls direct chat access:

    - `pairing` (default)
    - `allowlist`
    - `open` (requires `allowFrom` to include `"*"`)
    - `disabled`

    `allowFrom` accepts E.164-style numbers (normalized internally).

    `allowFrom` is a DM sender access-control list. It does not gate explicit outbound sends to WhatsApp group JIDs or `@newsletter` channel JIDs.

    Multi-account override: `channels.whatsapp.accounts.<id>.dmPolicy` (and `allowFrom`) take precedence over channel-level defaults for that account.

    Runtime behavior details:

    - pairings are persisted in channel allow-store and merged with configured `allowFrom`
    - scheduled automation and heartbeat recipient fallback use explicit delivery targets or configured `allowFrom`; DM pairing approvals are not implicit cron or heartbeat recipients
    - if no allowlist is configured, the linked self number is allowed by default
    - OpenClaw never auto-pairs outbound `fromMe` DMs (messages you send to yourself from the linked device)

  
**Group policy + allowlists:**

Group access has two layers:

    1. **Group membership allowlist** (`channels.whatsapp.groups`)
       - if `groups` is omitted, all groups are eligible
       - if `groups` is present, it acts as a group allowlist (`"*"` allowed)

    2. **Group sender policy** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`)
       - `open`: sender allowlist bypassed
       - `allowlist`: sender must match `groupAllowFrom` (or `*`)
       - `disabled`: block all group inbound

    Sender allowlist fallback:

    - if `groupAllowFrom` is unset, runtime falls back to `allowFrom` when available
    - sender allowlists are evaluated before mention/reply activation

    Note: if no `channels.whatsapp` block exists at all, runtime group-policy fallback is `allowlist` (with a warning log), even if `channels.defaults.groupPolicy` is set.

  
**Mentions + /activation:**

Group replies require mention by default.

    Mention detection includes:

    - explicit WhatsApp mentions of the bot identity
    - configured mention regex patterns (`agents.list[].groupChat.mentionPatterns`, fallback `messages.groupChat.mentionPatterns`)
    - inbound voice-note transcripts for authorized group messages
    - implicit reply-to-bot detection (reply sender matches bot identity)

    Security note:

    - quote/reply only satisfies mention gating; it does **not** grant sender authorization
    - with `groupPolicy: "allowlist"`, non-allowlisted senders are still blocked even if they reply to an allowlisted user's message

    Session-level activation command:

    - `/activation mention`
    - `/activation always`

    `activation` updates session state (not global config). It is owner-gated.

  
## Personal-number and self-chat behavior

When the linked self number is also present in `allowFrom`, WhatsApp self-chat safeguards activate:

- skip read receipts for self-chat turns
- ignore mention-JID auto-trigger behavior that would otherwise ping yourself
- if `messages.responsePrefix` is unset, self-chat replies default to `[{identity.name}]` or `[openclaw]`

## Message normalization and context

### Inbound envelope + reply context

Incoming WhatsApp messages are wrapped in the shared inbound envelope.

    If a quoted reply exists, context is appended in this form:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    Reply metadata fields are also populated when available (`ReplyToId`, `ReplyToBody`, `ReplyToSender`, sender JID/E.164).
    When the quoted reply target is downloadable media, OpenClaw saves it through
    the normal inbound media store and exposes it as `MediaPath`/`MediaType` so
    the agent can inspect the referenced image instead of only seeing
    `<media:image>`.

  ### Media placeholders and location/contact extraction

Media-only inbound messages are normalized with placeholders such as:

    - `<media:image>`
    - `<media:video>`
    - `<media:audio>`
    - `<media:document>`
    - `<media:sticker>`

    Authorized group voice notes are transcribed before mention gating when the
    body is only `<media:audio>`, so saying the bot mention in the voice note can
    trigger the reply. If the transcript still does not mention the bot, the
    transcript is kept in pending group history instead of the raw placeholder.

    Location bodies use terse coordinate text. Location labels/comments and contact/vCard details are rendered as fenced untrusted metadata, not inline prompt text.

  ### Pending group history injection

For groups, unprocessed messages can be buffered and injected as context when the bot is finally triggered.

    - default limit: `50`
    - config: `channels.whatsapp.historyLimit`
    - fallback: `messages.groupChat.historyLimit`
    - `0` disables

    Injection markers:

    - `[Chat messages since your last reply - for context]`
    - `[Current message - respond to this]`

  ### Read receipts

Read receipts are enabled by default for accepted inbound WhatsApp messages.

    Disable globally:

    ```json5
    {
      channels: {
        whatsapp: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    Per-account override:

    ```json5
    {
      channels: {
        whatsapp: {
          accounts: {
            work: {
              sendReadReceipts: false,
            },
          },
        },
      },
    }
    ```

    Self-chat turns skip read receipts even when globally enabled.

  ## Delivery, chunking, and media

### Text chunking

- default chunk limit: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.chunkMode = "length" | "newline"`
    - `newline` mode prefers paragraph boundaries (blank lines), then falls back to length-safe chunking

  ### Outbound media behavior

- supports image, video, audio (PTT voice-note), and document payloads
    - audio media is sent through the Baileys `audio` payload with `ptt: true`, so WhatsApp clients render it as a push-to-talk voice note
    - reply payloads preserve `audioAsVoice`; TTS voice-note output for WhatsApp stays on this PTT path even when the provider returns MP3 or WebM
    - native Ogg/Opus audio is sent as `audio/ogg; codecs=opus` for voice-note compatibility
    - non-Ogg audio, including Microsoft Edge TTS MP3/WebM output, is transcoded with `ffmpeg` to 48 kHz mono Ogg/Opus before PTT delivery
    - `/tts latest` sends the latest assistant reply as one voice note and suppresses repeat sends for the same reply; `/tts chat on|off|default` controls auto-TTS for the current WhatsApp chat
    - animated GIF playback is supported via `gifPlayback: true` on video sends
    - captions are applied to the first media item when sending multi-media reply payloads, except PTT voice notes send the audio first and visible text separately because WhatsApp clients do not render voice-note captions consistently
    - media source can be HTTP(S), `file://`, or local paths

  ### Media size limits and fallback behavior

- inbound media save cap: `channels.whatsapp.mediaMaxMb` (default `50`)
    - outbound media send cap: `channels.whatsapp.mediaMaxMb` (default `50`)
    - per-account overrides use `channels.whatsapp.accounts.<accountId>.mediaMaxMb`
    - images are auto-optimized (resize/quality sweep) to fit limits
    - on media send failure, first-item fallback sends text warning instead of dropping the response silently

  ## Reply quoting

WhatsApp supports native reply quoting, where outbound replies visibly quote the inbound message. Control it with `channels.whatsapp.replyToMode`.

| Value      
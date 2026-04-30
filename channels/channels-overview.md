---
domain: channels
topic: "Channels Overview: Built-In and Plugin Channels Supported by OpenClaw"
type: reference
keywords:
  - channels overview
  - messaging channels
  - channel list
  - built-in channels
  - plugin channels
related:
  - channels/telegram
  - channels/discord
  - channels/slack
source: channels/index.md
---

OpenClaw supports a wide range of messaging channels. Built-in channels are always available; plugin channels require installing additional plugins.

## Built-In Channels

| Channel | Notes |
|---------|-------|
| Telegram | Fastest to set up; recommended for first-time users |
| WhatsApp | Requires QR code scan |
| Discord | Guild-based; supports slash commands |
| Slack | Socket Mode; no public URL needed |
| iMessage | macOS only |
| Signal | Requires signal-cli |
| Google Chat | Requires Google app setup |
| IRC | Text-only |

## Plugin Channels (installed separately)

Matrix, Microsoft Teams, Mattermost, Feishu/Lark, Twitch, QQBot, WeChat, Zalo, Tlon, Nextcloud Talk, Synology Chat, Nostr, Line, and more.

OpenClaw can talk to you on any chat app you already use. Each channel connects via the Gateway.
Text is supported everywhere; media and reactions vary by channel.

## Delivery notes

- Telegram replies that contain markdown image syntax, such as `![alt](url)`,
  are converted into media replies on the final outbound path when possible.
- Slack multi-person DMs route as group chats, so group policy, mention
  behavior, and group-session rules apply to MPIM conversations.
- WhatsApp setup is install-on-demand: onboarding can show the setup flow before
  Baileys runtime dependencies are staged, and the Gateway loads the WhatsApp
  runtime only when the channel is actually active.

## Supported channels

- [BlueBubbles](/channels/bluebubbles) — **Recommended for iMessage**; uses the BlueBubbles macOS server REST API with full feature support (bundled plugin; edit, unsend, effects, reactions, group management — edit currently broken on macOS 26 Tahoe).
- [Discord](/channels/discord) — Discord Bot API + Gateway; supports servers, channels, and DMs.
- [Feishu](/channels/feishu) — Feishu/Lark bot via WebSocket (bundled plugin).
- [Google Chat](/channels/googlechat) — Google Chat API app via HTTP webhook.
- [iMessage (legacy)](/channels/imessage) — Legacy macOS integration via imsg CLI (deprecated, use BlueBubbles for new setups).
- [IRC](/channels/irc) — Classic IRC servers; channels + DMs with pairing/allowlist controls.
- [LINE](/channels/line) — LINE Messaging API bot (bundled plugin).
- [Matrix](/channels/matrix) — Matrix protocol (bundled plugin).
- [Mattermost](/channels/mattermost) — Bot API + WebSocket; channels, groups, DMs (bundled plugin).
- [Microsoft Teams](/channels/msteams) — Bot Framework; enterprise support (bundled plugin).
- [Nextcloud Talk](/channels/nextcloud-talk) — Self-hosted chat via Nextcloud Talk (bundled plugin).
- [Nostr](/channels/nostr) — Decentralized DMs via NIP-04 (bundled plugin).
- [QQ Bot](/channels/qqbot) — QQ Bot API; private chat, group chat, and rich media (bundled plugin).
- [Signal](/channels/signal) — signal-cli; privacy-focused.
- [Slack](/channels/slack) — Bolt SDK; workspace apps.
- [Synology Chat](/channels/synology-chat) — Synology NAS Chat via outgoing+incoming webhooks (bundled plugin).
- [Telegram](/channels/telegram) — Bot API via grammY; supports groups.
- [Tlon](/channels/tlon) — Urbit-based messenger (bundled plugin).
- [Twitch](/channels/twitch) — Twitch chat via IRC connection (bundled plugin).
- [Voice Call](/plugins/voice-call) — Telephony via Plivo or Twilio (plugin, installed separately).
- [WebChat](/web/webchat) — Gateway WebChat UI over WebSocket.
- [WeChat](/channels/wechat) — Tencent iLink Bot plugin via QR login; private chats only (external plugin).
- [WhatsApp](/channels/whatsapp) — Most popular; uses Baileys and requires QR pairing.
- [Yuanbao](/channels/yuanbao) — Tencent Yuanbao bot (external plugin).
- [Zalo](/channels/zalo) — Zalo Bot API; Vietnam's popular messenger (bundled plu

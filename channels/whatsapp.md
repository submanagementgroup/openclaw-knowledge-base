---
domain: channels
topic: "WhatsApp Channel: QR Code Pairing, allowFrom Access Control, Groups, and Feature Reference"
type: procedure
keywords:
  - WhatsApp
  - whatsapp channel
  - allowFrom
  - QR code
  - Baileys
  - WhatsApp Web
  - WhatsApp groups
related:
  - channels/telegram
  - channels/signal
  - gateway/configuration-overview
source: channels/whatsapp.md
---

WhatsApp channel for OpenClaw via Baileys (unofficial WhatsApp Web API). Requires scanning a QR code with your WhatsApp account.

## Quick Setup

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],  // E.164 format phone numbers
    }
  }
}
```

Then start the Gateway and scan the QR code that appears.

## Access Control

- `allowFrom`: list of E.164 phone numbers (`"+1..."`)
- Groups: use `groups` key with `requireMention: true`

Status: production-ready via WhatsApp Web (Baileys). Gateway owns linked session(s).

## Install (on demand)

- Onboarding (`openclaw onboard`) and `openclaw channels add --channel whatsapp`
  prompt to install the WhatsApp plugin the first time you select it.
- `openclaw channels login --channel whatsapp` also offers the install flow when
  the plugin is not present yet.
- Dev channel + git checkout: defaults to the local plugin path.
- Stable/Beta: uses the npm package `@openclaw/whatsapp` when a current package
  is published.

Manual install stays available:

```bash
openclaw plugins install @openclaw/whatsapp
```

If npm reports the OpenClaw-owned package as deprecated or missing, use a
current packaged OpenClaw build or a local checkout until the npm package train
catches up.

    Default DM policy is pairing for unknown senders.

    Cross-channel diagnostics and repair playbooks.

    Full channel config patterns and examples.

## Quick setup

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

```bash
openclaw gateway
```

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    Pairing requests expire after 1 hour. Pending requests are capped at 3 per channel.

OpenClaw recommends running WhatsApp on a separate number when possible. (The channel metadata and setup flow are optimized for that setup, but personal-number setups are also supported.)

## Deployment patterns

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

    Onboarding supports personal-number mode and writes a self-chat-friendly baseline:

    - `dmPolicy: "allowlist"`
    - `allowFrom` includes your personal number
    - `selfChatMode: true`

    In runtime, self-chat protections key off the linked self number and `allowFrom`.

    The messaging platform channel is WhatsApp Web-based (`Baileys`) in current OpenClaw channel architecture.

    There is no separate Twilio WhatsApp messaging channel in the built-in chat-channel registry.

## Runtime model

- Gateway owns the WhatsApp socket and reconnect loop.
- The reconnect watchdog uses WhatsApp Web transport activity, not only inbound app-message volume, so a quiet linked-device session is not restarted solely because nobody has sent a message recently. A longer application-silence cap still forces a reconnect if transport frames keep arriving but no application messages are handled for the watchdog window; after a transient reconnect for a recently active session, that application-silence check uses the normal message timeout for the first recovery window.
- Baileys socket timings are explicit under `web.whatsapp.*`: `keepAliveIntervalMs` controls WhatsApp Web application pings, `connectTimeoutMs` controls the opening handshake timeout, and `defaultQueryTimeoutMs` controls Baileys query timeouts.
- Outbound sends require an active WhatsApp listener for the target account.
- Status and broadcast chats are ignored (`@status`, `@broadcast`).
- The reconnect watchdog follows WhatsApp Web transport activity, not only inbound app-message volume: quiet linked-device sessions stay up while transport frames continue, but a transport stall forces reconnect well before the later remote disconnect path.
- Direct chats use DM session rules (`session.dmScope`; default `main` collapses DMs to the agent main session).
- Group sessions are isolated (`agent:<agentId>:whatsapp:group:<jid>`).
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
        messageReceived: tr

---
domain: channels
topic: "Signal Channel Setup"
type: procedure
keywords:
  - Signal
  - signal bot
  - signal channel
  - signal-cli
  - Signal messenger
source: channels/signal.md
---

Status: external CLI integration. Gateway talks to `signal-cli` over HTTP — either native daemon (JSON-RPC + SSE) or bbernhard/signal-cli-rest-api container (REST + WebSocket).

## Prerequisites

- OpenClaw installed on your server (Linux flow below tested on Ubuntu 24).
- One of:
  - `signal-cli` available on the host (native mode), **or**
  - `bbernhard/signal-cli-rest-api` Docker container (container mode).
- A phone number that can receive one verification SMS (for SMS registration path).
- Browser access for Signal captcha (`signalcaptchas.org`) during registration.

## Quick setup (beginner)

1. Use a **separate Signal number** for the bot (recommended).
2. Install `signal-cli` (Java required if you use the JVM build).
3. Choose one setup path:
   - **Path A (QR link):** `signal-cli link -n "OpenClaw"` and scan with Signal.
   - **Path B (SMS register):** register a dedicated number with captcha + SMS verification.
4. Configure OpenClaw and restart the gateway.
5. Send a first DM and approve pairing (`openclaw pairing approve signal `).

Minimal config:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

Field reference:

| Field       | Description                                       |
| ----------- | ------------------------------------------------- |
| `account`   | Bot phone number in E.164 format (`+15551234567`) |
| `cliPath`   | Path to `signal-cli` (`signal-cli` if on `PATH`)  |
| `dmPolicy`  | DM access policy (`pairing` recommended)          |
| `allowFrom` | Phone numbers or `uuid:<id>` values allowed to DM |

## What it is

- Signal channel via `signal-cli` (not embedded libsignal).
- Deterministic routing: replies always go back to Signal.
- DMs share the agent's main session; groups are isolated (`agent:<agentId>:signal:group:<groupId>`).

## Config writes

By default, Signal is allowed to write config updates triggered by `/config set|unset` (requires `commands.config: true`).

Disable with:

```json5
{
  channels: { signal: { configWrites: false } },
}
```

## The number model (important)

- The gateway connects to a **Signal device** (the `signal-cli` account).
- If you run the bot on **your personal Signal account**, it will ignore your own messages (loop protection).
- For "I text the bot and it replies," use a **separate bot number**.

## Setup path A: link existing Signal account (QR)

1. Install `signal-cli` (JVM or native build).
2. Link a bot account:
   - `signal-cli link -n "OpenClaw"` then scan the QR in Signal.
3. Configure Signal and start the gateway.

Example:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

Multi-account support: use `channels.signal.accounts` with per-account config and optional `name`. See [`gateway/configuration`](/gateway/config-channels#multi-account-all-channels) for the shared pattern.

## Setup path B: register dedicated bot number (SMS, Linux)

Use this when you want a dedicated bot number instead of linking an existing Signal app account.

1. Get a number that can receive SMS (or voice verification for landlines).
   - Use a dedicated bot number to avoid account/session conflicts.
2. Install `signal-cli` on the gateway host:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

If you use the JVM build (`signal-cli-${VERSION}.tar.gz`), install JRE 25+ first.
Keep `signal-cli` updated; upstream notes that old releases can break as Signal server APIs change.

3. Register and verify the number:

```bash
signal-cli -a + register
```

If captcha is required:

1. Open `https://signalcaptchas.org/registration/generate.html`.
2. Complete captcha, copy the `signalcaptcha://...` link target from "Open Signal".
3. Run from the same external IP as the browser session when possible.
4. Run registration again immediately (captcha tokens expire quickly):

```bash
signal-cli -a + register --captcha ''
signal-cli -a + verify 
```

4. Configure OpenClaw, restart gateway, verify channel:

```bash
# If you run the gateway as a user systemd service:
systemctl --user restart openclaw-gateway.service

# Then verify:
openclaw doctor
openclaw channels status --probe
```

5. Pair your DM sender:
   - Send any message to the bot number.
   - Approve code on the server: `openclaw pairing approve signal `.
   - Save the bot number as a contact on your phone to avoid "Unknown contact".

> **Note:** Registering a phone number account with `signal-cli` can de-authenticate the main Signal app session for that number. Prefer a dedicated bot number, or use QR link mode if you need to keep your existing phone app setup.


Upstream references:

- `signal-cli` README: `https://github.com/AsamK/signal-cli`
- Captcha flow: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- Linking flow: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## External daemon mode (httpUrl)

If you want to manage `signal-cli` yourself (slow JVM cold starts, container init, or shared CPUs), run the daemon separately and point OpenClaw at it:

```json5
{
  channels: {
    signal: {
      httpUrl: "http://127.0.0.1:8080",
      autoStart: false,
    },
  },
}
```

This skips auto-spawn and the startup wait inside OpenClaw. For slow starts when auto-spawning, set `channels.signal.startupTimeoutMs`.

## Container mode (bbernhard/signal-cli-rest-api)

Instead of running `signal-cli` natively, you can use the [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) Docker container. This wraps `signal-cli` behind a REST API and WebSocket interface.

Requirements:

- The container **must** run with `MODE=json-rpc` for real-time message receiving.
- Register or link your Signal account inside the container before connecting OpenClaw.

Example `docker-compose.yml` service:

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

OpenClaw config:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      httpUrl: "http://signal-cli:8080",
      autoStart: false,
      apiMode: "container", // or "auto" to detect automatically
    },
  },
}
```

The `apiMode` field controls which protocol OpenClaw uses:

| Value         | Behavior                                                                             |
| ------------- | ------------------------------------------------------------------------------------ |
| `"auto"`      | (Default) Probes both transports; streaming validates container WebSocket receive    |
| `"native"`    | Force native signal-cli (JSON-RPC at `/api/v1/rpc`, SSE at `/api/v1/events`)         |
| `"container"` | Force bbernhard container (REST at `/v2/send`, WebSocket at `/v1/receive/{account}`) |

When `apiMode` is `"auto"`, OpenClaw caches the detected mode for 30 seconds to avoid repeated probes. Container receive is only selected for streaming after `/v1/receive/{account}` upgrades to WebSocket, which requires `MODE=json-rpc`.

Container mode supports the same Signal channel operations as native mode where the container exposes matching APIs: sends, receives, attachments, typing indicators, read/viewed receipts, reactions, groups, and styled text. OpenClaw translates its native Signal RPC calls into the container's REST payloads, including `group.{base64(internal_id)}` group IDs and `text_mode: "styled"` for formatted text.

Operational notes:

- Use `autoStart: false` with container mode. OpenClaw should not spawn a native daemon when `apiMode: "container"` is selected.
- Use `MODE=json-rpc` for receiving. `MODE=normal` can make `/v1/about` look healthy, but `/v1/receive/{account}` does not WebSocket-upgrade, so OpenClaw will not select container receive streaming in `auto` mode.
- Set `apiMode: "container"` when you know the `httpUrl` points at bbernhard's REST API. Set `apiMode: "native"` when you know it points at native `signal-cli` JSON-RPC/SSE. Use `"auto"` when the deployment may vary.
- Container attachment downloads honor the same media byte limits as native mode. Oversized responses are rejected before being fully buffered when the server sends `Content-Length`, and while streaming otherwise.

## Access control (DMs + groups)

DMs:

- Default: `channels.signal.dmPolicy = "pairing"`.
- Unknown senders receive a pairing code; messages are ignored until approved (codes expire after 1 hour).
- Approve via:
  - `openclaw pairing list signal`
  - `openclaw pairing approve signal `
- Pairing is the default token exchange for Signal DMs. Details: [Pairing](/channels/pairing)
- UUID-only senders (from `sourceUuid`) are stored as `uuid:<id>` in `channels.signal.allowFrom`.

Groups:

- `channels.signal.groupPolicy = open | allowlist | disabled`.
- `channels.signal.groupAllowFrom` controls which groups or senders can trigger group replies when `allowlist` is set; entries can be Signal group IDs (raw, `group:<id>`, or `signal:group:<id>`), sender phone numbers, `uuid:<id>` values, or `*`.
- `channels.signal.groups["<group-id>" | "*"]` can override group behavior with `requireMention`, `tools`, and `toolsBySender`.
- Use `channels.signal.accounts.<id>.groups` for per-account overrides in multi-account setups.
- Allowlisting a Signal group through `groupAllowFrom` does not disable mention gating by itself. A specifically configured `channels.signal.groups["<group-id>"]` entry processes every group message unless `requireMention=true` is set.
- Runtime note: if `channels.signal` is completely missing, runtime falls back to `groupPolicy="allowlist"` for group checks (even if `channels.defaults.groupPolicy` is set).

## How it works (behavior)

- Native mode: `signal-cli` runs as a daemon; the gateway reads events via SSE.
- Container mode: the gateway sends via REST API and receives via WebSocket.
- Inbound messages are normalized into the shared channel envelope.
- Replies always route back to the same number or group.

## Media + limits

- Outbound text is chunked to `channels.signal.textChunkLimit` (default 4000).
- Optional newline chunking: set `channels.signal.chunkMode="newline"` to split on blank lines (paragraph boundaries) before length chunking.
- Attachments supported (base64 fetched from `signal-cli`).
- Voice-note attachments use the `signal-cli` filename as a MIME fallback when `contentType` is missing, so audio transcription can still classify AAC voice memos.
- Default media cap: `channels.signal.mediaMaxMb` (default 8).
- Use `channels.signal.ignoreAttachments` to skip downloading media.
- Group history context uses `channels.signal.historyLimit` (or `channels.signal.accounts.*.historyLimit`), falling back to `messages.groupChat.historyLimit`. Set `0` to disable (default 50).

## Typing + read receipts

- **Typing indicators**: OpenClaw sends typing signals via `signal-cli sendTyping` and refreshes them while a reply is running.
- **Read receipts**: when `channels.signal.sendReadReceipts` is true, OpenClaw forwards read receipts for allowed DMs.
- Signal-cli does not expose read receipts for groups.

## Reactions (message tool)

- Use `message action=react` with `channel=signal`.
- Targets: sender E.164 or UUID (use `uuid:<id>` from pairing output; bare UUID works too).
- `messageId` is the Signal timestamp for the message you're reacting to.
- Group reactions re
---
domain: channels
topic: "Channel Pairing: Linking Sessions to Channel Conversations"
type: concept
keywords:
  - channel pairing
  - session pairing
  - pairing
  - channel link
related:
  - channels/channel-routing
  - concepts/session
source: channels/pairing.md
---

Channel pairing links an OpenClaw session to a specific channel conversation. Used for associating web/CLI sessions with a messaging channel.

“Pairing” is OpenClaw’s explicit access approval step.
It is used in two places:

1. **DM pairing** (who is allowed to talk to the bot)
2. **Node pairing** (which devices/nodes are allowed to join the gateway network)

Security context: [Security](/gateway/security)

## 1) DM pairing (inbound chat access)

When a channel is configured with DM policy `pairing`, unknown senders get a short code and their message is **not processed** until you approve.

Default DM policies are documented in: [Security](/gateway/security)

`dmPolicy: "open"` is public only when the effective DM allowlist includes `"*"`.
Setup and validation require that wildcard for public-open configs. If existing
state contains `open` with concrete `allowFrom` entries, runtime still admits
only those senders, and pairing-store approvals do not widen `open` access.

Pairing codes:

- 8 characters, uppercase, no ambiguous chars (`0O1I`).
- **Expire after 1 hour**. The bot only sends the pairing message when a new request is created (roughly once per hour per sender).
- Pending DM pairing requests are capped at **3 per channel** by default; additional requests are ignored until one expires or is approved.

### Approve a sender

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

If no command owner is configured yet, approving a DM pairing code also bootstraps
`commands.ownerAllowFrom` to the approved sender, such as `telegram:123456789`.
That gives first-time setups an explicit owner for privileged commands and exec
approval prompts. After an owner exists, later pairing approvals only grant DM
access; they do not add more owners.

Supported channels: `bluebubbles`, `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `matrix`, `mattermost`, `msteams`, `nextcloud-talk`, `nostr`, `openclaw-weixin`, `signal`, `slack`, `synology-chat`, `telegram`, `twitch`, `whatsapp`, `zalo`, `zalouser`.

### Where the state lives

Stored under `~/.openclaw/credentials/`:

- Pending requests: `<channel>-pairing.json`
- Approved allowlist store:
  - Default account: `<channel>-allowFrom.json`
  - Non-default account: `<channel>-<accountId>-allowFrom.json`

Account scoping behavior:

- Non-default accounts read/write only their scoped allowlist file.
- Default account uses the channel-scoped unscoped allowlist file.

Treat these as sensitive (they gate access to your assistant).

The pairing allowlist store is for DM access. Group authorization is separate.
Approving a DM pairing code does not automatically allow that sender to run group
commands or control the bot in groups. First-owner bootstrap is separate config
state in `commands.ownerAllowFrom`, and group chat delivery still follows the
channel's group allowlists (for example `groupAllowFrom`, `groups`, or per-group
or per-topic overrides depending on the channel).

## 2) Node device pairing (iOS/Android/macOS/headless nodes)

Nodes connect to the Gateway as **devices** with `role: node`. The Gateway
creates a device pairing request that must be approved.

### Pair via Telegram (recommended for iOS)

If you use the `device-pair` plugin, you can do first-time device pairing entirely from Telegram:

1. In Telegram, message your bot: `/pair`
2. The bot replies with two messages: an instruction message and a separate **setup code** message (easy to copy/paste in Telegram).
3. On your phone, open the OpenClaw iOS app → Settings → Gateway.
4. Paste the setup code and connect.
5. Back in Telegram: `/pair pending` (review request IDs, role, and scopes), then approve.

The setup code is a base64-encoded JSON payload that contains:

- `url`: the Gateway WebSocket URL (`ws://...` or `wss://...`)
- `bootstrapToken`: a short-lived single-device bootstrap token used for the initial pairing handshake

That bootstrap token carries the built-in pairing bootstrap profile:

- primary handed-off `node` token stays `scopes: []`
- any handed-off `operator` token stays bounded to the bootstrap allowlist:
  `operator.approvals`, `operator.read`, `operator.talk.secrets`, `operator.write`
- bootstrap scope checks are role-prefixed, not one flat scope pool:
  operator scope entries only satisfy operator requests, and non-operator roles
  must still request scopes under their own role prefix
- later token rotation/revocation remains bounded by both the device's approved
  role contract and the caller session's operator scopes

Treat the setup code like a password while it is valid.

### Approve a node device

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

If the same device retries with different auth details (for example different
role/scopes/public key), the previous pending request is superseded and a new
`requestId` is created.

An already paired device does not get broader access silently. If it reconnects asking for more scopes or a broader role, OpenClaw keeps the existing approval as-is and creates a fresh pending upgrade reques

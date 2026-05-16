---
domain: channels
topic: "Matrix Channel Setup and Configuration"
type: procedure
keywords:
  - Matrix
  - matrix chat
  - Element
  - matrix setup
  - MATRIX_HOMESERVER
  - matrix bot
source: channels/matrix.md
---

Matrix is a downloadable channel plugin for OpenClaw.
It uses the official `matrix-js-sdk` and supports DMs, rooms, threads, media, reactions, polls, location, and E2EE.

## Install

Install Matrix from ClawHub before configuring the channel:

```bash
openclaw plugins install @openclaw/matrix
```

Bare plugin specs try ClawHub first, then npm fallback. To force the registry source, use `openclaw plugins install clawhub:@openclaw/matrix` or `openclaw plugins install npm:@openclaw/matrix`.

From a local checkout:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

`plugins install` registers and enables the plugin, so no separate `openclaw plugins enable matrix` step is needed. The plugin still does nothing until you configure the channel below. See [Plugins](/tools/plugin) for general plugin behavior and install rules.

## Setup

1. Create a Matrix account on your homeserver.
2. Configure `channels.matrix` with either `homeserver` + `accessToken`, or `homeserver` + `userId` + `password`.
3. Restart the gateway.
4. Start a DM with the bot, or invite it to a room (see [auto-join](#auto-join) - fresh invites only land when `autoJoin` allows them).

### Interactive setup

```bash
openclaw channels add
openclaw configure --section channels
```

The wizard asks for: homeserver URL, auth method (access token or password), user ID (password auth only), optional device name, whether to enable E2EE, and whether to configure room access and auto-join.

If matching `MATRIX_*` env vars already exist and the selected account has no saved auth, the wizard offers an env-var shortcut. To resolve room names before saving an allowlist, run `openclaw channels resolve --channel matrix "Project Room"`. When E2EE is enabled, the wizard writes the config and runs the same bootstrap as [`openclaw matrix encryption setup`](#encryption-and-verification).

### Minimal config

Token-based:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Password-based (the token is cached after first login):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

### Auto-join

`channels.matrix.autoJoin` defaults to `off`. With the default, the bot will not appear in new rooms or DMs from fresh invites until you join manually.

OpenClaw cannot tell at invite time whether an invited room is a DM or a group, so all invites - including DM-style invites - go through `autoJoin` first. `dm.policy` only applies later, after the bot has joined and the room has been classified.

> **Note:** Set `autoJoin: "allowlist"` plus `autoJoinAllowlist` to restrict which invites the bot accepts, or `autoJoin: "always"` to accept every invite.

`autoJoinAllowlist` only accepts stable targets: `!roomId:server`, `#alias:server`, or `*`. Plain room names are rejected; alias entries are resolved against the homeserver, not against state claimed by the invited room.


```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": { requireMention: true },
      },
    },
  },
}
```

To accept every invite, use `autoJoin: "always"`.

### Allowlist target formats

DM and room allowlists are best populated with stable IDs:

- DMs (`dm.allowFrom`, `groupAllowFrom`, `groups.<room>.users`): use `@user:server`. Display names are ignored by default because they are mutable; set `dangerouslyAllowNameMatching: true` only when you explicitly need compatibility with display-name entries.
- Room allowlist keys (`groups`, legacy `rooms`): use `!room:server` or `#alias:server`. Plain room names are ignored by default; set `dangerouslyAllowNameMatching: true` only when you explicitly need compatibility with joined-room name lookup.
- Invite allowlists (`autoJoinAllowlist`): use `!room:server`, `#alias:server`, or `*`. Plain room names are rejected.

### Account ID normalization

The wizard converts a friendly name into a normalized account ID. For example, `Ops Bot` becomes `ops-bot`. Punctuation is escaped in scoped env-var names so that two accounts cannot collide: `-` → `_X2D_`, so `ops-prod` maps to `MATRIX_OPS_X2D_PROD_*`.

### Cached credentials

Matrix stores cached credentials under `~/.openclaw/credentials/matrix/`:

- default account: `credentials.json`
- named accounts: `credentials-<account>.json`

When cached credentials exist there, OpenClaw treats Matrix as configured even if the access token is not in the config file - that covers setup, `openclaw doctor`, and channel-status probes.

### Environment variables

Used when the equivalent config key is not set. The default account uses unprefixed names; named accounts use the account ID inserted before the suffix.

| Default account       | Named account (`` is the normalized account ID) |
| --------------------- | --------------------------------------------------- |
| `MATRIX_HOMESERVER`   | `MATRIX__HOMESERVER`                            |
| `MATRIX_ACCESS_TOKEN` | `MATRIX__ACCESS_TOKEN`                          |
| `MATRIX_USER_ID`      | `MATRIX__USER_ID`                               |
| `MATRIX_PASSWORD`     | `MATRIX__PASSWORD`                              |
| `MATRIX_DEVICE_ID`    | `MATRIX__DEVICE_ID`                             |
| `MATRIX_DEVICE_NAME`  | `MATRIX__DEVICE_NAME`                           |
| `MATRIX_RECOVERY_KEY` | `MATRIX__RECOVERY_KEY`                          |

For account `ops`, the names become `MATRIX_OPS_HOMESERVER`, `MATRIX_OPS_ACCESS_TOKEN`, and so on. The recovery-key env vars are read by recovery-aware CLI flows (`verify backup restore`, `verify device`, `verify bootstrap`) when you pipe the key in via `--recovery-key-stdin`.

`MATRIX_HOMESERVER` cannot be set from a workspace `.env`; see [Workspace `.env` files](/gateway/security).

## Configuration example

A practical baseline with DM pairing, room allowlist, and E2EE:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

## Streaming previews

Matrix reply streaming is opt-in. `streaming` controls how OpenClaw delivers the in-flight assistant reply; `blockStreaming` controls whether each completed block is preserved as its own Matrix message.

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

To keep live answer previews but hide interim tool/progress lines, use object
form:

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "partial",
        preview: {
          toolProgress: false,
        },
      },
    },
  },
}
```

| `streaming`       | Behavior                                                                                                                                                            |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"off"` (default) | Wait for the full reply, send once. `true` ↔ `"partial"`, `false` ↔ `"off"`.                                                                                        |
| `"partial"`       | Edit one normal text message in place as the model writes the current block. Stock Matrix clients may notify on the first preview, not the final edit.              |
| `"quiet"`         | Same as `"partial"` but the message is a non-notifying notice. Recipients only get a notification once a per-user push rule matches the finalized edit (see below). |

`blockStreaming` is independent of `streaming`:

| `streaming`             | `blockStreaming: true`                                              | `blockStreaming: false` (default)                    |
| ----------------------- | ------------------------------------------------------------------- | ---------------------------------------------------- |
| `"partial"` / `"quiet"` | Live draft for the current block, completed blocks kept as messages | Live draft for the current block, finalized in place |
| `"off"`                 | One notifying Matrix message per finished block                     | One notifying Matrix message for the full reply      |

Notes:

- If a preview grows past Matrix's per-event size limit, OpenClaw stops preview streaming and falls back to final-only delivery.
- Media replies always send attachments normally. If a stale preview can no longer be reused safely, OpenClaw redacts it before sending the final media reply.
- Tool-progress preview updates are enabled by default when Matrix preview streaming is active. Set `streaming.preview.toolProgress: false` to keep preview edits for answer text but leave tool progress on the normal delivery path.
- Preview edits cost extra Matrix API calls. Leave `streaming: "off"` if you want the most conservative rate-limit profile.

## Approval metadata

Matrix native approval prompts are normal `m.room.message` events with OpenClaw-specific custom event content under `com.openclaw.approval`. Matrix permits custom event-content keys, so stock clients still render the text body while OpenClaw-aware clients can read the structured approval id, kind, state, available decisions, and exec/plugin details.

When an approval prompt is too long for one Matrix event, OpenClaw chunks the visible text and attaches `com.openclaw.approval` to the first chunk only. Reactions for allow/deny decisions are bound to that first event, so long prompts keep the same approval target as single-event prompts.

### Self-hosted push rules for quiet finalized previews

`streaming: "quiet"` only notifies recipients once a block or turn is finalized - a per-user push rule has to match the finalized preview marker. See [Matrix push rules for quiet previews](/channels/matrix-push-rules) for the full recipe (recipient token, pusher check, rule install, per-homeserver notes).

## Bot-to-bot rooms

By default, Matrix messages from other configured OpenClaw Matrix accounts are ignored.

Use `allowBots` when you intentionally want inter-agent Matrix traffic:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` accepts messages from other configured Matrix bot accounts in allowed rooms and DMs.
- `allowBots: "mentions"` accepts those messages only when they visibly mention this bot in rooms. DMs are still allowed.
- `groups.<room>.allowBots` overrides the account-level setting for one room.
- OpenClaw still ignores messages from the same Matrix user ID to avoid self-reply loops.
- Matrix does not expose a native bot flag here; OpenClaw treats "bot-authored" as "sent by another configured Matrix account on this OpenClaw gateway".

Use strict room allowlists and mention requirements when enabling bot-to-bot traffic in shared rooms.

## Encryption and verification

In encrypted (E2EE) rooms, outbound image events use `thumbnail_file` so image previews are encrypted alongside the full attachment. Unencrypted rooms still use plain `thumbnail_url`. No configuration is needed - the plugin detects E2EE state automatically.

All `openclaw matrix` commands accept `--verbose` (full diagnostics), `--json` (machine-readable output), and `--account <id>` (multi-account setups). Output is concise by default with quiet internal SDK logging. The examples below show the canonical form; add the flags as needed.

### Enable encryption

```bash
openclaw matrix encryption setup
```

Bootstraps secret storage and cross-signing, creates a room-key backup if needed, then prints status and next steps. Useful flags:

- `--recovery-key <key>` apply a recovery key before bootstrapping (prefer the stdin form documented below)
- `--force-reset-cross-signing` discard the current cross-signing identity and create a new one (use only intentionally)

For a new account, enable E2EE at creation time:

```bash
openclaw matrix account add \
  --homeserver https://matrix.example.org \
  --access-token syt_xxx \
  --enable-e2ee
```

`--encryption` is an alias for `--enable-e2ee`.

Manual config equivalent:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

### Status and trust signals

```bash
openclaw matrix verify status
openclaw matrix verify status --include-recovery-key --json
```

`verify status` reports three independent trust signals (`--verbose` shows all of them):

- `Locally trusted`: trusted by this client only
- `Cross-signing verified`: the SDK reports verification via cross-signing
- `Signed by owner`: signed by your own self-signing key (diagnostic only)

`Verified by owner` becomes `yes` only when `Cross-signing verified` is `yes`. Local trust or an owner signature alone is not enough.

`--allow-degraded-local-state` returns best-effort diagnostics without preparing the Matrix account first; useful for offline or partially-configured probes.

### Verify this device with a recovery key

The recovery key i
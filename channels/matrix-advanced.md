---
domain: channels
topic: "Matrix Advanced: Migration from Element, Push Rules, and Multi-Account Notes"
type: reference
keywords:
  - Matrix migration
  - Matrix push rules
  - Matrix element
  - Matrix multi-account
related:
  - channels/matrix
  - channels/mattermost
source:
  - channels/matrix-migration.md
  - channels/matrix-push-rules.md
---

Matrix migration from Element to native Matrix, and Matrix push rules configuration.

Upgrade from the previous public `matrix` plugin to the current implementation.

For most users, the upgrade is in place:

- the plugin stays `@openclaw/matrix`
- the channel stays `matrix`
- your config stays under `channels.matrix`
- cached credentials stay under `~/.openclaw/credentials/matrix/`
- runtime state stays under `~/.openclaw/matrix/`

You do not need to rename config keys or reinstall the plugin under a new name.

## What the migration does automatically

When the gateway starts, and when you run [`openclaw doctor --fix`](/gateway/doctor), OpenClaw tries to repair old Matrix state automatically.
Before any actionable Matrix migration step mutates on-disk state, OpenClaw creates or reuses a focused recovery snapshot.

When you use `openclaw update`, the exact trigger depends on how OpenClaw is installed:

- source installs run `openclaw doctor --fix` during the update flow, then restart the gateway by default
- package-manager installs update the package, run a non-interactive doctor pass, then rely on the default gateway restart so startup can finish Matrix migration
- if you use `openclaw update --no-restart`, startup-backed Matrix migration is deferred until you later run `openclaw doctor --fix` and restart the gateway

Automatic migration covers:

- creating or reusing a pre-migration snapshot under `~/Backups/openclaw-migrations/`
- reusing your cached Matrix credentials
- keeping the same account selection and `channels.matrix` config
- moving the oldest flat Matrix sync store into the current account-scoped location
- moving the oldest flat Matrix crypto store into the current account-scoped location when the target account can be resolved safely
- extracting a previously saved Matrix room-key backup decryption key from the old rust crypto store, when that key exists locally
- reusing the most complete existing token-hash storage root for the same Matrix account, homeserver, and user when the access token changes later
- scanning sibling token-hash storage roots for pending encrypted-state restore metadata when the Matrix access token changed but the account/device identity stayed the same
- restoring backed-up room keys into the new crypto store on the next Matrix startup

Snapshot details:

- OpenClaw writes a marker file at `~/.openclaw/matrix/migration-snapshot.json` after a successful snapshot so later startup and repair passes can reuse the same archive.
- These automatic Matrix migration snapshots back up config + state only (`includeWorkspace: false`).
- If Matrix only has warning-only migration state, for example because `userId` or `accessToken` is still missing, OpenClaw does not create the snapshot yet because no Matrix mutation is actionable.
- If the snapshot step fails, OpenClaw skips Matrix migration for that run instead of mutating state without a recovery point.

About multi-account upgrades:

- the oldest flat Matrix store (`~/.openclaw/matrix/bot-storage.json` and `~/.openclaw/matrix/crypto/`) came from a single-store layout, so OpenClaw can only migrate it into one resolved Matrix account target
- already account-scoped legacy Matrix stores are detected and prepared per configured Matrix account

## What the migration cannot do automatically

The previous public Matrix plugin did **not** automatically create Matrix room-key backups. It persisted local crypto state and requested device verification, but it did not guarantee that your room keys were backed up to the homeserver.

That means some encrypted installs can only be migrated partially.

OpenClaw cannot automatically recover:

- local-only room keys that were never backed up
- encrypted state when the target Matrix account cannot be resolved yet because `homeserver`, `userId`, or `accessToken` are still unavailable
- automatic migration of one shared flat Matrix store when multiple Matrix accounts are configured but `channels.matrix.defaultAccount` is not set
- custom plugin path installs that are pinned to a repo path instead of 

## Push Rules

When `channels.matrix.streaming` is `"quiet"`, OpenClaw edits a single preview event in place and marks the finalized edit with a custom content flag. Matrix clients notify on the final edit only if a per-user push rule matches that flag. This page is for operators who self-host Matrix and want to install that rule for each recipient account.

If you only want stock Matrix notification behavior, use `streaming: "partial"` or leave streaming off. See [Matrix channel setup](/channels/matrix#streaming-previews).

## Prerequisites

- recipient user = the person who should receive the notification
- bot user = the OpenClaw Matrix account that sends the reply
- use the recipient user's access token for the API calls below
- match `sender` in the push rule against the bot user's full MXID
- the recipient account must already have working pushers — quiet preview rules only work when normal Matrix push delivery is healthy

## Steps

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

    Reuse an existing client session token where possible. To mint a fresh one:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

If no pushers come back, fix normal Matrix push delivery for this account before continuing.

    OpenClaw marks finalized text-only preview edits with `content["com.openclaw.finalized_preview"] = true`. Install a rule that matches that marker plus the bot MXID as sender:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    Replace before running:

    - `https://matrix.example.org`: your homeserver base URL
    - `$USER_ACCESS_TOKEN`: the recipient user's access token
    - `openclaw-finalized-preview-botname`: a rule ID unique per bot per recipient (pattern: `openclaw-finalized-preview-<botname>`)
    - `@bot:example.org`: your OpenClaw bot MXID, not the recipient's

```bash
curl

---
domain: channels
topic: "Matrix Advanced Configuration and E2E Encryption"
type: reference
keywords:
  - Matrix
  - matrix E2E
  - matrix encryption
  - matrix rooms
  - matrix bridge
source: channels/matrix.md
---

s sensitive - pipe it via stdin instead of passing it on the command line. Set `MATRIX_RECOVERY_KEY` (or `MATRIX__RECOVERY_KEY` for a named account):

```bash
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
```

The command reports three states:

- `Recovery key accepted`: Matrix accepted the key for secret storage or device trust.
- `Backup usable`: room-key backup can be loaded with the trusted recovery material.
- `Device verified by owner`: this device has full Matrix cross-signing identity trust.

It exits non-zero when full identity trust is incomplete, even if the recovery key unlocked backup material. In that case, finish self-verification from another Matrix client:

```bash
openclaw matrix verify self
```

`verify self` waits for `Cross-signing verified: yes` before it exits successfully. Use `--timeout-ms <ms>` to tune the wait.

The literal-key form `openclaw matrix verify device "<recovery-key>"` is also accepted, but the key ends up in your shell history.

### Bootstrap or repair cross-signing

```bash
openclaw matrix verify bootstrap
```

`verify bootstrap` is the repair and setup command for encrypted accounts. In order, it:

- bootstraps secret storage, reusing an existing recovery key when possible
- bootstraps cross-signing and uploads missing public keys
- marks and cross-signs the current device
- creates a server-side room-key backup if one does not already exist

If the homeserver requires UIA to upload cross-signing keys, OpenClaw tries no-auth first, then `m.login.dummy`, then `m.login.password` (requires `channels.matrix.password`).

Useful flags:

- `--recovery-key-stdin` (pair with `printf '%s\n' "$MATRIX_RECOVERY_KEY" | …`) or `--recovery-key <key>`
- `--force-reset-cross-signing` to discard the current cross-signing identity (intentional only)

### Room-key backup

```bash
openclaw matrix verify backup status
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
```

`backup status` shows whether a server-side backup exists and whether this device can decrypt it. `backup restore` imports backed-up room keys into the local crypto store; if the recovery key is already on disk you can omit `--recovery-key-stdin`.

To replace a broken backup with a fresh baseline (accepts losing unrecoverable old history; can also recreate secret storage if the current backup secret is unloadable):

```bash
openclaw matrix verify backup reset --yes
```

Add `--rotate-recovery-key` only when you intentionally want the previous recovery key to stop unlocking the fresh backup baseline.

### Listing, requesting, and responding to verifications

```bash
openclaw matrix verify list
```

Lists pending verification requests for the selected account.

```bash
openclaw matrix verify request --own-user
openclaw matrix verify request --user-id @ops:example.org --device-id ABCDEF
```

Sends a verification request from this OpenClaw account. `--own-user` requests self-verification (you accept the prompt in another Matrix client of the same user); `--user-id`/`--device-id`/`--room-id` target someone else. `--own-user` cannot be combined with the other targeting flags.

For lower-level lifecycle handling - typically while shadowing inbound requests from another client - these commands act on a specific request `<id>` (printed by `verify list` and `verify request`):

| Command                                    | Purpose                                                             |
| ------------------------------------------ | ------------------------------------------------------------------- |
| `openclaw matrix verify accept <id>`       | Accept an inbound request                                           |
| `openclaw matrix verify start <id>`        | Start the SAS flow                                                  |
| `openclaw matrix verify sas <id>`          | Print the SAS emoji or decimals                                     |
| `openclaw matrix verify confirm-sas <id>`  | Confirm that the SAS matches what the other client shows            |
| `openclaw matrix verify mismatch-sas <id>` | Reject the SAS when the emoji or decimals do not match              |
| `openclaw matrix verify cancel <id>`       | Cancel; takes optional `--reason <text>` and `--code <matrix-code>` |

`accept`, `start`, `sas`, `confirm-sas`, `mismatch-sas`, and `cancel` all accept `--user-id` and `--room-id` as DM follow-up hints when the verification is anchored to a specific direct-message room.

### Multi-account notes

Without `--account <id>`, Matrix CLI commands use the implicit default account. If you have multiple named accounts and have not set `channels.matrix.defaultAccount`, they will refuse to guess and ask you to choose. When E2EE is disabled or unavailable for a named account, errors point at that account's config key, for example `channels.matrix.accounts.assistant.encryption`.

### Startup behavior

With `encryption: true`, `startupVerification` defaults to `"if-unverified"`. On startup an unverified device requests self-verification in another Matrix client, skipping duplicates and applying a cooldown (24 hours by default). Tune with `startupVerificationCooldownHours` or disable with `startupVerification: "off"`.

    Startup also runs a conservative crypto bootstrap pass that reuses the current secret storage and cross-signing identity. If bootstrap state is broken, OpenClaw attempts a guarded repair even without `channels.matrix.password`; if the homeserver requires password UIA, startup logs a warning and stays non-fatal. Already-owner-signed devices are preserved.

    See [Matrix migration](/channels/matrix-migration) for the full upgrade flow.

  ### Verification notices

Matrix posts verification lifecycle notices into the strict DM verification room as `m.notice` messages: request, ready (with "Verify by emoji" guidance), start/completion, and SAS (emoji/decimal) details when available.

    Incoming requests from another Matrix client are tracked and auto-accepted. For self-verification, OpenClaw starts the SAS flow automatically and confirms its own side once emoji verification is available - you still need to compare and confirm "They match" in your Matrix client.

    Verification system notices are not forwarded to the agent chat pipeline.

  ### Deleted or invalid Matrix device

If `verify status` says the current device is no longer listed on the homeserver, create a new OpenClaw Matrix device. For password login:

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --user-id '@assistant:example.org' \
  --password '<password>' \
  --device-name OpenClaw-Gateway
```

    For token auth, create a fresh access token in your Matrix client or admin UI, then update OpenClaw:

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --access-token '<token>'
```

    Replace `assistant` with the account ID from the failed command, or omit `--account` for the default account.

  ### Device hygiene

Old OpenClaw-managed devices can accumulate. List and prune:

```bash
openclaw matrix devices list
openclaw matrix devices prune-stale
```

  ### Crypto store

Matrix E2EE uses the official `matrix-js-sdk` Rust crypto path with `fake-indexeddb` as the IndexedDB shim. Crypto state persists to `crypto-idb-snapshot.json` (restrictive file permissions).

    Encrypted runtime state lives under `~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/` and includes the sync store, crypto store, recovery key, IDB snapshot, thread bindings, and startup verification state. When the token changes but the account identity stays the same, OpenClaw reuses the best existing root so prior state remains visible.

  ## Profile management

Update the Matrix self-profile for the selected account:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

You can pass both options in one call. Matrix accepts `mxc://` avatar URLs directly; when you pass `http://` or `https://`, OpenClaw uploads the file first and stores the resolved `mxc://` URL into `channels.matrix.avatarUrl` (or the per-account override).

## Threads

Matrix supports native Matrix threads for both automatic replies and message-tool sends. Two independent knobs control behavior:

### Session routing (`sessionScope`)

`dm.sessionScope` decides how Matrix DM rooms map to OpenClaw sessions:

- `"per-user"` (default): all DM rooms with the same routed peer share one session.
- `"per-room"`: each Matrix DM room gets its own session key, even when the peer is the same.

Explicit conversation bindings always win over `sessionScope`, so bound rooms and threads keep their chosen target session.

### Reply threading (`threadReplies`)

`threadReplies` decides where the bot posts its reply:

- `"off"`: replies are top-level. Inbound threaded messages stay on the parent session.
- `"inbound"`: reply inside a thread only when the inbound message was already in that thread.
- `"always"`: reply inside a thread rooted at the triggering message; that conversation is routed through a matching thread-scoped session from the first trigger onward.

`dm.threadReplies` overrides this for DMs only - for example, keep room threads isolated while keeping DMs flat.

### Thread inheritance and slash commands

- Inbound threaded messages include the thread root message as extra agent context.
- Message-tool sends auto-inherit the current Matrix thread when targeting the same room (or the same DM user target), unless an explicit `threadId` is provided.
- DM user-target reuse only kicks in when the current session metadata proves the same DM peer on the same Matrix account; otherwise OpenClaw falls back to normal user-scoped routing.
- `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`, and thread-bound `/acp spawn` all work in Matrix rooms and DMs.
- Top-level `/focus` creates a new Matrix thread and binds it to the target session when `threadBindings.spawnSessions` is enabled.
- Running `/focus` or `/acp spawn --thread here` inside an existing Matrix thread binds that thread in place.

When OpenClaw detects a Matrix DM room colliding with another DM room on the same shared session, it posts a one-time `m.notice` in that room pointing to the `/focus` escape hatch and suggesting a `dm.sessionScope` change. The notice only appears when thread bindings are enabled.

## ACP conversation bindings

Matrix rooms, DMs, and existing Matrix threads can be turned into durable ACP workspaces without changing the chat surface.

Fast operator flow:

- Run `/acp spawn codex --bind here` inside the Matrix DM, room, or existing thread you want to keep using.
- In a top-level Matrix DM or room, the current DM/room stays the chat surface and future messages route to the spawned ACP session.
- Inside an existing Matrix thread, `--bind here` binds that current thread in place.
- `/new` and `/reset` reset the same bound ACP session in place.
- `/acp close` closes the ACP session and removes the binding.

Notes:

- `--bind here` does not create a child Matrix thread.
- `threadBindings.spawnSessions` gates `/acp spawn --thread auto|here`, where OpenClaw needs to create or bind a child Matrix thread.

### Thread binding config

Matrix inherits global defaults from `session.threadBindings`, and also supports per-channel overrides:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSessions`
- `threadBindings.defaultSpawnContext`

Matrix thread-bound session spawns default on:

- Set `threadBindings.spawnSessions: false` to block top-level `/focus` and `/acp spawn --thread auto|here` from creating/binding Matrix threads.
- Set `threadBindings.defaultSpawnContext: "isolated"` when native subagent thread spawns should not fork the parent transcript.

## Reactions

Matrix supports outbound reactions, inbound reaction notifications, and ack reactions.

Outbound reaction tooling is gated by `channels.matrix.actions.reactions`:

- `react` adds a reaction to a Matrix event.
- `reactions` lists the current reaction summary for a Matrix event.
- `emoji=""` removes the bot's own reactions on that event.
- `remove: true` removes only the specified emoji reaction from the bot.

**Resolution order** (first defined value wins):

| Setting                 | Order                                                                            |
| ----------------------- | -------------------------------------------------------------------------------- |
| `ackReaction`           | per-account → channel → `messages.ackReaction` → agent identity emoji fallback   |
| `ackReactionScope`      | per-account → channel → `messages.ackReactionScope` → default `"group-mentions"` |
| `reactionNotifications` | per-account → channel → default `"own"`                                          |

`reactionNotifications: "own"` forwards added `m.reaction` events when they target bot-authored Matrix messages; `"off"` disables reaction system events. Reaction removals are not synthesized into system events because Matrix surfaces those as redactions, not as standalone `m.reaction` removals.

## History context

- `channels.matrix.historyLimit` controls how many recent room messages are included as `InboundHistory` when a Matrix room message triggers the agent. Falls back to `messages.groupChat.historyLimit`; if both are unset, the effective default is `0`. Set `0` to disable.
- Matrix room history is room-only. DMs keep using normal session history.
- Matrix room history is pending-only: OpenClaw buffers room messages that did not trigger a reply yet, then snapshots that window when a mention or other trigger arrives.
- The current trigger message is not included in `InboundHistory`; it stays in the main inbound bod
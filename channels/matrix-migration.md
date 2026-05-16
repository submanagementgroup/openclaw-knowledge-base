---
domain: channels
topic: "Matrix Migration Guide"
type: procedure
keywords:
  - Matrix
  - matrix migration
  - matrix upgrade
  - matrix version migration
source: channels/matrix-migration.md
---

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
- custom plugin path installs that are pinned to a repo path instead of the standard Matrix package
- a missing recovery key when the old store had backed-up keys but did not keep the decryption key locally

Current warning scope:

- custom Matrix plugin path installs are surfaced by both gateway startup and `openclaw doctor`

If your old installation had local-only encrypted history that was never backed up, some older encrypted messages may remain unreadable after the upgrade.

## Recommended upgrade flow

1. Update OpenClaw and the Matrix plugin normally.
   Prefer plain `openclaw update` without `--no-restart` so startup can finish the Matrix migration immediately.
2. Run:

   ```bash
   openclaw doctor --fix
   ```

   If Matrix has actionable migration work, doctor will create or reuse the pre-migration snapshot first and print the archive path.

3. Start or restart the gateway.
4. Check current verification and backup state:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. Put the recovery key for the Matrix account you are repairing in an account-specific environment variable. For a single default account, `MATRIX_RECOVERY_KEY` is fine. For multiple accounts, use one variable per account, for example `MATRIX_RECOVERY_KEY_ASSISTANT`, and add `--account assistant` to the command.

6. If OpenClaw tells you a recovery key is needed, run the command for the matching account:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify backup restore --recovery-key-stdin --account assistant
   ```

7. If this device is still unverified, run the command for the matching account:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify device --recovery-key-stdin --account assistant
   ```

   If the recovery key is accepted and backup is usable, but `Cross-signing verified`
   is still `no`, complete self-verification from another Matrix client:

   ```bash
   openclaw matrix verify self
   ```

   Accept the request in another Matrix client, compare the emoji or decimals,
   and type `yes` only when they match. The command exits successfully only
   after `Cross-signing verified` becomes `yes`.

8. If you are intentionally abandoning unrecoverable old history and want a fresh backup baseline for future messages, run:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

9. If no server-side key backup exists yet, create one for future recoveries:

   ```bash
   openclaw matrix verify bootstrap
   ```

## How encrypted migration works

Encrypted migration is a two-stage process:

1. Startup or `openclaw doctor --fix` creates or reuses the pre-migration snapshot if encrypted migration is actionable.
2. Startup or `openclaw doctor --fix` inspects the old Matrix crypto store through the active Matrix plugin install.
3. If a backup decryption key is found, OpenClaw writes it into the new recovery-key flow and marks room-key restore as pending.
4. On the next Matrix startup, OpenClaw restores backed-up room keys into the new crypto store automatically.

If the old store reports room keys that were never backed up, OpenClaw warns instead of pretending recovery succeeded.

## Common messages and what they mean

### Upgrade and detection messages

`Matrix plugin upgraded in place.`

- Meaning: the old on-disk Matrix state was detected and migrated into the current layout.
- What to do: nothing unless the same output also includes warnings.

`Matrix migration snapshot created before applying Matrix upgrades.`

- Meaning: OpenClaw created a recovery archive before mutating Matrix state.
- What to do: keep the printed archive path until you confirm migration succeeded.

`Matrix migration snapshot reused before applying Matrix upgrades.`

- Meaning: OpenClaw found an existing Matrix migration snapshot marker and reused that archive instead of creating a duplicate backup.
- What to do: keep the printed archive path until you confirm migration succeeded.

`Legacy Matrix state detected at ... but channels.matrix is not configured yet.`

- Meaning: old Matrix state exists, but OpenClaw cannot map it to a current Matrix account because Matrix is not configured.
- What to do: configure `channels.matrix`, then rerun `openclaw doctor --fix` or restart the gateway.

`Legacy Matrix state detected at ... but the new account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- Meaning: OpenClaw found old state, but it still cannot determine the exact current account/device root.
- What to do: start the gateway once with a working Matrix login, or rerun `openclaw doctor --fix` after cached credentials exist.

`Legacy Matrix state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- Meaning: OpenClaw found one shared flat Matrix store, but it refuses to guess which named Matrix account should receive it.
- What to do: set `channels.matrix.defaultAccount` to the intended account, then rerun `openclaw doctor --fix` or restart the gateway.

`Matrix legacy sync store not migrated because the target already exists (...)`

- Meaning: the new account-scoped location already has a sync or crypto store, so OpenClaw did not overwrite it automatically.
- What to do: verify that the current account is the correct one before manually removing or moving the conflicting target.

`Failed migrating Matrix legacy sync store (...)` or `Failed migrating Matrix legacy crypto store (...)`

- Meaning: OpenClaw tried to move old Matrix state but the filesystem operation failed.
- What to do: inspect filesystem permissions and disk state, then rerun `openclaw doctor --fix`.

`Legacy Matrix encrypted state detected at ... but channels.matrix is not configured yet.`

- Meaning: OpenClaw found an old encrypted Matrix store, but there is no current Matrix config to attach it to.
- What to do: configure `channels.matrix`, then rerun `openclaw doctor --fix` or restart the gateway.

`Legacy Matrix encrypted state detected at ... but the account-scoped target could not be resolved yet (need homeserver, userId, and access token for channels.matrix...).`

- Meaning: the encrypted store exists, but OpenClaw cannot safely decide which current account/device it belongs to.
- What to do: start the gateway once with a working Matrix login, or rerun `openclaw doctor --fix` after cached credentials are available.

`Legacy Matrix encrypted state detected at ... but multiple Matrix accounts are configured and channels.matrix.defaultAccount is not set.`

- Meaning: OpenClaw found one shared flat legacy crypto store, but it refuses to guess which named Matrix account should receive it.
- What to do: set `channels.matrix.defaultAccount` to the intended account, then rerun `openclaw doctor --fix` or restart the gateway.

`Matrix migration warnings are present, but no on-disk Matrix mutation is actionable yet. No pre-migration snapshot was needed.`

- Meaning: OpenClaw detected old Matrix state, but the migration is still blocked on missing identity or credential data.
- What to do: finish Matrix login or config setup, then rerun `openclaw doctor --fix` or restart the gateway.

`Legacy Matrix encrypted state was detected, but the Matrix plugin helper is unavailable. Install or repair @openclaw/matrix so OpenClaw can inspect the old rust crypto store before upgrading.`

- Meaning: OpenClaw found old encrypted Matrix state, but it could not load the helper entrypoint from the Matrix plugin that normally inspects that store.
- What to do: reinstall or repair the Matrix plugin (`openclaw plugins install @openclaw/matrix`, or `openclaw plugins install ./path/to/local/matrix-plugin` for a repo checkout), then rerun `openclaw doctor --fix` or restart the gateway.

`Matrix plugin he
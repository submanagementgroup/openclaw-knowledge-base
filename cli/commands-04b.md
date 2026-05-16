---
domain: cli
topic: "CLI Commands (cont): nodes, pairing, proxy..."
type: reference
keywords:
  - nodes
  - pairing
  - proxy
  - qr
  - reset
  - CLI commands
  - openclaw CLI
source: 
  - cli/nodes.md
  - cli/pairing.md
  - cli/proxy.md
  - cli/qr.md
  - cli/reset.md
  - cli/sandbox.md
  - cli/secrets.md
  - cli/security.md
---
## Why this is needed

When you update sandbox configuration:

- Existing runtimes continue running with old settings.
- Runtimes are only pruned after 24h of inactivity.
- Regularly-used agents keep old runtimes alive indefinitely.

Use `openclaw sandbox recreate` to force removal of old runtimes. They are recreated automatically with current settings when next needed.

> **Note:** Prefer `openclaw sandbox recreate` over manual backend-specific cleanup. It uses the Gateway's runtime registry and avoids mismatches when scope or session keys change.


## Registry migration

OpenClaw stores sandbox runtime metadata as one JSON shard per container/browser entry under the sandbox state directory. Older installs may still have monolithic legacy files:

- `~/.openclaw/sandbox/containers.json`
- `~/.openclaw/sandbox/browsers.json`

Regular sandbox runtime reads do not rewrite those files. Run `openclaw doctor --fix` to migrate valid legacy entries into the sharded registry directories. Invalid legacy files are quarantined so one bad old registry cannot hide current runtime entries.

## Configuration

Sandbox settings live in `~/.openclaw/openclaw.json` under `agents.defaults.sandbox` (per-agent overrides go in `agents.list[].sandbox`):

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // off, non-main, all
        "backend": "docker", // docker, ssh, openshell
        "scope": "agent", // session, agent, shared
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... more Docker options
        },
        "prune": {
          "idleHours": 24, // Auto-prune after 24h idle
          "maxAgeDays": 7, // Auto-prune after 7 days
        },
      },
    },
  },
}
```

## Related

- [CLI reference](/cli)
- [Sandboxing](/gateway/sandboxing)
- [Agent workspace](/concepts/agent-workspace)
- [Doctor](/gateway/doctor): checks sandbox setup.


## openclaw secrets Command

# `openclaw secrets`

Use `openclaw secrets` to manage SecretRefs and keep the active runtime snapshot healthy.

Command roles:

- `reload`: gateway RPC (`secrets.reload`) that re-resolves refs and swaps runtime snapshot only on full success (no config writes).
- `audit`: read-only scan of configuration/auth/generated-model stores and legacy residues for plaintext, unresolved refs, and precedence drift (exec refs are skipped unless `--allow-exec` is set).
- `configure`: interactive planner for provider setup, target mapping, and preflight (TTY required).
- `apply`: execute a saved plan (`--dry-run` for validation only; dry-run skips exec checks by default, and write mode rejects exec-containing plans unless `--allow-exec` is set), then scrub targeted plaintext residues.

Recommended operator loop:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

If your plan includes `exec` SecretRefs/providers, pass `--allow-exec` on both dry-run and write apply commands.

Exit code note for CI/gates:

- `audit --check` returns `1` on findings.
- unresolved refs return `2`.

Related:

- Secrets guide: [Secrets Management](/gateway/secrets)
- Credential surface: [SecretRef Credential Surface](/reference/secretref-credential-surface)
- Security guide: [Security](/gateway/security)

## Reload runtime snapshot

Re-resolve secret refs and atomically swap runtime snapshot.

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Notes:

- Uses gateway RPC method `secrets.reload`.
- If resolution fails, gateway keeps last-known-good snapshot and returns an error (no partial activation).
- JSON response includes `warningCount`.

Options:

- `--url <url>`
- `--token <token>`
- `--timeout <ms>`
- `--json`

## Audit

Scan OpenClaw state for:

- plaintext secret storage
- unresolved refs
- precedence drift (`auth-profiles.json` credentials shadowing `openclaw.json` refs)
- generated `agents/*/agent/models.json` residues (provider `apiKey` values and sensitive provider headers)
- legacy residues (legacy auth store entries, OAuth reminders)

Header residue note:

- Sensitive provider header detection is name-heuristic based (common auth/credential header names and fragments such as `authorization`, `x-api-key`, `token`, `secret`, `password`, and `credential`).

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

Exit behavior:

- `--check` exits non-zero on findings.
- unresolved refs exit with higher-priority non-zero code.

Report shape highlights:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`, `skippedExecRefs`, `resolvabilityComplete`
- `summary`: `plaintextCount`, `unresolvedRefCount`, `shadowedRefCount`, `legacyResidueCount`
- finding codes:
  - `PLAINTEXT_FOUND`
  - `REF_UNRESOLVED`
  - `REF_SHADOWED`
  - `LEGACY_RESIDUE`

## Configure (interactive helper)

Build provider and SecretRef changes interactively, run preflight, and optionally apply:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

Flow:

- Provider setup first (`add/edit/remove` for `secrets.providers` aliases).
- Credential mapping second (select fields and assign `{source, provider, id}` refs).
- Preflight and optional apply last.

Flags:

- `--providers-only`: configure `secrets.providers` only, skip credential mapping.
- `--skip-provider-setup`: skip provider setup and map credentials to existing providers.
- `--agent <id>`: scope `auth-profiles.json` target discovery and writes to one agent store.
- `--allow-exec`: allow exec SecretRef checks during preflight/apply (may execute provider commands).

Notes:

- Requires an interactive TTY.
- You cannot combine `--providers-only` with `--skip-provider-setup`.
- `configure` targets secret-bearing fields in `openclaw.json` plus `auth-profiles.json` for the selected agent scope.
- `configure` supports creating new `auth-profiles.json` mappings directly in the picker flow.
- Canonical supported surface: [SecretRef Credential Surface](/reference/secretref-credential-surface).
- It performs preflight resolution before apply.
- If preflight/apply includes exec refs, keep `--allow-exec` set for both steps.
- Generated plans default to scrub options (`scrubEnv`, `scrubAuthProfilesForProviderTargets`, `scrubLegacyAuthJson` all enabled).
- Apply path is one-way for scrubbed plaintext values.
- Without `--apply`, CLI still prompts `Apply this plan now?` after preflight.
- With `--apply` (and no `--yes`), CLI prompts an extra irreversible confirmation.
- `--json` prints the plan + preflight report, but the command still requires an interactive TTY.

Exec provider safety note:

- Homebrew installs often expose symlinked binaries under `/opt/homebrew/bin/*`.
- Set `allowSymlinkCommand: true` only when needed for trusted package-manager paths, and pair it with `trustedDirs` (for example `["/opt/homebrew"]`).
- On Windows, if ACL verification is unavailable for a provider path, OpenClaw fails closed. For trusted paths only, set `allowInsecurePath: true` on that provider to bypass path security checks.

## Apply a saved plan

Apply or preflight a plan generated previously:

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

Exec behavior:

- `--dry-run` validates preflight without writing files.
- exec SecretRef checks are skipped by default in dry-run.
- write mode rejects plans that contain exec SecretRefs/providers unless `--allow-exec` is set.
- Use `--allow-exec` to opt in to exec provider checks/execution in either mode.

Plan contract details (allowed target paths, validation rules, and failure semantics):

- [Secrets Apply Plan Contract](/gateway/secrets-plan-contract)

What `apply` may update:

- `openclaw.json` (SecretRef targets + provider upserts/deletes)
- `auth-profiles.json` (provider-target scrubbing)
- legacy `auth.json` residues
- `~/.openclaw/.env` known secret keys whose values were migrated

## Why no rollback backups

`secrets apply` intentionally does not write rollback backups containing old plaintext values.

Safety comes from strict preflight + atomic-ish apply with best-effort in-memory restore on failure.

## Example

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

If `audit --check` still reports plaintext findings, update the remaining reported target paths and rerun audit.

## Related

- [CLI reference](/cli)
- [Secrets management](/gateway/secrets)


## openclaw security Command

# `openclaw security`

Security tools (audit + optional fixes).

Related:

- Security guide: [Security](/gateway/security)

## Audit

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --fix
openclaw security audit --json
```

Plain `security audit` stays on the cold config/filesystem/read-only path. It does not discover plugin runtime security collectors by default, so routine audits do not load every installed plugin runtime. Use `--deep` to include best-effort live Gateway probes and plugin-owned security audit collectors; explicit internal callers may also opt into those plugin-owned collectors when they already have an appropriate runtime scope.

The audit warns when multiple DM senders share the main session and recommends **secure DM mode**: `session.dmScope="per-channel-peer"` (or `per-account-channel-peer` for multi-account channels) for shared inboxes.
This is for cooperative/shared inbox hardening. A single Gateway shared by mutually untrusted/adversarial operators is not a recommended setup; split trust boundaries with separate gateways (or separate OS users/hosts).
It also emits `security.trust_model.multi_user_heuristic` when config suggests likely shared-user ingress (for example open DM/group policy, configured group targets, or wildcard sender rules), and reminds you that OpenClaw is a personal-assistant trust model by default.
For intentional shared-user setups, the audit guidance is to sandbox all sessions, keep filesystem access workspace-scoped, and keep personal/private identities or credentials off that runtime.
It also warns when small models (`<=300B`) are used without sandboxing and with web/browser tools enabled.
For webhook ingress, it warns when `hooks.token` reuses the Gateway token, when `hooks.token` is short, when `hooks.path="/"`, when `hooks.defaultSessionKey` is unset, when `hooks.allowedAgentIds` is unrestricted, when request `sessionKey` overrides are enabled, and when overrides are enabled without `hooks.allowedSessionKeyPrefixes`.
It also warns when sandbox Docker settings are configured while sandbox mode is off, when `gateway.nodes.denyCommands` uses ineffective pattern-like/unknown entries (exact node command-name matching only, not shell-text filtering), when `gateway.nodes.allowCommands` explicitly enables dangerous node commands, when global `tools.profile="minimal"` is overridden by agent tool profiles, when write/edit tools are disabled but `exec` is still available without a constraining sandbox filesystem boundary, when open groups expose runtime/filesystem tools without sandbox/workspace guards, and when installed plugin tools may be reachable under permissive tool policy.
It also flags `gateway.allowRealIpFallback=true` (header-spoofing risk if proxies are misconfigured) and `discovery.mdns.mode="full"` (metadata leakage via mDNS TXT records).
It also warns when sandbox browser uses Docker `bridge` network without `sandbox.browser.cdpSourceRange`.
It also flags dangerous sandbox Docker network modes (including `host` and `container:*` namespace joins).
It also warns when existing sandbox browser Docker containers have missing/stale hash labels (for example pre-migration containers missing `openclaw.browserConfigEpoch`) and recommends `openclaw sandbox recreate --browser --all`.
It also warns when npm-based plugin/hook install records are unpinned, missing integrity metadata, or drift from currently installed package versions.
It warns when channel allowlists rely on mutable names/emails/tags instead of stable IDs (Discord, Slack, Google Chat, Microsoft Teams, Mattermost, IRC scopes where applicable).
It warns when `gateway.auth.mode="none"` leaves Gateway HTTP APIs reachable without a shared secret (`/tools/invoke` plus any enabled `/v1/*` endpoint).
Settings prefixed with `dangerous`/`dangerously` are explicit break-glass operator overrides; enabling one is not, by itself, a security vulnerability report.
For the complete dangerous-parameter inventory, see the "Insecure or dangerous flags summary" section in [Security](/gateway/security).

SecretRef behavior:

- `security audit` resolves supported SecretRefs in read-only mode for its targeted paths.
- If a SecretRef is unavailable in the current command path, audit continues and reports `secretDiagnostics` (instead of crashing).
- `--token` and `--password` only override deep-probe auth for that command invocation; they do not rewrite config or SecretRef mappings.

## JSON output

Use `--json` for CI/policy checks:

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

If `--fix` and `--json` are combined, output includes both fix actions and final report:

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## What `--fix` changes

`--fix` applies safe, deterministic remediations:

- flips common `groupPolicy="open"` to `groupPolicy="allowlist"` (including account variants in supported channels)
- when WhatsApp group policy flips to `allowlist`, seeds `groupAllowFrom` from
  the stored `allowFrom` file when that list exists and config does not already
  define `allowFrom`
- sets `logging.redactSensitive` from `"off"` to `"tools"`
- tightens permissions for state/config and common sensitive files
  (`credentials/*.json`, `auth-profiles.json`, `sessions.json`, session
  `*.jsonl`)
- also tightens config include files referenced from `openclaw.json`
- uses `chmod` on POSIX hosts and `icacls` resets on Windows

`--fix` does **not**:

- rotate tokens/passwords/API keys
- disable tools (`gateway`, `cron`, `exec`, etc.)
- change gateway bind/auth/network exposure choices
- remove or rewrite plugins/skills

## Related

- [CLI reference](/cli)
- [Security audit](/gateway/security)

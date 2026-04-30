---
domain: cli
topic: "openclaw setup: Initialize Config and Workspace"
type: procedure
keywords:
  - openclaw setup
  - openclaw onboard
  - first run setup
  - workspace initialization
  - wizard setup
  - non-interactive setup
  - remote gateway setup
  - import migration
source: cli/setup.md
related:
  - cli/cli-overview
  - getting-started/onboarding-wizard
  - install/installation-methods
---

`openclaw setup` initializes `~/.openclaw/openclaw.json` and the agent workspace. Plain `openclaw setup` initializes without the full onboarding flow; use `--wizard` to run the full wizard.

## Examples

```bash
# Minimal init (no wizard)
openclaw setup

# Set workspace path
openclaw setup --workspace ~/.openclaw/workspace

# Full onboarding wizard
openclaw setup --wizard

# Onboarding with migration from Hermes
openclaw setup --wizard --import-from hermes --import-source ~/.hermes

# Non-interactive remote mode
openclaw setup --non-interactive --mode remote \
  --remote-url wss://gateway-host:18789 \
  --remote-token <token>
```

## Options

- `--workspace <dir>`: agent workspace directory (stored as `agents.defaults.workspace`)
- `--wizard`: run onboarding wizard
- `--non-interactive`: run onboarding without prompts
- `--mode <local|remote>`: onboarding mode
- `--import-from <provider>`: migration provider to run during onboarding
- `--import-source <path>`: source agent home for `--import-from`
- `--import-secrets`: import supported secrets during onboarding migration
- `--remote-url <url>`: remote Gateway WebSocket URL
- `--remote-token <token>`: remote Gateway token

## Notes

- Onboarding auto-runs when any onboarding flags are present.
- If Hermes state is detected, interactive onboarding can offer migration automatically.
- Import onboarding requires a fresh setup; use [`openclaw migrate`](/cli/migrate) for dry-run plans and backup mode outside onboarding.

## Related

- [CLI overview](/cli)
- [Onboarding wizard](/getting-started/onboarding-wizard)
- [Migration](/install/migration)

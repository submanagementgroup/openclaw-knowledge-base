---
domain: install
topic: "Nix: Declarative OpenClaw Install with Home Manager and nix-openclaw"
type: procedure
keywords:
  - nix
  - NixOS
  - Home Manager
  - nix-openclaw
  - declarative install
  - reproducible install
  - OPENCLAW_NIX_MODE
  - flake openclaw
  - rollback openclaw
  - nix mode
  - OPENCLAW_STATE_DIR
source: install/nix.md
related:
  - install/installation-methods
  - gateway/gateway-runbook
---

Install OpenClaw declaratively with [nix-openclaw](https://github.com/openclaw/nix-openclaw) — a batteries-included Home Manager module. All dependencies are pinned; instant rollback is available via `home-manager switch --rollback`.

## What You Get

- Gateway + macOS app + tools (whisper, spotify, cameras) — all pinned
- Launchd service that survives reboots
- Plugin system with declarative config
- Instant rollback: `home-manager switch --rollback`

## Quick Start

1. **Install Determinate Nix** — follow the [Determinate Nix installer](https://github.com/DeterminateSystems/nix-installer) if Nix is not already installed.

2. **Create a local flake** — use the agent-first template from the nix-openclaw repo:
   ```bash
   mkdir -p ~/code/openclaw-local
   # Copy templates/agent-first/flake.nix from the nix-openclaw repo
   ```

3. **Configure secrets** — set up your messaging bot token and model provider API key. Plain files at `~/.secrets/` work fine.

4. **Fill in template placeholders and switch:**
   ```bash
   home-manager switch
   ```

5. **Verify** — confirm the launchd service is running and your bot responds to messages.

See the [nix-openclaw README](https://github.com/openclaw/nix-openclaw) for full module options and examples.

## Nix Mode Runtime Behavior

When `OPENCLAW_NIX_MODE=1` is set (automatic with nix-openclaw), OpenClaw enters a deterministic mode that disables auto-install flows.

Set manually when needed:

```bash
export OPENCLAW_NIX_MODE=1
```

On macOS, the GUI app does not automatically inherit shell environment variables. Enable Nix mode via defaults instead:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### What Changes in Nix Mode

- Auto-install and self-mutation flows are disabled
- Missing dependencies surface Nix-specific remediation messages
- UI surfaces a read-only Nix mode banner

### Config and State Paths

OpenClaw reads JSON5 config from `OPENCLAW_CONFIG_PATH` and stores mutable data in `OPENCLAW_STATE_DIR`. Set these explicitly to Nix-managed locations so runtime state and config stay out of the immutable store.

| Variable | Default |
|----------|---------|
| `OPENCLAW_HOME` | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR` | `~/.openclaw` |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json` |

## Related

- [Installation methods](/install/installation-methods)
- [Gateway runbook](/gateway/gateway-runbook)

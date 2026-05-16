---
domain: install
topic: "Alternative Installation Methods: Bun, Nix, macOS VM"
type: procedure
keywords:
  - bun
  - nix
  - macOS VM
  - alternative install
  - nix flake
source: 
  - install/bun.md
  - install/nix.md
  - install/macos-vm.md
---

## Bun

> **Note:** Bun is **not recommended for gateway runtime** (known issues with WhatsApp and Telegram). Use Node for production.


Bun is an optional local runtime for running TypeScript directly (`bun run ...`, `bun --watch ...`). The default package manager remains `pnpm`, which is fully supported and used by docs tooling. Bun cannot use `pnpm-lock.yaml` and will ignore it.

## Install

**Install dependencies**

```sh
    bun install
    ```

    `bun.lock` / `bun.lockb` are gitignored, so there is no repo churn. To skip lockfile writes entirely:

    ```sh
    bun install --no-save
    ```

  
**Build and test**

```sh
    bun run build
    bun run vitest run
    ```
  
## Lifecycle scripts

Bun blocks dependency lifecycle scripts unless explicitly trusted. For this repo, the commonly blocked scripts are not required:

- `baileys` `preinstall` -- checks Node major >= 20 (OpenClaw defaults to Node 24 and still supports Node 22 LTS, currently `22.16+`)
- `protobufjs` `postinstall` -- emits warnings about incompatible version schemes (no build artifacts)

If you hit a runtime issue that requires these scripts, trust them explicitly:

```sh
bun pm trust baileys protobufjs
```

## Caveats

Some scripts still hardcode pnpm (for example `docs:build`, `ui:*`, `protocol:check`). Run those via pnpm for now.

## Related

- [Install overview](/install)
- [Node.js](/install/node)
- [Updating](/install/updating)


## Nix

Install OpenClaw declaratively with **[nix-openclaw](https://github.com/openclaw/nix-openclaw)** - the first-party, batteries-included Home Manager module.

> **Note:** The [nix-openclaw](https://github.com/openclaw/nix-openclaw) repo is the source of truth for Nix installation. This page is a quick overview.


## What you get

- Gateway + macOS app + tools (whisper, spotify, cameras) -- all pinned
- Launchd service that survives reboots
- Plugin system with declarative config
- Instant rollback: `home-manager switch --rollback`

## Quick start

**Install Determinate Nix**

If Nix is not already installed, follow the [Determinate Nix installer](https://github.com/DeterminateSystems/nix-installer) instructions.
  
**Create a local flake**

Use the agent-first template from the nix-openclaw repo:
    ```bash
    mkdir -p ~/code/openclaw-local
    # Copy templates/agent-first/flake.nix from the nix-openclaw repo
    ```
  
**Configure secrets**

Set up your messaging bot token and model provider API key. Plain files at `~/.secrets/` work fine.
  
**Fill in template placeholders and switch**

```bash
    home-manager switch
    ```
  
**Verify**

Confirm the launchd service is running and your bot responds to messages.
  
See the [nix-openclaw README](https://github.com/openclaw/nix-openclaw) for full module options and examples.

## Nix-mode runtime behavior

When `OPENCLAW_NIX_MODE=1` is set (automatic with nix-openclaw), OpenClaw enters a deterministic mode for Nix-managed installs. Other Nix packages can set the same mode; nix-openclaw is the first-party reference.

You can also set it manually:

```bash
export OPENCLAW_NIX_MODE=1
```

On macOS, the GUI app does not automatically inherit shell environment variables. Enable Nix mode via defaults instead:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### What changes in Nix mode

- Auto-install and self-mutation flows are disabled
- `openclaw.json` is treated as immutable. Startup-derived defaults stay runtime-only, and config writers such as setup, onboarding, mutating `openclaw update`, plugin install/update/uninstall/enable, `doctor --fix`, `doctor --generate-gateway-token`, and `openclaw config set` refuse to edit the file.
- Agents should edit the Nix source instead. For nix-openclaw, use the agent-first [Quick Start](https://github.com/openclaw/nix-openclaw#quick-start) and set config under `programs.openclaw.config` or `instances.<name>.config`.
- Missing dependencies surface Nix-specific remediation messages
- UI surfaces a read-only Nix mode banner

### Config and state paths

OpenClaw reads JSON5 config from `OPENCLAW_CONFIG_PATH` and stores mutable data in `OPENCLAW_STATE_DIR`. When running under Nix, set these explicitly to Nix-managed locations so runtime state and config stay out of the immutable store.

| Variable               | Default                                 |
| ---------------------- | --------------------------------------- |
| `OPENCLAW_HOME`        | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR`   | `~/.openclaw`                           |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json`     |

### Service PATH discovery

The launchd/systemd gateway service auto-discovers Nix-profile binaries so
plugins and tools that shell out to `nix`-installed executables work without
manual PATH setup:

- When `NIX_PROFILES` is set, every entry is added to the service PATH in
  right-to-left precedence (matches Nix shell precedence - rightmost wins).
- When `NIX_PROFILES` is unset, `~/.nix-profile/bin` is added as a fallback.

This applies to both macOS launchd and Linux systemd service environments.

## Related

**nix-openclaw:** Source-of-truth Home Manager module and full setup guide.
  

**Setup wizard:** Non-Nix CLI setup walkthrough.
  

**Docker:** Containerized setup as a non-Nix alternative.
  

**Updating:** Updating Home Manager-managed installs alongside the package.
  



## macOS VM

## Recommended default (most users)

- **Small Linux VPS** for an always-on Gateway and low cost. See [VPS hosting](/vps).
- **Dedicated hardware** (Mac mini or Linux box) if you want full control and a **residential IP** for browser automation. Many sites block data center IPs, so local browsing often works better.
- **Hybrid:** keep the Gateway on a cheap VPS, and connect your Mac as a **node** when you need browser/UI automation. See [Nodes](/nodes) and [Gateway remote](/gateway/remote).

Use a macOS VM when you specifically need macOS-only capabilities such as iMessage or want strict isolation from your daily Mac.

## macOS VM options

### Local VM on your Apple Silicon Mac (Lume)

Run OpenClaw in a sandboxed macOS VM on your existing Apple Silicon Mac using [Lume](https://cua.ai/docs/lume).

This gives you:

- Full macOS environment in isolation (your host stays clean)
- iMessage support via `imsg` (the default local path is impossible on Linux/Windows)
- Instant reset by cloning VMs
- No extra hardware or cloud costs

### Hosted Mac providers (cloud)

If you want macOS in the cloud, hosted Mac providers work too:

- [MacStadium](https://www.macstadium.com/) (hosted Macs)
- Other hosted Mac vendors also work; follow their VM + SSH docs

Once you have SSH access to a macOS VM, continue at step 6 below.

---

## Quick path (Lume, experienced users)

1. Install Lume
2. `lume create openclaw --os macos --ipsw latest`
3. Complete Setup Assistant, enable Remote Login (SSH)
4. `lume run openclaw --no-display`
5. SSH in, install OpenClaw, configure channels
6. Done

---

## What you need (Lume)

- Apple Silicon Mac (M1/M2/M3/M4)
- macOS Sequoia or later on the host
- ~60 GB free disk space per VM
- ~20 minutes

---

## 1) Install Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

If `~/.local/bin` isn't in your PATH:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

Verify:

```bash
lume --version
```

Docs: [Lume Installation](https://cua.ai/docs/lume/guide/getting-started/installation)

---

## 2) Create the macOS VM

```bash
lume create openclaw --os macos --ipsw latest
```

This downloads macOS and creates the VM. A VNC window opens automatically.

> **Note:** The download can take a while depending on your connection.


---

## 3) Complete Setup Assistant

In the VNC window:

1. Select language and region
2. Skip Apple ID (or sign in if you want iMessage later)
3. Create a user account (remember the username and password)
4. Skip all optional features

After setup completes, enable SSH:

1. Open System Settings → General → Sharing
2. Enable "Remote Login"

---

## 4) Get the VM IP address

```bash
lume get openclaw
```

Look for the IP address (usually `192.168.64.x`).

---

## 5) SSH into the VM

```bash
ssh youruser@192.168.64.X
```

Replace `youruser` with the account you created, and the IP with your VM's IP.

---

## 6) Install OpenClaw

Inside the VM:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

Follow the onboarding prompts to set up your model provider (Anthropic, OpenAI, etc.).

---

## 7) Configure channels

Edit the config file:

```bash
nano ~/.openclaw/openclaw.json
```

Add your channels:

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
  },
}
```

Then login to WhatsApp (scan QR):

```bash
openclaw channels login
```

---

## 8) Run the VM headlessly

Stop the VM and restart without display:

```bash
lume stop openclaw
lume run openclaw --no-display
```

The VM runs in the background. OpenClaw's daemon keeps the gateway running.

To check status:

```bash
ssh youruser@192.168.64.X "openclaw status"
```

---

## Bonus: iMessage integration

This is the killer feature of running on macOS. Use [iMessage](/channels/imessage) with `imsg` to add Messa
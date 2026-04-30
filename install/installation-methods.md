---
domain: install
topic: "Installation Methods: npm, One-Line Installer, Bun, Nix, and Node Requirements"
type: procedure
keywords:
  - install
  - npm install
  - openclaw install
  - installer
  - Node.js
  - Bun
  - Nix
related:
  - getting-started/quickstart
  - install/docker
  - install/updating
source:
  - install/index.md
  - install/installer.md
  - install/node.md
  - install/bun.md
---

OpenClaw install methods: npm (recommended), one-line installer, Bun, Nix, Docker, and platform-specific packages.

## Install Methods

### npm (Recommended)
```bash
npm install -g openclaw@latest
```
Node 24 recommended; Node 22 LTS (22.14+) minimum.

### One-Line Installer
```bash
curl -fsSL https://install.openclaw.dev | sh
```

### Bun
Bun is **not recommended for gateway runtime** (known issues with WhatsApp and Telegram). Use Node for production.

Bun is an optional local runtime for running TypeScript directly (`bun run ...`, `bun --watch ...`). The default package manager remains `pnpm`, which is fully supported and used by docs tooling. Bun cannot use `pnpm-lock.yaml` and will ignore it.

## Install

    ```sh
    bun install
    ```

    `bun.lock` / `bun.lockb` are gitignored, so there is no repo churn. To skip lockfile writes entirely:

    ```sh
    bun install --no-save
    ```

    ```sh
    bun run build
    bun run vitest run
    ```

## Lifecycle scripts

Bun blocks dependency lifecycle scripts unless explicitly trusted. For this repo, the commonly blocked scripts are not required:

- `@whiskeysockets/baileys` `preinstall` -- checks Node major >= 20 (OpenClaw defaults to Node 24 and still supports Node 22 LTS, currently `22.14+`)
- `protobufjs` `postinstall` -- emits warnings about incompatible version 

### Nix
Install OpenClaw declaratively with **[nix-openclaw](https://github.com/openclaw/nix-openclaw)** — a batteries-included Home Manager module.

The [nix-openclaw](https://github.com/openclaw/nix-openclaw) repo is the source of truth for Nix installation. This page is a quick overview.

## What you get

- Gateway + macOS app + tools (whisper, spotify, cameras) -- all pinned
- Launchd service that survives reboots
- Plugin system with declarative config
- Instant rollback: `home-manager switch --rollback`

## Quick start

    If Nix is not already installed, follow the [Determinate Nix installer](https://github.com/DeterminateSystems/nix-installer) instructions.

    Use the agent-first template from the nix-openclaw repo:
    ```bash
    mkdir -p ~/code/openclaw-local
    # Copy templates/agent-first/flake.nix from the nix-openclaw repo
    ```

    Set up your messaging bot token and model provider API key. Plain files at `~/.secrets/` work fine.

    ```bash
    home-manager switch
    

## Installer Details

OpenClaw ships three installer scripts, served from `openclaw.ai`.

| Script                             | Platform             | What it does                                                                                                   |
| ---------------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------- |
| [`install.sh`](#installsh)         | macOS / Linux / WSL  | Installs Node if needed, installs OpenClaw via npm (default) or git, and can run onboarding.                   |
| [`install-cli.sh`](#install-clish) | macOS / Linux / WSL  | Installs Node + OpenClaw into a local prefix (`~/.openclaw`) with npm or git checkout modes. No root required. |
| [`install.ps1`](#installps1)       | Windows (PowerShell) | Installs Node if needed, installs OpenClaw via npm (default) or git, and can run onboarding.                   |

## Quick commands

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --help
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --help
    ```

    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```

    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -Tag beta -NoOnboard -DryRun
    ```

If install succeeds but `openclaw` is not found in a new terminal, see [Node.js troubleshooting](/install/node#troubleshooting).

---

<a id="installsh"></a>

## install.sh

Recommended for most interactive installs on macOS/Linux/WSL.

### Flow (install.sh)

    Supports macOS and Linux (including WSL). If macOS is detected, installs Homebrew if missing.

    Checks Node version and installs Node 24 if needed (Homebrew on macOS, NodeSource setup scripts on Linux apt/dnf/yum). OpenClaw still supports Node 22 LTS, currently `22.14+`, for compatibility.

    Installs Git if missing.

    - `npm` method (default): global npm install
    - `git` method: clone/update repo, install deps with pnpm, build, then install wrapper at `~/.local/bin/openclaw`

    - Refreshes a loaded gateway service best-effort (`openclaw gateway install --force`, then restart)
    - Runs `openclaw doctor --non-interactive` on upgrades and git installs (best effort)
    - Attempts onboarding when appropriate (TTY available, onboarding not disabled, and bootstrap/config checks pass)
    - Defaults `SHARP_IGNORE_GLOBAL_LIBVIPS=1`

### Source checkout detection

If run inside an OpenClaw checkout (`package.json` + `pnpm-workspace.yaml`), the script offers:

- use checkout (`git`), or
- use global install (`npm`)

If no TTY is available and no install method is set, it defaults to `npm` and warns.

The script exits with code `2` for invalid method selection or invalid `--install-method` values.

### Examples (install.sh)

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --version main
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --dry-run
    ```

| Flag                                  | Description                                                |
| ------------------------------------- | ---------------------------------------------------------- |
| `--install-method npm\|git`           | Choose install method (default: `npm`).

## Node Version Requirements

OpenClaw requires **Node 22.14 or newer**. **Node 24 is the default and recommended runtime** for installs, CI, and release workflows. Node 22 remains supported via the active LTS line. The [installer script](/install#alternative-install-methods) will detect and install Node automatically — this page is for when you want to set up Node yourself and make sure everything is wired up correctly (versions, PATH, global installs).

## Check your version

```bash
node -v
```

If this prints `v24.x.x` or higher, you're on the recommended default. If it prints `v22.14.x` or higher, you're on the supported Node 22 LTS path, but we still recommend upgrading to Node 24 when convenient. If Node isn't installed or the version is too old, pick an install method below.

## Install Node

    **Homebrew** (recommended):

    ```bash
    brew install node
    ```

    Or download the macOS installer from [nodejs.org](https://nodejs.org/).

    **Ubuntu / Debian:**

    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt-get install -y nodejs
    ```

    **Fedora / RHEL:**

    ```bash
    sudo dnf install nodejs
    ```

    Or use a version manager (see below).

    **winget** (recommended):

    ```powershell
    winget install OpenJS.NodeJS.LTS
    ```

    **Chocolatey:**

    ```powershell
    choco install nodejs-lts
    ```

    Or download the Windows installer from [nodejs.org](https://nodejs.org/).

  Version managers let you switch between Node versions easily. Popular options:

- [**fnm**](https://github.com/Schniz/fnm) — fast, cross-platform
- [**nvm**](https://github.com/nvm-sh/nvm) — widely used on macOS/Linux
- [**mise**](https://mise.jdx.dev/) — polyglot (Node, Python, Ruby, etc.)

Example with fnm:

```bash
fnm install 24
fnm use 24
```

  Make sure your version manager is initialized in your shell startup file (`~/.zshrc` or `~/.bashrc`). If it isn't, `openclaw` may not be found in new terminal sessions because the PATH won't 

## System requirements

- **Node 24** (recommended) or Node 22.14+ — the installer script handles this automatically
- **macOS, Linux, or Windows** — both native Windows and WSL2 are supported; WSL2 is more stable. See [Windows](/platforms/windows).
- `pnpm` is only needed if you build from source

## Recommended: installer script

The fastest way to install. It detects your OS, installs Node if needed, installs OpenClaw, and launches onboarding.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```

    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```

To install without running onboarding:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```

    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```

For all flags and CI/automation options, see [Installer internals](/install/installer).

## Alternative install methods

### Local prefix installer (`install-cli.sh`)

Use this when you want OpenClaw and Node kept under a local prefix such as
`~/.openclaw`, without depending on a system-wide Node install:

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

It supports npm installs by default, plus git-checkout installs under the same
prefix flow. Full reference: [Installer internals](/install/installer#install-clish).

Already installed? Switch between package and git installs with
`openclaw update --channel dev` and `openclaw update --channel stable`. See
[Updating](/install/updating#switch-between-npm-and-git-installs).

### npm, pnpm, or bun

If you already manage Node yourself:

    ```bash
    npm install -g openclaw@latest
    openclaw onboard --install-daemon
    ```

    ```bash
    pnpm add -g openclaw@latest
    pnpm approve-builds -g
    openclaw onboard --install-daemon
    ```

    pnpm requires explicit approval for packages with build scripts. Run `pnpm approve-builds -g` after the first install.

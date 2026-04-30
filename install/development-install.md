---
domain: install
topic: "Development Channels: Installing Pre-Release Builds and Dev Mode"
type: reference
keywords:
  - development channels
  - pre-release
  - beta
  - dev install
  - exe dev
related:
  - install/installation-methods
source:
  - install/development-channels.md
  - install/exe-dev.md
---

Installing development/pre-release builds and contributing to OpenClaw development.

## Development Channels

# Development channels

OpenClaw ships three update channels:

- **stable**: npm dist-tag `latest`. Recommended for most users.
- **beta**: npm dist-tag `beta` when it is current; if beta is missing or older than
  the latest stable release, the update flow falls back to `latest`.
- **dev**: moving head of `main` (git). npm dist-tag: `dev` (when published).
  The `main` branch is for experimentation and active development. It may contain
  incomplete features or breaking changes. Do not use it for production gateways.

We usually ship stable builds to **beta** first, test them there, then run an
explicit promotion step that moves the vetted build to `latest` without
changing the version number. Maintainers can also publish a stable release
directly to `latest` when needed. Dist-tags are the source of truth for npm
installs.

## Switching channels

```bash
openclaw update --channel stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` persists your choice in config (`update.channel`) and aligns the
install method:

- **`stable`** (package installs): updates via npm dist-tag `latest`.
- **`beta`** (package installs): prefers npm dist-tag `beta`, but falls back to
  `latest` when `beta` is missing or older than the current stable tag.
- **`stable`** (git installs): checks out the latest stable git tag.
- **`beta`** (git installs): prefers the latest beta git tag, but falls back to
  the latest stable git tag when beta is missing or older.
- **`dev`**: ensures a git checkout (default `~/openclaw`, override with
  `OPENCLAW_GIT_DIR`), switches to `main`, rebases on upstream, builds, and
  installs the global CLI from that checkout.

If you want stable and dev in parallel, keep two clones and point your gateway at the stable one.

## One-off version or tag targeting

Use `--tag` to target a specific dist-tag, version, or package spec for a single
update **without** changing your persisted channel:

```bash
# Install a specific version
openclaw update --tag 2026.4.1-beta.1

# Install from the beta dist-tag (one-off, does not persist)
openclaw update --tag beta

# Install from GitHub main branch (npm tarball)
openclaw update --tag main

# Install a specific npm package spec
openclaw update --tag openclaw@2026.4.1-beta.1
```

Notes:

- `--tag` applies to **package (npm) installs only**. Git installs ignore it.
- The tag is not persisted. Your next `openclaw update` uses your configured
  channel as usual.
- Downgrade protection: if the target version is older than your current version,
  OpenClaw prompts for confirmation (skip with `--yes`).
- `--channel beta` is different from `--tag beta`: the channel flow can fall back
  to stable/latest when beta is missing or older, while `--tag beta` targets the
  raw `beta` dist-tag for that one run.

## Dry run

Preview what `openclaw update` would do without making changes:

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

The dry run shows the effective channel, target version, planned actions, and
whether a downgrade confirmation would be required.

## Plugins and channels

When you switch channels with `openclaw update`, OpenClaw also syncs plugin
sources:

- `dev` prefers bundled plugins from the git checkout.
- `stable` and `beta` restore npm-installed plugin packages.
- npm-installed plugins are updated after the core update completes.

## Checking current status

```bash
openclaw update status
```

Shows the active channel, install kind (git or package), current version, and
source (config, git tag, git branch, or default).

## Tagging best practices

- Tag releases you want git checkouts to land on (`vYYYY.M.D` for stable,
  `vYYYY.M.D-beta.N` for beta).
- `vYYYY.M.D.beta.N` is also recognized for compatibility, but prefer `-beta.N`.
- Legacy `vYYYY.M.D-<patch>` tags are still recognized as stable (non-beta).
- Keep tags immutable: never move or

## Exe Dev Mode

Goal: OpenClaw Gateway running on an exe.dev VM, reachable from your laptop via: `https://<vm-name>.exe.xyz`

This page assumes exe.dev's default **exeuntu** image. If you picked a different distro, map packages accordingly.

## Beginner quick path

1. [https://exe.new/openclaw](https://exe.new/openclaw)
2. Fill in your auth key/token as needed
3. Click on "Agent" next to your VM and wait for Shelley to finish provisioning
4. Open `https://<vm-name>.exe.xyz/` and authenticate with the configured shared secret (this guide uses token auth by default, but password auth works too if you switch `gateway.auth.mode`)
5. Approve any pending device pairing requests with `openclaw devices approve <requestId>`

## What you need

- exe.dev account
- `ssh exe.dev` access to [exe.dev](https://exe.dev) virtual machines (optional)

## Automated install with Shelley

Shelley, [exe.dev](https://exe.dev)'s agent, can install OpenClaw instantly with our
prompt. The prompt used is as below:

```
Set up OpenClaw (https://docs.openclaw.ai/install) on this VM. Use the non-interactive and accept-risk flags for openclaw onboarding. Add the supplied auth or token as needed. Configure nginx to forward from the default port 18789 to the root location on the default enabled site config, making sure to enable Websocket support. Pairing is done by "openclaw devices list" and "openclaw devices approve <request id>". Make sure the dashboard shows that OpenClaw's health is OK. exe.dev handles forwarding from port 8000 to port 80/443 and HTTPS for us, so the final "reachable" should be <vm-name>.exe.xyz, without port specification.
```

## Manual installation

## 1) Create the VM

From your device:

```bash
ssh exe.dev new
```

Then connect:

```bash
ssh <vm-name>.exe.xyz
```

Keep this VM **stateful**. OpenClaw stores `openclaw.json`, per-agent `auth-profiles.json`, sessions, and channel/provider state under `~/.openclaw/`, plus the workspace under `~/.openclaw/workspace/`.

## 2) Install prerequisites (on the VM)

```bash
sudo apt-get update
sudo apt-get install -y git curl jq ca-certificates openssl
```

## 3) Install OpenClaw

Run the OpenClaw install script:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

## 4) Setup nginx to proxy OpenClaw to port 8000

Edit `/etc/nginx/sites-enabled/default` with

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 8000;
    listen [::]:8000;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings for long-lived connections
        proxy_read_timeout 86400s;

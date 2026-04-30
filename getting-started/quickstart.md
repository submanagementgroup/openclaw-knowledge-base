---
domain: getting-started
topic: "OpenClaw Quickstart: Install, Start Gateway, and Connect a Channel"
type: procedure
keywords:
  - quickstart
  - install
  - npm
  - openclaw onboard
  - gateway startup
  - first run
  - getting started
related:
  - getting-started/what-is-openclaw
  - install/installation-methods
  - gateway/gateway-runbook
source:
  - start/getting-started.md
  - start/quickstart.md
  - start/setup.md
---

OpenClaw installs via npm or the one-line installer. The minimal path is `npm install -g openclaw@latest` followed by `openclaw onboard --install-daemon` to start the Gateway as a background service.

## Install Options

**npm (recommended for most users):**
```bash
npm install -g openclaw@latest
```
Node 24 is recommended; Node 22 LTS (22.14+) is the minimum.

**One-line installer:**
```bash
curl -fsSL https://install.openclaw.dev | sh
```

## First-Run Steps

```bash
# Interactive onboarding (recommended)
openclaw onboard --install-daemon

# Manual startup
openclaw gateway --port 18789

# Open the browser dashboard
openclaw dashboard
```

## Verify the Gateway is Running

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
```

Healthy output shows: `Runtime: running`, `Connectivity probe: ok`, and capabilities matching your config.

## Connect a Channel (Fastest: Telegram)

1. Create a Telegram bot via [@BotFather](https://t.me/botfather)
2. Add the bot token to config:
   ```json5
   { "channels": { "telegram": { "botToken": "YOUR_TOKEN" } } }
   ```
3. Restart the Gateway and message your bot

Install OpenClaw, run onboarding, and chat with your AI assistant — all in
about 5 minutes. By the end you will have a running Gateway, configured auth,
and a working chat session.

## What you need

- **Node.js** — Node 24 recommended (Node 22.14+ also supported)
- **An API key** from a model provider (Anthropic, OpenAI, Google, etc.) — onboarding will prompt you

Check your Node version with `node --version`.
**Windows users:** both native Windows and WSL2 are supported. WSL2 is more
stable and recommended for the full experience. See [Windows](/platforms/windows).
Need to install Node? See [Node setup](/install/node).

## Quick setup

        ```bash
        curl -fsSL https://openclaw.ai/install.sh | bash
        ```

        ```powershell
        iwr -useb https://openclaw.ai/install.ps1 | iex
        ```

    Other install methods (Docker, Nix, npm): [Install](/install).

    ```bash
    openclaw onboard --install-daemon
    ```

    The wizard walks you through choosing a model provider, setting an API key,
    and configuring the Gateway. It takes about 2 minutes.

    See [Onboarding (CLI)](/start/wizard) for the full reference.

    ```bash
    openclaw gateway status
    ```

    You should see the Gateway listening on port 18789.

    ```bash
    openclaw dashboard
    ```

    This opens the Control UI in your browser. If it loads, everything is working.

    Type a message in the Control UI chat and you should get an AI reply.

    Want to chat from your phone instead? The fastest channel to set up is
    [Telegram](/channels/telegram) (just a bot token). See [Channels](/channels)
    for all options.

  If you maintain a localized or customized dashboard build, point
  `gateway.controlUi.root` to a directory that contains your built static
  assets and `index.html`.

```bash
mkdir -p "$HOME/.openclaw/control-ui-custom"
# Copy your built static files into that directory.
```

Then set:

```json
{
  "gateway": {
    "controlUi": {
      "enabled": true,

If you are setting up for the first time, start with [Getting Started](/start/getting-started).
For onboarding details, see [Onboarding (CLI)](/start/wizard).

## TL;DR

Pick a setup workflow based on how often you want updates and whether you want to run the Gateway yourself:

- **Tailoring lives outside the repo:** keep your config and workspace in `~/.openclaw/openclaw.json` and `~/.openclaw/workspace/` so repo updates don't touch them.
- **Stable workflow (recommended for most):** install the macOS app and let it run the bundled Gateway.
- **Bleeding edge workflow (dev):** run the Gateway yourself via `pnpm gateway:watch`, then let the macOS app attach in Local mode.

## Prereqs (from source)

- Node 24 recommended (Node 22 LTS, currently `22.14+`, still supported)
- `pnpm` preferred (or Bun if you intentionally use the [Bun workflow](/install/bun))
- Docker (optional; only for containerized setup/e2e — see [Docker](/install/docker))

## Tailoring strategy (so updates do not hurt)

If you want “100% tailored to me” _and_ easy updates, keep your customization in:

- **Config:** `~/.openclaw/openclaw.json` (JSON/JSON5-ish)
- **Workspace:** `~/.openclaw/workspace` (skills, prompts, memories; make it a private git repo)

Bootstrap once:

```bash
openclaw setup
```

From inside this repo, use the local CLI entry:

```bash
openclaw setup
```

If you don’t have a global install yet, run it via `pnpm openclaw setup` (or `bun run openclaw setup` if you are using the Bun workflow).

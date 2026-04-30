---
domain: getting-started
topic: "What Is OpenClaw: Self-Hosted AI Agent Gateway"
type: concept
keywords:
  - OpenClaw
  - gateway
  - self-hosted
  - AI agent
  - multi-channel
  - Pi agent
  - overview
related:
  - getting-started/quickstart
  - getting-started/installation-overview
  - concepts/architecture
source:
  - index.md
  - start/openclaw.md
  - start/lore.md
---

OpenClaw is a self-hosted, multi-channel gateway for AI agents (such as the Pi agent) that runs on any OS. It connects Discord, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Google Chat, Zalo, and more to AI agents — giving you a personal AI assistant accessible from any chat app.

## What OpenClaw Is and Who It Is For

OpenClaw is for developers and power users who want a personal AI assistant they can message from anywhere, without giving up control of their data. It is self-hosted (runs on your own machine or server), MIT-licensed, and open source.

**Core architecture:**
- One Gateway process handles all channel connections and agent routing
- The Gateway bridges messaging apps to AI agents like Pi
- Built-in channels plus bundled/external channel plugins extend coverage
- Mobile nodes (iOS and Android) pair with the Gateway for voice, camera, and Canvas

## Key Capabilities

- **Multi-channel**: Discord, iMessage, Signal, Slack, Telegram, WhatsApp, Google Chat, Matrix, Teams, and many more via plugins
- **Agent-native**: tool use, sessions, memory, and multi-agent routing built in
- **Self-hosted**: full data sovereignty; runs on Mac, Linux, Windows, Raspberry Pi, VPS
- **Extensible**: plugin system for channels, providers, and tools

## How it Works

```
Chat apps + plugins → Gateway → Pi agent
                              → CLI
                              → Web Control UI
                              → macOS app
                              → iOS and Android nodes
```

The Gateway is the single source of truth for sessions, routing, and channel connections.

# Building a personal assistant with OpenClaw

OpenClaw is a self-hosted gateway that connects Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo, and more to AI agents. This guide covers the "personal assistant" setup: a dedicated WhatsApp number that behaves like your always-on AI assistant.

## ⚠️ Safety first

You’re putting an agent in a position to:

- run commands on your machine (depending on your tool policy)
- read/write files in your workspace
- send messages back out via WhatsApp/Telegram/Discord/Mattermost and other bundled channels

Start conservative:

- Always set `channels.whatsapp.allowFrom` (never run open-to-the-world on your personal Mac).
- Use a dedicated WhatsApp number for the assistant.
- Heartbeats now default to every 30 minutes. Disable until you trust the setup by setting `agents.defaults.heartbeat.every: "0m"`.

## Prerequisites

- OpenClaw installed and onboarded — see [Getting Started](/start/getting-started) if you haven't done this yet
- A second phone number (SIM/eSIM/prepaid) for the assistant

## The two-phone setup (recommended)

You want this:

If you link your personal WhatsApp to OpenClaw, every message to you becomes “agent input”. That’s rarely what you want.

## 5-minute quick start

1. Pair WhatsApp Web (shows QR; scan with the assistant phone):

```bash
openclaw channels login
```

2. Start the Gateway (leave it running):

```bash
openclaw gateway --port 18789
```

3. Put a minimal config in `~/.openclaw/openclaw.json`:

```json5
{
  gateway: { mode: "local" },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

Now message the assistant number from your allowlisted phone.

When onboarding finishes, OpenClaw auto-opens the dashboard and prints a clean (non-tokenized) link. If the dashboard prompts for auth, paste the configured shared secret into Control UI settings. Onboarding uses a token by default (`gateway.auth.token`), but password auth works too if you switched `gateway.auth.mode` to `password`. To reopen later: `openclaw dashboard`.

## Give the agent a workspace (AGENTS)

OpenClaw reads operating instructions and “memory” from its workspace directory.

By default, OpenClaw uses `~/.openclaw/workspace` as the agent workspace, and will create it (plus starter `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`) automatically on setup/first agent run. `BOOTSTRAP.md` is only created when the workspace is brand new (it should not come back after you delete it). `MEMORY.md` is optional (not auto-created); when present, it is loaded for normal sessions. Subagent sessions only inject `AGENTS.md` and `TOOLS.md`.

Treat this folder like OpenClaw's memory and make it a git repo (ideally private) so your `AGENTS.md` and memory files are backed up. If git is installed, brand-new workspaces are auto-initialized.

```bash
openclaw setup
```

Full workspace layout + backup guide: [Agent workspace](/concepts/agent-workspace)

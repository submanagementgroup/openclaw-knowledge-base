---
domain: concepts
topic: "Agent Overview: Pi Agent, Sessions, Workspace, and Architecture"
type: concept
keywords:
  - agent
  - Pi agent
  - agent lifecycle
  - architecture
  - sessions
  - workspace
  - agent routing
related:
  - concepts/agent-loop
  - concepts/multi-agent
  - concepts/session
source:
  - concepts/agent.md
  - concepts/architecture.md
  - concepts/features.md
---

An OpenClaw agent is a Pi agent instance running inside the Gateway. Agents have isolated sessions, a workspace directory, memory files, and a set of tools. The Gateway manages agent lifecycle, routing, and session persistence.

## What Is an Agent

OpenClaw runs a **single embedded agent runtime** — one agent process per
Gateway, with its own workspace, bootstrap files, and session store. This page
covers that runtime contract: what the workspace must contain, which files get
injected, and how sessions bootstrap against it.

## Workspace (required)

OpenClaw uses a single agent workspace directory (`agents.defaults.workspace`) as the agent’s **only** working directory (`cwd`) for tools and context.

Recommended: use `openclaw setup` to create `~/.openclaw/openclaw.json` if missing and initialize the workspace files.

Full workspace layout + backup guide: [Agent workspace](/concepts/agent-workspace)

If `agents.defaults.sandbox` is enabled, non-main sessions can override this with
per-session workspaces under `agents.defaults.sandbox.workspaceRoot` (see
[Gateway configuration](/gateway/configuration)).

## Bootstrap files (injected)

Inside `agents.defaults.workspace`, OpenClaw expects these user-editable files:

- `AGENTS.md` — operating instructions + “memory”
- `SOUL.md` — persona, boundaries, tone
- `TOOLS.md` — user-maintained tool notes (e.g. `imsg`, `sag`, conventions)
- `BOOTSTRAP.md` — one-time first-run ritual (deleted after completion)
- `IDENTITY.md` — agent name/vibe/emoji
- `USER.md` — user profile + preferred address

On the first turn of a new session, OpenClaw injects the contents of these files directly into the agent context.

Blank files are skipped. Large files are trimmed and truncated with a marker so prompts stay lean (read the file for full content).

If a file is missing, OpenClaw injects a single “missing file” marker line (and `openclaw setup` will create a safe default template).

`BOOTSTRAP.md` is only created for a **brand new workspace** (no other bootstrap files present). If you delete it after completing the ritual, it should not be recreated on later restarts.

To disable bootstrap file creation entirely (for pre-seeded workspaces), set:

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## Built-in tools

Core tools (read/exec/edit/write and related system tools) are always available,
subject to tool policy. `apply_patch` is optional and gated by
`tools.exec.applyPatch`. `TOOLS.md` does **not** control which tools exist; it’s
guidance for how _you_ want them used.

## Skills

OpenClaw loads skills from these locations (highest precedence first):

- Workspace: `<workspace>/skills`
- Project agent skills: `<workspace>/.agents/skills`
- Personal agent skills: `~/.agents/skills`
- Managed/local: `~/.openclaw/skills`
- Bundled (shipped with the install)
- Extra skill folders: `skills.load.extraDirs`

Skills can be gated by config/env (see `skills` in [Gateway configuration](/gateway/configuration)).

## Runtime boundaries

The embedded agent runtime is built on the Pi agent core (models, tools, and
prompt pipeline). Session management, discovery, tool wiring, and channel
delivery are OpenClaw-owned layers on top of that core.

## Sessions

Session

## Architecture Overview

## Overview

- A single long‑lived **Gateway** owns all messaging surfaces (WhatsApp via
  Baileys, Telegram via grammY, Slack, Discord, Signal, iMessage, WebChat).
- Control-plane clients (macOS app, CLI, web UI, automations) connect to the
  Gateway over **WebSocket** on the configured bind host (default
  `127.0.0.1:18789`).
- **Nodes** (macOS/iOS/Android/headless) also connect over **WebSocket**, but
  declare `role: node` with explicit caps/commands.
- One Gateway per host; it is the only place that opens a WhatsApp session.
- The **canvas host** is served by the Gateway HTTP server under:
  - `/__openclaw__/canvas/` (agent-editable HTML/CSS/JS)
  - `/__openclaw__/a2ui/` (A2UI host)
    It uses the same port as the Gateway (default `18789`).

## Components and flows

### Gateway (daemon)

- Maintains provider connections.
- Exposes a typed WS API (requests, responses, server‑push events).
- Validates inbound frames against JSON Schema.
- Emits events like `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`.

### Clients (mac app / CLI / web admin)

- One WS connection per client.
- Send requests (`health`, `status`, `send`, `agent`, `system-presence`).
- Subscribe to events (`tick`, `agent`, `presence`, `shutdown`).

### Nodes (macOS / iOS / Android / headless)

- Connect to the **same WS server** with `role: node`.
- Provide a device identity in `connect`; pairing is **device‑based** (role `node`) and
  approval lives in the device pairing store.
- Expose commands like `canvas.*`, `camera.*`, `screen.record`, `location.get`.

Protocol details:

- [Gateway protocol](/gateway/protocol)

### WebChat

- Static UI that uses the Gateway WS API for chat history and sends.
- In remote setups, connects through the same SSH/Tailscale tunnel as other
  clients.

## Connection lifecycle (single client)

## Wire protocol (summary)

- Transport: WebSocket, text frames with JSON payloads.
- First frame **must** be `connect`.
- After handshake:
  - Requests: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - Events: `{type:"event", event, payload, seq?, stateVersion?}`
- `hello-ok.features.methods` / `events` are discovery metadata, not a
  generated dump of every callable helper route.
- Shared-secret auth uses `connect.params.auth.token` or
  `connect.params.auth.password`, depending on the configured gateway auth mode.
- Identity-bearing modes such as Tailscale Serve
  (`gateway.auth.allowTailscale: true`) or non-loopback
  `gateway.auth.mode: "trusted-proxy"` satisfy auth from request headers
  instead of `connect.params.auth.*`.
- Private-ingress `gateway.auth.mode: "none"` disables shared-secret auth
  entirely; keep that mode off public/untrusted ingress.
- Idempotency keys are required for side‑effecting methods (`send`, `agent`) to
  safely retry; the server keeps a short‑lived dedupe cache.
- Nodes must include `role: "node"` plus caps/commands/permissions in `connect`.

## Pairing + local trust

- All WS client

## Feature List

## Highlights

    Discord, iMessage, Signal, Slack, Telegram, WhatsApp, WebChat, and more with a single Gateway.

    Bundled plugins add Matrix, Nextcloud Talk, Nostr, Twitch, Zalo, and more without separate installs in normal current releases.

    Multi-agent routing with isolated sessions.

    Images, audio, video, documents, and image/video generation.

    Web Control UI and macOS companion app.

    iOS and Android nodes with pairing, voice/chat, and rich device commands.

## Full list

**Channels:**

- Built-in channels include Discord, Google Chat, iMessage (legacy), IRC, Signal, Slack, Telegram, WebChat, and WhatsApp
- Bundled plugin channels include BlueBubbles for iMessage, Feishu, LINE, Matrix, Mattermost, Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Synology Chat, Tlon, Twitch, Zalo, and Zalo Personal
- Optional separately installed channel plugins include Voice Call and third-party packages such as WeChat
- Third-party channel plugins can extend the Gateway further, such as WeChat
- Group chat support with mention-based activation
- DM safety with allowlists and pairing

**Agent:**

- Embedded agent runtime with tool streaming
- Multi-agent routing with isolated sessions per workspace or sender
- Sessions: direct chats collapse into shared `main`; groups are isolated
- Streaming and chunking for long responses

**Auth and providers:**

- 35+ model providers (Anthropic, OpenAI, Google, and more)
- Subscription auth via OAuth (e.g. OpenAI Codex)
- Custom and self-hosted provider support (vLLM, SGLang, Ollama, and any OpenAI-compatible or Anthropic-compatible endpoint)

**Media:**

- Images, audio, video, and documents in and out
- Shared image generation and video generation capability surfaces
- Voice note transcription
- Text-to-speech with multiple providers

**Apps and interfaces:**

- WebChat and browser Control UI
- macOS menu bar companion app
- iOS node with pairing, Canvas, camera, screen recording, location, and voice
- Android node with

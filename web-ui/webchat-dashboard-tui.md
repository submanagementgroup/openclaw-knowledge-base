---
domain: web-ui
topic: "WebChat, Dashboard, and TUI: Web and Terminal Interfaces for OpenClaw"
type: reference
keywords:
  - WebChat
  - dashboard
  - TUI
  - terminal UI
  - web surfaces
  - openclaw web
related:
  - web-ui/control-ui
source:
  - web/webchat.md
  - web/dashboard.md
  - web/tui.md
  - web/index.md
---

WebChat and additional web surfaces for OpenClaw.

## WebChat

Status: the macOS/iOS SwiftUI chat UI talks directly to the Gateway WebSocket.

## What it is

- A native chat UI for the gateway (no embedded browser and no local static server).
- Uses the same sessions and routing rules as other channels.
- Deterministic routing: replies always go back to WebChat.

## Quick start

1. Start the gateway.
2. Open the WebChat UI (macOS/iOS app) or the Control UI chat tab.
3. Ensure a valid gateway auth path is configured (shared-secret by default,
   even on loopback).

## How it works (behavior)

- The UI connects to the Gateway WebSocket and uses `chat.history`, `chat.send`, and `chat.inject`.
- `chat.history` is bounded for stability: Gateway may truncate long text fields, omit heavy metadata, and replace oversized entries with `[chat.history omitted: message too large]`.
- `chat.history` follows the active transcript branch for modern append-only session files, so abandoned rewrite branches and superseded prompt copies are not rendered in WebChat.
- Control UI coalesces duplicate in-flight submits for the same session, message, and attachments before generating a new `chat.send` run id; the Gateway still dedupes repeated requests that reuse the same idempotency key.
- `chat.history` is also display-normalized: runtime-only OpenClaw context,
  inbound envelope wrappers, inline delivery directive tags
  such as `[[reply_to_*]]` and `[[audio_as_voice]]`, plain-text tool-call XML
  payloads (including `<tool_call>...</tool_call>`,
  `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`,
  `<function_calls>...</function_calls>`, and truncated tool-call blocks), and
  leaked ASCII/full-width model control tokens are stripped from visible text,
  and assistant entries whose whole visible text is only the exact silent
  token `NO_REPLY` / `no_reply` are omitted.
- Reasoning-flagged reply payloads (`isReasoning: true`) are excluded from WebChat assistant content, transcript replay text, and audio content blocks, so thinking-only payloads do not surface as visible assistant messages or playable audio.
- `chat.inject` appends an assistant note directly to the transcript and broadcasts it to the UI (no agent run).
- Aborted runs can keep partial assistant output visible in the UI.
- Gateway persists aborted partial assistant text into transcript history when buffered output exists, and marks those entries with abort metadata.
- History is always fetched from the gateway (no local file watching).
- If the gateway is unreachable, WebChat is read-only.

## Control UI agents tools panel

- The Control UI `/agents` Tools panel has two separate views:
  - **Available Right Now** uses `tools.effective(sessionKey=...)` and shows what the current
    session can actually use at runtime, including core, plugin, and channel-owned tools.
  - **Tool Configuration** uses `tools.catalog` and stays focused on profiles, overrides, and
    catalog semantics.
- Runtime availability is session-scoped. Switching sessions on the same agent can change the
  **Available Right Now** list.
- The config editor does not imply runtime availability; effective access still follows policy
  precedence (`allow`/`deny`, per-agent and provider/channel overrides).

## Remote use

- Remote mode tunnels the gateway WebSocket over SSH/Tailscale.
- You do not need to run a separate WebChat server.

## Configuration reference (WebChat)

Full configuration: [Configuration](/gateway/configuration)

WebChat options:

- `gateway.webchat.chatHistoryMaxChars`: maximum character count for text fields in `chat.history` responses. When a transcript entry exceeds this limit, Gateway truncates long text fields and may replace oversized messages with a placeholder. Per-request `maxChars` can also be sent by the client to override this default for a single `chat.history` call.

Related global options:

- `gateway.port`, `gateway.bind`: WebSocket host/port.
- `gateway.auth.mode`, `gateway.auth.token`, `gateway.auth.password`:
  shared-secret WebSocket auth.
- `gateway.auth.allowTailscale`: browser Control UI chat tab can use Tailscale
  Serve identity headers when enabled.
- `gateway.auth.mode: "trusted-proxy"`: reverse-proxy auth for browser clients behind an identity-aware **non-loopback** proxy source (see [Trusted Proxy Auth](/gateway/trusted-proxy-auth)).
- `gateway.remote.url`, `gateway.remote.token`, `gateway.remote.password`: remote gateway target.
- `session.*`: session storage and main key defaults.

## Related

- [Control UI](/web/control-ui)
- [Dashboard](/web/dashboard)

## Dashboard

The Gateway dashboard is the browser Control UI served at `/` by default
(override with `gateway.controlUi.basePath`).

Quick open (local Gateway):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (or [http://localhost:18789/](http://localhost:18789/))
- With `gateway.tls.enabled: true`, use `https://127.0.0.1:18789/` and
  `wss://127.0.0.1:18789` for the WebSocket endpoint.

Key references:

- [Control UI](/web/control-ui) for usage and UI capabilities.
- [Tailscale](/gateway/tailscale) for Serve/Funnel automation.
- [Web surfaces](/web) for bind modes and security notes.

Authentication is enforced at the WebSocket handshake via the configured gateway
auth path:

- `connect.params.auth.token`
- `connect.params.auth.password`
- Tailscale Serve identity headers when `gateway.auth.allowTailscale: true`
- trusted-proxy identity headers when `gateway.auth.mode: "trusted-proxy"`

See `gateway.auth` in [Gateway configuration](/gateway/configuration).

Security note: the Control UI is an **admin surface** (chat, config, exec approvals).
Do not expose it publicly. The UI keeps dashboard URL tokens in sessionStorage
for the current browser tab session and selected gateway URL, and strips them from the URL after load.
Prefer localhost, Tailscale Serve, or an SSH tunnel.

## Fast path (recommended)

- After onboarding, the CLI auto-opens the dashboard and prints a clean (non-tokenized) link.
- Re-open anytime: `openclaw dashboard` (copies link, opens browser if possible, shows SSH hint if headless).
- If the UI prompts for shared-secret auth, paste the configured token or
  password into Control UI settings.

## Auth basics (local vs remote)

- **Localhost**: open `http://127.0.0.1:18789/`.
- **Gateway TLS**: when `gateway.tls.enabled: true`, dashboard/status links use
  `https://` and Control UI WebSocket links use `wss://`.
- **Shared-secret token source**: `gateway.auth.token` (or
  `OPENCLAW_GATEWAY_TOKEN`); `openclaw dashboard` can pass it via URL fragment
  for one-time bootstrap, and the Control UI keeps it in sessionStorage for the
  current browser tab session and selected gateway URL instead of localStorage.
- If `gateway.auth.token` is SecretRef-managed, `openclaw dashboard`
  prints/copies/opens a non-tokenized URL by design. This avoids exposing
  externally managed tokens in shell logs, clipboard history, or browser-launch
  arguments.
- If `gateway.auth.token` is configured as a SecretRef and is unresolved in your
  current shell, `openclaw dashboard` still prints a non-tokenized URL plus
  actionable auth setup guidance.
- **Shared-secret password**: use the configured `gateway.auth.password` (or
  `OPENCLAW_GATEWAY_PASSWORD`). The dashboard does not persist passwords across
  reloads.
- **Identity-bearing modes**: Tailscale Serve can satisfy Control UI/WebSocket
  auth via identity headers when `gateway.auth.allowTailscale: true`, and a
  non-loopback identity-aware reverse proxy can satisfy
  `gateway.auth.mode: "trusted-proxy"`. In 

## TUI (Terminal UI)

## Quick start

### Gateway mode

1. Start the Gateway.

```bash
openclaw gateway
```

2. Open the TUI.

```bash
openclaw tui
```

3. Type a message and press Enter.

Remote Gateway:

```bash
openclaw tui --url ws://<host>:<port> --token <gateway-token>
```

Use `--password` if your Gateway uses password auth.

### Local mode

Run the TUI without a Gateway:

```bash
openclaw chat
# or
openclaw tui --local
```

Notes:

- `openclaw chat` and `openclaw terminal` are aliases for `openclaw tui --local`.
- `--local` cannot be combined with `--url`, `--token`, or `--password`.
- Local mode uses the embedded agent runtime directly. Most local tools work, but Gateway-only features are unavailable.
- `openclaw` and `openclaw crestodian` also use this TUI shell, with Crestodian as the local setup and repair chat backend.

## What you see

- Header: connection URL, current agent, current session.
- Chat log: user messages, assistant replies, system notices, tool cards.
- Status line: connection/run state (connecting, running, streaming, idle, error).
- Footer: connection state + agent + session + model + think/fast/verbose/trace/reasoning + token counts + deliver.
- Input: text editor with autocomplete.

## Mental model: agents + sessions

- Agents are unique slugs (e.g. `main`, `research`). The Gateway exposes the list.
- Sessions belong to the current agent.
- Session keys are stored as `agent:<agentId>:<sessionKey>`.
  - If you type `/session main`, the TUI expands it to `agent:<currentAgent>:main`.
  - If you type `/session agent:other:main`, you switch to that agent session explicitly.
- Session scope:
  - `per-sender` (default): each agent has many sessions.
  - `global`: the TUI always uses the `global` session (the picker may be empty).
- The current agent + session are always visible in the footer.

## Sending + delivery

- Messages are sent to the Gateway; delivery to providers is off by default.
- Turn delivery on:
  - `/deliver on`
  - or the Settings panel
  - or start with `openclaw tui --deliver`

## Pickers + overlays

- Model picker: list available models and set the session override.
- Agent picker: choose a different agent.
- Session picker: shows only sessions for the current agent.
- Settings: toggle deliver, tool output expansion, and thinking visibility.

## Keyboard shortcuts

- Enter: send message
- Esc: abort active run
- Ctrl+C: clear input (press twice to exit)
- Ctrl+D: exit
- Ctrl+L: model picker
- Ctrl+G: agent picker
- Ctrl+P: session picker
- Ctrl+O: toggle tool output expansion
- Ctrl+T: toggle thinking visibility (reloads history)

## Slash commands

Core:

- `/help`
- `/status`
- `/agent <id>` (or `/agents`)
- `/session <key>` (or `/sessions`)
- `/model <provider/model>` (or `/models`)

Session controls:

- `/think <off|minimal|low|medium|high>`
- `/fast <status|on|off>`
- `/verbose <on|full|off>`
- `/trace <on|off>`
- `/reasoning <on|off|stream>`
- `/usage <off|tokens|full>`
- `/elevated <on|off|ask|full>` (alias: `/elev`)
- `/activation <mention|always>`
- `/deliver <on|off>`

Session lifecycle:

- `/new` or `/reset` (reset the session)
- `/abort` (abort the active run)
- `/settings`
- `/exit`

Local mode only:

- `/auth [provider]` opens the provider auth/login flow inside the TUI.

Other Gateway slash commands (for example, `/context`) are forwarded to the Gateway and shown as system output. See [Slash commands](/tools/slash-commands).

## Local shell commands

- Prefix a line with `!` to run a local shell command on the TUI host.
- The TUI prompts once per session to allow local execution; declining keeps `!` disabled for the session.
- Commands run in a fresh, non-interactive shell in the TUI working directory (no persistent `cd`/env).
- Local shell commands receive `OPENCLAW_SHELL=tui-local` in their environment.
- A lone `!` is sent as a normal message; leading spaces do not trigger local exec.

## Repair configs from the local TUI

Use local mode when the current config already validates and you want the
embedded agent to inspect it on the same machine, compare it against the docs,
and help repair drift without depending on a running Gateway.

If `openclaw config validate` is already failing, start with `openclaw configure`
or `openclaw doctor --fix` first. `openclaw chat` does not bypass the invalid-
config guard.

Typical loop:

1. Start local mode:

```bash
openclaw chat
```

2. Ask the agent what you want checked, for example:

```text
Compare my gateway auth config with the docs and suggest the smallest fix.
```

3. Use local shell commands for exact evidence and validation:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

4. Apply narrow changes with `openclaw config set` or `openclaw configure`, then rerun `!openclaw config validate`.
5. If Doctor recommends an automatic migration or repair, review it and run `!openclaw doctor --fix`.

Tips:

- Prefer `openclaw config set` or `openclaw configure

## Web Surfaces Overview

The Gateway serves a small **browser Control UI** (Vite + Lit) from the same port as the Gateway WebSocket:

- default: `http://<host>:18789/`
- with `gateway.tls.enabled: true`: `https://<host>:18789/`
- optional prefix: set `gateway.controlUi.basePath` (e.g. `/openclaw`)

Capabilities live in [Control UI](/web/control-ui). The rest of this page focuses on bind modes, security, and web-facing surfaces.

## Webhooks

When `hooks.enabled=true`, the Gateway also exposes a small webhook endpoint on the same HTTP server.
See [Gateway configuration](/gateway/configuration) → `hooks` for auth + payloads.

## Config (default-on)

The Control UI is **enabled by default** when assets are present (`dist/control-ui`).
You can control it via config:

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // basePath optional
  },
}
```

## Tailscale access

### Integrated Serve (recommended)

Keep the Gateway on loopback and let Tailscale Serve proxy it:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

Then start the gateway:

```bash
openclaw gateway
```

Open:

- `https://<magicdns>/` (or your configured `gateway.controlUi.basePath`)

### Tailnet bind + token

```json5
{
  gateway: {
    bind: "tailnet",
    controlUi: { enabled: true },
    auth: { mode: "token", token: "your-token" },
  },
}
```

Then start the gateway (this non-loopback example uses shared-secret token
auth):

```bash
openclaw gateway
```

Open:

- `http://<tailscale-ip>:18789/` (or your configured `gateway.controlUi.basePath`)

### Public internet (Funnel)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password" }, // or OPENCLAW_GATEWAY_PASSWORD
  },
}
```

## Security notes

- Gateway auth is required by default (token, password, trusted-proxy, or Tailscale Serve identity headers when enabled).
- Non-loopback binds still **require** gateway auth. In practice that means token/password auth or an identity-aware reverse proxy with `gateway.auth.mode: "trusted-proxy"`.
- The wizard creates shared-secret auth by default and usually generates a
  gateway token (even on loopback).
- In shared-secret mode, the UI sends `connect.params.auth.token` or
  `connect.params.auth.password`.
- When `gateway.tls.enabled: true`, local dashboard and status helpers render
  `https://` dashboard URLs and `wss://` WebSocket URLs.
- In identity-bearing modes such as Tailscale Serve or `trusted-proxy`, the
  WebSocket auth check is satisfied from request headers instead.
- For non-loopback Control UI deployments, set `gateway.controlUi.allowedOrigins`
  explicitly (full origins). Without it, gateway startup is refused by default.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` enables
  Host-header origin fallback mode, but is a dangerous security downgrade.
- With Serve, Tailscale identity headers can satisfy Control UI/WebSocket auth
  when `gateway.auth.allowTailscale` is `true` (no token/password required).
  HTTP API endpoints do not use those Tailscale identity headers; they follow
  the gateway's normal HTTP auth mode instead. Set
  `gateway.auth.allowTailscale: false` to require explicit credentials. See
  [Tailscale](/gateway/tailscale) and [Security](/gateway/security). This
  tokenless flow assumes the gateway host is trusted.
- `gateway.tailscale.mode: "funnel"` requires `gateway.auth.mode: "password"` (shared password).

## Building the UI

The Gateway serves static files from `dist/control-ui`. Build them with:

```bash
pnpm ui:build
```

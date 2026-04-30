---
domain: gateway
topic: "Remote Gateway Access: SSH Tunnel, Tailscale, HTTPS Proxy, and Remote CLI"
type: procedure
keywords:
  - remote gateway
  - tailscale
  - SSH tunnel
  - remote access
  - OPENCLAW_HOST
  - reverse proxy
  - remote CLI
related:
  - gateway/authentication
  - gateway/gateway-runbook
  - install/vps-hosting
source:
  - gateway/remote.md
  - gateway/tailscale.md
  - gateway/remote-gateway-readme.md
---

Run the Gateway on a remote server and connect to it from anywhere via SSH port forwarding, Tailscale, or direct HTTPS with a trusted proxy.

## Access Methods

### SSH Port Forwarding (simplest)

```bash
# On your local machine
ssh -L 18789:localhost:18789 user@your-server
# Now access Gateway at http://localhost:18789
```

### Tailscale (recommended for persistent remote access)

1. Install Tailscale on server and client
2. Set `gateway.host` to the Tailscale IP:
   ```json5
   { gateway: { host: "100.x.y.z" } }
   ```
3. Access at `http://100.x.y.z:18789`

### HTTPS Reverse Proxy

Use nginx/Caddy in front of the Gateway:
```
location / { proxy_pass http://127.0.0.1:18789; }
```
Enable `gateway.tls` or terminate TLS at the proxy.

## Remote CLI Connection

```bash
# Point CLI at remote gateway
OPENCLAW_HOST=http://your-server:18789 openclaw status
OPENCLAW_HOST=http://your-server:18789 OPENCLAW_TOKEN=yourtoken openclaw agent "hello"
```

This repo supports “remote over SSH” by keeping a single Gateway (the master) running on a dedicated host (desktop/server) and connecting clients to it.

- For **operators (you / the macOS app)**: SSH tunneling is the universal fallback.
- For **nodes (iOS/Android and future devices)**: connect to the Gateway **WebSocket** (LAN/tailnet or SSH tunnel as needed).

## The core idea

- The Gateway WebSocket binds to **loopback** on your configured port (defaults to 18789).
- For remote use, you forward that loopback port over SSH (or use a tailnet/VPN and tunnel less).

## Common VPN and tailnet setups

Think of the **Gateway host** as where the agent lives. It owns sessions, auth profiles, channels, and state. Your laptop, desktop, and nodes connect to that host.

### Always-on Gateway in your tailnet

Run the Gateway on a persistent host (VPS or home server) and reach it via **Tailscale** or SSH.

- **Best UX:** keep `gateway.bind: "loopback"` and use **Tailscale Serve** for the Control UI.
- **Fallback:** keep loopback plus SSH tunnel from any machine that needs access.
- **Examples:** [exe.dev](/install/exe-dev) (easy VM) or [Hetzner](/install/hetzner) (production VPS).

Ideal when your laptop sleeps often but you want the agent always-on.

### Home desktop runs the Gateway

The laptop does **not** run the agent. It connects remotely:

- Use the macOS app's **Remote over SSH** mode (Settings → General → OpenClaw runs).
- The app opens and manages the tunnel, so WebChat and health checks just work.

Runbook: [macOS remote access](/platforms/mac/remote).

### Laptop runs the Gateway

Keep the Gateway local but expose it safely:

- SSH tunnel to the laptop from other machines, or
- Tailscale Serve the Control UI and keep the Gateway loopback-only.

Guides: [Tailscale](/gateway/tailscale) and [Web overview](/web).

## Command flow (what runs where)

One gateway service owns state + channels. Nodes are peripherals.

Flow example (Telegram → node):

- Telegram message arrives at the **Gateway**.
- Gateway runs the **agent** and decides whether to call a node tool.
- Gateway calls the **node** over the Gateway WebSocket (`node.*` RPC).
- Node returns the result; Gateway replies back out to Telegram.

Notes:

- **Nodes do not run the gateway service.** Only one gateway should run per host unless you intentionally run isolated profiles (see [Multiple gateways](/gateway/multiple-gateways)).
- macOS app “node mode” is just a node client over the Gateway WebSocket.

## SSH tunnel (CLI + tools)

Create a local tunnel to the remote Gateway WS:

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

With the tunnel up:

- `openclaw health` and `openclaw status --deep` now reach the remote gateway via `ws://127.0.0.1:18789`.
- `openclaw gateway status`, `openclaw gateway health`, `openclaw gateway probe`, and `openclaw gateway call` can also target the forwarded URL via `--url` when needed.

Replace `18789` with your configured `gateway.port` (or `--port` or `OP

OpenClaw can auto-configure Tailscale **Serve** (tailnet) or **Funnel** (public) for the
Gateway dashboard and WebSocket port. This keeps the Gateway bound to loopback while
Tailscale provides HTTPS, routing, and (for Serve) identity headers.

## Modes

- `serve`: Tailnet-only Serve via `tailscale serve`. The gateway stays on `127.0.0.1`.
- `funnel`: Public HTTPS via `tailscale funnel`. OpenClaw requires a shared password.
- `off`: Default (no Tailscale automation).

Status and audit output use **Tailscale exposure** for this OpenClaw Serve/Funnel
mode. `off` means OpenClaw is not managing Serve or Funnel; it does not mean the
local Tailscale daemon is stopped or logged out.

## Auth

Set `gateway.auth.mode` to control the handshake:

- `none` (private ingress only)
- `token` (default when `OPENCLAW_GATEWAY_TOKEN` is set)
- `password` (shared secret via `OPENCLAW_GATEWAY_PASSWORD` or config)
- `trusted-proxy` (identity-aware reverse proxy; see [Trusted Proxy Auth](/gateway/trusted-proxy-auth))

When `tailscale.mode = "serve"` and `gateway.auth.allowTailscale` is `true`,
Control UI/WebSocket auth can use Tailscale identity headers
(`tailscale-user-login`) without supplying a token/password. OpenClaw verifies
the identity by resolving the `x-forwarded-for` address via the local Tailscale
daemon (`tailscale whois`) and matching it to the header before accepting it.
OpenClaw only treats a request as Serve when it arrives from loopback with
Tailscale’s `x-forwarded-for`, `x-forwarded-proto`, and `x-forwarded-host`
headers.
For Control UI operator sessions that include browser device identity, this
verified Serve path also skips the device-pairing round trip. It does not bypass
browser device identity: device-less clients are still rejected, and node-role
or non-Control UI WebSocket connections still follow the normal pairing and
auth checks.
HTTP API endpoints (for example `/v1/*`, `/tools/invoke`, and `/api/channels/*`)
do **not** use Tailscale identity-header auth. The

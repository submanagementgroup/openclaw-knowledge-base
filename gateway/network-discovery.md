---
domain: gateway
topic: "Gateway Network Model, Multiple Gateways, mDNS Bonjour, and DNS-SD Discovery"
type: reference
keywords:
  - network model
  - multiple gateways
  - mDNS
  - Bonjour
  - DNS-SD
  - gateway discovery
  - local network
related:
  - gateway/gateway-runbook
  - gateway/remote-access
source:
  - gateway/network-model.md
  - gateway/multiple-gateways.md
  - gateway/bonjour.md
  - gateway/discovery.md
---

OpenClaw supports network discovery via mDNS/Bonjour and DNS-SD. Multiple gateways can run simultaneously on different ports.

## Network Model

> This content has been merged into [Network](/network#core-model). See that page for the current guide.

Most operations flow through the Gateway (`openclaw gateway`), a single long-running
process that owns channel connections and the WebSocket control plane.

## Core rules

- One Gateway per host is recommended. It is the only process allowed to own the WhatsApp Web session. For rescue bots or strict isolation, run multiple gateways with isolated profiles and ports. See [Multiple gateways](/gateway/multiple-gateways).
- Loopback first: the Gateway WS defaults to `ws://127.0.0.1:18789`. The wizard creates shared-secret auth by default and usually generates a token, even for loopback. For non-loopback access, use a valid gateway auth path: shared-secret token/password auth, or a correctly configured non-loopback `trusted-proxy` deployment. Tailnet/mobile setups usually work best through Tailscale Serve or another `wss://` endpoint instead of raw tailnet `ws://`.
- Nodes connect to the Gateway WS over LAN, tailnet, or SSH as needed. The
  legacy TCP bridge has been removed.
- Canvas host is served by the Gateway HTTP server on the **same port** as the Gateway (default `18789`):
  - `/__openclaw__/canvas/`
  - `/__openclaw__/a2ui/`
    When `gateway.auth` is configured and the Gateway binds beyond loopback, these routes are protected by Gateway auth. Node clients use node-scoped capability URLs tied to their active WS session. See [Gateway configuration](/gateway/configuration) (`canvasHost`, `gateway`).
- Remote use is typically SSH tunnel or tailnet VPN. See [Remote access](/gateway/remote) and [Discovery](/gateway/discovery).

## Related

- [Remote access](/gateway/remote)
- [Trusted proxy auth](/gateway/trusted-proxy-auth)
- [Gateway protocol](/gateway/protocol)

## Multiple Gateways

Most setups should use one Gateway because a single Gateway can handle multiple messaging connections and agents. If you need stronger isolation or redundancy (e.g., a rescue bot), run separate Gateways with isolated profiles/ports.

## Best recommended setup

For most users, the simplest rescue-bot setup is:

- keep the main bot on the default profile
- run the rescue bot on `--profile rescue`
- use a completely separate Telegram bot for the rescue account
- keep the rescue bot on a different base port such as `19789`

This keeps the rescue bot isolated from the main bot so it can debug or apply
config changes if the primary bot is down. Leave at least 20 ports between
base ports so the derived browser/canvas/CDP ports never collide.

## Rescue-Bot Quickstart

Use this as the default path unless you have a strong reason to do something
else:

```bash
# Rescue bot (separate Telegram bot, separate profile, port 19789)
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

If your main bot is already running, that is usually all you need.

During `openclaw --profile rescue onboard`:

- use the separate Telegram bot token
- keep the `rescue` profile
- use a base port at least 20 higher than the main bot
- accept the default rescue workspace unless you already manage one yourself

If onboarding already installed the rescue service for you, the final
`gateway install` is not needed.

## Why this works

The rescue bot stays independent because it has its own:

- profile/config
- state directory
- workspace
- base port (plus derived ports)
- Telegram bot token

For most setups, use a completely separate Telegram bot for the rescue profile:

- easy to keep operator-only
- separate bot token and identity
- independent from the main bot's channel/app install
- simple DM-based recovery path when the main bot is broken

## What `--profile rescue onboard` Changes

`openclaw --profile rescue onboard` uses the normal onboarding flow, but it
writes everything into a separate profile.

In practice, that means the rescue bot gets its own:

- config file
- state directory
- workspace (by default `~/.openclaw/workspace-rescue`)
- managed service name

The prompts are otherwise the same as normal onboarding.

## General multi-gateway setup

The rescue-bot layout above is the easiest default, but the same isolation
pattern works for any pair or group of Gateways on one host.

For a more general setup, give each extra Gateway its own named profile and its
own base port:

```bash
# main (default profile)
openclaw setup
openclaw gateway --port 18789

# extra gateway
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

If you want both Gateways to use named profiles, that also works:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

Services follow the same pattern:

```bash
openclaw gateway instal

## mDNS/Bonjour Discovery

OpenClaw advertises itself via mDNS so nodes and CLIs can auto-discover the Gateway on the local network.

```json5
{
  discovery: {
    mdns: { enabled: true, name: "my-openclaw" },
    dns: { enabled: false }
  }
}
```

# Bonjour / mDNS discovery

OpenClaw uses Bonjour (mDNS / DNS‑SD) to discover an active Gateway (WebSocket endpoint).
Multicast `local.` browsing is a **LAN-only convenience**. The bundled `bonjour`
plugin owns LAN advertising and is enabled by default. For cross-network discovery,
the same beacon can also be published through a configured wide-area DNS-SD domain.
Discovery is still best-effort and does **not** replace SSH or Tailnet-based connectivity.

## Wide-area Bonjour (Unicast DNS-SD) over Tailscale

If the node and gateway are on different networks, multicast mDNS won’t cross the
boundary. You can keep the same discovery UX by switching to **unicast DNS‑SD**
("Wide‑Area Bonjour") over Tailscale.

High‑level steps:

1. Run a DNS server on the gateway host (reachable over Tailnet).
2. Publish DNS‑SD records for `_openclaw-gw._tcp` under a dedicated zone
   (example: `openclaw.internal.`).
3. Configure Tailscale **split DNS** so your chosen domain resolves via that
   DNS server for clients (including iOS).

OpenClaw supports any discovery domain; `openclaw.internal.` is just an example.
iOS/Android nodes browse both `local.` and your configured wide‑area domain.

### Gateway config (recommended)

```json5
{
  gateway: { bind: "tailnet" }, // tailnet-only (recommended)
  discovery: { wideArea: { enabled: true } }, // enables wide-area DNS-SD publishing
}
```

### One-time DNS server setup (gateway host)

```bash
openclaw dns setup --apply
```

This installs CoreDNS and configures it to:

- listen on port 53 only on the gateway’s Tailscale interfaces
- serve your chosen domain (example: `openclaw.internal.`) from `~/.openclaw/dns/<domain>.db`

Validate from a tailnet‑connected machine:

```bash
dns-sd -B _openclaw-gw._tcp openclaw.internal.
dig @<TAILNET_IPV4> -p 53 _openclaw-gw._tcp.openclaw.internal PTR +short
```

### Tailscale DNS settings

In the Tailscale admin console:

- Add a nameserver pointing at the gateway’s tailnet IP (UDP/TCP 53).
- Add split DNS 

## DNS-SD Wide-Area Discovery

# Discovery & transports

OpenClaw has two distinct problems that look similar on the surface:

1. **Operator remote control**: the macOS menu bar app controlling a gateway running elsewhere.
2. **Node pairing**: iOS/Android (and future nodes) finding a gateway and pairing securely.

The design goal is to keep all network discovery/advertising in the **Node Gateway** (`openclaw gateway`) and keep clients (mac app, iOS) as consumers.

## Terms

- **Gateway**: a single long-running gateway process that owns state (sessions, pairing, node registry) and runs channels. Most setups use one per host; isolated multi-gateway setups are possible.
- **Gateway WS (control plane)**: the WebSocket endpoint on `127.0.0.1:18789` by default; can be bound to LAN/tailnet via `gateway.bind`.
- **Direct WS transport**: a LAN/tailnet-facing Gateway WS endpoint (no SSH).
- **SSH transport (fallback)**: remote control by forwarding `127.0.0.1:18789` over SSH.
- **Legacy TCP bridge (removed)**: older node transport (see
  [Bridge protocol](/gateway/bridge-protocol)); no longer advertised for
  discovery and no longer part of current builds.

Protocol details:

- [Gateway protocol](/gateway/protocol)
- [Bridge protocol (legacy)](/gateway/bridge-protocol)

## Why we keep both "direct" and SSH

- **Direct WS** is the best UX on the same network and within a tailnet:
  - auto-discovery on LAN via Bonjour
  - pairing tokens + ACLs owned by the gateway
  - no shell access required; protocol surface can stay tight and auditable
- **SSH** remains the universal fallback:
  - works anywhere you have SSH access (even across unrelated networks)
  - survives multicast/mDNS issues
  - requires no new inbound ports besides SSH

## Discovery inputs (how clients learn where the gateway is)

### 1) Bonjour / DNS-SD discovery

Multicast Bonjour is best-effort and does not cross networks. OpenClaw can also browse the
same gateway beacon via a configured wide-area DNS-SD domain, so discovery can cover:

- `loca

---
domain: gateway
topic: "Gateway Discovery and Pairing"
type: procedure
keywords:
  - Bonjour
  - mDNS
  - discovery
  - pairing
  - gateway pairing
  - QR code
  - local network discovery
source: ['gateway/bonjour.md', 'gateway/discovery.md', 'gateway/pairing.md']
---

## Bonjour (mDNS) Discovery

OpenClaw can use Bonjour (mDNS / DNS-SD) to discover an active Gateway (WebSocket endpoint).
Multicast `local.` browsing is a **LAN-only convenience**. The bundled `bonjour`
plugin owns LAN advertising. It auto-starts on macOS hosts and is opt-in on
Linux, Windows, and containerized Gateway deployments. For cross-network discovery, the same
beacon can also be published through a configured wide-area DNS-SD domain. Discovery
is still best-effort and does **not** replace SSH or Tailnet-based connectivity.

## Wide-area Bonjour (Unicast DNS-SD) over Tailscale

If the node and gateway are on different networks, multicast mDNS won't cross the
boundary. You can keep the same discovery UX by switching to **unicast DNS-SD**
("Wide-Area Bonjour") over Tailscale.

High-level steps:

1. Run a DNS server on the gateway host (reachable over Tailnet).
2. Publish DNS-SD records for `_openclaw-gw._tcp` under a dedicated zone
   (example: `openclaw.internal.`).
3. Configure Tailscale **split DNS** so your chosen domain resolves via that
   DNS server for clients (including iOS).

OpenClaw supports any discovery domain; `openclaw.internal.` is just an example.
iOS/Android nodes browse both `local.` and your configured wide-area domain.

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

- listen on port 53 only on the gateway's Tailscale interfaces
- serve your chosen domain (example: `openclaw.internal.`) from `~/.openclaw/dns/<domain>.db`

Validate from a tailnet-connected machine:

```bash
dns-sd -B _openclaw-gw._tcp openclaw.internal.
dig @ -p 53 _openclaw-gw._tcp.openclaw.internal PTR +short
```

### Tailscale DNS settings

In the Tailscale admin console:

- Add a nameserver pointing at the gateway's tailnet IP (UDP/TCP 53).
- Add split DNS so your discovery domain uses that nameserver.

Once clients accept tailnet DNS, iOS nodes and CLI discovery can browse
`_openclaw-gw._tcp` in your discovery domain without multicast.

### Gateway listener security (recommended)

The Gateway WS port (default `18789`) binds to loopback by default. For LAN/tailnet
access, bind explicitly and keep auth enabled.

For tailnet-only setups:

- Set `gateway.bind: "tailnet"` in `~/.openclaw/openclaw.json`.
- Restart the Gateway (or restart the macOS menubar app).

## What advertises

Only the Gateway advertises `_openclaw-gw._tcp`. LAN multicast advertising is
provided by the bundled `bonjour` plugin when the plugin is enabled; wide-area
DNS-SD publishing remains Gateway-owned.

## Service types

- `_openclaw-gw._tcp` - gateway transport beacon (used by macOS/iOS/Android nodes).

## TXT keys (non-secret hints)

The Gateway advertises small non-secret hints to make UI flows convenient:

- `role=gateway`
- `displayName=<friendly name>`
- `lanHost=<hostname>.local`
- `gatewayPort=<port>` (Gateway WS + HTTP)
- `gatewayTls=1` (only when TLS is enabled)
- `gatewayTlsSha256=<sha256>` (only when TLS is enabled and fingerprint is available)
- `canvasPort=<port>` (only when the canvas host is enabled; currently the same as `gatewayPort`)
- `transport=gateway`
- `tailnetDns=<magicdns>` (mDNS full mode only, optional hint when Tailnet is available)
- `sshPort=<port>` (mDNS full mode only; wide-area DNS-SD may omit it)
- `cliPath=<path>` (mDNS full mode only; wide-area DNS-SD still writes it as a remote-install hint)

Security notes:

- Bonjour/mDNS TXT records are **unauthenticated**. Clients must not treat TXT as authoritative routing.
- Clients should route using the resolved service endpoint (SRV + A/AAAA). Treat `lanHost`, `tailnetDns`, `gatewayPort`, and `gatewayTlsSha256` as hints only.
- SSH auto-targeting should likewise use the resolved service host, not TXT-only hints.
- TLS pinning must never allow an advertised `gatewayTlsSha256` to override a previously stored pin.
- iOS/Android nodes should treat discovery-based direct connects as **TLS-only** and require explicit user confirmation before trusting a first-time fingerprint.

## Debugging on macOS

Useful built-in tools:

- Browse instances:

  ```bash
  dns-sd -B _openclaw-gw._tcp local.
  ```

- Resolve one instance (replace `<instance>`):

  ```bash
  dns-sd -L "<instance>" _openclaw-gw._tcp local.
  ```

If browsing works but resolving fails, you're usually hitting a LAN policy or
mDNS resolver issue.

## Debugging in Gateway logs

The Gateway writes a rolling log file (printed on startup as
`gateway log file: ...`). Look for `bonjour:` lines, especially:

- `bonjour: advertise failed ...`
- `bonjour: suppressing ciao cancellation ...`
- `bonjour: ... name conflict resolved` / `hostname conflict resolved`
- `bonjour: watchdog detected non-announced service ...`
- `bonjour: disabling advertiser after ... failed rest

## Gateway Discovery

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

## Why we keep both direct and SSH

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

- `local.` on the same LAN
- a configured unicast DNS-SD domain for cross-network discovery

Target direction:

- The **gateway** advertises its WS endpoint via Bonjour when the bundled
  `bonjour` plugin is enabled. The plugin auto-starts on macOS hosts and is
  opt-in elsewhere.
- Clients browse and show a "pick a gateway" list, then store the chosen endpoint.

Troubleshooting and beacon details: [Bonjour](/gateway/bonjour).

#### Service beacon details

- Service types:
  - `_openclaw-gw._tcp` (gateway transport beacon)
- TXT keys (non-secret):
  - `role=gateway`
  - `transport=gateway`
  - `displayName=<friendly name>` (operator-configured display name)
  - `lanHost=<hostname>.local`
  - `gatewayPort=18789` (Gateway WS + HTTP)
  - `gatewayTls=1` (only when TLS is enabled)
  - `gatewayTlsSha256=<sha256>` (only when TLS is enabled and fingerprint is available)
  - `canvasPort=<port>` (canvas host port; currently the same as `gatewayPort` when the canvas host is enabled)
  - `tailnetDns=<magicdns>` (optional hint; auto-detected when Tailscale is available)
  - `sshPort=<port>` (mDNS full mode only; wide-area DNS-SD may omit it, in which case SSH defaults stay at `22`)
  - `cliPath=<path>` (mDNS full mode only; wide-area DNS-SD still writes it as a remote-install hint)

Security notes:

- Bonjour/mDNS TXT records are **unauthenticated**. Clients must treat TXT values as UX hints only.
- Routing (host/port) should prefer the **resolved service endpoint** (SRV + A/AAAA) over TXT-provided `lanHost`, `tailnetDns`, or `gatewayPort`.
- TLS pinning must never allow an advertised `gatewayTlsSha256` to override a previously stored pin.
- iOS/Android nodes should require an explicit "trust this fingerprint" confirmation before storing a first-time pin (out-of-band verification) whenever the chosen route is secure/TLS-based.

Enable/disable/override:

- `openclaw plugins enable bonjour` enables LAN multicast advertising.
- `OPENCLAW_DISABLE_BONJOUR=1` disables advertising.
- When the Bonjour plugin is enabled and `OPENCLAW_DISABLE_BONJOUR` is unset,
  Bonjour advertises on normal hosts and auto-disables inside detected containers.
  Empty-config macOS Gateway startup enables the plugin automatically; Linux,
  Windows, and containerized deployments need explicit enablement.
  Use `0` only on host, macvlan, or another mDNS-capable network; use `1` to
  force-disable.
- `gateway.bind` in `~/.openclaw/openclaw.json` controls the Gateway bind mode.
- `OPENCLAW_SSH_PORT` overrides the SSH port advertised when `sshPort` is emitted.
- `OPENCLAW_TAILNET_DNS` publishes a `tailnetDns` hint (MagicDNS).
- `OPENCLAW_CLI_PATH` overrides the advertised CLI path.

### 2) Tailnet (cross-network)

For London/Vienna style setups, Bonjour won't help. The recommended "direct" target is:

- Tailscale MagicDNS name (preferred) or a stable tailnet IP.

If the gateway can detect it is running under Tailscale, it publishes `tailnetDns` as an optional hint for clients (including wide-area beacons).

The macOS app now prefers MagicDNS names over raw Tailscale IPs for gateway discovery. This improves reliability when tailnet IPs change (for example after node restarts or CGNAT reassignment), because MagicDNS names resolve to the current IP automatically.

For mobile node pairing, discovery hints do not relax transport security on tailnet/public routes:

- iOS/Android still require a secure first-time tailnet/public connect path (`wss://` or Tailscale Serve/Funnel).
- A discovered raw tailnet IP is a routing hint, not permission to use plaintext remote `ws://`.
- Private LAN direct-connect `ws://` remains supported.
- If you want the simplest Tailscale path for mobile nodes, use Tailscale Serve so discovery and the setup code both resolve to the same secure MagicDNS endpoint.

### 3) Manual / SSH target

When there is no direct route (or direct is disabled), clients can always connect via SSH by forwarding the loopback gateway port.

See [Remote access](/gateway/remote).

## Transport selection (client policy)

Recommended client behavior:

1. If a paired direct endpoint is configured and reachable, use it.
2. Else, if discovery finds a gateway on `local.` or the configured wide-area domain, offer a one-tap "Use this gateway" choice and save it as the direct endpoint.
3. Else, if a tailnet DNS/IP is configured, try direct.
   For mobile nodes on tailnet/public routes, direct means a secure endpoint, not plaintext remote `ws://`.
4. Else, fall back to SSH.

## Pairing + auth (direct transport)

The gateway is the source of truth for node/client admission.

- Pairing requests are created/approved/rejected in the gateway (see [Gateway pairing](/gateway/pairing)).
- The gateway enforces:
  - auth (token / keypair)
  - scopes/ACLs (the gateway is not a raw proxy to every method)
  - rate limits

## Responsibilities by component

- **Gateway**: advertises discovery beacons, owns pairing decisions, and hosts the WS endpoint.
- **macOS app**: helps you pick a gateway, shows pairing prompts, and uses SSH only as a fallback.
- **iOS/Android nodes**: browse Bonjour as a convenience and connect to the paired Gateway WS.

## Related

- [Remote access](/gateway/remote)
- [Tailscale](/gateway/tailscale)
- [Bonjour discovery](/gateway/bonjour)


## Pairing

In Gateway-owned pairing, the **Gateway** is the source of truth for which nodes
are allowed to join. UIs (macOS app, future clients) are just frontends that
approve or reject pending requests.

**Important:** WS nodes use **device pairing** (role `node`) during `connect`.
`node.pair.*` is a separate pairing store and does **not** gate the WS handshake.
Only clients that explicitly call `node.pair.*` use this flow.

## Concepts

- **Pending request**: a node asked to join; requires approval.
- **Paired node**: approved node with an issued auth token.
- **Transport**: the Gateway WS endpoint forwards requests but does not decide
  membership. (Legacy TCP bridge support has been removed.)

## How pairing works

1. A node connects to the Gateway WS and requests pairing.
2. The Gateway stores a **pending request** and emits `node.pair.requested`.
3. You approve or reject the request (CLI or UI).
4. On approval, the Gateway issues a **new token** (tokens are rotated on re-pair).
5. The node reconnects using the token and is now "paired".

Pending requests expire automatically after **5 minutes**.

## CLI workflow (headless friendly)

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes status
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name "Living Room iPad"
```

`nodes status` shows paired/connected nodes and their capabilities.

## API surface (gateway protocol)

Events:

- `node.pair.requested` - emitted when a new pending request is created.
- `node.pair.resolved` - emitted when a request is approved/rejected/expired.

Methods:

- `node.pair.request` - create or reuse a pending request.
- `node.pair.list` - list pending + paired nodes (`operator.pairing`).
- `node.pair.approve` - approve a pending request (issues token).
- `node.pair.reject` - reject a pending request.
- `node.pair.remove` - remove a stale paired node entry.
- `node.pair.verify` - verify `{ nodeId, token }`.

Notes:

- `node.pair.request` is idempotent per node: repeated calls return the same
  pending request.
- Repeated requests for the same pending node also refresh the stored node
  metadata and the latest allowlisted declared command snapshot for operator visibility.
- Approval **always** generates a fresh token; no token is ever returned from
  `node.pair.request`.
- Operator scope levels and approval-time checks are summarized in
  [Operator scopes](/gateway/operator-scopes).
- Requests may include `silent: true` as a hint for auto-approval flows.
- `node.pair.approve` uses the pending request's declared commands to enforce
  extra approval scopes:
  - commandless request: `operator.pairing`
  - non-exec command request: `operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which` request:
    `operator.pairing` + `operator.admin`

> **Note:** Node pairing is a trust and identity flow plus token issuance. It does **not** pin the live node command surface per node.

- Live node commands come from what the node declares on connect after the gateway's global node command policy (`gateway.nodes.allowCommands` and `denyCommands`) is applied.
- Per-node `system.run` allow and ask policy lives on the node in `exec.approvals.node.*`, not in the pairing record.



## Node command gating (2026.3.31+)

> **Note:** **Breaking change:** Starting with `2026.3.31`, node commands are disabled until node pairing is approved. Device pairing alone is no longer enough to expose declared node commands.


When a node connects for the first time, pairing is requested automatically. Until the pairing request is approved, all pending node commands from that node are filtered and will not execute. Once trust is established through pairing approval, the node's declared commands become available subject to the normal command policy.

This means:

- Nodes that were previously relying on device pairing alone to expose commands must now complete node pairing.
- Commands queued before pairing approval are dropped, not deferred.

## Node event trust boundaries (2026.3.31+)

> **Note:** **Breaking change:** Node-originated runs now stay on a reduced trusted surface.


Node-originated summaries and related session events are restricted to the intended trusted surface. Notification-driven or node-triggered flows that previously relied on broader host or session tool access may need adjustment. This hardening ensures that node events cannot escalate into host-level tool access beyond what the node's trust boundary permits.

Durable node presence updates follow the same identity boundary. The `node.presence.alive` event is
accepted only from authenticated node device sessions and updates pairing metadata only when the
device/node identity is already paired. Self-declared `client.id` values are not enough to write
last-seen state.

## Auto-approval (macOS app)

The macOS app can optionally attempt a **silent approval** when:

- the request is marked `silent`, and
- the app can verify an SSH connection to the gateway host using the same user.

If silent approval fails, it falls back to the normal "Approve/Reject" prompt.

## Trusted-CIDR device auto-approval

WS device pairing for `role: node` remains manual by default. For private
node networks where the Gateway already trusts the network path, operators can
opt in with explicit CIDRs or exact IPs:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Security boundary:

- Disabled when `gateway.nodes.pairing.autoApproveCidrs` is unset.
- No blanket LAN or private-network auto-approve mode exists.
- Only fresh `role: node` device pairing with no requested scopes is eligible.
- Operator, browser, Control UI, and WebChat clients stay manual.
- Role, scope, metadata, and public-key upgrades stay manual.
- Same-host loopback trusted-proxy header paths are not eligible because that
  path can be spoofed by local callers.

## Metadata-upgrade auto-approval

When an already paired device reconnects with only non-sensitive metadata
changes (for example, display name or client platform hints), OpenClaw treats
that as a `metadata-upgrade`. Silent auto-approval is narrow: it applies only
to trusted non-browser local reconnects that already proved possession of local
or shared credentials, including same-host native app reconnects after OS
version metadata changes. Browser/Control UI clients and remote clients still
use the explicit re-approval flow. Scope upgrades (read to write/admin) and
public key changes are **not** eligible for metadata-upgrade auto-approval -
they stay as explicit re-approval requests.

## QR pairing helpers

`/pair qr` renders the pairing payload as structured media so mobile and
browser clients can scan it directly.

Deleting a device also sweeps any stale pending pairing requests for that
device id, so `nodes pending` does not show orphaned rows after a revoke.

## Locality and forwarded headers

Gateway pairing treats a connection as loopback only when both the raw socket
and any upstream proxy evidence agree. If a request arrives on loopback but
carries `X-Forwarded-For` / `X-Forwarded-Host` / `X-Forwarded-Proto` headers
that point at a non-local origin, that forwarded-header evidence disqualifies
the loopback locality claim. The pairing path then requires explicit approval
instead of silently treating the request as a same-host connect. See
[Trusted Proxy Auth](/gateway/trusted-proxy-auth) for the equivalent rule on
operator auth.

## Storage (local, private)

Pairing state is stored under the Gateway state directory (default `~/.openclaw`):

- `~/.openclaw/nodes/paired.json`
- `~/.openclaw/nodes/pending.json`

If you override `OPENCLAW_STATE_DIR`, the `nodes/` folder moves with it.

Security notes:

- Tokens are secrets; treat `paired.json` as sensitive.
- Rotating a token requires re-approval (or deleting the node entry).

## Transport behavior

- The transport is **stateless**; it does not store membership.
- If the Gateway is offline or pairing is disabled, nodes cannot pair.
- If the Gateway is in remote mode, pairing still happens against the remote Gateway's store.

## Related

- [Channel pairing](/channels/pairing)
- [Nodes](/nodes)
- [Devices CLI](/cli/devices)

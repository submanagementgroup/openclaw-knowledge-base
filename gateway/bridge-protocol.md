---
domain: gateway
topic: "Bridge Protocol: Historical TCP Bridge (Removed — Reference Only)"
type: reference
keywords:
  - bridge protocol
  - TCP bridge
  - bridge pairing
  - node pairing legacy
  - bridge JSONL
  - bridge TLS
  - canvas invoke
  - camera invoke
  - exec.finished
  - node legacy protocol
source: gateway/bridge-protocol.md
related:
  - gateway/protocol
  - nodes/nodes-overview
---

The TCP bridge has been **removed**. Current OpenClaw builds do not ship the bridge listener. `bridge.*` config keys are no longer in the schema. This page is historical reference only. Use the [Gateway Protocol](/gateway/protocol) for all node/operator clients.

## Why It Existed

- **Security boundary**: the bridge exposed a small allowlist instead of the full gateway API surface.
- **Pairing + node identity**: node admission was owned by the gateway, tied to a per-node token.
- **Discovery UX**: nodes could discover gateways via Bonjour on LAN or connect directly over a tailnet.
- **Loopback WS**: the full WS control plane stayed local unless tunneled via SSH.

## Historical Transport

- TCP, one JSON object per line (JSONL).
- Optional TLS (when `bridge.tls.enabled` was true).
- Historical default listener port: `18790` (current builds do not start a TCP bridge).

When TLS was enabled, discovery TXT records included `bridgeTls=1` plus `bridgeTlsSha256` as a non-secret hint. Bonjour/mDNS TXT records are unauthenticated; clients must not treat the advertised fingerprint as an authoritative pin without explicit user intent or out-of-band verification.

## Historical Handshake and Pairing

1. Client sent `hello` with node metadata + token (if already paired).
2. If not paired, gateway replied `error` (`NOT_PAIRED`/`UNAUTHORIZED`).
3. Client sent `pair-request`.
4. Gateway waited for approval, then sent `pair-ok` and `hello-ok`.

`hello-ok` returned `serverName` and could include `canvasHostUrl`.

## Historical Frame Types

**Client → Gateway:**
- `req` / `res`: scoped gateway RPC (chat, sessions, config, health, voicewake, skills.bins)
- `event`: node signals (voice transcript, agent request, chat subscribe, exec lifecycle)

**Gateway → Client:**
- `invoke` / `invoke-res`: node commands (`canvas.*`, `camera.*`, `screen.record`, `location.get`, `sms.send`)
- `event`: chat updates for subscribed sessions
- `ping` / `pong`: keepalive

Legacy allowlist enforcement lived in `src/gateway/server-bridge.ts` (removed).

## Exec Lifecycle Events (Legacy)

Nodes could emit `exec.finished` or `exec.denied` events. (Legacy nodes may still emit `exec.started`.)

Payload fields (all optional unless noted):

- `sessionKey` (required): agent session to receive the system event
- `runId`: unique exec id for grouping
- `command`: raw or formatted command string
- `exitCode`, `timedOut`, `success`, `output`: completion details (finished only)
- `reason`: denial reason (denied only)

## Historical Tailnet Usage

- Bind the bridge to a tailnet IP: `bridge.bind: "tailnet"` in `~/.openclaw/openclaw.json` — historical only; `bridge.*` is no longer valid.
- Bonjour does **not** cross networks; manual host/port or wide-area DNS-SD was needed for remote discovery.

## Versioning

The bridge was implicit v1 (no min/max negotiation). Current node/operator clients use the WebSocket [Gateway Protocol](/gateway/protocol).

## Related

- [Gateway protocol](/gateway/protocol)
- [Nodes overview](/nodes/nodes-overview)

---
domain: gateway
topic: "Gateway WebSocket RPC Protocol: Transport, Roles, Methods, and Device Pairing"
type: reference
keywords:
  - gateway protocol
  - WebSocket RPC
  - RPC methods
  - gateway API
  - device pairing
  - operator role
  - node role
related:
  - gateway/gateway-runbook
  - gateway/authentication
  - nodes/nodes-overview
source: gateway/protocol.md
---

The OpenClaw Gateway protocol is a WebSocket RPC protocol used by CLI, nodes, and plugins to communicate with the Gateway. Understanding it is needed for building integrations or debugging low-level connectivity issues.

## Transport and Framing

- WebSocket at `ws://127.0.0.1:18789/rpc` (or configured port)
- JSON-framed messages with `method`, `params`, and optional `id`
- Bidirectional: client sends requests, Gateway sends events and responses

## Roles and Scopes

| Role | Description |
|------|-------------|
| `operator` | Full control: config, channels, sessions, agents |
| `node` | Node capabilities: camera, audio, location, exec |
| `observer` | Read-only: receive events |

## Common RPC Methods

- `agent` — run an agent turn
- `agent.wait` — wait for a run to complete
- `channels.status` — channel health
- `config.get` / `config.set` — configuration management
- `sessions.list` / `sessions.get` — session management
- `models.list` — available models

## Device Pairing

Nodes (iOS, Android) pair with the Gateway using QR code scanning. After pairing, the device gets a signed identity that the Gateway trusts for subsequent connections.

The Gateway WS protocol is the **single control plane + node transport** for
OpenClaw. All clients (CLI, web UI, macOS app, iOS/Android nodes, headless
nodes) connect over WebSocket and declare their **role** + **scope** at
handshake time.

## Transport

- WebSocket, text frames with JSON payloads.
- First frame **must** be a `connect` request.
- Pre-connect frames are capped at 64 KiB. After a successful handshake, clients
  should follow the `hello-ok.policy.maxPayload` and
  `hello-ok.policy.maxBufferedBytes` limits. With diagnostics enabled,
  oversized inbound frames and slow outbound buffers emit `payload.large` events
  before the gateway closes or drops the affected frame. These events keep
  sizes, limits, surfaces, and safe reason codes. They do not keep the message
  body, attachment contents, raw frame body, tokens, cookies, or secret values.

## Handshake (connect)

Gateway → Client (pre-connect challenge):

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Client → Gateway:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway → Client:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 3,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

While the Gateway is still finishing startup sidecars, the `connect` request can
return a retryable `UNAVAILABLE` error with `details.reason` set to
`"startup-sidecars"` and `retryAfterMs`. Clients should retry that response
within their overall connection budget instead of surfacing it as a terminal
handshake failure.

`server`, `features`, `snapshot`, and `policy` are all required by the schema
(`src/gateway/protocol/schema/frames.ts`). `auth` is also required and reports
the negotiated role/scopes. `canvasHostUrl` is optional.

When no device token is issued, `hello-ok.auth` reports the negotiated
permissions without token fields:

```json
{
  "auth": {
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Trusted same-process backend clients (`client.id: "gateway-client"`,
`client.mode: "backend"`) may omit `device` on direct loopback connections when
they authenticate with the shared gateway token/password. This path is reserved
for internal control-plane RPCs and keeps stale CLI/device pairing baselines from
blocking local backend work such as subagent session updates. Remote clients,
browser-origin clients, node clients, and explicit device-token/device-identity
clients still use the normal pairing and scope-upgrade checks.

When a device token is issued, `hello-ok` also includes:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

During trusted bootstrap handoff, `hello-ok.auth` may also include additional
bounded role entries in `deviceTokens`:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

For the built-in node/operator bootstrap flow, the primary node token stays
`scopes: []` and any handed-off operator token stays bounded to the bootstrap
operator allowlist (`operator.approvals`, `operator.read`,
`operator.talk.secrets`, `operator.write`). Bootstrap scope checks stay
role-prefixed: operator entries only satisfy operator requests, and non-operator
roles still need scopes under their own role prefix.

### Node example

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "s

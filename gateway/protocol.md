---
domain: gateway
topic: "OpenClaw Gateway Protocol Reference"
type: reference
keywords:
  - gateway protocol
  - WebSocket
  - message format
  - events
  - API protocol
  - bridge protocol
  - JSON protocol
source: gateway/protocol.md
---

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
    "maxProtocol": 4,
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
    "protocol": 4,
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
the negotiated role/scopes. `pluginSurfaceUrls` is optional and maps plugin
surface names, such as `canvas`, to scoped hosted URLs.

Scoped plugin surface URLs may expire. Nodes can call
`node.pluginSurface.refresh` with `{ "surface": "canvas" }` to receive a fresh
entry in `pluginSurfaceUrls`. The experimental Canvas plugin refactor does not
support the deprecated `canvasHostUrl`, `canvasCapability`, or
`node.canvas.capability.refresh` compatibility path; current native clients and
gateways must use plugin surfaces.

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
    "maxProtocol": 4,
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
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
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

## Framing

- **Request**: `{type:"req", id, method, params}`
- **Response**: `{type:"res", id, ok, payload|error}`
- **Event**: `{type:"event", event, payload, seq?, stateVersion?}`

Side-effecting methods require **idempotency keys** (see schema).

## Roles + scopes

For the full operator scope model, approval-time checks, and shared-secret
semantics, see [Operator scopes](/gateway/operator-scopes).

### Roles

- `operator` = control plane client (CLI/UI/automation).
- `node` = capability host (camera/screen/canvas/system.run).

### Scopes (operator)

Common scopes:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` with `includeSecrets: true` requires `operator.talk.secrets`
(or `operator.admin`).

Plugin-registered gateway RPC methods may request their own operator scope, but
reserved core admin prefixes (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) always resolve to `operator.admin`.

Method scope is only the first gate. Some slash commands reached through
`chat.send` apply stricter command-level checks on top. For example, persistent
`/config set` and `/config unset` writes require `operator.admin`.

`node.pair.approve` also has an extra approval-time scope check on top of the
base method scope:

- commandless requests: `operator.pairing`
- requests with non-exec node commands: `operator.pairing` + `operator.write`
- requests that include `system.run`, `system.run.prepare`, or `system.which`:
  `operator.pairing` + `operator.admin`

### Caps/commands/permissions (node)

Nodes declare capability claims at connect time:

- `caps`: high-level capability categories such as `camera`, `canvas`, `screen`,
  `location`, `voice`, and `talk`.
- `commands`: command allowlist for invoke.
- `permissions`: granular toggles (e.g. `screen.record`, `camera.capture`).

The Gateway treats these as **claims** and enforces server-side allowlists.

## Presence

- `system-presence` returns entries keyed by device identity.
- Presence entries include `deviceId`, `roles`, and `scopes` so UIs can show a single row per device
  even when it connects as both **operator** and **node**.
- `node.list` includes optional `lastSeenAtMs` and `lastSeenReason` fields. Connected nodes report
  their current connection time as `lastSeenAtMs` with reason `connect`; paired nodes can also report
  durable background presence when a trusted node event updates their pairing metadata.

### Node background alive event

Nodes may call `node.event` with `event: "node.presence.alive"` to record that a paired node was
alive during a background wake without marking it connected.

```json
{
  "event": "node.presence.alive",
  "payloadJSON": "{\"trigger\":\"silent_push\",\"sentAtMs\":1737264000000,\"displayName\":\"Peter's iPhone\",\"version\":\"2026.4.28\",\"platform\":\"iOS 18.4.0\",\"deviceFamily\":\"iPhone\",\"modelIdentifier\":\"iPhone17,1\",\"pushTransport\":\"relay\"}"
}
```

`trigger` is a closed enum: `background`, `silent_push`, `bg_app_refresh`,
`significant_location`, `manual`, or `connect`. Unknown trigger strings are normalized to
`background` by the gateway before persistence. The event is durable only for authenticated node
device sessions; device-less or unpaired sessions return `handled: false`.

Successful gateways return a structured result:

```json
{
  "ok": true,
  "event": "node.presence.alive",
  "handled": true,
  "reason": "persisted"
}
```

Older gateways may still return `{ "ok": true }` for `node.event`; clients should treat that as an
acknowledged RPC, not as durable presence persistence.

## Broadcast event scoping

Server-pushed WebSocket broadcast events are scope-gated so that pairing-scoped or node-only sessions do not passively receive session content.

- **Chat, agent, and tool-result frames** (including streamed `agent` events and tool call results) require at least `operator.read`. Sessions without `operator.read` skip these frames entirely.
- **Plugin-defined `plugin.*` broadcasts** are gated to `operator.write` or `operator.admin`, depending on how the plugin registered them.
- **Status and transport events** (`heartbeat`, `presence`, `tick`, connect/disconnect lifecycle, etc.) remain unrestricted so transport health stays observable to every authenticated session.
- **Unknown broadcast event families** are scope-gated by default (fail-closed) unless a registered handler explicitly relaxes them.

Each client connection keeps its own per-client sequence number so broadcasts preserve monotonic ordering on that socket even when different clients see different scope-filtered subsets of the event stream.

## Common RPC method families

The public WS surface is broader than the handshake/auth examples above. This
is not a generated dump — `hello-ok.features.methods` is a conservative
discovery list built from `src/gateway/server-methods-list.ts` plus loaded
plugin/channel method exports. Treat it as feature discovery, not a full
enumeration of `src/gateway/server-methods/*.ts`.

### System and identity

- `health` returns the cached or freshly probed gateway health snapshot.
    - `diagnostics.stability` returns the recent bounded diagnostic stability recorder. It keeps operational metadata such as event names, counts, byte sizes, memory readings, queue/session state, channel/plugin names, and session ids. It does not keep chat text, webhook bodies, tool outputs, raw request or response bodies, tokens, cookies, or secret values. Operator read scope is required.
    - `status` returns the `/status`-style gateway summary; sensitive fields are included only for admin-scoped operator clients.
    - `gateway.identity.get` returns the gateway device identity used by relay and pairing flows.
    - `system-presence` returns the current presence snapshot for connected operator/node devices.
    - `system-event` appends a system event and can update/broadcast presence context.
    - `last-heartbeat` returns the latest persisted heartbeat event.
    - `set-heartbeats` toggles heartbeat processing on the gateway.

  ### Models and usage

- `models.list` returns the runtime-allowed model catalog. Pass `{ "view": "configured" }` for picker-sized configured models (`agents.defaults.models` first, then `models.providers.*.models`), or `{ "view": "all" }` for the full catalog.
    - `usage.status` returns provider usage windows/remaining quota summaries.
    - `usage.cost` returns aggregated cost usage summaries for a date range.
    - `doctor.memory.status` returns vector-memory / cached embedding readiness for the active default agent workspace. Pass `{ "probe": true }` or `{ "deep": true }` only when the caller explicitly wants a live embedding provider ping.
    - `doctor.memory.remHarness` returns a bounded, read-only REM harness preview for remote control-plane clients. It can include workspace paths, memory snippets, rendered grounded markdown, and deep promotion candidates, so callers need `operator.read`.
    - `sessions.usage` returns per-session usage summaries.
    - `sessions.usage.timeseries` returns timeseries usage for one session.
    - `sessions.usage.logs` returns usage log entries for one session.

  ### Channels and login helpers

- `channels.status` returns built-in + bundled channel/plugin status summaries.
    - `channels.logout` logs out a specific channel/account where the channel supports logout.
    - `web.login.start` starts a QR/web login flow for the current QR-capable web channel provider.
    - `web.login.wait` waits for that QR/web login flow to complete and starts the channel on success.
    - `push.test` sends a test APNs push to a registered iOS node.
    - `voicewake.get` returns the stored wake-word triggers.
    - `voicewake.set` updates wake-word triggers and broadcasts the change.

  ### Messaging and logs

- `send` is the direct outbound-delivery RPC for channel/account/thread-targeted sends outside the chat runner.
    - `logs.tail` returns the configured gateway file-log tail with cursor/limit and max-byte controls.

  ### Talk and TTS

- `talk.catalog` returns the read-only Talk provider catalog for speech, streaming transcription, and realtime voice. It includes provider ids,
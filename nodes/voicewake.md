---
domain: nodes
topic: "Voice Wake: Global Wake Words Owned by the Gateway, Sync, and Routing"
type: reference
keywords:
  - voice wake
  - wake words
  - voicewake.get
  - voicewake.set
  - voicewake.changed
  - voicewake routing
  - wake word sync
  - VoiceWakeManager
  - voicewake.json
  - voice wake routing config
source: nodes/voicewake.md
related:
  - nodes/nodes-overview
  - platforms/macos-app-features
  - nodes/troubleshooting
---

OpenClaw treats wake words as a **single global list** owned by the **Gateway**. There are no per-node custom wake words. Any node/app UI may edit the list; changes are persisted by the Gateway and broadcast to all connected clients.

## Storage (Gateway Host)

Wake words are stored on the gateway machine at:

```
~/.openclaw/settings/voicewake.json
```

Shape:

```json
{
  "triggers": ["openclaw", "claude", "computer"],
  "updatedAtMs": 1730000000000
}
```

## Gateway Protocol Methods

| Method | Direction | Description |
|--------|-----------|-------------|
| `voicewake.get` | Client → Gateway | Returns `{ triggers: string[] }` |
| `voicewake.set` | Client → Gateway | Sets `{ triggers: string[] }`, returns updated list |
| `voicewake.changed` event | Gateway → All clients | Broadcast when triggers change |
| `voicewake.routing.get` | Client → Gateway | Returns `{ config: VoiceWakeRoutingConfig }` |
| `voicewake.routing.set` | Client → Gateway | Sets routing config |
| `voicewake.routing.changed` event | Gateway → All clients | Broadcast when routing changes |

Triggers are normalized (trimmed, empties dropped). Empty lists fall back to defaults. Count and length limits are enforced.

## Routing Configuration

`VoiceWakeRoutingConfig` shape:

```json
{
  "version": 1,
  "defaultTarget": { "mode": "current" },
  "routes": [
    { "trigger": "robot wake", "target": { "sessionKey": "agent:main:main" } }
  ],
  "updatedAtMs": 1730000000000
}
```

Route targets support exactly one of:
- `{ "mode": "current" }`
- `{ "agentId": "main" }`
- `{ "sessionKey": "agent:main:main" }`

## Platform Behavior

### macOS App

- Uses the global list to gate `VoiceWakeRuntime` triggers.
- Editing "Trigger words" in Voice Wake settings calls `voicewake.set`; the broadcast keeps other clients in sync.

### iOS Node

- Uses the global list for `VoiceWakeManager` trigger detection.
- Editing Wake Words in Settings calls `voicewake.set` over the Gateway WS.

### Android Node

- Voice Wake is currently **disabled** on Android runtime.
- Android voice uses manual mic capture in the Voice tab instead of wake-word triggers.

## Events: Who Receives Them

Both `voicewake.changed` and `voicewake.routing.changed` are broadcast to:
- All WebSocket clients (macOS app, WebChat, etc.)
- All connected nodes (iOS/Android), plus an initial push when a node connects

## macOS-Specific Toggle

macOS and iOS keep local **Voice Wake enabled/disabled** toggles (local UX + permissions differ). Android Voice Wake is always off.

## Related

- [Nodes overview](/nodes/nodes-overview)
- [Voice wake on macOS](/platforms/macos-app-features)
- [Node troubleshooting](/nodes/troubleshooting)

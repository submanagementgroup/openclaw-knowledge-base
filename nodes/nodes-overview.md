---
domain: nodes
topic: "Nodes Overview: iOS/Android Hardware Nodes for Camera, Audio, Voice, and Location"
type: concept
keywords:
  - nodes
  - iOS node
  - Android node
  - camera
  - microphone
  - voice wake
  - node pairing
  - hardware nodes
related:
  - platforms/ios-android
  - tools/tts
  - concepts/agent-loop
source:
  - nodes/index.md
  - nodes/audio.md
  - nodes/camera.md
  - nodes/talk.md
---

OpenClaw nodes are companion apps (iOS, Android, Raspberry Pi) that extend the Gateway with hardware capabilities: camera, microphone, location, and voice.

## Node Overview

A **node** is a companion device (macOS/iOS/Android/headless) that connects to the Gateway **WebSocket** (same port as operators) with `role: "node"` and exposes a command surface (e.g. `canvas.*`, `camera.*`, `device.*`, `notifications.*`, `system.*`) via `node.invoke`. Protocol details: [Gateway protocol](/gateway/protocol).

Legacy transport: [Bridge protocol](/gateway/bridge-protocol) (TCP JSONL;
historical only for current nodes).

macOS can also run in **node mode**: the menubar app connects to the Gateway’s
WS server and exposes its local canvas/camera commands as a node (so
`openclaw nodes …` works against this Mac). In remote gateway mode, browser
automation is handled by the CLI node host (`openclaw node run` or the
installed node service), not by the native app node.

Notes:

- Nodes are **peripherals**, not gateways. They don’t run the gateway service.
- Telegram/WhatsApp/etc. messages land on the **gateway**, not on nodes.
- Troubleshooting runbook: [/nodes/troubleshooting](/nodes/troubleshooting)

## Pairing + status

**WS nodes use device pairing.** Nodes present a device identity during `connect`; the Gateway
creates a device pairing request for `role: node`. Approve via the devices CLI (or UI).

Quick CLI:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

If a node retries with changed auth details (role/scopes/public key), the prior
pending request is superseded and a new `requestId` is created. Re-run
`openclaw devices list` before approving.

Notes:

- `nodes status` marks a node as **paired** when its device pairing role includes `node`.
- The device pairing record is the durable approved-role contract. Token
  rotation stays inside that contract; it cannot upgrade a paired node into a
  different role that pairing approval never granted.
- `node.pair.*` (CLI: `openclaw nodes pending/approve/reject/remove/rename`) is a separate gateway-owned
  node pairing store; it does **not** gate the WS `connect` handshake.
- `openclaw nodes remove --node <id|name|ip>` deletes stale entries from that
  separate gateway-owned node pairing store.
- Approval scope follows the pending request's declared commands:
  - commandless request: `operator.pairing`
  - non-exec node commands: `operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`: `operator.pairing` + `operator.admin`

## Remote node host (system.run)

Use a **node host** when your Gateway runs on one machine and you want commands
to execute on another. The model still talks to the **gateway**; the gateway
forwards `exec` calls to the **node host** when `host=node` is selected.

### What runs where

- **Gateway host**: receives messages, runs the model, routes tool calls.
- **Node host**: executes `system.run`/`system.which` on the node machine.
- **Approvals**: enforced on the node host via `~/.openclaw/exec-approvals.json`.

Approval note:

- Approval-backed node runs bind exact request context.
- For direct shell/runtime file executions, OpenClaw also best-effort binds one concrete local
  file operand and denies the run if that file changes before execution.
- If OpenClaw cannot identify exactly one concrete local file for an interpreter/runtime command,
  approval-backed execution is denied instead of pretending full runtime coverage. Use sandboxing,
  separate hosts, or an explicit trusted allowlist/full workflow for broader interpreter semantics.

### Start a node host (foreground)

On the node machine:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

### Remote gateway via SSH tunnel (loopback bind)

If the Gateway binds to loopback (`gateway.bind=loopback`, default in local mode),
remote node hosts cannot connect directly. Create an SSH tunnel and point the
node host at the local end of the tunnel.

Example (node host -> gateway host):

```bash
# 

## Audio (Microphone + TTS Playback)

# Audio / Voice Notes (2026-01-17)

## What works

- **Media understanding (audio)**: If audio understanding is enabled (or auto‑detected), OpenClaw:
  1. Locates the first audio attachment (local path or URL) and downloads it if needed.
  2. Enforces `maxBytes` before sending to each model entry.
  3. Runs the first eligible model entry in order (provider or CLI).
  4. If it fails or skips (size/timeout), it tries the next entry.
  5. On success, it replaces `Body` with an `[Audio]` block and sets `{{Transcript}}`.
- **Command parsing**: When transcription succeeds, `CommandBody`/`RawBody` are set to the transcript so slash commands still work.
- **Verbose logging**: In `--verbose`, we log when transcription runs and when it replaces the body.

## Auto-detection (default)

If you **don’t configure models** and `tools.media.audio.enabled` is **not** set to `false`,
OpenClaw auto-detects in this order and stops at the first working option:

1. **Active reply model** when its provider supports audio understanding.
2. **Local CLIs** (if installed)
   - `sherpa-onnx-offline` (requires `SHERPA_ONNX_MODEL_DIR` with encoder/decoder/joiner/tokens)
   - `whisper-cli` (from `whisper-cpp`; uses `WHISPER_CPP_MODEL` or the bundled tiny model)
   - `whisper` (Python CLI; downloads models automatically)
3. **Gemini CLI** (`gemini`) using `read_many_files`
4. **Provider auth**
   - Configured `models.providers.*` entries that support audio are tried first
   - Bundled fallback order: OpenAI → Groq → xAI → Deepgram → Google → SenseAudio → ElevenLabs → Mistral

To disable auto-detection, set `tools.media.audio.enabled: false`.
To customize, set `tools.media.audio.models`.
Note: Binary detection is best-effort across macOS/Linux/Windows; ensure the CLI is on `PATH` (we expand `~`), or set an explicit CLI model with a full command path.

## Config examples

### Provider + CLI fallback (OpenAI + Whisper CLI)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        maxBytes: 20971520,
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          {
            type: "cli",
            command: "whisper",
            args: ["--model", "base", "{{MediaPath}}"],
            timeoutSeconds: 45,
          },
        ],
      },
    },
  },
}
```

### Provider-only with scope gating

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        scope: {
          default: "allow",
          rules: [{ action: "deny", match: { chatType: "group" } }],
        },
        models: [{ provider: "openai", model: "gpt-4o-mini-transcribe" }],
      },
    },
  },
}
```

### Provider-only (Deepgram)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "deepgram", model: "nova-3" }],
      },
    },
  },
}
```

### Provider-only (Mistral Voxtral)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", mod

## Camera

OpenClaw supports **camera capture** for agent workflows:

- **iOS node** (paired via Gateway): capture a **photo** (`jpg`) or **short video clip** (`mp4`, with optional audio) via `node.invoke`.
- **Android node** (paired via Gateway): capture a **photo** (`jpg`) or **short video clip** (`mp4`, with optional audio) via `node.invoke`.
- **macOS app** (node via Gateway): capture a **photo** (`jpg`) or **short video clip** (`mp4`, with optional audio) via `node.invoke`.

All camera access is gated behind **user-controlled settings**.

## iOS node

### User setting (default on)

- iOS Settings tab → **Camera** → **Allow Camera** (`camera.enabled`)
  - Default: **on** (missing key is treated as enabled).
  - When off: `camera.*` commands return `CAMERA_DISABLED`.

### Commands (via Gateway `node.invoke`)

- `camera.list`
  - Response payload:
    - `devices`: array of `{ id, name, position, deviceType }`

- `camera.snap`
  - Params:
    - `facing`: `front|back` (default: `front`)
    - `maxWidth`: number (optional; default `1600` on the iOS node)
    - `quality`: `0..1` (optional; default `0.9`)
    - `format`: currently `jpg`
    - `delayMs`: number (optional; default `0`)
    - `deviceId`: string (optional; from `camera.list`)
  - Response payload:
    - `format: "jpg"`
    - `base64: "<...>"`
    - `width`, `height`
  - Payload guard: photos are recompressed to keep the base64 payload under 5 MB.

- `camera.clip`
  - Params:
    - `facing`: `front|back` (default: `front`)
    - `durationMs`: number (default `3000`, clamped to a max of `60000`)
    - `includeAudio`: boolean (default `true`)
    - `format`: currently `mp4`
    - `deviceId`: string (optional; from `camera.list`)
  - Response payload:
    - `format: "mp4"`
    - `base64: "<...>"`
    - `durationMs`
    - `hasAudio`

### Foreground requirement

Like `canvas.*`, the iOS node only allows `camera.*` commands in the **foreground**. Background invocations return `NODE_BACKGROUND_UNAVAILABLE`.

### CLI helper (temp files + MEDIA)

The easiest way to get attachments is via the CLI helper, which writes decoded media to a temp file and prints `MEDIA:<path>`.

Examples:

```bash
openclaw nodes camera snap --node <id>               # default: both front + back (2 MEDIA lines)
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

Notes:

- `nodes camera snap` defaults to **both** facings to give the agent both views.
- Output files are temporary (in the OS temp directory) unless you build your own wrapper.

## Android node

### Android user setting (default on)

- Android Settings sheet → **Camera** → **Allow Camera** (`camera.enabled`)
  - Default: **on** (missing key is treated as enabled).
  - When off: `camera.*` commands return `CAMERA_DISABLED`.

### Permissions

- Android requires runtime permissions:
  - `CAMERA` for both `camera.snap` and `camera.clip`.
  - `RECORD_AUDIO` for `cam

## Talk (Voice Interface)

Talk mode is a continuous voice conversation loop:

1. Listen for speech
2. Send transcript to the model (main session, chat.send)
3. Wait for the response
4. Speak it via the configured Talk provider (`talk.speak`)

## Behavior (macOS)

- **Always-on overlay** while Talk mode is enabled.
- **Listening → Thinking → Speaking** phase transitions.
- On a **short pause** (silence window), the current transcript is sent.
- Replies are **written to WebChat** (same as typing).
- **Interrupt on speech** (default on): if the user starts talking while the assistant is speaking, we stop playback and note the interruption timestamp for the next prompt.

## Voice directives in replies

The assistant may prefix its reply with a **single JSON line** to control voice:

```json
{ "voice": "<voice-id>", "once": true }
```

Rules:

- First non-empty line only.
- Unknown keys are ignored.
- `once: true` applies to the current reply only.
- Without `once`, the voice becomes the new default for Talk mode.
- The JSON line is stripped before TTS playback.

Supported keys:

- `voice` / `voice_id` / `voiceId`
- `model` / `model_id` / `modelId`
- `speed`, `rate` (WPM), `stability`, `similarity`, `style`, `speakerBoost`
- `seed`, `normalize`, `lang`, `output_format`, `latency_tier`
- `once`

## Config (`~/.openclaw/openclaw.json`)

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

Defaults:

- `interruptOnSpeech`: true
- `silenceTimeoutMs`: when unset, Talk keeps the platform default pause window before sending the transcript (`700 ms on macOS and Android, 900 ms on iOS`)
- `provider`: selects the active Talk provider. Use `elevenlabs`, `mlx`, or `system` for the macOS-local playback paths.
- `providers.<provider>.voiceId`: falls back to `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID` for ElevenLabs (or first ElevenLabs voice when API key is available).
- `providers.elevenlabs.modelId`: defaults to `eleven_v3` when unset.
- `providers.mlx.modelId`: defaults to `mlx-community/Soprano-80M-bf16` when unset.
- `providers.elevenlabs.apiKey`: falls back to `ELEVENLABS_API_KEY` (or gateway shell profile if available).
- `speechLocale`: optional BCP 47 locale id for on-device Talk speech recognition on iOS/macOS. Leave unset to use the device default.
- `outputFormat`: defaults to `pcm_44100` on macOS/iOS and `pcm_24000` on Android (set `mp3_*` to force MP3 streaming)

## macOS UI

- Menu bar toggle: **Talk**
- Config tab: **Talk Mode** group (voice id + interrupt toggle)
- Overlay:
  - **Listening**: cloud pulses with mic level
  - **Thinking**: sinking animation
  - **Speaking**: radiating rings
  - Click cloud: stop speaking
  - Click X: exit Talk mode

## Android UI

- Voice tab toggle: **Talk**
- Manual **Mic** and **Talk** are mutually exclusive runtime capture modes.
- Manual Mic stops when the app leaves the foreground or the user leaves the Voice tab.
- Talk Mode keeps running until toggled off or the Android node disconnects, and uses Android's microphone foreground-service type while active.

## Notes

- Requires Speech + Microphone permissions.
- Uses `chat.send` against session key `main`.
- The gateway resolves Talk playback through `talk.speak` using the active Talk provider. Android falls back to local system TTS only when that RPC is unavailable.
- macOS local MLX playback uses the bundled `openclaw-mlx-tts` helper when present, or an executable on `PATH`. Set `OPENCLAW_MLX_TTS_BIN` to point at a custom helper binary during development.
- `stability` for `eleven_v3` is validated to `0.0`, `0.5`, or `1.0`; other models accept `0..1`.
- `latency_tier` is validated to `0..4` when set.
- Android supports `pcm_16000`, `pcm_22050`, `pcm_24000`, and `pcm_44100` output formats for low-latency AudioTrack streaming.

## Related

- [Voice wake](/nodes/voicewake)
- [Audio and voice notes](/nodes/audio)
- [Media understanding](/nodes/media-understanding)

## Voice Wake

OpenClaw treats **wake words as a single global list** owned by the **Gateway**.

- There are **no per-node custom wake words**.
- **Any node/app UI may edit** the list; changes are persisted by the Gateway and broadcast to everyone.
- macOS and iOS keep local **Voice Wake enabled/disabled** toggles (local UX + permissions differ).
- Android currently keeps Voice Wake off and uses a manual mic flow in the Voice tab.

## Storage (Gateway host)

Wake words are stored on the gateway machine at:

- `~/.openclaw/settings/voicewake.json`

Shape:

```json
{ "triggers": ["openclaw", "claude", "computer"], "updatedAtMs": 1730000000000 }
```

## Protocol

### Methods

- `voicewake.get` → `{ triggers: string[] }`
- `voicewake.set` with params `{ triggers: string[] }` → `{ triggers: string[] }`

Notes:

- Triggers are normalized (trimmed, empties dropped). Empty lists fall back to defaults.
- Limits are enforced for safety (count/length caps).

### Routing methods (trigger → target)

- `voicewake.routing.get` → `{ config: VoiceWakeRoutingConfig }`
- `voicewake.routing.set` with params `{ config: VoiceWakeRoutingConfig }` → `{ config: VoiceWakeRoutingConfig }`

`VoiceWakeRoutingConfig` shape:

```json
{
  "version": 1,
  "defaultTarget": { "mode": "current" },
  "routes": [{ "trigger": "robot wake", "target": { "sessionKey": "agent:main:main" } }],
  "updatedAtMs": 1730000000000
}
```

Route targets support exactly one of:

- `{ "mode": "current" }`
- `{ "agentId": "main" }`
- `{ "sessionKey": "agent:main:main" }`

### Events

- `voicewake.changed` payload `{ triggers: string[] }`
- `voicewake.routing.changed` payload `{ config: VoiceWakeRoutingConfig }`

Who receives it:

- All WebSocket clients (macOS app, WebChat, etc.)
- All connected nodes (iOS/Android), and also on node connect as an initial “current state” push.

## Client behavior

### macOS app

- Uses the global list to gate `VoiceWakeRuntime` triggers.
- Editing “Trigger words” in Voice Wake settings calls `voicewake.set` and then relies on the broadcast to keep other clients in sync.

### iOS node

- Uses the global list for `VoiceWakeManager` trigger detection.
- Editing Wake Words in Settings calls `voicewake.set` (over the Gateway WS) and also keeps local wake-word detection responsive.

### Android node

- Voice Wake is currently disabled in Android runtime/Settings.
- Android voice uses manual mic capture in the Voice tab instead of wake-word triggers.

## Related

- [Talk mode](/nodes/talk)
- [Audio and voice notes](/nodes/audio)
- [Media understanding](/nodes/media-understanding)

## Location Commands

## TL;DR

- `location.get` is a node command (via `node.invoke`).
- Off by default.
- Android app settings use a selector: Off / While Using.
- Separate toggle: Precise Location.

## Why a selector (not just a switch)

OS permissions are multi-level. We can expose a selector in-app, but the OS still decides the actual grant.

- iOS/macOS may expose **While Using** or **Always** in system prompts/Settings.
- Android app currently supports foreground location only.
- Precise location is a separate grant (iOS 14+ “Precise”, Android “fine” vs “coarse”).

Selector in UI drives our requested mode; actual grant lives in OS settings.

## Settings model

Per node device:

- `location.enabledMode`: `off | whileUsing`
- `location.preciseEnabled`: bool

UI behavior:

- Selecting `whileUsing` requests foreground permission.
- If OS denies requested level, revert to the highest granted level and show status.

## Permissions mapping (node.permissions)

Optional. macOS node reports `location` via the permissions map; iOS/Android may omit it.

## Command: `location.get`

Called via `node.invoke`.

Params (suggested):

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

Response payload:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

Errors (stable codes):

- `LOCATION_DISABLED`: selector is off.
- `LOCATION_PERMISSION_REQUIRED`: permission missing for requested mode.
- `LOCATION_BACKGROUND_UNAVAILABLE`: app is backgrounded but only While Using allowed.
- `LOCATION_TIMEOUT`: no fix in time.
- `LOCATION_UNAVAILABLE`: system failure / no providers.

## Background behavior

- Android app denies `location.get` while backgrounded.
- Keep OpenClaw open when requesting location on Android.
- Other node platforms may differ.

## Model/tooling integration

- Tool surface: `nodes` tool adds `location_get` action (node required).
- CLI: `openclaw nodes location get --node <id>`.
- Agent guidelines: only call when user enabled location and understands the scope.

## UX copy (suggested)

- Off: “Location sharing is disabled.”
- While Using: “Only when OpenClaw is open.”
- Precise: “Use precise GPS location. Toggle off to share approximate location.”

## Related

- [Channel location parsing](/channels/location)
- [Camera capture](/nodes/camera)
- [Talk mode](/nodes/talk)

## Troubleshooting Nodes

Use this page when a node is visible in status but node tools fail.

## Command ladder

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Then run node specific checks:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

Healthy signals:

- Node is connected and paired for role `node`.
- `nodes describe` includes the capability you are calling.
- Exec approvals show expected mode/allowlist.

## Foreground requirements

`canvas.*`, `camera.*`, and `screen.*` are foreground only on iOS/Android nodes.

Quick check and fix:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

If you see `NODE_BACKGROUND_UNAVAILABLE`, bring the node app to the foreground and retry.

## Permissions matrix

| Capability                   | iOS                                     | Android                                      | macOS node app                | Typical failure code           |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | ----------------------------- | ------------------------------ |
| `camera.snap`, `camera.clip` | Camera (+ mic for clip audio)           | Camera (+ mic for clip audio)                | Camera (+ mic for clip audio) | `*_PERMISSION_REQUIRED`        |
| `screen.record`              | Screen Recording (+ mic optional)       | Screen capture prompt (+ mic optional)       | Screen Recording              | `*_PERMISSION_REQUIRED`        |
| `location.get`               | While Using or Always (depends on mode) | Foreground/Background location based on mode | Location permission           | `LOCATION_PERMISSION_REQUIRED` |
| `system.run`                 | n/a (node host path)                    | n/a (node host path)                         | Exec approvals re

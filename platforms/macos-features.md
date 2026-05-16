---
domain: platforms
topic: "macOS App Features: Menu Bar, Permissions, Voice, Canvas"
type: reference
keywords:
  - macOS features
  - menu bar
  - permissions
  - voice overlay
  - canvas
  - peekaboo
  - bundled gateway
source: 
  - platforms/mac/bundled-gateway.md
  - platforms/mac/canvas.md
  - platforms/mac/child-process.md
  - platforms/mac/dev-setup.md
  - platforms/mac/health.md
  - platforms/mac/icon.md
  - platforms/mac/logging.md
  - platforms/mac/menu-bar.md
  - platforms/mac/peekaboo.md
  - platforms/mac/permissions.md
  - platforms/mac/remote.md
  - platforms/mac/signing.md
  - platforms/mac/skills.md
  - platforms/mac/voice-overlay.md
  - platforms/mac/voicewake.md
  - platforms/mac/webchat.md
  - platforms/mac/xpc.md
---

macOS-specific features and configuration for the OpenClaw desktop app.


## Bundled Gateway

OpenClaw.app no longer bundles Node/Bun or the Gateway runtime. The macOS app
expects an **external** `openclaw` CLI install, does not spawn the Gateway as a
child process, and manages a per-user launchd service to keep the Gateway
running (or attaches to an existing local Gateway if one is already running).

## Install the CLI (required for local mode)

Node 24 is the default runtime on the Mac. Node 22 LTS, currently `22.16+`, still works for compatibility. Then install `openclaw` globally:

```bash
npm install -g openclaw@<version>
```

The macOS app's **Install CLI** button runs the same global install flow the app
uses internally: it prefers npm first, then pnpm, then bun if that is the only
detected package manager. Node remains the recommended Gateway runtime.

## Launchd (Gateway as LaunchAgent)

Label:

- `ai.openclaw.gateway` (or `ai.openclaw.<profile>`; legacy `com.openclaw.*` may remain)

Plist location (per-user):

- `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
  (or `~/Library/LaunchAgents/ai.openclaw.<profile>.plist`)

Manager:

- The macOS app owns LaunchAgent install/update in Local mode.
- The CLI can also install it: `openclaw gateway install`.

Behavior:

- "OpenClaw Active" enables/disables the LaunchAgent.
- App quit does **not** stop the gateway (launchd keeps it alive).
- If a Gateway is already running on the configured port, the app attaches to
  it instead of starting a new one.

Logging:

- launchd stdout/err: `/tmp/openclaw/openclaw-gateway.log`

## Version compatibility

The macOS app checks the gateway version against its own version. If they're
incompatible, update the global CLI to match the app version.

## Smoke check

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

Then:

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```

## Related

- [macOS app](/platforms/macos)
- [Gateway runbook](/gateway)


## Canvas

The macOS app embeds an agent-controlled **Canvas panel** using `WKWebView`. It
is a lightweight visual workspace for HTML/CSS/JS, A2UI, and small interactive
UI surfaces.

## Where Canvas lives

Canvas state is stored under Application Support:

- `~/Library/Application Support/OpenClaw/canvas/<session>/...`

The Canvas panel serves those files via a **custom URL scheme**:

- `openclaw-canvas://<session>/<path>`

Examples:

- `openclaw-canvas://main/` → `<canvasRoot>/main/index.html`
- `openclaw-canvas://main/assets/app.css` → `<canvasRoot>/main/assets/app.css`
- `openclaw-canvas://main/widgets/todo/` → `<canvasRoot>/main/widgets/todo/index.html`

If no `index.html` exists at the root, the app shows a **built-in scaffold page**.

## Panel behavior

- Borderless, resizable panel anchored near the menu bar (or mouse cursor).
- Remembers size/position per session.
- Auto-reloads when local canvas files change.
- Only one Canvas panel is visible at a time (session is switched as needed).

Canvas can be disabled from Settings → **Allow Canvas**. When disabled, canvas
node commands return `CANVAS_DISABLED`.

## Agent API surface

Canvas is exposed via the **Gateway WebSocket**, so the agent can:

- show/hide the panel
- navigate to a path or URL
- evaluate JavaScript
- capture a snapshot image

CLI examples:

```bash
openclaw nodes canvas present --node <id>
openclaw nodes canvas navigate --node <id> --url "/"
openclaw nodes canvas eval --node <id> --js "document.title"
openclaw nodes canvas snapshot --node <id>
```

Notes:

- `canvas.navigate` accepts **local canvas paths**, `http(s)` URLs, and `file://` URLs.
- If you pass `"/"`, the Canvas shows the local scaffold or `index.html`.

## A2UI in Canvas

A2UI is hosted by the Gateway canvas host and rendered inside the Canvas panel.
When the Gateway advertises a Canvas host, the macOS app auto-navigates to the
A2UI host page on first open.

Default A2UI host URL:

```
http://<gateway-host>:18789/__openclaw__/a2ui/
```

### A2UI commands (v0.8)

Canvas currently accepts **A2UI v0.8** server→client messages:

- `beginRendering`
- `surfaceUpdate`
- `dataModelUpdate`
- `deleteSurface`

`createSurface` (v0.9) is not supported.

CLI example:

```bash
cat > /tmp/a2ui-v0.8.jsonl <<'EOFA2'
{"surfaceUpdate":{"surfaceId":"main","components":[{"id":"root","component":{"Column":{"children":{"explicitList":["title","content"]}}}},{"id":"title","component":{"Text":{"text":{"literalString":"Canvas (A2UI v0.8)"},"usageHint":"h1"}}},{"id":"content","component":{"Text":{"text":{"literalString":"If you can read this, A2UI push works."},"usageHint":"body"}}}]}}
{"beginRendering":{"surfaceId":"main","root":"root"}}
EOFA2

openclaw nodes canvas a2ui push --jsonl /tmp/a2ui-v0.8.jsonl --node <id>
```

Quick smoke:

```bash
openclaw nodes canvas a2ui push --node <id> --text "Hello from A2UI"
```

## Triggering agent runs from Canvas

Canvas can trigger new agent runs via deep links:

- `openclaw://agent?...`

Example (in JS):

```js
window.location.href = "openclaw://agent?message=Review%20this%20design";
```

The app prompts for confirmation unless a valid key is provided.

## Security notes

- Canvas scheme blocks directory traversal; files must live under the session root.
- Local Canvas content uses a custom scheme (no loopback server required).
- External `http(s)` URLs are allowed only when explicitly navigated.

## Related

- [macOS app](/platforms/macos)
- [WebChat](/web/webchat)


## Child Process

The macOS app **manages the Gateway via launchd** by default and does not spawn
the Gateway as a child process. It first tries to attach to an already-running
Gateway on the configured port; if none is reachable, it enables the launchd
service via the external `openclaw` CLI (no embedded runtime). This gives you
reliable auto-start at login and restart on crashes.

Child-process mode (Gateway spawned directly by the app) is **not in use** today.
If you need tighter coupling to the UI, run the Gateway manually in a terminal.

## Default behavior (launchd)

- The app installs a per-user LaunchAgent labeled `ai.openclaw.gateway`
  (or `ai.openclaw.<profile>` when using `--profile`/`OPENCLAW_PROFILE`; legacy `com.openclaw.*` is supported).
- When Local mode is enabled, the app ensures the LaunchAgent is loaded and
  starts the Gateway if needed.
- Logs are written to the launchd gateway log path (visible in Debug Settings).

Common commands:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Replace the label with `ai.openclaw.<profile>` when running a named profile.

## Unsigned dev builds

`scripts/restart-mac.sh --no-sign` is for fast local builds when you don't have
signing keys. To prevent launchd from pointing at an unsigned relay binary, it:

- Writes `~/.openclaw/disable-launchagent`.

Signed runs of `scripts/restart-mac.sh` clear this override if the marker is
present. To reset manually:

```bash
rm ~/.openclaw/disable-launchagent
```

## Attach-only mode

To force the macOS app to **never install or manage launchd**, launch it with
`--attach-only` (or `--no-launchd`). This sets `~/.openclaw/disable-launchagent`,
so the app only attaches to an already running Gateway. You can toggle the same
behavior in Debug Settings.

## Remote mode

Remote mode never starts a local Gateway. The app uses an SSH tunnel to the
remote host and connects over that tunnel.

## Why we prefer launchd

- Auto-start at login.
- Built-in restart/KeepAlive semantics.
- Predictable logs and supervision.

If a true child-process mode is ever needed again, it should be documented as a
separate, explicit dev-only mode.

## Related

- [macOS app](/platforms/macos)
- [Gateway runbook](/gateway)


## Dev Setup

# macOS developer setup

Build and run the OpenClaw macOS application from source.

## Prerequisites

Before building the app, ensure you have the following installed:

1. **Xcode 26.2+**: Required for Swift development.
2. **Node.js 24 & pnpm**: Recommended for the gateway, CLI, and packaging scripts. Node 22 LTS, currently `22.16+`, remains supported for compatibility.

## 1. Install Dependencies

Install the project-wide dependencies:

```bash
pnpm install
```

## 2. Build and Package the App

To build the macOS app and package it into `dist/OpenClaw.app`, run:

```bash
./scripts/package-mac-app.sh
```

If you don't have an Apple Developer ID certificate, the script will automatically use **ad-hoc signing** (`-`).

For dev run modes, signing flags, and Team ID troubleshooting, see the macOS app README:
[https://github.com/openclaw/openclaw/blob/main/apps/macos/README.md](https://github.com/openclaw/openclaw/blob/main/apps/macos/README.md)

> **Note**: Ad-hoc signed apps may trigger security prompts. If the app crashes immediately with "Abort trap 6", see the [Troubleshooting](#troubleshooting) section.

## 3. Install the CLI

The macOS app expects a global `openclaw` CLI install to manage background tasks.

**To install it (recommended):**

1. Open the OpenClaw app.
2. Go to the **General** settings tab.
3. Click **"Install CLI"**.

Alternatively, install it manually:

```bash
npm install -g openclaw@<version>
```

`pnpm add -g openclaw@<version>` and `bun add -g openclaw@<version>` also work.
For the Gateway runtime, Node remains the recommended path.

## Troubleshooting

### Build fails: toolchain or SDK mismatch

The macOS app build expects the latest macOS SDK and Swift 6.2 toolchain.

**System dependencies (required):**

- **Latest macOS version available in Software Update** (required by Xcode 26.2 SDKs)
- **Xcode 26.2** (Swift 6.2 toolchain)

**Checks:**

```bash
xcodebuild -version
xcrun swift --version
```

If versions don't match, update macOS/Xcode and re-run the build.

### App crashes on permission grant

If the app crashes when you try to allow **Speech Recognition** or **Microphone** access, it may be due to a corrupted TCC cache or signature mismatch.

**Fix:**

1. Reset the TCC permissions:

   ```bash
   tccutil reset All ai.openclaw.mac.debug
   ```

2. If that fails, change the `BUNDLE_ID` temporarily in [`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh) to force a "clean slate" from macOS.

### Gateway "Starting..." indefinitely

If the gateway status stays on "Starting...", check if a zombie process is holding the port:

```bash
openclaw gateway status
openclaw gateway stop

# If you're not using a LaunchAgent (dev mode / manual runs), find the listener:
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

If a manual run is holding the port, stop that process (Ctrl+C). As a last resort, kill the PID you found above.

## Related

- [macOS app](/platforms/macos)
- [Install overview](/install)


## Health

# Health Checks on macOS

How to see whether the linked channel is healthy from the menu bar app.

## Menu bar

- Status dot now reflects Baileys health:
  - Green: linked + socket opened recently.
  - Orange: connecting/retrying.
  - Red: logged out or probe failed.
- Secondary line reads "linked · auth 12m" or shows the failure reason.
- "Run Health Check" menu item triggers an on-demand probe.

## Settings

- General tab gains a Health card showing: linked auth age, session-store path/count, last check time, last error/status code, and buttons for Run Health Check / Reveal Logs.
- Uses a cached snapshot so the UI loads instantly and falls back gracefully when offline.
- **Channels tab** surfaces channel status + controls for WhatsApp/Telegram (login QR, logout, probe, last disconnect/error).

## How the probe works

- App runs `openclaw health --json` via `ShellExecutor` every ~60s and on demand. The probe loads creds and reports status without sending messages.
- Cache the last good snapshot and the last error separately to avoid flicker; show the timestamp of each.

## When in doubt

- You can still use the CLI flow in [Gateway health](/gateway/health) (`openclaw status`, `openclaw status --deep`, `openclaw health --json`) and tail `/tmp/openclaw/openclaw-*.log` for `web-heartbeat` / `web-reconnect`.

## Related

- [Gateway health](/gateway/health)
- [macOS app](/platforms/macos)


## Icon

# Menu Bar Icon States

Author: steipete · Updated: 2025-12-06 · Scope: macOS app (`apps/macos`)

- **Idle:** Normal icon animation (blink, occasional wiggle).
- **Paused:** Status item uses `appearsDisabled`; no motion.
- **Voice trigger (big ears):** Voice wake detector calls `AppState.triggerVoiceEars(ttl: nil)` when the wake word is heard, keeping `earBoostActive=true` while the utterance is captured. Ears scale up (1.9x), get circular ear holes for readability, then drop via `stopVoiceEars()` after 1s of silence. Only fired from the in-app voice pipeline.
- **Working (agent running):** `AppState.isWorking=true` drives a "tail/leg scurry" micro-motion: faster leg wiggle and slight offset while work is in-flight. Currently toggled around WebChat agent runs; add the same toggle around other long tasks when you wire them.

Wiring points

- Voice wake: runtime/tester call `AppState.triggerVoiceEars(ttl: nil)` on trigger and `stopVoiceEars()` after 1s of silence to match the capture window.
- Agent activity: set `AppStateStore.shared.setWorking(true/false)` around work spans (already done in WebChat agent call). Keep spans short and reset in `defer` blocks to avoid stuck animations.

Shapes & sizes

- Base icon drawn in `CritterIconRenderer.makeIcon(blink:legWiggle:earWiggle:earScale:earHoles:)`.
- Ear scale defaults to `1.0`; voice boost sets `earScale=1.9` and toggles `earHoles=true` without changing overall frame (18×18 pt template image rendered into a 36×36 px Retina backing store).
- Scurry uses leg wiggle up to ~1.0 with a small horizontal jiggle; it's additive to any existing idle wiggle.

Behavioral notes

- No external CLI/broker toggle for ears/working; keep it internal to the app's own signals to avoid accidental flapp
---
domain: platforms
topic: "macOS Gateway Lifecycle: launchd, Health Checks, and WebChat Embedding"
type: reference
keywords:
  - macOS gateway
  - launchd
  - LaunchAgent
  - macOS Gateway lifecycle
  - bundled gateway
  - macOS health checks
  - webchat macOS
  - cli install macOS
  - ai.openclaw.gateway
  - disable-launchagent
source:
  - platforms/mac/bundled-gateway.md
  - platforms/mac/child-process.md
  - platforms/mac/health.md
  - platforms/mac/webchat.md
related:
  - platforms/macos-app
  - gateway/gateway-runbook
  - gateway/health-diagnostics-logging
---

The macOS app manages the Gateway via launchd as an external service — it does not bundle or spawn the Gateway as a child process. The app attaches to a running Gateway on the configured port, or enables the launchd service if none is reachable.

## Install the CLI (Required for Local Mode)

Node 24 is the default runtime. Node 22 LTS (`22.14+`) also works. Install globally:

```bash
npm install -g openclaw@<version>
```

The macOS app's **Install CLI** button runs the same flow: it prefers npm, then pnpm, then bun.

## launchd Gateway (LaunchAgent)

**Label:** `ai.openclaw.gateway` (or `ai.openclaw.<profile>`; legacy `com.openclaw.*` also supported)

**Plist location:** `~/Library/LaunchAgents/ai.openclaw.gateway.plist`

**Behavior:**
- "OpenClaw Active" enables/disables the LaunchAgent.
- App quit does **not** stop the gateway (launchd keeps it alive).
- If a Gateway is already running on the configured port, the app attaches instead of starting a new one.

**Log file:** `/tmp/openclaw/openclaw-gateway.log`

**Manual control:**

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Replace the label with `ai.openclaw.<profile>` when running a named profile.

## Version Compatibility

The macOS app checks the gateway version against its own version. If they're incompatible, update the global CLI to match the app version.

## Smoke Check

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 OPENCLAW_SKIP_CANVAS_HOST=1 \
  openclaw gateway --port 18999 --bind loopback

# In another terminal:
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```

## Attach-Only Mode

Force the macOS app to never install or manage launchd by launching with `--attach-only` (or `--no-launchd`). This sets `~/.openclaw/disable-launchagent`. Toggle the same behavior in Debug Settings.

For unsigned dev builds, `scripts/restart-mac.sh --no-sign` writes `~/.openclaw/disable-launchagent` automatically. Signed runs clear this override. To reset manually:

```bash
rm ~/.openclaw/disable-launchagent
```

## Remote Mode

Remote mode never starts a local Gateway. The app uses an SSH tunnel to the remote host and connects over that tunnel.

## Health Checks (macOS App)

The app runs `openclaw health --json` via ShellExecutor every ~60s and on demand.

**Menu bar indicators:**
- Green dot: linked + socket opened recently
- Orange dot: connecting/retrying
- Red dot: logged out or probe failed
- Secondary line reads "linked · auth 12m" or shows failure reason

**Settings Health card** (General tab) shows: linked auth age, session-store path/count, last check time, last error, and buttons for Run Health Check and Reveal Logs.

**Channels tab** surfaces channel status and controls for WhatsApp/Telegram (login QR, logout, probe, last disconnect/error).

When in doubt, use the CLI:

```bash
openclaw status
openclaw status --deep
openclaw health --json
```

## WebChat Embedding (macOS App)

The macOS menu bar app embeds WebChat as a native SwiftUI view connecting to the Gateway. Defaults to the **main session** for the selected agent with a session switcher.

- **Local mode:** connects directly to the local Gateway WebSocket.
- **Remote mode:** forwards the Gateway control port over SSH and uses that tunnel.

**Gateway WS methods used:** `chat.history`, `chat.send`, `chat.abort`, `chat.inject`

**Events:** `chat`, `agent`, `presence`, `tick`, `health`

`chat.history` strips inline directive tags, tool-call XML payloads, leaked model control tokens, and pure silent-token rows (`NO_REPLY`/`no_reply`) from visible text.

**Debug:**

```bash
# Launch with webchat auto-opened
dist/OpenClaw.app/Contents/MacOS/OpenClaw --webchat

# Logs (subsystem: ai.openclaw, category: WebChatSwiftUI)
./scripts/clawlog.sh
```

## Related

- [macOS app](/platforms/macos-app)
- [Gateway runbook](/gateway/gateway-runbook)
- [Gateway health](/gateway/health-diagnostics-logging)

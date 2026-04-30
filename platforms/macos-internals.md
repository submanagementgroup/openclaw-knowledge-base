---
domain: platforms
topic: "macOS Internals: Dev Setup, Code Signing, XPC, Child Process, and Logging"
type: reference
keywords:
  - macOS dev
  - code signing
  - XPC
  - macOS internals
  - macOS child process
related:
  - platforms/macos-app
source:
  - platforms/mac/dev-setup.md
  - platforms/mac/signing.md
  - platforms/mac/xpc.md
---

macOS development setup, signing, XPC, and internals for developers building the macOS app.

## Dev Setup

# macOS developer setup

Build and run the OpenClaw macOS application from source.

## Prerequisites

Before building the app, ensure you have the following installed:

1. **Xcode 26.2+**: Required for Swift development.
2. **Node.js 24 & pnpm**: Recommended for the gateway, CLI, and packaging scripts. Node 22 LTS, currently `22.14+`, remains supported for compatibility.

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

If versions don’t match, update macOS/Xcode and re-run the build.

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

## Signing

# mac signing (debug builds)

This app is usually built from [`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh), which now:

- sets a stable debug bundle identifier: `ai.openclaw.mac.debug`
- writes the Info.plist with that bundle id (override via `BUNDLE_ID=...`)
- calls [`scripts/codesign-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/codesign-mac-app.sh) to sign the main binary and app bundle so macOS treats each rebuild as the same signed bundle and keeps TCC permissions (notifications, accessibility, screen recording, mic, speech). For stable permissions, use a real signing identity; ad-hoc is opt-in and fragile (see [macOS permissions](/platforms/mac/permissions)).
- uses `CODESIGN_TIMESTAMP=auto` by default; it enables trusted timestamps for Developer ID signatures. Set `CODESIGN_TIMESTAMP=off` to skip timestamping (offline debug builds).
- inject build metadata into Info.plist: `OpenClawBuildTimestamp` (UTC) and `OpenClawGitCommit` (short hash) so the About pane can show build, git, and debug/release channel.
- **Packaging defaults to Node 24**: the script runs TS builds and the Control UI build. Node 22 LTS, currently `22.14+`, remains supported for compatibility.
- reads `SIGN_IDENTITY` from the environment. Add `export SIGN_IDENTITY="Apple Development: Your Name (TEAMID)"` (or your Developer ID Application cert) to your shell rc to always sign with your cert. Ad-hoc signing requires explicit opt-in via `ALLOW_ADHOC_SIGNING=1` or `SIGN_IDENTITY="-"` (not recommended for permission testing).
- runs a Team ID audit after signing and fails if any Mach-O inside the app bundle is signed by a different Team ID. Set `SKIP_TEAM_ID_CHECK=1` to bypass.

## Usage

```bash
# from repo root
scripts/package-mac-app.sh               # auto-selects identity; errors if none found
SIGN_IDENTITY="Developer ID Application: Your Name" scripts/package-mac-app.sh   # real cert
ALLOW_ADHOC_SIGNING=1 scripts/package-mac-app.sh    # ad-hoc (permissions will not stick)
SIGN_IDENTITY="-" scripts/package-mac-app.sh        # explicit ad-hoc (same caveat)
DISABLE_LIBRARY_VALIDATION=1 scripts/package-mac-app.sh   # dev-only Sparkle Team ID mismatch workaround
```

### Ad-hoc Signing Note

When signing with `SIGN_IDENTITY="-"` (ad-hoc), the script automatically disables the **Hardened Runtime** (`--options runtime`). This is necessary to prevent crashes when the app attempts to load embedded frameworks (like Sparkle) that do not share the same Team ID. Ad-hoc signatures also break TCC permission persistence; see [macOS permissions](/platforms/mac/permissions) for recovery steps.

## Build metadata for About

`package-mac-app.sh` stamps the bundle with:

- `OpenClawBuildTimestamp`: ISO8601 UTC at package time
- `OpenClawGitCommit`: short git hash (or `unknown` if unavailable)

The About tab reads these keys to show version, build date, git commit, and whether it’s a debug build (via `#if DEBUG`). Run the packager to refresh these values after code changes.

## Why

TCC permissions are tied to the bundle identifier _and_ code signature. Unsigned debug builds with changing UUIDs were causing macOS to forget grants after each rebuild. Signing the binaries (ad‑hoc by default) and keeping a fixed bundle id/path (`dist/OpenClaw.app`) preserves the grants between builds, matching the VibeTunnel approach.

## Related

- [macOS app](/platforms/macos)
- [macOS permissions](/platforms/mac/permissions)

## XPC

# OpenClaw macOS IPC architecture

**Current model:** a local Unix socket connects the **node host service** to the **macOS app** for exec approvals + `system.run`. A `openclaw-mac` debug CLI exists for discovery/connect checks; agent actions still flow through the Gateway WebSocket and `node.invoke`. UI automation uses PeekabooBridge.

## Goals

- Single GUI app instance that owns all TCC-facing work (notifications, screen recording, mic, speech, AppleScript).
- A small surface for automation: Gateway + node commands, plus PeekabooBridge for UI automation.
- Predictable permissions: always the same signed bundle ID, launched by launchd, so TCC grants stick.

## How it works

### Gateway + node transport

- The app runs the Gateway (local mode) and connects to it as a node.
- Agent actions are performed via `node.invoke` (e.g. `system.run`, `system.notify`, `canvas.*`).

### Node service + app IPC

- A headless node host service connects to the Gateway WebSocket.
- `system.run` requests are forwarded to the macOS app over a local Unix socket.
- The app performs the exec in UI context, prompts if needed, and returns output.

Diagram (SCI):

```
Agent -> Gateway -> Node Service (WS)
                      |  IPC (UDS + token + HMAC + TTL)
                      v
                  Mac App (UI + TCC + system.run)
```

### PeekabooBridge (UI automation)

- UI automation uses a separate UNIX socket named `bridge.sock` and the PeekabooBridge JSON protocol.
- Host preference order (client-side): Peekaboo.app → Claude.app → OpenClaw.app → local execution.
- Security: bridge hosts require an allowed TeamID; DEBUG-only same-UID escape hatch is guarded by `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (Peekaboo convention).
- See: [PeekabooBridge usage](/platforms/mac/peekaboo) for details.

## Operational flows

- Restart/rebuild: `SIGN_IDENTITY="Apple Development: <Developer Name> (<TEAMID>)" scripts/restart-mac.sh`
  - Kills existing instances
  - Swift build + package
  - Writes/bootstraps/kickstarts the LaunchAgent
- Single instance: app exits early if another instance with the same bundle ID is running.

## Hardening notes

- Prefer requiring a TeamID match for all privileged surfaces.
- PeekabooBridge: `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (DEBUG-only) may allow same-UID callers for local development.
- All communication remains local-only; no network sockets are exposed.
- TCC prompts originate only from the GUI app bundle; keep the signed bundle ID stable across rebuilds.
- IPC hardening: socket mode `0600`, token, peer-UID checks, HMAC challenge/response, short TTL.

## Related

- [macOS app](/platforms/macos)
- [macOS IPC flow (Exec approvals)](/tools/exec-approvals-advanced#macos-ipc-flow)

## Child Process

# Gateway lifecycle on macOS

The macOS app **manages the Gateway via launchd** by default and does not spawn
the Gateway as a child process. It first tries to attach to an already‑running
Gateway on the configured port; if none is reachable, it enables the launchd
service via the external `openclaw` CLI (no embedded runtime). This gives you
reliable auto‑start at login and restart on crashes.

Child‑process mode (Gateway spawned directly by the app) is **not in use** today.
If you need tighter coupling to the UI, run the Gateway manually in a terminal.

## Default behavior (launchd)

- The app installs a per‑user LaunchAgent labeled `ai.openclaw.gateway`
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

`scripts/restart-mac.sh --no-sign` is for fast local builds when you don’t have
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

- Auto‑start at login.
- Built‑in restart/KeepAlive semantics.
- Predictable logs and supervision.

If a true child‑process mode is ever needed again, it should be documented as a
separate, explicit dev‑only mode.

## Related

- [macOS app](/platforms/macos)
- [Gateway runbook](/gateway)

## Logging (macOS)

# Logging (macOS)

## Rolling diagnostics file log (Debug pane)

OpenClaw routes macOS app logs through swift-log (unified logging by default) and can write a local, rotating file log to disk when you need a durable capture.

- Verbosity: **Debug pane → Logs → App logging → Verbosity**
- Enable: **Debug pane → Logs → App logging → “Write rolling diagnostics log (JSONL)”**
- Location: `~/Library/Logs/OpenClaw/diagnostics.jsonl` (rotates automatically; old files are suffixed with `.1`, `.2`, …)
- Clear: **Debug pane → Logs → App logging → “Clear”**

Notes:

- This is **off by default**. Enable only while actively debugging.
- Treat the file as sensitive; don’t share it without review.

## Unified logging private data on macOS

Unified logging redacts most payloads unless a subsystem opts into `privacy -off`. Per Peter's write-up on macOS [logging privacy shenanigans](https://steipete.me/posts/2025/logging-privacy-shenanigans) (2025) this is controlled by a plist in `/Library/Preferences/Logging/Subsystems/` keyed by the subsystem name. Only new log entries pick up the flag, so enable it before reproducing an issue.

## Enable for OpenClaw (`ai.openclaw`)

- Write the plist to a temp file first, then install it atomically as root:

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

- No reboot is required; logd notices the file quickly, but only new log lines will include private payloads.
- View the richer output with the existing helper, e.g. `./scripts/clawlog.sh --category WebChat --last 5m`.

## Disable after debugging

- Remove the override: `sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`.
- Optionally run `sudo log config --reload` to force logd to drop the override immediately.
- Remember this surface can include phone numbers and message bodies; keep the plist in place only while you actively need the extra detail.

## Related

- [macOS app](/platforms/macos)
- [Gateway logging](/gateway/logging)

## Remote Access (macOS)

# Remote OpenClaw (macOS ⇄ remote host)

This flow lets the macOS app act as a full remote control for an OpenClaw gateway running on another host (desktop/server). It’s the app’s **Remote over SSH** (remote run) feature. All features—health checks, Voice Wake forwarding, and Web Chat—reuse the same remote SSH configuration from _Settings → General_.

## Modes

- **Local (this Mac)**: Everything runs on the laptop. No SSH involved.
- **Remote over SSH (default)**: OpenClaw commands are executed on the remote host. The mac app opens an SSH connection with `-o BatchMode` plus your chosen identity/key and a local port-forward.
- **Remote direct (ws/wss)**: No SSH tunnel. The mac app connects to the gateway URL directly (for example, via Tailscale Serve or a public HTTPS reverse proxy).

## Remote transports

Remote mode supports two transports:

- **SSH tunnel** (default): Uses `ssh -N -L ...` to forward the gateway port to localhost. The gateway will see the node’s IP as `127.0.0.1` because the tunnel is loopback.
- **Direct (ws/wss)**: Connects straight to the gateway URL. The gateway sees the real client IP.

In SSH tunnel mode, discovered LAN/tailnet hostnames are saved as
`gateway.remote.sshTarget`. The app keeps `gateway.remote.url` on the local
tunnel endpoint, for example `ws://127.0.0.1:18789`, so CLI, Web Chat, and
the local node-host service all use the same safe loopback transport.

Browser automation in remote mode is owned by the CLI node host, not by the
native macOS app node. The app starts the installed node host service when
possible; if you need browser control from that Mac, install/start it with
`openclaw node install ...` and `openclaw node start` (or run
`openclaw node run ...` in the foreground), then target that browser-capable
node.

## Prereqs on the remote host

1. Install Node + pnpm and build/install the OpenClaw CLI (`pnpm install && pnpm build && pnpm link --global`).
2. Ensure `openclaw` is on PATH for non-interactive shells (symlink in

## Skills (macOS)

The macOS app surfaces OpenClaw skills via the gateway; it does not parse skills locally.

## Data source

- `skills.status` (gateway) returns all skills plus eligibility and missing requirements
  (including allowlist blocks for bundled skills).
- Requirements are derived from `metadata.openclaw.requires` in each `SKILL.md`.

## Install actions

- `metadata.openclaw.install` defines install options (brew/node/go/uv).
- The app calls `skills.install` to run installers on the gateway host.
- Built-in dangerous-code `critical` findings block `skills.install` by default; suspicious findings still warn only. The dangerous override exists on the gateway request, but the default app flow stays fail-closed.
- If every install option is `download`, the gateway surfaces all download
  choices.
- Otherwise, the gateway picks one preferred installer using the current
  install preferences and host binaries: Homebrew first when
  `skills.install.preferBrew` is enabled and `brew` exists, then `uv`, then the
  configured node manager from `skills.install.nodeManager`, then later
  fallbacks like `go` or `download`.
- Node install labels reflect the configured node manager, including `yarn`.

## Env/API keys

- The app stores keys in `~/.openclaw/openclaw.json` under `skills.entries.<skillKey>`.
- `skills.update` patches `enabled`, `apiKey`, and `env`.

## Remote mode

- Install + config updates happen on the gateway host (not the local Mac).

## Related

- [Skills](/tools/skills)
- [macOS app](/platforms/macos)

## Health (macOS)

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

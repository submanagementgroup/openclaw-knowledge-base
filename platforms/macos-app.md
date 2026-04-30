---
domain: platforms
topic: "macOS App: Menu Bar, Permissions, Bundled Gateway, and macOS-Specific Features"
type: procedure
keywords:
  - macOS app
  - menu bar
  - macOS Gateway
  - macOS permissions
  - bundled gateway
  - macOS OpenClaw
related:
  - platforms/ios
  - install/installation-methods
source:
  - platforms/macos.md
  - platforms/mac/menu-bar.md
  - platforms/mac/permissions.md
---

The OpenClaw macOS app provides a menu bar interface for controlling the Gateway and accessing features like Canvas, Peekaboo, and voice overlay.

## macOS App Overview

The macOS app is the **menu‑bar companion** for OpenClaw. It owns permissions,
manages/attaches to the Gateway locally (launchd or manual), and exposes macOS
capabilities to the agent as a node.

## What it does

- Shows native notifications and status in the menu bar.
- Owns TCC prompts (Notifications, Accessibility, Screen Recording, Microphone,
  Speech Recognition, Automation/AppleScript).
- Runs or connects to the Gateway (local or remote).
- Exposes macOS‑only tools (Canvas, Camera, Screen Recording, `system.run`).
- Starts the local node host service in **remote** mode (launchd), and stops it in **local** mode.
- Optionally hosts **PeekabooBridge** for UI automation.
- Installs the global CLI (`openclaw`) on request via npm, pnpm, or bun (the app prefers npm, then pnpm, then bun; Node remains the recommended Gateway runtime).

## Local vs remote mode

- **Local** (default): the app attaches to a running local Gateway if present;
  otherwise it enables the launchd service via `openclaw gateway install`.
- **Remote**: the app connects to a Gateway over SSH/Tailscale and never starts
  a local process.
  The app starts the local **node host service** so the remote Gateway can reach this Mac.
  The app does not spawn the Gateway as a child process.
  Gateway discovery now prefers Tailscale MagicDNS names over raw tailnet IPs,
  so the Mac app recovers more reliably when tailnet IPs change.

## Launchd control

The app manages a per‑user LaunchAgent labeled `ai.openclaw.gateway`
(or `ai.openclaw.<profile>` when using `--profile`/`OPENCLAW_PROFILE`; legacy `com.openclaw.*` still unloads).

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Replace the label with `ai.openclaw.<profile>` when running a named profile.

If the LaunchAgent isn’t installed, enable it from the app or run
`openclaw gateway install`.

## Node capabilities (mac)

The macOS app presents itself as a node. Common commands:

- Canvas: `canvas.present`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`
- Camera: `camera.snap`, `camera.clip`
- Screen: `screen.snapshot`, `screen.record`
- System: `system.run`, `system.notify`

The node reports a `permissions` map so agents can decide what’s allowed.

Node service + app IPC:

- When the headless node host service is running (remote mode), it connects to the Gateway WS as a node.
- `system.run` executes in the macOS app (UI/TCC context) over a local Unix socket; prompts + output stay in-app.

Diagram (SCI):

```
Gateway -> Node Service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac App (UI + TCC + system.run)
```

## Exec approvals (system.run)

`system.run` is controlled by **Exec approvals** in the macOS app (Settings → Exec approvals).
Security + ask + allowlist are stored locally on the Mac in:

```
~/.openclaw/exec-approvals.json
```

Example:

```json
{
  "version": 1,
  "defaults": {
    "security": "deny",
    "ask": "on-miss"
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [{ "pattern": "/opt/homebrew/bin/rg" }]
    }
  }
}
```

Notes:

- `allowlist` entries are glob patterns for resolved binary paths, or bare command names for PATH-invoked commands.
- Raw shell command text that contains shell control or expansion syntax (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) is treated as an allowlist miss and requires explicit approval (or allowlisting the shell binary).
- Choosing “Always Allow” in the prompt adds that command to the allowlist.
- `system.run` environment overrides are filtered (drops `PATH`, `DYLD_*`, `LD_*`, `NODE_OPTIONS`, `PYTHON*`, `PERL*`, `RUBYOPT`, `SHELLOPTS`, `PS4`) and then merged with the app’s environment.
- For shell wrappers (`bash|sh|zsh ... -c/-lc`), request-scoped environment overrides are reduced to a small explicit allowlist (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_

## Menu Bar

# Menu Bar Status Logic

## What is shown

- We surface the current agent work state in the menu bar icon and in the first status row of the menu.
- Health status is hidden while work is active; it returns when all sessions are idle.
- The “Nodes” block in the menu lists **devices** only (paired nodes via `node.list`), not client/presence entries.
- A “Usage” section appears under Context when provider usage snapshots are available.

## State model

- Sessions: events arrive with `runId` (per-run) plus `sessionKey` in the payload. The “main” session is the key `main`; if absent, we fall back to the most recently updated session.
- Priority: main always wins. If main is active, its state is shown immediately. If main is idle, the most recently active non‑main session is shown. We do not flip‑flop mid‑activity; we only switch when the current session goes idle or main becomes active.
- Activity kinds:
  - `job`: high‑level command execution (`state: started|streaming|done|error`).
  - `tool`: `phase: start|result` with `toolName` and `meta/args`.

## IconState enum (Swift)

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)` (debug override)

### ActivityKind → glyph

- `exec` → 💻
- `read` → 📄
- `write` → ✍️
- `edit` → 📝
- `attach` → 📎
- default → 🛠️

### Visual mapping

- `idle`: normal critter.
- `workingMain`: badge with glyph, full tint, leg “working” animation.
- `workingOther`: badge with glyph, muted tint, no scurry.
- `overridden`: uses the chosen glyph/tint regardless of activity.

## Status row text (menu)

- While work is active: `<Session role> · <activity label>`
  - Examples: `Main · exec: pnpm test`, `Other · read: apps/macos/Sources/OpenClaw/AppState.swift`.
- When idle: falls back to the health summary.

## Event ingestion

- Source: control‑channel `agent` events (`ControlChannel.handleAgentEvent`).
- Parsed fields:
  - `stream: "job"` with `data.state` for start/stop.
  - `stream: "tool"` with `data.phase`, `name`, optional `meta`/`args`.
- Labels:
  - `exec`: first line of `args.command`.
  - `read`/`write`: shortened path.
  - `edit`: path plus inferred change kind from `meta`/diff counts.
  - fallback: tool name.

## Debug override

- Settings ▸ Debug ▸ “Icon override” picker:
  - `System (auto)` (default)
  - `Working: main` (per tool kind)
  - `Working: other` (per tool kind)
  - `Idle`
- Stored via `@AppStorage("iconOverride")`; mapped to `IconState.overridden`.

## Testing checklist

- Trigger main session job: verify icon switches immediately and status row shows main label.
- Trigger non‑main session job while main idle: icon/status shows non‑main; stays stable until it finishes.
- Start main while other active: icon flips to main instantly.
- Rapid tool bursts: ensure badge does not flicker (TTL grace on tool results).
- Health row reappears once all sessions idle.

## Related

- [macOS app](/platforms/macos)
- [Menu bar icon](/platforms/mac/icon)

## Permissions

macOS permission grants are fragile. TCC associates a permission grant with the
app's code signature, bundle identifier, and on-disk path. If any of those change,
macOS treats the app as new and may drop or hide prompts.

## Requirements for stable permissions

- Same path: run the app from a fixed location (for OpenClaw, `dist/OpenClaw.app`).
- Same bundle identifier: changing the bundle ID creates a new permission identity.
- Signed app: unsigned or ad-hoc signed builds do not persist permissions.
- Consistent signature: use a real Apple Development or Developer ID certificate
  so the signature stays stable across rebuilds.

Ad-hoc signatures generate a new identity every build. macOS will forget previous
grants, and prompts can disappear entirely until the stale entries are cleared.

## Recovery checklist when prompts disappear

1. Quit the app.
2. Remove the app entry in System Settings -> Privacy & Security.
3. Relaunch the app from the same path and re-grant permissions.
4. If the prompt still does not appear, reset TCC entries with `tccutil` and try again.
5. Some permissions only reappear after a full macOS restart.

Example resets (replace bundle ID as needed):

```bash
sudo tccutil reset Accessibility ai.openclaw.mac
sudo tccutil reset ScreenCapture ai.openclaw.mac
sudo tccutil reset AppleEvents
```

## Files and folders permissions (Desktop/Documents/Downloads)

macOS may also gate Desktop, Documents, and Downloads for terminal/background processes. If file reads or directory listings hang, grant access to the same process context that performs file operations (for example Terminal/iTerm, LaunchAgent-launched app, or SSH process).

Workaround: move files into the OpenClaw workspace (`~/.openclaw/workspace`) if you want to avoid per-folder grants.

If you are testing permissions, always sign with a real certificate. Ad-hoc
builds are only acceptable for quick local runs where permissions do not matter.

## Related

- [macOS app](/platforms/macos)
- [macOS signing](/platforms/mac/signing)

## Bundled Gateway

OpenClaw.app no longer bundles Node/Bun or the Gateway runtime. The macOS app
expects an **external** `openclaw` CLI install, does not spawn the Gateway as a
child process, and manages a per‑user launchd service to keep the Gateway
running (or attaches to an existing local Gateway if one is already running).

## Install the CLI (required for local mode)

Node 24 is the default runtime on the Mac. Node 22 LTS, currently `22.14+`, still works for compatibility. Then install `openclaw` globally:

```bash
npm install -g openclaw@<version>
```

The macOS app’s **Install CLI** button runs the same global install flow the app
uses internally: it prefers npm first, then pnpm, then bun if that is the only
detected package manager. Node remains the recommended Gateway runtime.

## Launchd (Gateway as LaunchAgent)

Label:

- `ai.openclaw.gateway` (or `ai.openclaw.<profile>`; legacy `com.openclaw.*` may remain)

Plist location (per‑user):

- `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
  (or `~/Library/LaunchAgents/ai.openclaw.<profile>.plist`)

Manager:

- The macOS app owns LaunchAgent install/update in Local mode.
- The CLI can also install it: `openclaw gateway install`.

Behavior:

- “OpenClaw Active” enables/disables the LaunchAgent.
- App quit does **not** stop the gateway (launchd keeps it alive).
- If a Gateway is already running on the configured port, the app attaches to
  it instead of starting a new one.

Logging:

- launchd stdout/err: `/tmp/openclaw/openclaw-gateway.log`

## Version compatibility

The macOS app checks the gateway version against its own version. If they’re
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

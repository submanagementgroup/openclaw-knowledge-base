---
domain: tools
topic: "Browser Advanced: Remote CDP, Browserless, Login Persistence, WSL2, and Linux Troubleshooting"
type: reference
keywords:
  - browser remote CDP
  - browserless
  - browser login
  - browser WSL2
  - browser Linux
  - browser profiles
related:
  - tools/browser
source:
  - tools/browser-control.md
  - tools/browser-login.md
  - tools/browser-linux-troubleshooting.md
---

Browser advanced: profiles, remote CDP, browserless, WSL2, Linux troubleshooting, and session management.

## Browser Control API

For setup, configuration, and troubleshooting, see [Browser](/tools/browser).
This page is the reference for the local control HTTP API, the `openclaw browser`
CLI, and scripting patterns (snapshots, refs, waits, debug flows).

## Control API (optional)

For local integrations only, the Gateway exposes a small loopback HTTP API:

- Status/start/stop: `GET /`, `POST /start`, `POST /stop`
- Tabs: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`
- Snapshot/screenshot: `GET /snapshot`, `POST /screenshot`
- Actions: `POST /navigate`, `POST /act`
- Hooks: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- Downloads: `POST /download`, `POST /wait/download`
- Permissions: `POST /permissions/grant`
- Debugging: `GET /console`, `POST /pdf`
- Debugging: `GET /errors`, `GET /requests`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- Network: `POST /response/body`
- State: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- State: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- Settings: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

All endpoints accept `?profile=<name>`. `POST /start?headless=true` requests a
one-shot headless launch for local managed profiles without changing persisted
browser config; attach-only, remote CDP, and existing-session profiles reject
that override because OpenClaw does not launch those browser processes.

If shared-secret gateway auth is configured, browser HTTP routes require auth too:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` or HTTP Basic auth with that password

Notes:

- This standalone loopback browser API does **not** consume trusted-proxy or
  Tailscale Serve identity headers.
- If `gateway.auth.mode` is `none` or `trusted-proxy`, these loopback browser
  routes do not inherit those identity-bearing modes; keep them loopback-only.

### `/act` error contract

`POST /act` uses a structured error response for route-level validation and
policy failures:

```json
{ "error": "<message>", "code": "ACT_*" }
```

Current `code` values:

- `ACT_KIND_REQUIRED` (HTTP 400): `kind` is missing or unrecognized.
- `ACT_INVALID_REQUEST` (HTTP 400): action payload failed normalization or validation.
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): `selector` was used with an unsupported action kind.
- `ACT_EVALUATE_DISABLED` (HTTP 403): `evaluate` (or `wait --fn`) is disabled by config.
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): top-level or batched `targetId` conflicts with request target.
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): action is not supported for existing-session profiles.

Other runtime failures may still return `{ "error": "<message>" }` without a
`code` field.

### Playwright requirement

Some features (navigate/act/AI snapshot/role snapshot, element screenshots,
PDF) require Play

## Browser Login (Session Persistence)

# Browser login + X/Twitter posting

## Manual login (recommended)

When a site requires login, **sign in manually** in the **host** browser profile (the openclaw browser).

Do **not** give the model your credentials. Automated logins often trigger anti‑bot defenses and can lock the account.

Back to the main browser docs: [Browser](/tools/browser).

## Which Chrome profile is used?

OpenClaw controls a **dedicated Chrome profile** (named `openclaw`, orange‑tinted UI). This is separate from your daily browser profile.

For agent browser tool calls:

- Default choice: the agent should use its isolated `openclaw` browser.
- Use `profile="user"` only when existing logged-in sessions matter and the user is at the computer to click/approve any attach prompt.
- If you have multiple user-browser profiles, specify the profile explicitly instead of guessing.

Two easy ways to access it:

1. **Ask the agent to open the browser** and then log in yourself.
2. **Open it via CLI**:

```bash
openclaw browser start
openclaw browser open https://x.com
```

If you have multiple profiles, pass `--browser-profile <name>` (the default is `openclaw`).

## X/Twitter: recommended flow

- **Read/search/threads:** use the **host** browser (manual login).
- **Post updates:** use the **host** browser (manual login).

## Sandboxing + host browser access

Sandboxed browser sessions are **more likely** to trigger bot detection. For X/Twitter (and other strict sites), prefer the **host** browser.

If the agent is sandboxed, the browser tool defaults to the sandbox. To allow host control:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        browser: {
          allowHostControl: true,
        },
      },
    },
  },
}
```

Then target the host browser:

```bash
openclaw browser open https://x.com --browser-profile openclaw --target host
```

Or disable sandboxing for the agent that posts updates.

## Related

- [Browser](/tools/browser)
- [Browser Linux troubleshooting](/tools/browser-linux-troubleshooting)
- [Browser WSL2 troubleshooting](/tools/browser-wsl2-windows-remote-cdp-troubleshooting)

## Linux Troubleshooting

## Problem: "Failed to start Chrome CDP on port 18800"

OpenClaw's browser control server fails to launch Chrome/Brave/Edge/Chromium with the error:

```
{"error":"Error: Failed to start Chrome CDP on port 18800 for profile \"openclaw\"."}
```

### Root cause

On Ubuntu (and many Linux distros), the default Chromium installation is a **snap package**. Snap's AppArmor confinement interferes with how OpenClaw spawns and monitors the browser process.

The `apt install chromium` command installs a stub package that redirects to snap:

```
Note, selecting 'chromium-browser' instead of 'chromium'
chromium-browser is already the newest version (2:1snap1-0ubuntu2).
```

This is NOT a real browser - it's just a wrapper.

Other common Linux launch failures:

- `The profile appears to be in use by another Chromium process` means Chrome
  found stale `Singleton*` lock files in the managed profile directory. OpenClaw
  removes those locks and retries once when the lock points at a dead or
  different-host process.
- `Missing X server or $DISPLAY` means a visible browser was explicitly
  requested on a host without a desktop session. By default, local managed
  profiles now fall back to headless mode on Linux when `DISPLAY` and
  `WAYLAND_DISPLAY` are both unset. If you set `OPENCLAW_BROWSER_HEADLESS=0`,
  `browser.headless: false`, or `browser.profiles.<name>.headless: false`,
  remove that headed override, set `OPENCLAW_BROWSER_HEADLESS=1`, start `Xvfb`,
  run `openclaw browser start --headless` for a one-shot managed launch, or run
  OpenClaw in a real desktop session.

### Solution 1: Install Google Chrome (Recommended)

Install the official Google Chrome `.deb` package, which is not sandboxed by snap:

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # if there are dependency errors
```

Then update your OpenClaw config (`~/.openclaw/openclaw.json`):

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### Solution 2: Use Snap Chromium with Attach-Only Mode

If you must use snap Chromium, configure OpenClaw to attach to a manually-started browser:

1. Update config:

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

2. Start Chromium manually:

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

3. Optionally create a systemd user service to auto-start Chrome:

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw Browser (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/o

## WSL2/Windows Remote CDP

In the common split-host setup, OpenClaw Gateway runs inside WSL2, Chrome runs on Windows, and browser control must cross the WSL2 and Windows boundary. The layered failure pattern from [issue #39369](https://github.com/openclaw/openclaw/issues/39369) means several independent problems can show up at once, which makes the wrong layer look broken first.

## Choose the right browser mode first

You have two valid patterns:

### Option 1: Raw remote CDP from WSL2 to Windows

Use a remote browser profile that points from WSL2 to a Windows Chrome CDP endpoint.

Choose this when:

- the Gateway stays inside WSL2
- Chrome runs on Windows
- you need browser control to cross the WSL2/Windows boundary

### Option 2: Host-local Chrome MCP

Use `existing-session` / `user` only when the Gateway itself runs on the same host as Chrome.

Choose this when:

- OpenClaw and Chrome are on the same machine
- you want the local signed-in browser state
- you do not need cross-host browser transport
- you do not need advanced managed/raw-CDP-only routes like `responsebody`, PDF
  export, download interception, or batch actions

For WSL2 Gateway + Windows Chrome, prefer raw remote CDP. Chrome MCP is host-local, not a WSL2-to-Windows bridge.

## Working architecture

Reference shape:

- WSL2 runs the Gateway on `127.0.0.1:18789`
- Windows opens the Control UI in a normal browser at `http://127.0.0.1:18789/`
- Windows Chrome exposes a CDP endpoint on port `9222`
- WSL2 can reach that Windows CDP endpoint
- OpenClaw points a browser profile at the address that is reachable from WSL2

## Why this setup is confusing

Several failures can overlap:

- WSL2 cannot reach the Windows CDP endpoint
- the Control UI is opened from a non-secure origin
- `gateway.controlUi.allowedOrigins` does not match the page origin
- token or pairing is missing
- the browser profile points at the wrong address

Because of that, fixing one layer can still leave a different error visible.

## Critical rule for the Contr

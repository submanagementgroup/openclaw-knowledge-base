---
domain: tools
topic: "Browser Tool Troubleshooting"
type: troubleshooting
keywords:
  - browser troubleshooting
  - browser Linux
  - browser WSL2
  - CDP errors
  - browser issues
source: 
  - tools/browser-linux-troubleshooting.md
  - tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
---

## Linux Browser Troubleshooting

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
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Enable with: `systemctl --user enable --now openclaw-browser.service`

### Verifying the Browser Works

Check status:

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
```

Test browsing:

```bash
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### Config reference

| Option                           | Description                                                          | Default                                                     |
| -------------------------------- | -------------------------------------------------------------------- | ----------------------------------------------------------- |
| `browser.enabled`                | Enable browser control                                               | `true`                                                      |
| `browser.executablePath`         | Path to a Chromium-based browser binary (Chrome/Brave/Edge/Chromium) | auto-detected (prefers default browser when Chromium-based) |
| `browser.headless`               | Run without GUI                                                      | `false`                                                     |
| `OPENCLAW_BROWSER_HEADLESS`      | Per-process override for local managed browser headless mode         | unset                                                       |
| `browser.noSandbox`              | Add `--no-sandbox` flag (needed for some Linux setups)               | `false`                                                     |
| `browser.attachOnly`             | Don't launch browser, only attach to existing                        | `false`                                                     |
| `browser.cdpPort`                | Chrome DevTools Protocol port                                        | `18800`                                                     |
| `browser.localLaunchTimeoutMs`   | Local managed Chrome discovery timeout                               | `15000`                                                     |
| `browser.localCdpReadyTimeoutMs` | Local managed post-launch CDP readiness timeout                      | `8000`                                                      |

On Raspberry Pi, older VPS hosts, or slow storage, raise
`browser.localLaunchTimeoutMs` when Chrome needs more time to expose its CDP HTTP
endpoint. Raise `browser.localCdpReadyTimeoutMs` when launch succeeds but
`openclaw browser start` still reports `not reachable after start`. Values must
be positive integers up to `120000` ms; invalid config values are rejected.

### Problem: "No Chrome tabs found for profile=\"user\""

You're using an `existing-session` / Chrome MCP profile. OpenClaw can see local Chrome,
but there are no open tabs available to attach to.

Fix options:

1. **Use the managed browser:** `openclaw browser start --browser-profile openclaw`
   (or set `browser.defaultProfile: "openclaw"`).
2. **Use Chrome MCP:** make sure local Chrome is running with at least one open tab, then retry with `--browser-profile user`.

Notes:

- `user` is host-only. For Linux servers, containers, or remote hosts, prefer CDP profiles.
- `user` / other `existing-session` profiles keep the current Chrome MCP limits:
  ref-driven actions, one-file upload hooks, no dialog timeout overrides, no
  `wait --load networkidle`, and no `responsebody`, PDF export, download
  interception, or batch actions.
- Local `openclaw` profiles auto-assign `cdpPort`/`cdpUrl`; only set those for remote CDP.
- Remote CDP profiles accept `http://`, `https://`, `ws://`, and `wss://`.
  Use HTTP(S) for `/json/version` discovery, or WS(S) when your browser
  service gives you a direct DevTools socket URL.

## Related

- [Browser](/tools/browser)
- [Browser login](/tools/browser-login)
- [Browser WSL2 troubleshooting](/tools/browser-wsl2-windows-remote-cdp-troubleshooting)


## WSL2/Windows Browser Troubleshooting

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

## Critical rule for the Control UI

When the UI is opened from Windows, use Windows localhost unless you have a deliberate HTTPS setup.

Use:

`http://127.0.0.1:18789/`

Do not default to a LAN IP for the Control UI. Plain HTTP on a LAN or tailnet address can trigger insecure-origin/device-auth behavior that is unrelated to CDP itself. See [Control UI](/web/control-ui).

## Validate in layers

Work top to bottom. Do not skip ahead.

### Layer 1: Verify Chrome is serving CDP on Windows

Start Chrome on Windows with remote debugging enabled:

```powershell
chrome.exe --remote-debugging-port=9222
```

From Windows, verify Chrome itself first:

```powershell
curl http://127.0.0.1:9222/json/version
curl http://127.0.0.1:9222/json/list
```

If this fails on Windows, OpenClaw is not the problem yet.

### Layer 2: Verify WSL2 can reach that Windows endpoint

From WSL2, test the exact address you plan to use in `cdpUrl`:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

Good result:

- `/json/version` returns JSON with Browser / Protocol-Version metadata
- `/json/list` returns JSON (empty array is fine if no pages are open)

If this fails:

- Windows is not exposing the port to WSL2 yet
- the address is wrong for the WSL2 side
- firewall / port forwarding / local proxying is still missing

Fix that before touching OpenClaw config.

### Layer 3: Configure the correct browser profile

For raw remote CDP, point OpenClaw at the address that is reachable from WSL2:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

Notes:

- use the WSL2-reachable address, not whatever only works on Windows
- keep `attachOnly: true` for externally managed browsers
- `cdpUrl` can be `http://`, `https://`, `ws://`, or `wss://`
- use HTTP(S) when you want OpenClaw to discover `/json/version`
- use WS(S) only when the browser provider gives you a direct DevTools socket URL
- test the same URL with `curl` before expecting OpenClaw to succeed

### Layer 4: Verify the Control UI layer separately

Open the UI from Windows:

`http://127.0.0.1:18789/`

Then verify:

- the page origin matches what `gateway.controlUi.allowedOrigins` expects
- token auth or pairing is configured correctly
- you are not debugging a Control UI auth problem as if it were a browser problem

Helpful page:

- [Control UI](/web/control-ui)

### Layer 5: Verify end-to-end browser control

From WSL2:

```bash
openclaw browser open https://example.com --browser-profile remote
openclaw browser tabs --browser-profile remote
```

Good result:

- the tab opens in Windows Chrome
- `openclaw browser tabs` returns the target
- later actions (`snapshot`, `screenshot`, `navigate`) work from the same profile

## Common misleading errors

Treat each message as a layer-specific clue:

- `control-ui-insecure-auth`
  - UI origin / secure-context problem, not a CDP transport problem
- `token_missing`
  - auth configuration problem
- `pairing required`
  - device approval problem
- `Remote CDP for profile "remote" is not reachable`
  - WSL2 cannot reach the configured `cdpUrl`
- `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable`
  - the HTTP endpoint answered, but the DevTools WebSocket still could not be opened
- stale viewport / dark-mode / locale / offline overrides after a remote session
  - run `openclaw browser stop --browser-profile remote`
  - this closes the active control session and releases Playwright/CDP emulation state without restarting the gateway or the external browser
- `gateway timeout after 1500ms`
  - often still CDP reachability or a slow/unreachable remote endpoint
- `No Chrome tabs found for profile="user"`
  - local Chrome MCP profile selected where no host-local tabs are available

## Fast triage checklist

1. Windows: does `curl http://127.0.0.1:9222/json/version` work?
2. WSL2: does `curl http://WINDOWS_HOST_OR_IP:9222/json/version` work?
3. OpenClaw config: does `browser.profiles.<name>.cdpUrl` use that exact WSL2-reachable address?
4. Control UI: are you opening `http://127.0.0.1:18789/` instead of a LAN IP?
5. Are you trying to use `existing-session` across WSL2 and Windows instead of raw remote CDP?

## Practical takeaway

The setup is usually viable. The hard part is that browser transport, Control UI origin security, and token/pairing can each fail independently while looking similar from the user side.

When in doubt:

- verify the Windows Chrome endpoint locally first
- verify the same endpoint from WSL2 second
- only then debug OpenClaw config or Control UI auth

## Related

- [Browser](/tools/browser)
- [Browser login](/tools/browser-login)
- [Browser Linux troubleshooting](/tools/browser-linux-troubleshooting)

---
domain: troubleshooting
topic: "WSL2 + Windows + Remote Chrome CDP Troubleshooting: Layered Debugging Guide"
type: troubleshooting
keywords:
  - WSL2 browser
  - Windows Chrome CDP
  - remote CDP
  - control-ui-insecure-auth
  - cdpUrl WSL2
  - attachOnly browser
  - CDP reachability
  - WSL2 gateway browser
  - Windows Chrome port 9222
  - token_missing browser
  - browser profile remote
source: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
related:
  - tools/browser
  - troubleshooting/general-troubleshooting
---

In the common split-host setup, the OpenClaw Gateway runs inside WSL2, Chrome runs on Windows, and browser control must cross the WSL2/Windows boundary. Multiple independent problems can show up at once, making the wrong layer look broken first. Validate in layers — do not skip ahead.

## Choose the Right Browser Mode First

### Option 1: Raw Remote CDP from WSL2 to Windows (Recommended for WSL2 + Windows)

Use a remote browser profile pointing from WSL2 to a Windows Chrome CDP endpoint.

Choose this when:
- The Gateway stays inside WSL2
- Chrome runs on Windows
- You need browser control to cross the WSL2/Windows boundary

### Option 2: Host-Local Chrome MCP

Use `existing-session` / `user` only when the Gateway itself runs on the same host as Chrome.

Choose this when:
- OpenClaw and Chrome are on the same machine
- You want the local signed-in browser state
- You do not need cross-host browser transport

**Do not use Chrome MCP to bridge WSL2 and Windows.** Chrome MCP is host-local, not a WSL2-to-Windows bridge.

## Working Architecture

```
WSL2:    Gateway on 127.0.0.1:18789
Windows: Control UI at http://127.0.0.1:18789/
Windows: Chrome with CDP on port 9222
WSL2:    Reaches Windows CDP endpoint
OpenClaw: browser profile points at WSL2-reachable CDP address
```

## Critical Rule for the Control UI

When the UI is opened from Windows, use Windows localhost:

```
http://127.0.0.1:18789/
```

Do **not** default to a LAN IP for the Control UI. Plain HTTP on a LAN or tailnet address can trigger insecure-origin/device-auth behavior unrelated to CDP itself.

## Validate in Layers

Work top to bottom. Do not skip.

### Layer 1: Verify Chrome is Serving CDP on Windows

Start Chrome on Windows with remote debugging:

```powershell
chrome.exe --remote-debugging-port=9222
```

From Windows, verify Chrome itself first:

```powershell
curl http://127.0.0.1:9222/json/version
curl http://127.0.0.1:9222/json/list
```

If this fails on Windows, OpenClaw is not the problem yet.

### Layer 2: Verify WSL2 Can Reach the Windows CDP Endpoint

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

Good result: `/json/version` returns JSON with Browser/Protocol-Version metadata; `/json/list` returns JSON (empty array is fine if no pages are open).

If this fails:
- Windows is not exposing the port to WSL2
- The address is wrong for the WSL2 side
- Firewall / port forwarding / local proxying is still missing

Fix that before touching OpenClaw config.

### Layer 3: Configure the Correct Browser Profile

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
- Use the WSL2-reachable address, not whatever only works on Windows
- Keep `attachOnly: true` for externally managed browsers
- `cdpUrl` can be `http://`, `https://`, `ws://`, or `wss://`
- Use HTTP(S) when you want OpenClaw to discover `/json/version`; use WS(S) only when the browser provider gives you a direct DevTools socket URL
- Test the same URL with `curl` before expecting OpenClaw to succeed

### Layer 4: Verify the Control UI Layer Separately

Open the UI from Windows: `http://127.0.0.1:18789/`

Verify:
- The page origin matches what `gateway.controlUi.allowedOrigins` expects
- Token auth or pairing is configured correctly
- You are not debugging a Control UI auth problem as if it were a browser problem

### Layer 5: Verify End-to-End Browser Control

```bash
openclaw browser open https://example.com --browser-profile remote
openclaw browser tabs --browser-profile remote
```

Good result: the tab opens in Windows Chrome and `openclaw browser tabs` returns the target.

## Common Misleading Errors

| Error message | Layer | Cause |
|---------------|-------|-------|
| `control-ui-insecure-auth` | Layer 4 | UI origin / secure-context problem, not a CDP transport problem |
| `token_missing` | Layer 4 | Auth configuration problem |
| `pairing required` | Layer 4 | Device approval problem |
| `Remote CDP for profile "remote" is not reachable` | Layer 2/3 | WSL2 cannot reach the configured `cdpUrl` |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable` | Layer 3 | HTTP endpoint answered, but DevTools WebSocket couldn't open |
| Stale viewport/dark-mode/locale after remote session | Layer 5 | Run `openclaw browser stop --browser-profile remote` to release Playwright/CDP state |
| `gateway timeout after 1500ms` | Layer 2/3 | CDP reachability or slow/unreachable remote endpoint |
| `No Chrome tabs found for profile="user"` | — | Local Chrome MCP profile selected where no host-local tabs are available |

## Fast Triage Checklist

1. Windows: does `curl http://127.0.0.1:9222/json/version` work?
2. WSL2: does `curl http://WINDOWS_HOST_OR_IP:9222/json/version` work?
3. OpenClaw config: does `browser.profiles.<name>.cdpUrl` use that exact WSL2-reachable address?
4. Control UI: are you opening `http://127.0.0.1:18789/` instead of a LAN IP?
5. Are you trying to use `existing-session` across WSL2 and Windows instead of raw remote CDP?

## Practical Takeaway

The setup is usually viable. The hard part is that browser transport, Control UI origin security, and token/pairing can each fail independently while looking similar. Verify the Windows Chrome endpoint locally first, then from WSL2, then debug OpenClaw config and Control UI auth.

## Related

- [Browser tool](/tools/browser)
- [General troubleshooting](/troubleshooting/general-troubleshooting)

---
domain: tools
topic: "Browser Tool: Advanced Configuration and Troubleshooting"
type: reference
keywords:
  - browser
  - browser config
  - browser CDP
  - browser remote
  - browser Linux
source: tools/browser.md
---

executablePath` follows the same local managed profile rule. Changing it on a
  running local managed profile marks that profile for restart/reconcile so the
  next launch uses the new binary.

Stopping behavior differs by profile mode:

- local managed profiles: `openclaw browser stop` stops the browser process that
  OpenClaw launched
- attach-only and remote CDP profiles: `openclaw browser stop` closes the active
  control session and releases Playwright/CDP emulation overrides (viewport,
  color scheme, locale, timezone, offline mode, and similar state), even
  though no browser process was launched by OpenClaw

Remote CDP URLs can include auth:

- Query tokens (e.g., `https://provider.example?token=<token>`)
- HTTP Basic auth (e.g., `https://user:pass@provider.example`)

OpenClaw preserves the auth when calling `/json/*` endpoints and when connecting
to the CDP WebSocket. Prefer environment variables or secrets managers for
tokens instead of committing them to config files.

## Node browser proxy (zero-config default)

If you run a **node host** on the machine that has your browser, OpenClaw can
auto-route browser tool calls to that node without any extra browser config.
This is the default path for remote gateways.

Notes:

- The node host exposes its local browser control server via a **proxy command**.
- Profiles come from the node's own `browser.profiles` config (same as local).
- `nodeHost.browserProxy.allowProfiles` is optional. Leave it empty for the legacy/default behavior: all configured profiles remain reachable through the proxy, including profile create/delete routes.
- If you set `nodeHost.browserProxy.allowProfiles`, OpenClaw treats it as a least-privilege boundary: only allowlisted profiles can be targeted, and persistent profile create/delete routes are blocked on the proxy surface.
- Disable if you don't want it:
  - On the node: `nodeHost.browserProxy.enabled=false`
  - On the gateway: `gateway.nodes.browser.mode="off"`

## Browserless (hosted remote CDP)

[Browserless](https://browserless.io) is a hosted Chromium service that exposes
CDP connection URLs over HTTPS and WebSocket. OpenClaw can use either form, but
for a remote browser profile the simplest option is the direct WebSocket URL
from Browserless' connection docs.

Example:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=",
        color: "#00AA00",
      },
    },
  },
}
```

Notes:

- Replace `` with your real Browserless token.
- Choose the region endpoint that matches your Browserless account (see their docs).
- If Browserless gives you an HTTPS base URL, you can either convert it to
  `wss://` for a direct CDP connection or keep the HTTPS URL and let OpenClaw
  discover `/json/version`.

### Browserless Docker on the same host

When Browserless is self-hosted in Docker and OpenClaw runs on the host, treat
Browserless as an externally managed CDP service:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "ws://127.0.0.1:3000",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

The address in `browser.profiles.browserless.cdpUrl` must be reachable from the
OpenClaw process. Browserless must also advertise a matching reachable endpoint;
set Browserless `EXTERNAL` to that same public-to-OpenClaw WebSocket base, such
as `ws://127.0.0.1:3000`, `ws://browserless:3000`, or a stable private Docker
network address. If `/json/version` returns `webSocketDebuggerUrl` pointing at
an address OpenClaw cannot reach, CDP HTTP can look healthy while the WebSocket
attach still fails.

Do not leave `attachOnly` unset for a loopback Browserless profile. Without
`attachOnly`, OpenClaw treats the loopback port as a local managed browser
profile and may report that the port is in use but not owned by OpenClaw.

## Direct WebSocket CDP providers

Some hosted browser services expose a **direct WebSocket** endpoint rather than
the standard HTTP-based CDP discovery (`/json/version`). OpenClaw accepts three
CDP URL shapes and picks the right connection strategy automatically:

- **HTTP(S) discovery** - `http://host[:port]` or `https://host[:port]`.
  OpenClaw calls `/json/version` to discover the WebSocket debugger URL, then
  connects. No WebSocket fallback.
- **Direct WebSocket endpoints** - `ws://host[:port]/devtools/<kind>/<id>` or
  `wss://...` with a `/devtools/browser|page|worker|shared_worker|service_worker/<id>`
  path. OpenClaw connects directly via a WebSocket handshake and skips
  `/json/version` entirely.
- **Bare WebSocket roots** - `ws://host[:port]` or `wss://host[:port]` with no
  `/devtools/...` path (e.g. [Browserless](https://browserless.io),
  [Browserbase](https://www.browserbase.com)). OpenClaw tries HTTP
  `/json/version` discovery first (normalising the scheme to `http`/`https`);
  if discovery returns a `webSocketDebuggerUrl` it is used, otherwise OpenClaw
  falls back to a direct WebSocket handshake at the bare root. If the advertised
  WebSocket endpoint rejects the CDP handshake but the configured bare root
  accepts it, OpenClaw falls back to that root as well. This lets a bare `ws://`
  pointed at a local Chrome still connect, since Chrome only accepts WebSocket
  upgrades on the specific per-target path from `/json/version`, while hosted
  providers can still use their root WebSocket endpoint when their discovery
  endpoint advertises a short-lived URL that is not suitable for Playwright CDP.

### Browserbase

[Browserbase](https://www.browserbase.com) is a cloud platform for running
headless browsers with built-in CAPTCHA solving, stealth mode, and residential
proxies.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    remoteCdpTimeoutMs: 3000,
    remoteCdpHandshakeTimeoutMs: 5000,
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=",
        color: "#F97316",
      },
    },
  },
}
```

Notes:

- [Sign up](https://www.browserbase.com/sign-up) and copy your **API Key**
  from the [Overview dashboard](https://www.browserbase.com/overview).
- Replace `` with your real Browserbase API key.
- Browserbase auto-creates a browser session on WebSocket connect, so no
  manual session creation step is needed.
- The free tier allows one concurrent session and one browser hour per month.
  See [pricing](https://www.browserbase.com/pricing) for paid plan limits.
- See the [Browserbase docs](https://docs.browserbase.com) for full API
  reference, SDK guides, and integration examples.

## Security

Key ideas:

- Browser control is loopback-only; access flows through the Gateway's auth or node pairing.
- The standalone loopback browser HTTP API uses **shared-secret auth only**:
  gateway token bearer auth, `x-openclaw-password`, or HTTP Basic auth with the
  configured gateway password.
- Tailscale Serve identity headers and `gateway.auth.mode: "trusted-proxy"` do
  **not** authenticate this standalone loopback browser API.
- If browser control is enabled and no shared-secret auth is configured, OpenClaw
  generates a runtime-only gateway token for that startup. Configure
  `gateway.auth.token`, `gateway.auth.password`, `OPENCLAW_GATEWAY_TOKEN`, or
  `OPENCLAW_GATEWAY_PASSWORD` explicitly if clients need a stable secret across
  restarts.
- OpenClaw does **not** auto-generate that token when `gateway.auth.mode` is
  already `password`, `none`, or `trusted-proxy`.
- Keep the Gateway and any node hosts on a private network (Tailscale); avoid public exposure.
- Treat remote CDP URLs/tokens as secrets; prefer env vars or a secrets manager.

Remote CDP tips:

- Prefer encrypted endpoints (HTTPS or WSS) and short-lived tokens where possible.
- Avoid embedding long-lived tokens directly in config files.

## Profiles (multi-browser)

OpenClaw supports multiple named profiles (routing configs). Profiles can be:

- **openclaw-managed**: a dedicated Chromium-based browser instance with its own user data directory + CDP port
- **remote**: an explicit CDP URL (Chromium-based browser running elsewhere)
- **existing session**: your existing Chrome profile via Chrome DevTools MCP auto-connect

Defaults:

- The `openclaw` profile is auto-created if missing.
- The `user` profile is built-in for Chrome MCP existing-session attach.
- Existing-session profiles are opt-in beyond `user`; create them with `--driver existing-session`.
- Local CDP ports allocate from **18800-18899** by default.
- Deleting a profile moves its local data directory to Trash.

All control endpoints accept `?profile=<name>`; the CLI uses `--browser-profile`.

## Existing session via Chrome DevTools MCP

OpenClaw can also attach to a running Chromium-based browser profile through the
official Chrome DevTools MCP server. This reuses the tabs and login state
already open in that browser profile.

Official background and setup references:

- [Chrome for Developers: Use Chrome DevTools MCP with your browser session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

Built-in profile:

- `user`

Optional: create your own custom existing-session profile if you want a
different name, color, or browser data directory.

Default behavior:

- The built-in `user` profile uses Chrome MCP auto-connect, which targets the
  default local Google Chrome profile.

Use `userDataDir` for Brave, Edge, Chromium, or a non-default Chrome profile.
`~` expands to your OS home directory:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

Then in the matching browser:

1. Open that browser's inspect page for remote debugging.
2. Enable remote debugging.
3. Keep the browser running and approve the connection prompt when OpenClaw attaches.

Common inspect pages:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

Live attach smoke test:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

What success looks like:

- `status` shows `driver: existing-session`
- `status` shows `transport: chrome-mcp`
- `status` shows `running: true`
- `tabs` lists your already-open browser tabs
- `snapshot` returns refs from the selected live tab

What to check if attach does not work:

- the target Chromium-based browser is version `144+`
- remote debugging is enabled in that browser's inspect page
- the browser showed and you accepted the attach consent prompt
- `openclaw doctor` migrates old extension-based browser config and checks that
  Chrome is installed locally for default auto-connect profiles, but it cannot
  enable browser-side remote debugging for you

Agent use:

- Use `profile="user"` when you need the user's logged-in browser state.
- If you use a custom existing-session profile, pass that explicit profile name.
- Only choose this mode when the user is at the computer to approve the attach
  prompt.
- the Gateway or node host can spawn `npx chrome-devtools-mcp@latest --autoConnect`

Notes:

- This path is higher-risk than the isolated `openclaw` profile because it can
  act inside your signed-in browser session.
- OpenClaw does not launch the browser for this driver; it only attaches.
- OpenClaw uses the official Chrome DevTools MCP `--autoConnect` flow here. If
  `userDataDir` is set, it is passed through to target that user data directory.
- Existing-session can attach on the selected host or through a connected
  browser node. If Chrome lives elsewhere and no browser node is connected, use
  remote CDP or a node host instead.

### Custom Chrome MCP launch

Override the spawned Chrome DevTools MCP server per profile when the default
`npx chrome-devtools-mcp@latest` flow is not what you want (offline hosts,
pinned versions, vendored binaries):

| Field        | What it does                                                                                                               |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `mcpCommand` | Executable to spawn instead of `npx`. Resolved as-is; absolute paths are honored.                                          |
| `mcpArgs`    | Argument array passed verbatim to `mcpCommand`. Replaces the default `chrome-devtools-mcp@latest --autoConnect` arguments. |

When `cdpUrl` is set on an existing-session profile, OpenClaw skips
`--autoConnect` and forwards the endpoint to Chrome MCP automatically:

- `http(s)://...` → `--browserUrl <url>` (DevTools HTTP discovery endpoint).
- `ws(s)://...` → `--wsEndpoint <url>` (direct CDP WebSocket).

Endpoint flags and `userDataDir` cannot be combined: when `cdpUrl` is set,
`userDataDir` is ignored for Chrome MCP launch, since Chrome MCP attaches to
the running browser behind the endpoint rather than opening a profile
directory.

### Existing-session feature limitations

Compared to the managed `openclaw` profile, existing-session drivers are more constrained:

- **Screenshots** - page captures and `--ref` element captures work; CSS `--element` selectors do not. `--full-page` cannot combine with `--ref` or `--element`. Playwright is not required for page or ref-based element screenshots.
- **Actions** - `click`, `type`, `hover`, `scrollIntoView`, `drag`, and `select` require snapshot refs (no CSS selectors). `click-coords` clicks vis
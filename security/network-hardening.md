---
domain: security
topic: "Gateway Network Hardening: Bind Modes, Reverse Proxy, Auth, Tailscale, and Docker Firewall"
type: reference
keywords:
  - gateway bind
  - reverse proxy
  - trustedProxies
  - X-Forwarded-For
  - HSTS
  - Tailscale Serve
  - Docker UFW
  - gateway auth
  - loopback
  - token auth
  - password auth
  - trusted-proxy auth
  - mDNS Bonjour
source: gateway/security/index.md
related:
  - security/security-model
  - security/configuration-hardening
  - gateway/trusted-proxy-auth
  - gateway/tailscale
  - gateway/network-model
---

Gateway network hardening covers bind mode, reverse proxy configuration, auth token rotation, Tailscale Serve identity, Docker firewall rules, mDNS discovery, and HSTS setup.

## Network Exposure (Bind, Port, Firewall)

The Gateway multiplexes **WebSocket + HTTP** on a single port (default `18789`). This HTTP surface includes the Control UI, canvas host (`/__openclaw__/canvas/`, `/__openclaw__/a2ui/`), and HTTP APIs.

Bind mode controls where the Gateway listens:

- `gateway.bind: "loopback"` (default): only local clients can connect.
- Non-loopback binds (`"lan"`, `"tailnet"`, `"custom"`) expand the attack surface. Use only with gateway auth and a real firewall.

Rules of thumb:
- Prefer Tailscale Serve over LAN binds (Serve keeps the Gateway on loopback; Tailscale handles access).
- If you must bind to LAN, firewall the port to a tight allowlist of source IPs.
- Never expose the Gateway unauthenticated on `0.0.0.0`.

## Reverse Proxy Configuration

If you run the Gateway behind a reverse proxy (nginx, Caddy, Traefik), configure `gateway.trustedProxies` for proper forwarded-client IP handling.

When the Gateway detects proxy headers from an address **not** in `trustedProxies`, it will not treat connections as local clients. If gateway auth is disabled, those connections are rejected — preventing authentication bypass where proxied connections would otherwise appear to come from localhost.

`gateway.trustedProxies` also feeds `gateway.auth.mode: "trusted-proxy"`, but that auth mode is stricter:
- trusted-proxy auth **fails closed on loopback-source proxies by default**
- same-host loopback reverse proxies can satisfy trusted-proxy auth only when `gateway.auth.trustedProxy.allowLoopback = true`; otherwise use token/password auth

```yaml
gateway:
  trustedProxies:
    - "10.0.0.1" # reverse proxy IP
  allowRealIpFallback: false  # Default false; only enable when proxy cannot provide X-Forwarded-For
  auth:
    mode: password
    password: ${OPENCLAW_GATEWAY_PASSWORD}
```

When `trustedProxies` is configured, the Gateway uses `X-Forwarded-For` to determine client IP. `X-Real-IP` is ignored unless `gateway.allowRealIpFallback: true` is explicitly set.

Good reverse proxy behavior (overwrite incoming forwarding headers):
```nginx
proxy_set_header X-Forwarded-For $remote_addr;
proxy_set_header X-Real-IP $remote_addr;
```

Bad reverse proxy behavior (append/preserve untrusted forwarding headers):
```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

## Gateway WebSocket Auth (Lock Down)

Gateway auth is **required by default**. If no valid gateway auth path is configured, the Gateway refuses WebSocket connections (fail-closed). Onboarding generates a token by default.

Set a token so all WS clients must authenticate:

```json5
{
  gateway: {
    auth: { mode: "token", token: "your-token" },
  },
}
```

Generate a token: `openclaw doctor --generate-gateway-token`

Auth modes:
- `gateway.auth.mode: "token"` — shared bearer token (recommended for most setups).
- `gateway.auth.mode: "password"` — password auth (prefer setting via env: `OPENCLAW_GATEWAY_PASSWORD`).
- `gateway.auth.mode: "trusted-proxy"` — trust an identity-aware reverse proxy to authenticate users and pass identity via headers.

**Token rotation checklist:**
1. Generate/set a new secret (`gateway.auth.token` or `OPENCLAW_GATEWAY_PASSWORD`).
2. Restart the Gateway (or restart the macOS app if it supervises the Gateway).
3. Update any remote clients (`gateway.remote.token` / `.password` on machines that call into the Gateway).
4. Verify you can no longer connect with the old credentials.

## Tailscale Serve Identity Headers

When `gateway.auth.allowTailscale` is `true` (default for Serve), OpenClaw accepts Tailscale Serve identity headers (`tailscale-user-login`) for Control UI/WebSocket authentication. OpenClaw verifies the identity by resolving the `x-forwarded-for` address through the local Tailscale daemon (`tailscale whois`) and matching it to the header.

**Important boundary notes:**
- Gateway HTTP bearer auth is effectively all-or-nothing operator access. Treat credentials that can call `/v1/chat/completions`, `/v1/responses`, or `/api/channels/*` as full-access operator secrets.
- HTTP API endpoints (`/v1/*`, `/tools/invoke`, `/api/channels/*`) do **not** use Tailscale identity-header auth.
- **Security rule:** do not forward these headers from your own reverse proxy. If you terminate TLS in front of the gateway, disable `gateway.auth.allowTailscale` and use shared-secret auth or Trusted Proxy Auth instead.

**Trust assumption:** tokenless Serve auth assumes the gateway host is trusted. Do not treat this as protection against hostile same-host processes.

## Docker Port Publishing with UFW

If running OpenClaw with Docker on a VPS, published container ports (`-p HOST:CONTAINER`) are routed through Docker's forwarding chains, not only host `INPUT` rules. Enforce rules in `DOCKER-USER`:

```bash
# /etc/ufw/after.rules (append as its own *filter section)
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
-A DOCKER-USER -s 127.0.0.0/8 -j RETURN
-A DOCKER-USER -s 10.0.0.0/8 -j RETURN
-A DOCKER-USER -s 172.16.0.0/12 -j RETURN
-A DOCKER-USER -s 192.168.0.0/16 -j RETURN
-A DOCKER-USER -s 100.64.0.0/10 -j RETURN
-A DOCKER-USER -p tcp --dport 80 -j RETURN
-A DOCKER-USER -p tcp --dport 443 -j RETURN
-A DOCKER-USER -m conntrack --ctstate NEW -j DROP
-A DOCKER-USER -j RETURN
COMMIT
```

Add a matching policy in `/etc/ufw/after6.rules` if Docker IPv6 is enabled. Avoid hardcoding interface names like `eth0`.

Quick validation after reload:
```bash
ufw reload
iptables -S DOCKER-USER
ip6tables -S DOCKER-USER
nmap -sT -p 1-65535 <public-ip> --open
```

## mDNS/Bonjour Discovery

When the bundled `bonjour` plugin is enabled, the Gateway broadcasts presence via mDNS (`_openclaw-gw._tcp` on port 5353) for local device discovery. In full mode, this includes `cliPath` (reveals username and install location) and `sshPort`.

Recommendations:
1. **Keep Bonjour disabled unless LAN discovery is needed.**
2. **Minimal mode** (recommended for exposed gateways): omit sensitive fields from mDNS broadcasts:
   ```json5
   { discovery: { mdns: { mode: "minimal" } } }
   ```
3. **Disable mDNS mode** if you want to keep the plugin but suppress local discovery:
   ```json5
   { discovery: { mdns: { mode: "off" } } }
   ```
4. **Environment variable:** `OPENCLAW_DISABLE_BONJOUR=1` to disable without config changes.

## HSTS and Origin Notes

- OpenClaw gateway is local/loopback first. If you terminate TLS at a reverse proxy, set HSTS on the proxy-facing HTTPS domain there.
- If the gateway itself terminates HTTPS, set `gateway.http.securityHeaders.strictTransportSecurity` to emit the HSTS header.
- For non-loopback Control UI deployments, `gateway.controlUi.allowedOrigins` is required by default.
- `gateway.controlUi.allowedOrigins: ["*"]` is an explicit allow-all browser-origin policy. Avoid it outside tightly controlled local testing.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` enables Host-header origin fallback mode; treat as a dangerous operator-selected policy.

## Control UI Over HTTP

The Control UI needs a **secure context** (HTTPS or localhost) to generate device identity. `gateway.controlUi.allowInsecureAuth` is a local compatibility toggle:
- On localhost, it allows Control UI auth without device identity when the page is loaded over non-secure HTTP.
- It does not bypass pairing checks.
- It does not relax remote (non-localhost) device identity requirements.

Prefer HTTPS (Tailscale Serve) or open the UI on `127.0.0.1`. For break-glass only, `gateway.controlUi.dangerouslyDisableDeviceAuth` disables device identity checks entirely — a severe security downgrade.

## Session Logs on Disk

OpenClaw stores session transcripts under `~/.openclaw/agents/<agentId>/sessions/*.jsonl`. Any process/user with filesystem access can read those logs. Treat disk access as the trust boundary:
- Keep permissions tight (`700` on dirs, `600` on files).
- Use full-disk encryption on the gateway host.
- If you need stronger isolation between agents, run them under separate OS users or separate hosts.

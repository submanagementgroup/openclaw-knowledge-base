---
domain: security
topic: "Network Proxy Security: nginx, Caddy, TLS Termination, and Reverse Proxy Config"
type: procedure
keywords:
  - network proxy
  - nginx
  - Caddy
  - TLS
  - reverse proxy
  - HTTPS
  - security proxy
related:
  - gateway/authentication
  - gateway/security-overview
source: security/network-proxy.md
---

Network proxy configuration for secure OpenClaw deployments: nginx, Caddy, and TLS termination.

# Network Proxy

OpenClaw can route runtime HTTP and WebSocket traffic through an operator-managed forward proxy. This is optional defense in depth for deployments that want central egress control, stronger SSRF protection, and better network auditability.

OpenClaw does not ship, download, start, configure, or certify a proxy. You run the proxy technology that fits your environment, and OpenClaw routes normal process-local HTTP and WebSocket clients through it.

## Why Use a Proxy?

A proxy gives operators one network control point for outbound HTTP and WebSocket traffic. That can be useful even outside SSRF hardening:

- Central policy: maintain one egress policy instead of relying on every application HTTP call site to get network rules right.
- Connect-time checks: evaluate the destination after DNS resolution and immediately before the proxy opens the upstream connection.
- DNS rebinding defense: reduce the gap between an application-level DNS check and the actual outbound connection.
- Broader JavaScript coverage: route ordinary `fetch`, `node:http`, `node:https`, WebSocket, axios, got, node-fetch, and similar clients through the same path.
- Auditability: log allowed and denied destinations at the egress boundary.
- Operational control: enforce destination rules, network segmentation, rate limits, or outbound allowlists without rebuilding OpenClaw.

Proxy routing is a process-level guardrail for normal HTTP and WebSocket egress. It gives operators a fail-closed path for routing supported JavaScript HTTP clients through their own filtering proxy, but it is not an OS-level network sandbox and does not make OpenClaw certify the proxy's destination policy.

## How OpenClaw Routes Traffic

When `proxy.enabled=true` and a proxy URL is configured, protected runtime processes such as `openclaw gateway run`, `openclaw node run`, and `openclaw agent --local` route normal HTTP and WebSocket egress through the configured proxy:

```text
OpenClaw process
  fetch                  -> operator-managed filtering proxy -> public internet
  node:http and https    -> operator-managed filtering proxy -> public internet
  WebSocket clients      -> operator-managed filtering proxy -> public internet
```

The public contract is the routing behavior, not the internal Node hooks used to implement it. OpenClaw Gateway control-plane WebSocket clients use a narrow direct path for local loopback Gateway RPC traffic when the Gateway URL uses `localhost` or a literal loopback IP such as `127.0.0.1` or `[::1]`. That control-plane path must be able to reach loopback Gateways even when the operator proxy blocks loopback destinations. Normal runtime HTTP and WebSocket requests still use the configured proxy.

Internally, OpenClaw uses two process-level routing hooks for this feature:

- Undici dispatcher routing covers `fetch`, undici-backed clients, and transports that provide their own undici dispatcher.
- `global-agent` routing covers Node core `node:http` and `node:https` callers, including many libraries layered on `http.request`, `https.request`, `http.get`, and `https.get`. Managed proxy mode forces that global agent so explicit Node HTTP agents do not accidentally bypass the operator proxy.

Some plugins own custom transports that need explicit proxy wiring even when process-level routing exists. For example, Telegram's Bot API transport uses its own HTTP/1 undici dispatcher and therefore honors process proxy env plus the managed `OPENCLAW_PROXY_URL` fallback in that owner-specific transport path.

The proxy URL itself must use `http://`. HTTPS destinations are still supported through the proxy with HTTP `CONNECT`; this only means OpenClaw expects a plain HTTP forward-proxy listener such as `http://127.0.0.1:3128`.

While the proxy is active, OpenClaw clears `no_proxy`, `NO_PROXY`, and `GLOBAL_AGENT_NO_PROXY`. Those bypass lists are destination-based, so leaving `localhost` or `127.0.0.1` there would let high-risk SSRF targets skip the filtering proxy.

On shutdown, OpenClaw restores the previous proxy environment and resets cached process routing state.

## Related Proxy Terms

- `proxy.enabled` / `proxy.proxyUrl`: outbound forward-proxy routing for OpenClaw runtime egress. This page documents that feature.
- `gateway.auth.mode: "trusted-proxy"`: inbound identity-aware reverse-proxy authentication for Gateway access. See [Trusted proxy auth](/gateway/trusted-proxy-auth).
- `openclaw proxy`: local debug proxy and capture inspector for development and support. See [openclaw proxy](/cli/proxy).
- Channel or provider-specific proxy settings: owner-specific overrides for a particular transport. Prefer the managed network proxy when the goal is central egress control across the runtime.

## Configuration

```yaml
proxy:
  enabled: true
  proxyUrl: http://127.0.0.1:3128
```

You can also provide the URL through the environment, while keeping `proxy.enabled=true` in config:

```bash
OPENCLAW_PROXY_URL=http://127.0.0.1:3128 openclaw gateway run
```

`proxy.proxyUrl` takes precedence over `OPENCLAW_PROXY_URL`.

If `enabled=true` but no valid proxy URL is configured, protected commands fail startup instead of falling back to direct network access.

For managed gateway services started with `openclaw gateway start`, prefer storing the URL in config:

```bash
openclaw config set proxy.enabled true
openclaw config set proxy.proxyUrl http://127.0.0.1:3128
openclaw gateway install --force
openclaw gateway start
```

The environment fallback is best for foreground runs. If you use it with an installed service, put `OPENCLAW_PROXY_URL` in the service durable environment, such as `$OPENCLAW_STATE_DIR/.env` or `~/.openclaw/.env`, then reinstall the service so launchd, systemd, or Scheduled Tasks starts the gateway with that value.

For `openclaw --container ...` commands, OpenClaw forwards `OPENCLAW_PROXY_URL` into the container-targeted child CLI when it is set. The URL must be reachable from inside the container; `127.0.0.1` refers to the container itself, not the host. OpenClaw rejects loopback proxy URLs for container-targeted commands unless you explicitly override that safety check.

## Proxy Requirements

The proxy policy is the security boundary. OpenClaw cannot verify that the proxy blocks the right targets.

Configure the proxy to:

- Bind only to loopback or a private trusted interface.
- Restrict access so only the OpenClaw process, host, container, or service account can use it.
- Resolve destinations itself and block destination IPs after DNS resolution.
- Apply policy at connect time for both plain HTTP requests and HTTPS `CONNECT` tunnels.
- Reject destination-based bypasses for loopback, private, link-local, metadata, multicast, reserved, or documentation ranges.
- Avoid hostname allowlists unless you fully trust the DNS resolution path.
- Log destination, decision, status, and reason without logging request bodies, authorization headers, cookies, or other secrets.
- Keep proxy policy under version control and review changes like security-sensitive configuration.

## Recommended Blocked Destinations

Use this denylist as the starting point for any forward proxy, firewall, or egress policy.

OpenClaw application-level classifier logic lives in `src/infra/net/ssrf.ts` and `src/shared/net/ip.ts`. The relevant parity hooks are `BLOCKED_HOSTNAMES`, `BLOCKED_IPV4_SPECIAL_USE_RANGES`, `BLOCKED_IPV6_SPECIAL_USE_RANGES`, `RFC2544_BENCHMARK_PREFIX`, and the embedded IPv4 sentinel handling for NAT64, 6to4, Teredo, ISATAP, and IPv4-mapped forms. Those files are useful references when maintaining an external proxy policy, but OpenClaw does not automatically export or enforce those rules in your proxy.

| Range or host                                                                        | Why to block                                         |
| ------------------------------------------------------------------------------------ | ----------------------------------------------------

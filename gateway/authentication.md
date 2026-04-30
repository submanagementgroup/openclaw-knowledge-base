---
domain: gateway
topic: "Gateway Authentication: Token Auth, Trusted Proxy, and OAuth Auth Profiles"
type: reference
keywords:
  - authentication
  - gateway token
  - trusted proxy
  - auth profiles
  - OAuth
  - OPENCLAW_TOKEN
  - proxy auth
related:
  - gateway/security-overview
  - gateway/secrets
  - security/security-overview
source:
  - gateway/authentication.md
  - gateway/trusted-proxy-auth.md
---

Gateway authentication controls which clients can connect to the Gateway RPC. By default the Gateway is local-only (localhost). Remote access requires a token or trusted proxy configuration.

## Local Token Auth

```json5
{
  gateway: {
    auth: {
      token: "your-secret-token",
    }
  }
}
```

The CLI uses this token automatically from `~/.openclaw/openclaw.json` or `OPENCLAW_TOKEN` env var.

## Trusted Proxy Authentication

For reverse proxy deployments (nginx, Caddy, Traefik):

```json5
{
  gateway: {
    auth: {
      trustedProxies: ["127.0.0.1", "::1"],
      proxyAuthHeader: "X-OpenClaw-User",  // proxy sets this after verifying identity
    }
  }
}
```

## OAuth and Credential Profiles

Auth profiles manage credentials for AI providers:
- OAuth-based (Anthropic, OpenAI Codex): stored in `~/.openclaw/auth-profiles.json`
- API key-based: stored in config or as SecretRefs
- Profile selection: `agents.defaults.authProfile` or `agents.list[].authProfile`

This page is the **model provider** authentication reference (API keys, OAuth, Claude CLI reuse, and Anthropic setup-token). For **gateway connection** authentication (token, password, trusted-proxy), see [Configuration](/gateway/configuration) and [Trusted Proxy Auth](/gateway/trusted-proxy-auth).

OpenClaw supports OAuth and API keys for model providers. For always-on gateway
hosts, API keys are usually the most predictable option. Subscription/OAuth
flows are also supported when they match your provider account model.

See [/concepts/oauth](/concepts/oauth) for the full OAuth flow and storage
layout.
For SecretRef-based auth (`env`/`file`/`exec` providers), see [Secrets Management](/gateway/secrets).
For credential eligibility/reason-code rules used by `models status --probe`, see
[Auth Credential Semantics](/auth-credential-semantics).

## Recommended setup (API key, any provider)

If you’re running a long-lived gateway, start with an API key for your chosen
provider.
For Anthropic specifically, API key auth is still the most predictable server
setup, but OpenClaw also supports reusing a local Claude CLI login.

1. Create an API key in your provider console.
2. Put it on the **gateway host** (the machine running `openclaw gateway`).

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. If the Gateway runs under systemd/launchd, prefer putting the key in
   `~/.openclaw/.env` so the daemon can read it:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

Then restart the daemon (or restart your Gateway process) and re-check:

```bash
openclaw models status
openclaw doctor
```

If you’d rather not manage env vars yourself, onboarding can store
API keys for daemon use: `openclaw onboard`.

See [Help](/help) for details on env inheritance (`env.shellEnv`,
`~/.openclaw/.env`, systemd/launchd).

## Anthropic: Claude CLI and token compatibility

Anthropic setup-token auth is still available in OpenClaw as a supported token
path. Anthropic staff has since told us that OpenClaw-style Claude CLI usage is
allowed again, so OpenClaw treats Claude CLI reuse and `claude -p` usage as
sanctioned for this integration unless Anthropic publishes a new policy. When
Claude CLI reuse is available on the host, that is now the preferred path.

For long-lived gateway hosts, an Anthropic API key is still the most predictable
setup. If you want to reuse an existing Claude login on the same host, use the
Anthropic Claude CLI path in onboarding/configure.

Recommended host setup for Claude CLI reuse:

```bash
# Run on the gateway host
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

This is a two-step setup:

1. Log Claude Code itself into Anthropic on the gateway host.
2. Tell OpenClaw to switch Anthropic model selection to the local `claude-cli`
   backend and store the matching OpenClaw auth profile.

If `claude` is not on `PATH`, either install Claude Code

## Trusted Proxy Auth Details

**Security-sensitive feature.** This mode delegates authentication entirely to your reverse proxy. Misconfiguration can expose your Gateway to unauthorized access. Read this page carefully before enabling.

## When to use

Use `trusted-proxy` auth mode when:

- You run OpenClaw behind an **identity-aware proxy** (Pomerium, Caddy + OAuth, nginx + oauth2-proxy, Traefik + forward auth).
- Your proxy handles all authentication and passes user identity via headers.
- You're in a Kubernetes or container environment where the proxy is the only path to the Gateway.
- You're hitting WebSocket `1008 unauthorized` errors because browsers can't pass tokens in WS payloads.

## When NOT to use

- If your proxy doesn't authenticate users (just a TLS terminator or load balancer).
- If there's any path to the Gateway that bypasses the proxy (firewall holes, internal network access).
- If you're unsure whether your proxy correctly strips/overwrites forwarded headers.
- If you only need personal single-user access (consider Tailscale Serve + loopback for simpler setup).

## How it works

    Your reverse proxy authenticates users (OAuth, OIDC, SAML, etc.).

    Proxy adds a header with the authenticated user identity (e.g., `x-forwarded-user: nick@example.com`).

    OpenClaw checks that the request came from a **trusted proxy IP** (configured in `gateway.trustedProxies`).

    OpenClaw extracts the user identity from the configured header.

    If everything checks out, the request is authorized.

## Control UI pairing behavior

When `gateway.auth.mode = "trusted-proxy"` is active and the request passes trusted-proxy checks, Control UI WebSocket sessions can connect without device pairing identity.

Implications:

- Pairing is no longer the primary gate for Control UI access in this mode.
- Your reverse proxy auth policy and `allowUsers` become the effective access control.
- Keep gateway ingress locked to trusted proxy IPs only (`gateway.trustedProxies` + firewall).

## Configuratio

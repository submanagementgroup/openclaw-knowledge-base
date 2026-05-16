---
domain: security
topic: "OpenClaw Security Model and Personal Assistant Trust Boundary"
type: concept
keywords:
  - security model
  - trust boundary
  - personal assistant
  - multi-tenant
  - gateway trust
  - node trust
  - sessionKey
  - operator scope
  - hardened baseline
  - security audit
source: gateway/security/index.md
related:
  - security/access-controls
  - security/network-hardening
  - security/prompt-injection
  - security/configuration-hardening
  - security/audit-checks-reference
  - gateway/operator-scopes
  - gateway/sandboxing
---

OpenClaw's security model assumes a **personal assistant** deployment: one trusted operator boundary per gateway, potentially many agents. Running one shared gateway for mutually adversarial users is not a supported security boundary. Use separate gateways (separate OS users/hosts) when isolation between adversarial users is required.

## Personal Assistant Trust Model

OpenClaw security guidance assumes:

- **Supported posture:** one user/trust boundary per gateway (prefer one OS user/host/VPS per boundary).
- **Not supported:** one shared gateway/agent used by mutually untrusted or adversarial users.
- If adversarial-user isolation is required, split by trust boundary — separate gateway + credentials, ideally separate OS users/hosts.
- If multiple untrusted users can message one tool-enabled agent, treat them as sharing the same delegated tool authority for that agent.

## Gateway and Node Trust Concept

Treat Gateway and node as one operator trust domain with different roles:

- **Gateway** is the control plane and policy surface (`gateway.auth`, tool policy, routing).
- **Node** is the remote execution surface paired to that Gateway (commands, device actions, host-local capabilities).
- A caller authenticated to the Gateway is trusted at Gateway scope. After pairing, node actions are trusted operator actions on that node.
- `sessionKey` is a routing/context selection key, **not** a per-user auth boundary.
- Exec approvals (allowlist + ask) are guardrails for operator intent, not hostile multi-tenant isolation.

## Trust Boundary Matrix

| Boundary or control | What it means | Common misread |
|---|---|---|
| `gateway.auth` (token/password/trusted-proxy/device auth) | Authenticates callers to gateway APIs | "Needs per-message signatures on every frame to be secure" |
| `sessionKey` | Routing key for context/session selection | "Session key is a user auth boundary" |
| Prompt/content guardrails | Reduce model abuse risk | "Prompt injection alone proves auth bypass" |
| `canvas.eval` / browser evaluate | Intentional operator capability when enabled | "Any JS eval primitive is automatically a vuln in this trust model" |
| Local TUI `!` shell | Explicit operator-triggered local execution | "Local shell convenience command is remote injection" |
| Node pairing and node commands | Operator-level remote execution on paired devices | "Remote device control should be treated as untrusted user access by default" |
| `gateway.nodes.pairing.autoApproveCidrs` | Opt-in trusted-network node enrollment policy | "A disabled-by-default allowlist is an automatic pairing vulnerability" |

## Not Vulnerabilities by Design

These patterns are commonly reported but are not security issues unless a real boundary bypass is demonstrated:

- Prompt-injection-only chains without a policy, auth, or sandbox bypass.
- Claims that assume hostile multi-tenant operation on one shared host or config.
- Claims that classify normal operator read-path access (`sessions.list`, `sessions.preview`, `chat.history`) as IDOR in a shared-gateway setup.
- Localhost-only deployment findings (e.g., HSTS on a loopback-only gateway).
- Reports that treat node pairing metadata as a hidden second per-command approval layer for `system.run`. The real execution boundary is still the gateway's global node command policy plus the node's own exec approvals.
- Reports that treat configured `gateway.nodes.pairing.autoApproveCidrs` as a vulnerability by itself. This setting is disabled by default, requires explicit CIDR/IP entries, and only applies to first-time `role: node` pairing with no requested scopes.
- "Missing per-user authorization" findings that treat `sessionKey` as an auth token.

## Quick Security Audit

Run regularly after changing config or exposing network surfaces:

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --fix
openclaw security audit --json
```

`security audit --fix` flips common open group policies to allowlists, restores `logging.redactSensitive: "tools"`, tightens state/config/include-file permissions, and uses Windows ACL resets on Windows.

## Hardened Baseline in 60 Seconds

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "replace-with-long-random-token" },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: { dmPolicy: "pairing", groups: { "*": { requireMention: true } } },
  },
}
```

This keeps the Gateway local-only, isolates DMs, and disables control-plane/runtime tools by default.

## Deployment and Host Trust

- If someone can modify Gateway host state/config (`~/.openclaw`, including `openclaw.json`), treat them as a trusted operator.
- For mixed-trust teams, split trust boundaries with separate gateways (or at minimum separate OS users/hosts).
- Recommended default: one user per machine/host (or VPS), one gateway for that user, one or more agents in that gateway.
- Session identifiers (`sessionKey`, session IDs, labels) are routing selectors, not authorization tokens.

## Security Audit Priority Order

When the audit prints findings:

1. **Anything "open" + tools enabled:** lock down DMs/groups first (pairing/allowlists), then tighten tool policy/sandboxing.
2. **Public network exposure** (LAN bind, Funnel, missing auth): fix immediately.
3. **Browser control remote exposure:** treat it like operator access (tailnet-only, pair nodes deliberately, avoid public exposure).
4. **Permissions:** make sure state/config/credentials/auth are not group/world-readable.
5. **Plugins:** only load what you explicitly trust.
6. **Model choice:** prefer modern, instruction-hardened models for any bot with tools.

## Security Audit Glossary

Each audit finding is keyed by a structured `checkId` (e.g., `gateway.bind_no_auth` or `tools.exec.security_full_configured`). Common critical severity classes:

- `fs.*` — filesystem permissions on state, config, credentials, auth profiles.
- `gateway.*` — bind mode, auth, Tailscale, Control UI, trusted-proxy setup.
- `hooks.*`, `browser.*`, `sandbox.*`, `tools.exec.*` — per-surface hardening.
- `plugins.*`, `skills.*` — plugin/skill supply chain and scan findings.
- `security.exposure.*` — cross-cutting checks where access policy meets tool blast radius.

See full catalog at [Security audit checks](/gateway/security/audit-checks).

## Reporting Security Issues

Found a vulnerability in OpenClaw? Report responsibly:

1. Email: security@openclaw.ai
2. Don't post publicly until fixed
3. Credit provided (unless you prefer anonymity)

---
domain: gateway
topic: "Gateway Security: Personal Assistant Model, Trust Boundaries, and Quick Hardening"
type: reference
keywords:
  - security
  - security model
  - trust boundaries
  - hardening
  - allowFrom
  - gateway token
  - security audit
related:
  - gateway/authentication
  - security/security-overview
  - gateway/sandboxing
source: gateway/security/index.md
---

OpenClaw security model, hardening steps, trust boundaries, and audit tooling. The security model is designed for a personal assistant, not a multi-tenant service — the key assumption is that the person running the Gateway is the person using it.

## Personal Assistant Security Model

OpenClaw is a personal assistant gateway. The security model assumes the operator and user are the same person. Multi-user deployments require explicit hardening (allowlists, tokens, reverse proxy auth).

## Quick Hardening (60 seconds)

```bash
# 1. Audit your current setup
openclaw security audit

# 2. Lock down channel access
# In openclaw.json:
# channels.telegram.allowFrom: ["@yourusername"]
# channels.whatsapp.allowFrom: ["+1yournumber"]

# 3. Set a gateway token
# gateway.auth.token: "your-secret-token"
```

## Trust Boundary Matrix

| Boundary | Trust Level | Risk |
|----------|-------------|------|
| Gateway ↔ local CLI | Full trust (same user) | Low |
| Gateway ↔ channel (inbound) | Controlled by allowFrom | Medium if open |
| Gateway ↔ node (paired) | Signed identity | Medium |
| Gateway ↔ internet | Zero trust | High if exposed |

**Personal assistant trust model.** This guidance assumes one trusted
  operator boundary per gateway (single-user, personal-assistant model).
  OpenClaw is **not** a hostile multi-tenant security boundary for multiple
  adversarial users sharing one agent or gateway. If you need mixed-trust or
  adversarial-user operation, split trust boundaries (separate gateway +
  credentials, ideally separate OS users or hosts).

## Scope first: personal assistant security model

OpenClaw security guidance assumes a **personal assistant** deployment: one trusted operator boundary, potentially many agents.

- Supported security posture: one user/trust boundary per gateway (prefer one OS user/host/VPS per boundary).
- Not a supported security boundary: one shared gateway/agent used by mutually untrusted or adversarial users.
- If adversarial-user isolation is required, split by trust boundary (separate gateway + credentials, and ideally separate OS users/hosts).
- If multiple untrusted users can message one tool-enabled agent, treat them as sharing the same delegated tool authority for that agent.

This page explains hardening **within that model**. It does not claim hostile multi-tenant isolation on one shared gateway.

## Quick check: `openclaw security audit`

See also: [Formal Verification (Security Models)](/security/formal-verification)

Run this regularly (especially after changing config or exposing network surfaces):

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --fix
openclaw security audit --json
```

`security audit --fix` stays intentionally narrow: it flips common open group
policies to allowlists, restores `logging.redactSensitive: "tools"`, tightens
state/config/include-file permissions, and uses Windows ACL resets instead of
POSIX `chmod` when running on Windows.

It flags common footguns (Gateway auth exposure, browser control exposure, elevated allowlists, filesystem permissions, permissive exec approvals, and open-channel tool exposure).

OpenClaw is both a product and an experiment: you’re wiring frontier-model behavior into real messaging surfaces and real tools. **There is no “perfectly secure” setup.** The goal is to be deliberate about:

- who can talk to your bot
- where the bot is allowed to act
- what the bot can touch

Start with the smallest access that still works, then widen it as you gain confidence.

### Deployment and host trust

OpenClaw assumes the host and config boundary are trusted:

- If someone can modify Gateway host state/config (`~/.openclaw`, including `openclaw.json`), treat them as a trusted operator.
- Running one Gateway for multiple mutually untrusted/adversarial operators is **not a recommended setup**.
- For mixed-trust teams, split trust boundaries with separate gateways (or at minimum separate OS users/hosts).
- Recommended default: one user per machine/host (or VPS), one gateway for that user, and one or more agents in that gateway.
- Inside one Gateway instance, authenticated operator access is a trusted control-plane role, not a per-user tenant role.
- Session identifiers (`sessionKey`, session IDs, labels) are routing selectors, not authorization tokens.
- If several people can message one tool-enabled agent, each of them can steer that same permission set. Per-user session/memory isolation helps privacy, but does not convert a shared agent into per-user host authorization.

### Shared Slack workspace: real risk

If "everyone in Slack can message the bot," the core risk is delegated tool authority:

- any allowed sender can induce tool calls (`exec`, browser, network/file tools) within the agent's policy;
- prompt/content injection from one sender can cause actions that affect shared state, devices, or outputs;
- if one shared agent has sensitive credentials/files, any allowed sender can potentially drive exfiltration via tool usage.

Use separate agents/gateways with minimal tools for team workflows; keep personal-data agents private.

### Company-shared agent: acceptable pattern

This is acceptable when everyone using that agent is in the same trust boundary (for example one company team) and the agent is strictly business-scoped.

- run it on a dedicated machine/VM/container;
- use a dedicated OS user + dedicated browser/profile/accounts for that runtime;
- do not sign that runtime into personal Apple/Google accounts or personal password-manager/browser profiles.

If you mix personal and company identities on the same runtime, you collapse the separation and increase personal-data exposure risk.

## Gateway and node trust concept

Treat Gateway and node as one operator trust domain, with different roles:

- **Gateway** is the control plane and policy surface (`gateway.auth`, tool policy, routing).
- **Node** is remote execution surface paired to that Gateway (commands, device actions, host-local capabilities).
- A caller authenticated to the Gateway is trusted at Gateway scope. After pairing, node actions are trusted o

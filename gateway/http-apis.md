---
domain: gateway
topic: "Gateway HTTP APIs: OpenAI-Compatible Completions, OpenResponses, and Tools Invoke"
type: reference
keywords:
  - OpenAI compatible API
  - HTTP API
  - chat completions
  - v1/chat/completions
  - tools invoke
  - REST API
  - programmatic access
related:
  - gateway/protocol
  - gateway/authentication
source:
  - gateway/openai-http-api.md
  - gateway/openresponses-http-api.md
  - gateway/tools-invoke-http-api.md
---

The OpenClaw Gateway exposes OpenAI-compatible HTTP endpoints and a tools-invoke HTTP API for programmatic access.

## OpenAI-Compatible Chat Completions

The Gateway serves `POST /v1/chat/completions` compatible with OpenAI's API format:

```bash
curl http://127.0.0.1:18789/v1/chat/completions \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"default","messages":[{"role":"user","content":"hello"}]}'
```

Use this endpoint with any OpenAI-compatible client or SDK.

OpenClaw’s Gateway can serve a small OpenAI-compatible Chat Completions endpoint.

This endpoint is **disabled by default**. Enable it in config first.

- `POST /v1/chat/completions`
- Same port as the Gateway (WS + HTTP multiplex): `http://<gateway-host>:<port>/v1/chat/completions`

When the Gateway’s OpenAI-compatible HTTP surface is enabled, it also serves:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/responses`

Under the hood, requests are executed as a normal Gateway agent run (same codepath as `openclaw agent`), so routing/permissions/config match your Gateway.

## Authentication

Uses the Gateway auth configuration.

Common HTTP auth paths:

- shared-secret auth (`gateway.auth.mode="token"` or `"password"`):
  `Authorization: Bearer <token-or-password>`
- trusted identity-bearing HTTP auth (`gateway.auth.mode="trusted-proxy"`):
  route through the configured identity-aware proxy and let it inject the
  required identity headers
- private-ingress open auth (`gateway.auth.mode="none"`):
  no auth header required

Notes:

- When `gateway.auth.mode="token"`, use `gateway.auth.token` (or `OPENCLAW_GATEWAY_TOKEN`).
- When `gateway.auth.mode="password"`, use `gateway.auth.password` (or `OPENCLAW_GATEWAY_PASSWORD`).
- When `gateway.auth.mode="trusted-proxy"`, the HTTP request must come from a
  configured trusted proxy source; same-host loopback proxies require explicit
  `gateway.auth.trustedProxy.allowLoopback = true`.
- If `gateway.auth.rateLimit` is configured and too many auth failures occur, the endpoint returns `429` with `Retry-After`.

## Security boundary (important)

Treat this endpoint as a **full operator-access** surface for the gateway instance.

- HTTP bearer auth here is not a narrow per-user scope model.
- A valid Gateway token/password for this endpoint should be treated like an owner/operator credential.
- Requests run through the same control-plane agent path as trusted operator actions.
- There is no separate non-owner/per-user tool boundary on this endpoint; once a caller passes Gateway auth here, OpenClaw treats that caller as a trusted operator for this gateway.
- For shared-secret auth modes (`token` and `password`), the endpoint restores the normal full operator defaults even if the caller sends a narrower `x-openclaw-scopes` header.
- Trusted identity-bearing HTTP modes (for example trusted proxy auth or `gateway.auth.mode="none"`) honor `x-openclaw-scopes` when present and otherwise fall back to the normal operator default scope set.
- If the target agent policy allows sensitive tools, this endpoint can use them.
- Keep this endpoint on loopback/tailnet/private ingress only; do not expose it directly to the public internet.

Auth matrix:

- `gateway.auth.mode="token"` or `"password"` + `Authorization: Bearer ...`
  - proves possession of the shared gateway operator secret
  - ignores narrower `x-openclaw-scopes`
  - restores the full default operator scope set:
    `operator.admin`, 

## OpenResponses HTTP API

OpenClaw’s Gateway can serve an OpenResponses-compatible `POST /v1/responses` endpoint.

This endpoint is **disabled by default**. Enable it in config first.

- `POST /v1/responses`
- Same port as the Gateway (WS + HTTP multiplex): `http://<gateway-host>:<port>/v1/responses`

Under the hood, requests are executed as a normal Gateway agent run (same codepath as
`openclaw agent`), so routing/permissions/config match your Gateway.

## Authentication, security, and routing

Operational behavior matches [OpenAI Chat Completions](/gateway/openai-http-api):

- use the matching Gateway HTTP auth path:
  - shared-secret auth (`gateway.auth.mode="token"` or `"password"`): `Authorization: Bearer <token-or-password>`
  - trusted-proxy auth (`gateway.auth.mode="trusted-proxy"`): identity-aware proxy headers from a configured trusted proxy source; same-host loopback proxies require explicit `gateway.auth.trustedProxy.allowLoopback = true`
  - private-ingress open auth (`gateway.auth.mode="none"`): no auth header
- treat the endpoint as full operator access for the gateway instance
- for shared-secret auth modes (`token` and `password`), ignore narrower bearer-declared `x-openclaw-scopes` values and restore the normal full operator defaults
- for trusted identity-bearing HTTP modes (for example trusted proxy auth or `gateway.auth.mode="none"`), honor `x-openclaw-scopes` when present and otherwise fall back to the normal operator default scope set
- select agents with `model: "openclaw"`, `model: "openclaw/default"`, `model: "openclaw/<agentId>"`, or `x-openclaw-agent-id`
- use `x-openclaw-model` when you want to override the selected agent's backend model
- use `x-openclaw-session-key` for explicit session routing
- use `x-openclaw-message-channel` when you want a non-default synthetic ingress channel context

Auth matrix:

- `gateway.auth.mode="token"` or `"password"` + `Authorization: Bearer ...`
  - proves possession of the shared gateway operator secret
  - ignores narrower `x-openclaw-scopes`
  - restores the full default operator scope set:
    `operator.admin`, `operator.approvals`, `operator.pairing`,
    `operator.read`, `operator.talk.secrets`, `operator.write`
  - treats chat turns on this endpoint as owner-sender turns
- trusted identity-bearing HTTP modes (for example trusted proxy auth, or `gateway.auth.mode="none"` on private ingress)
  - honor `x-openclaw-scopes` when the header is present
  - fall back to the normal operator default scope set when the he

## Tools Invoke HTTP API

# Tools Invoke (HTTP)

OpenClaw’s Gateway exposes a simple HTTP endpoint for invoking a single tool directly. It is always enabled and uses Gateway auth plus tool policy. Like the OpenAI-compatible `/v1/*` surface, shared-secret bearer auth is treated as trusted operator access for the whole gateway.

- `POST /tools/invoke`
- Same port as the Gateway (WS + HTTP multiplex): `http://<gateway-host>:<port>/tools/invoke`

Default max payload size is 2 MB.

## Authentication

Uses the Gateway auth configuration.

Common HTTP auth paths:

- shared-secret auth (`gateway.auth.mode="token"` or `"password"`):
  `Authorization: Bearer <token-or-password>`
- trusted identity-bearing HTTP auth (`gateway.auth.mode="trusted-proxy"`):
  route through the configured identity-aware proxy and let it inject the
  required identity headers
- private-ingress open auth (`gateway.auth.mode="none"`):
  no auth header required

Notes:

- When `gateway.auth.mode="token"`, use `gateway.auth.token` (or `OPENCLAW_GATEWAY_TOKEN`).
- When `gateway.auth.mode="password"`, use `gateway.auth.password` (or `OPENCLAW_GATEWAY_PASSWORD`).
- When `gateway.auth.mode="trusted-proxy"`, the HTTP request must come from a
  configured trusted proxy source; same-host loopback proxies require explicit
  `gateway.auth.trustedProxy.allowLoopback = true`.
- If `gateway.auth.rateLimit` is configured and too many auth failures occur, the endpoint returns `429` with `Retry-After`.

## Security boundary (important)

Treat this endpoint as a **full operator-access** surface for the gateway instance.

- HTTP bearer auth here is not a narrow per-user scope model.
- A valid Gateway token/password for this endpoint should be treated like an owner/operator credential.
- For shared-secret auth modes (`token` and `password`), the endpoint restores the normal full operator defaults even if the caller sends a narrower `x-openclaw-scopes` header.
- Shared-secret auth also treats direct tool invokes on this endpoint as owner-sender t

---
domain: gateway
topic: "OpenAI-Compatible and OpenResponses HTTP APIs"
type: reference
keywords:
  - openai HTTP API
  - openresponses API
  - REST API
  - completions API
  - HTTP gateway
  - API endpoint
  - tools-invoke
source: ['gateway/openai-http-api.md', 'gateway/openresponses-http-api.md']
---

## OpenAI-Compatible HTTP API

OpenClaw's Gateway can serve a small OpenAI-compatible Chat Completions endpoint.

This endpoint is **disabled by default**. Enable it in config first.

- `POST /v1/chat/completions`
- Same port as the Gateway (WS + HTTP multiplex): `http://<gateway-host>:<port>/v1/chat/completions`

When the Gateway's OpenAI-compatible HTTP surface is enabled, it also serves:

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
    `operator.admin`, `operator.approvals`, `operator.pairing`,
    `operator.read`, `operator.talk.secrets`, `operator.write`
  - treats chat turns on this endpoint as owner-sender turns
- trusted identity-bearing HTTP modes (for example trusted proxy auth, or `gateway.auth.mode="none"` on private ingress)
  - authenticate some outer trusted identity or deployment boundary
  - honor `x-openclaw-scopes` when the header is present
  - fall back to the normal operator default scope set when the header is absent
  - only lose owner semantics when the caller explicitly narrows scopes and omits `operator.admin`

See [Security](/gateway/security) and [Remote access](/gateway/remote).

## Agent-first model contract

OpenClaw treats the OpenAI `model` field as an **agent target**, not a raw provider model id.

- `model: "openclaw"` routes to the configured default agent.
- `model: "openclaw/default"` also routes to the configured default agent.
- `model: "openclaw/<agentId>"` routes to a specific agent.

Optional request headers:

- `x-openclaw-model: <provider/model-or-bare-id>` overrides the backend model for the selected agent.
- `x-openclaw-agent-id: <agentId>` remains supported as a compatibility override.
- `x-openclaw-session-key: <sessionKey>` fully controls session routing.
- `x-openclaw-message-channel: <channel>` sets the synthetic ingress channel context for channel-aware prompts and policies.

Compatibility aliases still accepted:

- `model: "openclaw:<agentId>"`
- `model: "agent:<agentId>"`

## Enabling the endpoint

Set `gateway.http.endpoints.chatCompletions.enabled` to `true`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

## Disabling the endpoint

Set `gateway.http.endpoints.chatCompletions.enabled` to `false`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: false },
      },
    },
  },
}
```

## Session behavior

By default the endpoint is **stateless per request** (a new session key is generated each call).

If the request includes an OpenAI `user` string, the Gateway derives a stable session key from it, so repeated calls can share an agent session.

## Why this surface matters

This is the highest-leverage compatibility set for self-hosted frontends and tooling:

- Most Open WebUI, LobeChat, and LibreChat setups expect `/v1/models`.
- Many RAG systems expect `/v1/embeddings`.
- Existing OpenAI chat clients can usually start with `/v1/chat/completions`.
- More agent-native clients increasingly prefer `/v1/responses`.

## Model list and agent routing

### What does `/v1/models` return?

An OpenClaw agent-target list.

    The returned ids are `openclaw`, `openclaw/default`, and `openclaw/<agentId>` entries.
    Use them directly as OpenAI `model` values.

  ### Does `/v1/models` list agents or sub-agents?

It lists top-level agent targets, not backend provider models and not sub-agents.

    Sub-agents remain internal execution top

## OpenResponses HTTP API

OpenClaw's Gateway can serve an OpenResponses-compatible `POST /v1/responses` endpoint.

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
  - fall back to the normal operator default scope set when the header is absent
  - only lose owner semantics when the caller explicitly narrows scopes and omits `operator.admin`

Enable or disable this endpoint with `gateway.http.endpoints.responses.enabled`.

The same compatibility surface also includes:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`

For the canonical explanation of how agent-target models, `openclaw/default`, embeddings pass-through, and backend model overrides fit together, see [OpenAI Chat Completions](/gateway/openai-http-api#agent-first-model-contract) and [Model list and agent routing](/gateway/openai-http-api#model-list-and-agent-routing).

## Session behavior

By default the endpoint is **stateless per request** (a new session key is generated each call).

If the request includes an OpenResponses `user` string, the Gateway derives a stable session key
from it, so repeated calls can share an agent session.

## Request shape (supported)

The request follows the OpenResponses API with item-based input. Current support:

- `input`: string or array of item objects.
- `instructions`: merged into the system prompt.
- `tools`: client tool definitions (function tools).
- `tool_choice`: filter or require client tools.
- `stream`: enables SSE streaming.
- `max_output_tokens`: best-effort output limit (provider dependent).
- `user`: stable session routing.

Accepted but **currently ignored**:

- `max_tool_calls`
- `reasoning`
- `metadata`
- `store`
- `truncation`

Supported:

- `previous_response_id`: OpenClaw reuses the earlier response session when the request stays within the same agent/user/requested-session scope.

## Items (input)

### `message`

Roles: `system`, `developer`, `user`, `assistant`.

- `system` and `developer` are appended to the system prompt.
- The most recent `user` or `function_call_output` item becomes the "current message."
- Earlier user/assistant messages are included as history for context.

### `function_call_output` (turn-based tools)

Send tool results back to the model:

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` and `item_reference`

Accepted for schema compatibility but ignored when building the prompt.

## Tools (client-side function tools)

Provide tools with `tools: [{ type: "function", function: { name, description?, parameters? } }]`.

If the agent decides to call a tool, the response returns a `function_call` output item.
You then send a follow-up request with `function_call_output` to continue the turn.

## Images (`input_image`)

Supports base64 or URL sources:

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

Allowed MIME types (current): `image/jpeg`, `image/png`, `image/gif`, `image/webp`, `image/heic`, `image/heif`.
Max size (current): 10MB.

## Files (`input_file`)

Supports base64 or URL sources:

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

Allowed MIME types (current): `text/plain`, `text/markdown`, `text/html`, `text/csv`,
`application/json`, `application/pdf`.

Max size (current): 5MB.

Current behavior:

- File content is decoded and added to the **system prompt**, not the user message,
  so it stays ephemeral (not persisted in session history).
- Decoded file text is wrapped as **untrusted external conten
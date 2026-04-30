---
domain: concepts
topic: "Model Failover: Backup Model Chains, Auth Profiles, and Rate Limit Recovery"
type: concept
keywords:
  - model failover
  - fallback model
  - backup model
  - auth profiles
  - rate limits
  - model reliability
related:
  - concepts/models
  - concepts/oauth
  - gateway/config-agents-reference
source: concepts/model-failover.md
---

Model failover automatically switches to a backup model when the primary model fails. Configure a fallback chain to ensure reliability across provider outages or rate limits.

OpenClaw handles failures in two stages:

1. **Auth profile rotation** within the current provider.
2. **Model fallback** to the next model in `agents.defaults.model.fallbacks`.

This doc explains the runtime rules and the data that backs them.

## Runtime flow

For a normal text run, OpenClaw evaluates candidates in this order:

    Resolve the active session model and auth-profile preference.

    Build the model candidate chain from the current model selection and the fallback policy for that selection source. Configured defaults, cron job primaries, and auto-selected fallback models can use configured fallbacks; explicit user session selections are strict.

    Try the current provider with auth-profile rotation/cooldown rules.

    If that provider is exhausted with a failover-worthy error, move to the next model candidate.

    Persist the selected fallback override before the retry starts so other session readers see the same provider/model the runner is about to use. The persisted model override is marked `modelOverrideSource: "auto"`.

    If the fallback candidate fails, roll back only the fallback-owned session override fields when they still match that failed candidate.

    If every candidate fails, throw a `FallbackSummaryError` with per-attempt detail and the soonest cooldown expiry when one is known.

This is intentionally narrower than "save and restore the whole session". The reply runner only persists the model-selection fields it owns for fallback:

- `providerOverride`
- `modelOverride`
- `modelOverrideSource`
- `authProfileOverride`
- `authProfileOverrideSource`
- `authProfileOverrideCompactionCount`

That prevents a failed fallback retry from overwriting newer unrelated session mutations such as manual `/model` changes or session rotation updates that happened while the attempt was running.

## Selection source policy

OpenClaw separates the selected provider/model from why it was selected. That source controls whether the fallback chain is allowed:

- **Configured default**: `agents.defaults.model.primary` uses `agents.defaults.model.fallbacks`.
- **Agent primary**: `agents.list[].model` is strict unless that agent model object includes its own `fallbacks`. Use `fallbacks: []` to make the strict behavior explicit, or provide a non-empty list to opt that agent into model fallback.
- **Auto fallback override**: a runtime fallback writes `providerOverride`, `modelOverride`, and `modelOverrideSource: "auto"` before retrying. That auto override can keep walking the configured fallback chain and is cleared by `/new`, `/reset`, and `sessions.reset`.
- **User session override**: `/model`, the model picker, `session_status(model=...)`, and `sessions.patch` write `modelOverrideSource: "user"`. That is an exact session selection. If the selected provider/model fails before producing a reply, OpenClaw reports the failure instead of answering from an unrelated configured fallback.
- **Legacy session override**: older session entries may have `modelOverride` without `modelOverrideSource`. OpenClaw treats those as user overrides so an explicit old selection is not silently converted into fallback behavior.
- **Cron payload model**: a cron job `payload.model` / `--model` is a job primary, not a user session override. It uses configured fallbacks unless the job provides `payload.fallbacks`; `payload.fallbacks: []` makes the cron run strict.

## Auth storage (keys + OAuth)

OpenClaw uses **auth profiles** for both API keys and OAuth tokens.

- Secrets live in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (legacy: `~/.openclaw/agent/auth-profiles.json`).
- Runtime auth-routing state lives in `~/.openclaw/agents/<agentId>/agent/auth-state.json`.
- Config `auth.profiles` / `auth.order` are **metadata + routing only** (no secrets).
- Legacy import-only OAuth file: `~/.openclaw/credentials/oauth.json` (imported into `auth-profiles.json` on first use).

More detail: [OAuth](/concepts/oauth)

Credential types:

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }` (+ `projectId`/`enterpriseUrl` for some providers)

## Profile IDs

OAuth logins create distinct profiles so multiple accounts can coexist.

- Default: `provider:default` when no email is available.
- OAuth with email: `provider:<email>` (for example `google-antigravity:user@gmail.com`).

Profiles live in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` under `profiles`.

## Rotation order

When a provider has multiple profiles, OpenClaw chooses an order like this:

    `auth.order[provider]` (if set).

    `auth.profiles` filtered by provider.

    Entries in `auth-profiles.json` for the provider.

If no explicit order is configured, OpenClaw uses a round‑robin order:

- **Primary key:** profile type (**OAuth before API keys**).
- **Secondary key:** `usageStats.lastUsed` (oldest first, within each type).
- **Cooldown/disabled profiles** are moved to the end, ordered by soonest expiry.

### Session stickiness (cache-friendly)

OpenClaw **pins the chosen auth profile per session** to keep provider caches warm. It does **not** rotate on every request. The pinned profile is reused until:

- the session is reset (`/new` / `/reset`)
- a compaction completes (compaction count increments)
- the profile is in cooldown/disabled

Manual selection via `/model …@<profileId>` sets a **user override** for that session and is not auto-rotated until a new session starts.

Auto-pinned profiles (selected by the session router) are treated as a **preference**: they are tried first, but OpenClaw may rotate to another profile on rate limits/timeouts. User-pinned profiles stay locked to that profile; if it fails and model fallbacks are configured, OpenClaw moves to the next model instead of switching profiles.

### Why OAuth can "look lost"

If you have both an OAuth profile and an API key profile for the same provider, round‑robin can switch between them across messages unless pinned. To force a single profile:

- Pin with `auth.order[provider] = ["provider:profileId"]`, or
- Use a per-session override via `/model …` with a profile override (when supported by your UI/chat surface).

## Cooldowns

When a profile fails due to auth/rate-limit errors (or a timeout that looks like rate limiting), OpenClaw marks it in cooldown and moves to the next profile.

    That rate-limit bucket is broader than plain `429`: it also includes provider messages such as `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`, `throttled`, `resource exhausted`, and periodic usage-window limits such as `weekly/monthly limit reached`.

    Format/invalid-request errors (for example Cloud Code Assist tool call ID validation failures) are treated as failover-worthy and use the same cooldowns. OpenAI-compatible stop-reason errors such as `Unhandled stop reason: error`, `stop reason: error`, and `reason: error` are classified as timeout/failover signals.

    Generic server text can also land in that timeout bucket when the source matches a known transient pattern. For example, the bare pi-ai stream-wrapper message `An unknown error occurred` is treated as failover-worthy for every provider because pi-ai emits it when provider streams end with `stopReason: "aborted"` or `stopReason: "error"` without specific details. JSON `api_error` payloads with transient server text such as `internal server error`, `unknown error, 520`, `upstream error`, or `backend error` are also treated as failover-worthy timeouts.

    OpenRouter-specific generic upstream text such as bare `Provider returned error` is treated as timeout only when the provider context is actually OpenRouter. Generic internal fallback text such as `LLM request failed with an unknown error.` stays conservative and does not trigger failover by itself.

    Some provider SDKs may otherwise sleep for a long `Retry-After` window before returning control to OpenClaw. For Stainless-based SDKs such as Anthropic and OpenAI, OpenClaw caps SDK-internal `retry-after-ms` / `retry-after` waits at 60 seconds by default and surfaces longer retryable responses immediately so this failover path can run. Tune or disable the cap with `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS`; see [Retry behavior](/concepts/retry).

    Rate-limit cooldowns can also be model-scoped:

    - OpenClaw records `cooldownModel` for rate-limit failures when the failing model id is known.
    - A sibling model on the same provider can still be tried when the cooldown is scoped to a different model.
    - Billing/disabled windows still block the whole profile across models.

Cooldowns use exponential backoff:

- 1 minute
- 5 minutes
- 25 minutes
- 1 hour (cap)

State is stored in `auth-state.json` under `usageStats`:

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
`

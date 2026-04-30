---
domain: gateway
topic: "Gateway Secrets Management: SecretRef Syntax, Providers, and secrets apply"
type: procedure
keywords:
  - secrets
  - SecretRef
  - secrets apply
  - secrets reload
  - 1Password
  - environment variables
  - credential storage
related:
  - gateway/configuration-overview
  - gateway/authentication
source:
  - gateway/secrets.md
  - gateway/secrets-plan-contract.md
---

OpenClaw supports SecretRefs so credentials are not stored as plaintext in `openclaw.json`. Secrets are resolved into an in-memory runtime snapshot at startup; the Gateway fails fast if a required secret cannot be resolved.

## SecretRef Syntax

```json5
{
  channels: {
    telegram: {
      botToken: "{{secret:TELEGRAM_BOT_TOKEN}}",  // resolve from env or secret store
    }
  },
  agents: {
    defaults: {
      model: "openai/gpt-4o",
    }
  }
}
```

## Secret Providers

- **Environment variables** (default): `secret:MY_VAR` reads `$MY_VAR`
- **1Password**: `secret:op://vault/item/field`
- **Hashicorp Vault**: configure `secrets.vault`
- **AWS Secrets Manager**: configure `secrets.aws`

## secrets apply

```bash
# Apply secrets from a file or provider
openclaw secrets apply

# Reload secrets without restart
openclaw secrets reload

# Audit active secrets
openclaw secrets audit
```

## Runtime Behavior

- Resolution is eager at activation, not lazy on request paths
- Startup fails fast when an active SecretRef cannot be resolved
- Reload uses atomic swap: full success or keep the last-known-good snapshot
- Inactive surfaces (disabled channels) do not block startup for unresolved refs

OpenClaw supports additive SecretRefs so supported credentials do not need to be stored as plaintext in configuration.

Plaintext still works. SecretRefs are opt-in per credential.

## Goals and runtime model

Secrets are resolved into an in-memory runtime snapshot.

- Resolution is eager during activation, not lazy on request paths.
- Startup fails fast when an effectively active SecretRef cannot be resolved.
- Reload uses atomic swap: full success, or keep the last-known-good snapshot.
- SecretRef policy violations (for example OAuth-mode auth profiles combined with SecretRef input) fail activation before runtime swap.
- Runtime requests read from the active in-memory snapshot only.
- After the first successful config activation/load, runtime code paths keep reading that active in-memory snapshot until a successful reload swaps it.
- Outbound delivery paths also read from that active snapshot (for example Discord reply/thread delivery and Telegram action sends); they do not re-resolve SecretRefs on each send.

This keeps secret-provider outages off hot request paths.

## Active-surface filtering

SecretRefs are validated only on effectively active surfaces.

- Enabled surfaces: unresolved refs block startup/reload.
- Inactive surfaces: unresolved refs do not block startup/reload.
- Inactive refs emit non-fatal diagnostics with code `SECRETS_REF_IGNORED_INACTIVE_SURFACE`.

    - Disabled channel/account entries.
    - Top-level channel credentials that no enabled account inherits.
    - Disabled tool/feature surfaces.
    - Web search provider-specific keys that are not selected by `tools.web.search.provider`. In auto mode (provider unset), keys are consulted by precedence for provider auto-detection until one resolves. After selection, non-selected provider keys are treated as inactive until selected.
    - Sandbox SSH auth material (`agents.defaults.sandbox.ssh.identityData`, `certificateData`, `knownHostsData`, plus per-agent overrides) is active only when the effective sandbox backend is `ssh` for the default agent or an enabled agent.
    - `gateway.remote.token` / `gateway.remote.password` SecretRefs are active if one of these is true:
      - `gateway.mode=remote`
      - `gateway.remote.url` is configured
      - `gateway.tailscale.mode` is `serve` or `funnel`
      - In local mode without those remote surfaces:
        - `gateway.remote.token` is active when token auth can win and no env/auth token is configured.
        - `gateway.remote.password` is active only when password auth can win and no env/auth password is configured.
    - `gateway.auth.token` SecretRef is inactive for startup auth resolution when `OPENCLAW_GATEWAY_TOKEN` is set, because env token input wins for that runtime.

## Gateway auth surface diagnostics

When a SecretRef is configured on `gateway.auth.token`, `gateway.auth.password`, `gateway.remote.token`, or `gateway.remote.password`, gateway startup/reload logs the surface state explicitly:

- `active`: the SecretRef is part of the effective auth surface and must resolve.
- `inactive`: the SecretRef is ignored for this runtime because another auth surface wins, or because remote auth is disabled/not active.

These entries are logged with `SECRETS_GATEWAY_AUTH_SURFACE` and include the reason used by the active-surface policy, so you can see why a credential was treated as active or inactive.

## Onboarding reference preflight

When onboarding runs in interactive mode and you choose SecretRef storage, OpenClaw runs preflight validation before saving:

- Env refs: validates env var name and confirms a non-empty value is visible during setup.
- Provider refs (`file` or `exec`): validates provider selection, resolves `id`, and checks resolved value type.
- Quickstart reuse path: when `gateway.auth.token` is already a SecretRef, onboarding resolves it before probe/dashboard bootstrap (for `env`, `file`, and `exec` refs) using the same fail-fast gate.

If validation fails, onboarding shows the error 

## Secrets Plan Contract (apply target/path rules)

This page defines the strict contract enforced by `openclaw secrets apply`.

If a target does not match these rules, apply fails before mutating configuration.

## Plan file shape

`openclaw secrets apply --from <plan.json>` expects a `targets` array of plan targets:

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

## Supported target scope

Plan targets are accepted for supported credential paths in:

- [SecretRef Credential Surface](/reference/secretref-credential-surface)

## Target type behavior

General rule:

- `target.type` must be recognized and must match the normalized `target.path` shape.

Compatibility aliases remain accepted for existing plans:

- `models.providers.apiKey`
- `skills.entries.apiKey`
- `channels.googlechat.serviceAccount`

## Path validation rules

Each target is validated with all of the following:

- `type` must be a recognized target type.
- `path` must be a non-empty dot path.
- `pathSegments` can be omitted. If provided, it must normalize to exactly the same path as `path`.
- Forbidden segments are rejected: `__proto__`, `prototype`, `constructor`.
- The normalized path must match the registered path shape for the target type.
- If `providerId` or `accountId` is set, it must match the id encoded in the path.
- `auth-profiles.json` targets require `agentId`.
- When creating a new `auth-profiles.json` mapping, include `authProfileProvider`.

## Failure behavior

If a target fails validation, apply exits wi

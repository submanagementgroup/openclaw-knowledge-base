---
domain: gateway
topic: "Model Provider Authentication"
type: reference
keywords:
  - API key
  - OAuth
  - auth-profiles.json
  - models auth
  - claude-cli
  - setup-token
  - models status
  - credential rotation
source: gateway/authentication.md
related:
  - gateway/secrets
  - concepts/oauth
---

OpenClaw supports OAuth and API keys for model provider authentication. Configure credentials via `auth-profiles.json`, environment variables, or `openclaw models auth` commands. For gateway hosts, API keys are the most predictable option.

## Recommended API Key Setup

Store your API key on the gateway host and verify with `openclaw models status`:

```bash
export ANTHROPIC_API_KEY="..."
openclaw models status
```

For daemons (systemd/launchd), put keys in `~/.openclaw/.env`:

```bash
cat >> ~/.openclaw/.env <<'EOF'
ANTHROPIC_API_KEY=...
EOF
```

Restart and verify: `openclaw models status && openclaw doctor`

## Anthropic Claude CLI Reuse

Anthropic Claude CLI reuse is supported and sanctioned. For long-lived gateway hosts, an Anthropic API key remains the most predictable setup, but you can reuse a local `claude` login:

```bash
# On the gateway host
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

If `claude` is not on `PATH`, set `agents.defaults.cliBackends.claude-cli.command` to the binary path.

Manual token paste for any provider:

```bash
openclaw models auth paste-token --provider openrouter
```

## auth-profiles.json Format

`auth-profiles.json` stores credentials using canonical `version` + `profiles` shape:

```json
{
  "version": 1,
  "profiles": {
    "openrouter:default": {
      "type": "api_key",
      "provider": "openrouter",
      "key": "OPENROUTER_API_KEY"
    }
  }
}
```

Run `openclaw doctor --fix` to migrate legacy flat files. Endpoint details (`baseUrl`, `api`, model ids) belong in `openclaw.json` under `models.providers.<id>`, not in `auth-profiles.json`.

SecretRef-backed credentials: `api_key` profiles support `keyRef: { source, provider, id }`, `token` profiles support `tokenRef`. OAuth-mode profiles do not support SecretRef.

## API Key Rotation (Gateway)

Priority order for key selection:
1. `OPENCLAW_LIVE_<PROVIDER>_KEY`
2. `<PROVIDER>_API_KEYS`
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*`

Rotation only triggers on rate-limit errors (429, `rate_limit`, `quota`, `resource exhausted`, `ThrottlingException`). Non-rate-limit errors are not retried with alternate keys.

## Controlling Active Credentials

Per-session (chat): `/model <alias>@<profileId>` — example: `/model claude-3-5@anthropic:work`

Per-agent (CLI):
```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Use `--agent <id>` to target a specific agent.

Live auth probes:

```bash
openclaw models status --probe
openclaw models status --check  # exits 1=expired/missing, 2=expiring
```

## Troubleshooting Auth

**"No credentials found"**: Configure an API key on the gateway host or set up setup-token, then run `openclaw models status`.

**Token expiring/expired**: Run `openclaw models status` to identify the expiring profile. Refresh via setup-token or migrate to an API key.

Rate-limit cooldowns can be model-scoped — a profile cooling down for one model can still work for sibling models on the same provider.

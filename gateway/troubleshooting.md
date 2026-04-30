---
domain: gateway
topic: "Gateway Troubleshooting: Port Conflicts, Channel Failures, Model Errors, and Diagnostics"
type: troubleshooting
keywords:
  - gateway troubleshooting
  - port conflict
  - channel not connecting
  - openclaw doctor
  - gateway error
  - debug gateway
related:
  - gateway/gateway-runbook
  - gateway/health
  - gateway/diagnostics
source: gateway/troubleshooting.md
---

Gateway troubleshooting: symptom-first diagnostics for common issues including port conflicts, channel connection failures, model errors, and performance problems.

## First Steps

```bash
openclaw gateway status          # overall status
openclaw doctor                  # full health check and diagnostics
openclaw logs --follow           # real-time log tail
openclaw channels status --probe # test channel connections
```

## Port Already In Use

```bash
openclaw gateway --force   # kill existing listener and start fresh
# or
lsof -i :18789             # find the process using the port
```

## Channel Not Connecting

1. Check `openclaw channels status --probe` for error details
2. Verify the bot token / API credentials are correct in config
3. For webhook-based channels (Discord, Slack socket mode): check firewall/outbound connectivity
4. Check channel-specific troubleshooting docs

## Model Errors

```bash
openclaw infer "test"    # quick model smoke test
openclaw models list     # verify model is available
```

## Logs and Diagnostics

```bash
# Enable debug logging
openclaw gateway --verbose

# Check log file
openclaw logs --lines 200

# Doctor
openclaw doctor
```

This page is the deep runbook. Start at [/help/troubleshooting](/help/troubleshooting) if you want the fast triage flow first.

## Command ladder

Run these first, in this order:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Expected healthy signals:

- `openclaw gateway status` shows `Runtime: running`, `Connectivity probe: ok`, and a `Capability: ...` line.
- `openclaw doctor` reports no blocking config/service issues.
- `openclaw channels status --probe` shows live per-account transport status and, where supported, probe/audit results such as `works` or `audit ok`.

## Split brain installs and newer config guard

Use this when a gateway service unexpectedly stops after an update, or logs show that one `openclaw` binary is older than the version that last wrote `openclaw.json`.

OpenClaw stamps config writes with `meta.lastTouchedVersion`. Read-only commands can still inspect a config written by a newer OpenClaw, but process and service mutations refuse to continue from an older binary. Blocked actions include gateway service start, stop, restart, uninstall, forced service reinstall, service-mode gateway startup, and `gateway --force` port cleanup.

```bash
which openclaw
openclaw --version
openclaw gateway status --deep
openclaw config get meta.lastTouchedVersion
```

    Fix `PATH` so `openclaw` resolves to the newer install, then rerun the action.

    Reinstall the intended gateway service from the newer install:

    ```bash
    openclaw gateway install --force
    openclaw gateway restart
    ```

    Remove stale system package or old wrapper entries that still point at an old `openclaw` binary.

For intentional downgrade or emergency recovery only, set `OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1` for the single command. Leave it unset for normal operation.

## Anthropic 429 extra usage required for long context

Use this when logs/errors include: `HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Look for:

- Selected Anthropic Opus/Sonnet model has `params.context1m: true`.
- Current Anthropic credential is not eligible for long-context usage.
- Requests fail only on long sessions/model runs that need the 1M beta path.

Fix options:

    Disable `context1m` for that model to fall back to the normal context window.

    Use an Anthropic credential that is eligible for long-context requests, or switch to an Anthropic API key.

    Configure fallback models so runs continue when Anthropic long-context requests are rejected.

Related:

- [Anthropic](/providers/anthropic)
- [Token use and costs](/reference/token-use)
- [Why am I seeing HTTP 429 from Anthropic?](/help/faq-first-run#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Local OpenAI-compatible backend passes direct probes but agent runs fail

Use this when:

- `curl ... /v1/models` works
- tiny direct `/v1/chat/completions` calls work
- OpenClaw model runs fail only on normal agent turns

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

Look for:

- direct tiny calls succeed, but OpenClaw runs fail only on larger prompts
- `model_not_found` or 404 errors even though direct `/v1/chat/completions`
  works with the same bare model id
- backend errors about `messages[].content` expecting a string
- intermittent `incomplete turn detected ... stopReason=stop payloads=0` warnings with an OpenAI-compatible local backend
- backend crashes that appear only with larger prompt-token counts or full agent runtime prompts

    - `model_not_found` with a local MLX/vLLM-style server → verify `baseUrl` includes `/v1`, `api` is `"openai-completions"` for `/v1/chat/completions` backends, and `models.providers.<provider>.models[].id` is the bare provider-local id. Select it with the provider prefix once, for example `mlx/mlx-community/Qwen3-30B-A3B-6bit`; keep the catalog entry as `mlx-community/Qwen3-30B-A3B-6bit`.
    - `messages[...].content: invalid type: sequence, expected a string` → backend rejects structured Chat Completions content parts. Fix: set `models.providers.<provider>.models[].compat.requiresStringContent: true`.
    - `incomplete turn detected ... stopReason=stop payloads=0` → the backend completed the Chat Completions request but returned no user-visible assistant text for that turn. OpenClaw retries replay-safe empty OpenAI-compatible turns once; persistent failures usually mean the backend is emitting empty/non-text content or suppressing final-answer text.
    - direct tiny requests succeed, but OpenClaw agent runs fail with backend/model crashes (for example Gemma on some `inferrs` builds) → OpenClaw transport is likely already correct; the backend is failing on the larger agent-runtime prompt shape.
    - failures shrink after disabling tools but do not disappear → tool schemas were part of the pressure, but the remaining issue is still upstream model/server capacity or a backend bug.

    1. Set `compat.requiresStringContent: true` for string-only Chat Completions backends.
    2. Set `compat.supportsTools: false` for models/backends that cannot handle OpenClaw's tool schema surface reliably.
    3. Lower prompt pressure where possible: smaller workspace bootstrap, shorter session history, lighter local model, or a backend with stronger long-context support.
    4. If tiny direct requests keep passing while OpenClaw agent turns still crash inside the backend, treat it as an upstream server/model limitation and file a repro there with the accepted payload shape.

Related:

- [Configuration](/gateway/configuration)
- [Local models](

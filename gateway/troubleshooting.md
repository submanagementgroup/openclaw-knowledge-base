---
domain: gateway
topic: "Gateway Troubleshooting Guide"
type: troubleshooting
keywords:
  - troubleshooting
  - gateway errors
  - startup failure
  - connection issues
  - debug
  - logs
  - openclaw doctor
source: gateway/troubleshooting.md
---

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

**Fix PATH**

Fix `PATH` so `openclaw` resolves to the newer install, then rerun the action.
  
**Reinstall the gateway service**

Reinstall the intended gateway service from the newer install:

    ```bash
    openclaw gateway install --force
    openclaw gateway restart
    ```

  
**Remove stale wrappers**

Remove stale system package or old wrapper entries that still point at an old `openclaw` binary.
  
> **Note:** For intentional downgrade or emergency recovery only, set `OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1` for the single command. Leave it unset for normal operation.


## Skill symlink skipped as path escape

Use this when logs include:

```text
Skipping escaped skill path outside its configured root: ... reason=symlink-escape
```

OpenClaw treats every skill root as a containment boundary. A symlink under
`~/.agents/skills`, `<workspace>/.agents/skills`, `<workspace>/skills`, or
`~/.openclaw/skills` is skipped when its real target resolves outside that root
unless the target is explicitly trusted.

Inspect the link:

```bash
ls -l ~/.agents/skills/<name>
realpath ~/.agents/skills/<name>
openclaw config get skills.load
```

If the target is intentional, configure both the direct skill root and the
allowed symlink target:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

Then start a new session or wait for the skills watcher to refresh. Restart the
gateway if the running process predates the config change.

Do not use broad targets such as `~`, `/`, or a whole synced project folder.
Keep `allowSymlinkTargets` scoped to the real skill root that contains trusted
`SKILL.md` directories.

Related:

- [Skills config](/tools/skills-config#symlinked-sibling-repos)
- [Configuration examples](/gateway/configuration-examples#symlinked-sibling-skill-repo)

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

**Disable context1m**

Disable `context1m` for that model to fall back to the normal context window.
  
**Use an eligible credential**

Use an Anthropic credential that is eligible for long-context requests, or switch to an Anthropic API key.
  
**Configure fallback models**

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

### Common signatures

- `model_not_found` with a local MLX/vLLM-style server → verify `baseUrl` includes `/v1`, `api` is `"openai-completions"` for `/v1/chat/completions` backends, and `models.providers.<provider>.models[].id` is the bare provider-local id. Select it with the provider prefix once, for example `mlx/mlx-community/Qwen3-30B-A3B-6bit`; keep the catalog entry as `mlx-community/Qwen3-30B-A3B-6bit`.
    - `messages[...].content: invalid type: sequence, expected a string` → backend rejects structured Chat Completions content parts. Fix: set `models.providers.<provider>.models[].compat.requiresStringContent: true`.
    - `validation.keys` or allowed message keys like `["role","content"]` → backend rejects OpenAI-style replay metadata on Chat Completions messages. Fix: set `models.providers.<provider>.models[].compat.strictMessageKeys: true`.
    - `incomplete turn detected ... stopReason=stop payloads=0` → the backend completed the Chat Completions request but returned no user-visible assistant text for that turn. OpenClaw retries replay-safe empty OpenAI-compatible turns once; persistent failures usually mean the backend is emitting empty/non-text content or suppressing final-answer text.
    - direct tiny requests succeed, but OpenClaw agent runs fail with backend/model crashes (for example Gemma on some `inferrs` builds) → OpenClaw transport is likely already correct; the backend is failing on the larger agent-runtime prompt shape.
    - failures shrink after disabling tools but do not disappear → tool schemas were part of the pressure, but the remaining issue is still upstream model/server capacity or a backend bug.

  ### Fix options

1. Set `compat.requiresStringContent: true` for string-only Chat Completions backends.
    2. Set `compat.strictMessageKeys: true` for strict Chat Completions backends that only accept `role` and `content` on each message.
    3. Set `compat.supportsTools: false` for models/backends that cannot handle OpenClaw's tool schema surface reliably.
    4. Lower prompt pressure where possible: smaller workspace bootstrap, shorter session history, lighter local model, or a backend with stronger long-context support.
    5. If tiny direct requests keep passing while OpenClaw agent turns still crash inside the backend, treat it as an upstream server/model limitation and file a repro there with the accepted payload shape.
  Related:

- [Configuration](/gateway/configuration)
- [Local models](/gateway/local-models)
- [OpenAI-compatible endpoints](/gateway/configuration-reference#openai-compatible-endpoints)

## No replies

If channels are up but nothing answers, check routing and policy before reconnecting anything.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Look for:

- Pairing pending for DM senders.
- Group mention gating (`requireMention`, `mentionPatterns`).
- Channel/group allowlist mismatches.

Common signatures:

- `drop guild message (mention required` → group message ignored until mention.
- `pairing request` → sender needs approval.
- `blocked` / `allowlist` → sender/channel was filtered by policy.

Related:

- [Channel troubleshooting](/channels/troubleshooting)
- [Groups](/channels/groups)
- [Pairing](/channels/pairing)

## Dashboard control UI connectivity

When dashboard/control UI will not connect, validate URL, auth mode, and secure context assumptions.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Look for:

- Correct probe URL and dashboard URL.
- Auth mode/token mismatch between client and gateway.
- HTTP usage where device identity is required.

### Connect / auth signatures

- `device identity required` → non-secure context or missing device auth.
    - `origin not allowed` → browser `Origin` is not in `gateway.controlUi.allowedOrigins` (or you are connecting from a non-loopback browser origin without an explicit allowlist).
    - `device nonce required` / `device nonce mismatch` → client is not completing the challenge-based device auth flow (`connect.challenge` + `device.nonce`).
    - `device signature invalid` / `device signature expired` → client signed the wrong payload (or stale timestamp) for the current handshake.
    - `AUTH_TOKEN_MISMATCH` with `canRetryWithDeviceToken=true` → client can do one trusted retry with cached device token.
    - That cached-token retry reuses the cached scope set stored with the paired device token. Explicit `deviceToken` / explicit `scopes` callers keep their requested scope set instead.
    - `AUTH_SCOPE_MISMATCH` → the device token was recognized, but its approved scopes do not cover this connect request; re-pair or approve the requested scope contract instead of rotating a shared gateway token.
    - Outside that retry path, connect auth precedence is explicit shared token/password first, then explicit `deviceToken`, then stored device token, then bootstrap token.
    - On the async Tailscale Serve Control UI path, failed attempts for the same `{scope, ip}` are serialized before the limiter records the failure. Two bad concurrent retries from the same client can therefore surface `retry later` on the second attempt instead of two plain mismatches.
    - `too many failed authentication attempts (retry later)` from a browser-origin loopback client → repeated failures from that same normalized `Origin` are locked out temporarily; another localhost origin uses a separate bucket.
    - repeated `unauthorized` after that retry → shared token/device token drift; refresh token config and re-approve/rotate device token if needed.
    - `gateway connect failed:` → wrong host/port/url target.

  ### Auth detail codes quick map

Use `error.details.code` from the failed `connect` response to pick the next action:

| Detail code                  | Meaning                                                                                                                                                                                      | Recommended action                                                                                                                                                                                                                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | Client did not send a required shared token.                                                                                                                                                 | Paste/set token in the client and retry. For dashboard paths: `openclaw config get gateway.auth.token` then paste into Control UI settings.                                                                                                                                              |
| `AUTH_TOKEN_MISMATCH`        | Shared token did not match gateway auth token.                                                                                                                                               | If `canRetryWithDeviceToken=true`, allow one trusted retry. Cached-token retries reuse stored approved scopes; explicit `deviceToken` / `scopes` callers keep requested scopes. If still failing, run the [token drift recovery checklist](/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | Cached per-device token is stale or revoked.                                                                                                                                                 | Rotate/re-approve device token using [devices CLI](/cli/devices), then reconnect.                                                                                                                                                                                                        |
| `AUTH_SCOPE_MISMATCH`        | Device token is valid, but its approved role/scopes do not cover this connect request.                        
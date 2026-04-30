---
domain: troubleshooting
topic: "Live Testing: Testing with Real Providers, Models, and Channels"
type: procedure
keywords:
  - live testing
  - real provider tests
  - live tests
  - network tests
  - provider testing
related:
  - troubleshooting/testing
source: help/testing-live.md
---

Live testing with real providers and models.

For quick start, QA runners, unit/integration suites, and Docker flows, see
[Testing](/help/testing). This page covers the **live** (network-touching) test
suites: model matrix, CLI backends, ACP, and media-provider live tests, plus
credential handling.

## Live: local profile smoke commands

Source `~/.profile` before ad hoc live checks so provider keys and local tool
paths match your shell:

```bash
source ~/.profile
```

Safe media smoke:

```bash
pnpm openclaw infer tts convert --local --json \
  --text "OpenClaw live smoke." \
  --output /tmp/openclaw-live-smoke.mp3
```

Safe voice-call readiness smoke:

```bash
pnpm openclaw voicecall setup --json
pnpm openclaw voicecall smoke --to "+15555550123"
```

`voicecall smoke` is a dry run unless `--yes` is also present. Use `--yes` only
when you intentionally want to place a real notify call. For Twilio, Telnyx, and
Plivo, a successful readiness check requires a public webhook URL; local-only
loopback/private fallbacks are rejected by design.

## Live: Android node capability sweep

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Script: `pnpm android:test:integration`
- Goal: invoke **every command currently advertised** by a connected Android node and assert command contract behavior.
- Scope:
  - Preconditioned/manual setup (the suite does not install/run/pair the app).
  - Command-by-command gateway `node.invoke` validation for the selected Android node.
- Required pre-setup:
  - Android app already connected + paired to the gateway.
  - App kept in foreground.
  - Permissions/capture consent granted for capabilities you expect to pass.
- Optional target overrides:
  - `OPENCLAW_ANDROID_NODE_ID` or `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Full Android setup details: [Android App](/platforms/android)

## Live: model smoke (profile keys)

Live tests are split into two layers so we can isolate failures:

- “Direct model” tells us the provider/model can answer at all with the given key.
- “Gateway smoke” tells us the full gateway+agent pipeline works for that model (sessions, history, tools, sandbox policy, etc.).

### Layer 1: Direct model completion (no gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Goal:
  - Enumerate discovered models
  - Use `getApiKeyForModel` to select models you have creds for
  - Run a small completion per model (and targeted regressions where needed)
- How to enable:
  - `pnpm test:live` (or `OPENCLAW_LIVE_TEST=1` if invoking Vitest directly)
- Set `OPENCLAW_LIVE_MODELS=modern` (or `all`, alias for modern) to actually run this suite; otherwise it skips to keep `pnpm test:live` focused on gateway smoke
- How to select models:
  - `OPENCLAW_LIVE_MODELS=modern` to run the modern allowlist (Opus/Sonnet 4.6+, GPT-5.2 + Codex, Gemini 3, DeepSeek V4, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` is an alias for the modern allowlist
  - or `OPENCLAW_LIVE_MODELS="openai/gpt-5.5,openai-codex/gpt-5.5,anthropic/claude-opus-4-6,..."` (comma allowlist)
  - Modern/all sweeps default to a curated high-signal cap; set `OPENCLAW_LIVE_MAX_MODELS=0` for an exhaustive modern sweep or a positive number for a smaller cap.
  - Exhaustive sweeps use `OPENCLAW_LIVE_TEST_TIMEOUT_MS` for the whole direct-model test timeout. Default: 60 minutes.
  - Direct-model probes run with 20-way parallelism by default; set `OPENCLAW_LIVE_MODEL_CONCURRENCY` to override.
- How to select providers:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (comma allowlist)
- Where keys come from:
  - By default: profile store and env fallbacks
  - Set `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` to enforce **profile store** only
- Why this exists:
  - Separates “provider API is broken / key is invalid” from “gateway agent pipeline is broken”
  - Contains small, isolated regressions (example: OpenAI Responses/Codex Responses reasoning replay + tool-call flows)

### Layer 2: Gateway + dev agent smoke (what "@openclaw" actually does)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Goal:
  - Spin up an in-process gateway
  - Create/patch a `agent:dev:*` session (model override per run)
  - Iterate models-with-keys and assert:
    - “meaningful” response (no tools)
    - a real tool invocation works (read probe)
    - optional extra tool probes (exec+read probe)
    - OpenAI regression paths (tool-call-only → follow-up) keep working
- Probe details (so you can explain failures quickly):
  - `read` probe: the test writes a nonce file in the workspace and asks the agent to `read` it and echo the nonce back.
  - `exec+read` probe: the test asks the agent to `exec`-write a nonce into a temp file, then `read` it back.
  - image probe: the test attaches a generated PNG (cat + randomized code) and expects the model to return `cat <CODE>`.
  - Implementation reference: `src/gateway/gateway-models.profiles.live.test.ts` and `src/gateway/live-image-probe.ts`.
- How to enable:
  - `pnpm test:live` (or `OPENCLAW_LIVE_TEST=1` if invoking Vitest directly)
- How to select models:
  - Default: modern allowlist (Opus/Sonnet 4.6+, GPT-5.2 + Codex, Gemini 3, DeepSeek V4, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` is an alias for the modern allowlist
  - Or set `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (or comma list) to narrow
  - Modern/all gateway sweeps default to a curated high-signal cap; set `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` for an exhaustive modern sweep or a positive number for a smaller cap.
- How to select providers (avoid “OpenRouter everything”):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (comma allowlist)
- Tool + image probes are always on in this live test:
  - `read` probe + `exec+read` probe (tool stress)
  - image probe runs when the model advertises image input support
  - Flow (high level):
    - Test generates a tiny PNG with “CAT” + random code (`src/gateway/live-image-probe.ts`)
    - Sends it via `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway parses attachments into `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Embedded agent forwards a multimodal user message to the model
    - Assertion: reply contains `cat` + the code (OCR tolerance: minor mistakes allowed)

To see what you can test on your machine (and the exact `provider/model` ids), run:

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI backend smoke (Claude, Codex, Gemini, or other local CLIs)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Goal: validate the Gateway + agent pipeline using a local CLI backend, without touching your default config.
- Backend-specific smoke defaults live with the owning extension's `cli-backend.ts` definition.
- Enable:
  - `pnpm test:live` (or `OPENCLAW_LIVE_TEST=1` if invoking Vitest directly)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Defaults:
  - Default provider/model: `claude-cli/claude-sonnet-4-6`
  - Command/args/image behavior come from the owning CLI backend plugin metadata.
- Overrides (optional):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.5"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` to send a real image attachment (paths are injected into the prompt). Docker recipes default this off unless explicitly requested.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` to pass image file paths as CLI args instead of prompt injection.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (or `"list"`) to control how image args are passed when `IMAGE_ARG` is set.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` to send a second turn and validate resume flow.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=1` to opt into the Claude Sonnet -> Opus same-session continuity probe when the selected model supports a switch target. Docker recipes default this off for aggregate reliability.
  - `OPENCLAW_LIVE_CLI_BACKEND_MCP_PROBE=1` to opt into the MCP/tool loopback probe. Docker recipes default this off unless explicitly requested.

Example:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.5" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Cheap Gemini MCP config smoke:

```bash
OPENCLAW_LIVE_TEST=1 \
  pnpm test:live src/agents/cli-runner/bundle-mcp.gemini.live.test.ts
```

This does not ask Gemini to generate a response. It writes the same system
settings OpenClaw gives Gemini, then runs `gemini --debug mcp list` to prove a
saved `transport: "streamable-http"` server is normalized to Gemini's HTTP MCP
shape and can connect to a local streamable-HTTP MCP server.

Dock

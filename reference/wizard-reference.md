---
domain: reference
topic: "Wizard Configuration Reference: All Onboarding Wizard Fields and Automation Options"
type: reference
keywords:
  - wizard reference
  - onboard wizard
  - wizard config
  - wizard fields
related:
  - getting-started/onboarding-wizard
source: reference/wizard.md
---

Onboarding wizard configuration reference: all wizard fields and automation options.

This is the full reference for `openclaw onboard`.
For a high-level overview, see [Onboarding (CLI)](/start/wizard).

## Flow details (local mode)

    - If `~/.openclaw/openclaw.json` exists, choose **Keep / Modify / Reset**.
    - Re-running onboarding does **not** wipe anything unless you explicitly choose **Reset**
      (or pass `--reset`).
    - CLI `--reset` defaults to `config+creds+sessions`; use `--reset-scope full`
      to also remove workspace.
    - If the config is invalid or contains legacy keys, the wizard stops and asks
      you to run `openclaw doctor` before continuing.
    - Reset uses `trash` (never `rm`) and offers scopes:
      - Config only
      - Config + credentials + sessions
      - Full reset (also removes workspace)

    - **Anthropic API key**: uses `ANTHROPIC_API_KEY` if present or prompts for a key, then saves it for daemon use.
    - **Anthropic API key**: preferred Anthropic assistant choice in onboarding/configure.
    - **Anthropic setup-token**: still available in onboarding/configure, though OpenClaw now prefers Claude CLI reuse when available.
    - **OpenAI Code (Codex) subscription (OAuth)**: browser flow; paste the `code#state`.
      - Sets `agents.defaults.model` to `openai-codex/gpt-5.5` when model is unset or already OpenAI-family.
    - **OpenAI Code (Codex) subscription (device pairing)**: browser pairing flow with a short-lived device code.
      - Sets `agents.defaults.model` to `openai-codex/gpt-5.5` when model is unset or already OpenAI-family.
    - **OpenAI API key**: uses `OPENAI_API_KEY` if present or prompts for a key, then stores it in auth profiles.
      - Sets `agents.defaults.model` to `openai/gpt-5.5` when model is unset, `openai/*`, or `openai-codex/*`.
    - **xAI (Grok) API key**: prompts for `XAI_API_KEY` and configures xAI as a model provider.
    - **OpenCode**: prompts for `OPENCODE_API_KEY` (or `OPENCODE_ZEN_API_KEY`, get it at https://opencode.ai/auth) and lets you pick the Zen or Go catalog.
    - **Ollama**: offers **Cloud + Local**, **Cloud only**, or **Local only** first. `Cloud only` prompts for `OLLAMA_API_KEY` and uses `https://ollama.com`; the host-backed modes prompt for the Ollama base URL, discover available models, and auto-pull the selected local model when needed; `Cloud + Local` also checks whether that Ollama host is signed in for cloud access.
    - More detail: [Ollama](/providers/ollama)
    - **API key**: stores the key for you.
    - **Vercel AI Gateway (multi-model proxy)**: prompts for `AI_GATEWAY_API_KEY`.
    - More detail: [Vercel AI Gateway](/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: prompts for Account ID, Gateway ID, and `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    - More detail: [Cloudflare AI Gateway](/providers/cloudflare-ai-gateway)
    - **MiniMax**: config is auto-written; hosted default is `MiniMax-M2.7`.
      API-key setup uses `minimax/...`, and OAuth setup uses
      `minimax-portal/...`.
    - More detail: [MiniMax](/providers/minimax)
    - **StepFun**: config is auto-written for StepFun standard or Step Plan on China or global endpoints.
    - Standard currently includes `step-3.5-flash`, and Step Plan also includes `step-3.5-flash-2603`.
    - More detail: [StepFun](/providers/stepfun)
    - **Synthetic (Anthropic-compatible)**: prompts for `SYNTHETIC_API_KEY`.
    - More detail: [Synthetic](/providers/synthetic)
    - **Moonshot (Kimi K2)**: config is auto-written.
    - **Kimi Coding**: config is auto-written.
    - More detail: [Moonshot AI (Kimi + Kimi Coding)](/providers/moonshot)
    - **Skip**: no auth configured yet.
    - Pick a default model from detected options (or enter provider/model manually). For best quality and lower prompt-injection risk, choose the strongest latest-generation model available in your provider stack.
    - Onboarding runs a model check and warns if the configured model is unknown or missing auth.
    - API key storage mode defaults to plaintext auth-profile values. Use `--secret-input-mode ref` to store env-backed refs instead (for example `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`).
    - Auth profiles live in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (API keys + OAuth). `~/.openclaw/credentials/oauth.json` is legacy import-only.
    - More detail: [/concepts/oauth](/concepts/oauth)

    Headless/server tip: complete OAuth on a machine with a browser, then copy
    that agent's `auth-profiles.json` (for example
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, or the matching
    `$OPENCLAW_STATE_DIR/...` path) to the gateway host. `credentials/oauth.json`
    is only a legacy import source.

    - Default `~/.openclaw/workspace` (configurable).
    - Seeds the workspace files needed for the agent bootstrap ritual.
    - Full workspace layout + backup guide: [Agent workspace](/concepts/agent-workspace)

    - Port, bind, auth mode, tailscale exposure.
    - Auth recommendation: keep **Token** even for loopback so local WS clients must authenticate.
    - In token mode, interactive setup offers:
      - **Generate/store plaintext token** (default)
      - **Use SecretRef** (opt-in)
      - Quickstart reuses existing `gateway.auth.token` SecretRefs across `env`, `file`, and `exec` providers for onboarding probe/dashboard bootstrap.
      - If that SecretRef is configured but cannot be resolved, onboarding fails early with a clear fix message instead of silently degrading runtime auth.
    - In password mode, interactive setup also supports plaintext or SecretRef storage.
    - Non-interactive token SecretRef path: `--gateway-token-ref-env <ENV_VAR>`.
      - Requires a non-empty env var in the onboarding process environment.
      - Cannot be combined with `--gateway-token`.
    - Disable auth only if you fully trust every local process.
    - Non‑loopback binds still require auth.

    - [WhatsApp](/channels/whatsapp): optional QR login.
    - [Telegram](/channels/telegram): bot token.
    - [Discord](/channels/discord): bot token.
    - [Google Chat](/channels/googlechat): service account JSON + webhook audience.
    - [Mattermost](/channels/mattermost) (plugin): bot token + base URL.
    - [Signal](/channels/signal): optional `signal-cli` install + account config.
    - [BlueBubbles](/channels/bluebubbles): **recommended for iMessage**; server URL + password + webhook.
    - [iMessage](/channels/imessage): legacy `imsg` CLI path + DB access.
    - DM security: default is pairing. First DM sends a code; approve via `openclaw pairing approve <channel> <code>` or use allowlists.

    - Pick a supported provider such as Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, or Tavily (or skip).
    - API-backed providers can use env vars or existing config for quick setup; key-free providers use their provider-specific prerequisites instead.
    - Skip with `--skip-search`.
    - Configure later: `openclaw configure --section web`.

    - macOS: LaunchAgent
      - Requires a logged-in user session; for headless, use a custom LaunchDaemon (not shipped).
    - Linux (and Windows via WSL2): systemd user unit
      - Onboarding attempts to enable lingering via `loginctl enable-linger <user>` so the Gateway stays up after logout.
      - May prompt for sudo (writes `/var/lib/systemd/linger`); it tries without sudo first.
    - **Runtime selection:** Node (recommended; required for WhatsApp/Telegram). Bun is **not recommended**.
    - If token auth requires a token and `gateway.auth.token` is SecretRef-managed, daemon install validates it but does not persist resolved plaintext token values into supervisor service environment metadata.
    - If token auth requires a token and the configured token SecretRef is unresolved, daemon install is blocked with actionable guidance.
    - If both `gateway.auth.token` and `gateway.auth.password` are configured and `gateway.auth.mode` is unset, da

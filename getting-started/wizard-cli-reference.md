---
domain: getting-started
topic: "Getting Started: Wizard Cli Reference"
type: procedure
keywords:
  - wizard-cli-reference
  - getting started
  - setup
  - quickstart
source: start/wizard-cli-reference.md
---

This page is the full reference for `openclaw onboard`.
For the short guide, see [Onboarding (CLI)](/start/wizard).

## What the wizard does

Local mode (default) walks you through:

- Model and auth setup (OpenAI Code subscription OAuth, Anthropic Claude CLI or API key, plus MiniMax, GLM, Ollama, Moonshot, StepFun, and AI Gateway options)
- Workspace location and bootstrap files
- Gateway settings (port, bind, auth, tailscale)
- Channels and providers (Telegram, WhatsApp, Discord, Google Chat, Mattermost, Signal, iMessage, and other bundled channel plugins)
- Daemon install (LaunchAgent, systemd user unit, or native Windows Scheduled Task with Startup-folder fallback)
- Health check
- Skills setup

Remote mode configures this machine to connect to a gateway elsewhere.
It does not install or modify anything on the remote host.

## Local flow details

**Existing config detection**

- If `~/.openclaw/openclaw.json` exists, choose Keep, Modify, or Reset.
    - Re-running the wizard does not wipe anything unless you explicitly choose Reset (or pass `--reset`).
    - CLI `--reset` defaults to `config+creds+sessions`; use `--reset-scope full` to also remove workspace.
    - If config is invalid or contains legacy keys, the wizard stops and asks you to run `openclaw doctor` before continuing.
    - Reset uses `trash` and offers scopes:
      - Config only
      - Config + credentials + sessions
      - Full reset (also removes workspace)

  
**Model and auth**

- Full option matrix is in [Auth and model options](#auth-and-model-options).

  
**Workspace**

- Default `~/.openclaw/workspace` (configurable).
    - Seeds workspace files needed for first-run bootstrap ritual.
    - Workspace layout: [Agent workspace](/concepts/agent-workspace).

  
**Gateway**

- Prompts for port, bind, auth mode, and tailscale exposure.
    - Recommended: keep token auth enabled even for loopback so local WS clients must authenticate.
    - In token mode, interactive setup offers:
      - **Generate/store plaintext token** (default)
      - **Use SecretRef** (opt-in)
    - In password mode, interactive setup also supports plaintext or SecretRef storage.
    - Non-interactive token SecretRef path: `--gateway-token-ref-env `.
      - Requires a non-empty env var in the onboarding process environment.
      - Cannot be combined with `--gateway-token`.
    - Disable auth only if you fully trust every local process.
    - Non-loopback binds still require auth.

  
**Channels**

- [WhatsApp](/channels/whatsapp): optional QR login
    - [Telegram](/channels/telegram): bot token
    - [Discord](/channels/discord): bot token
    - [Google Chat](/channels/googlechat): service account JSON + webhook audience
    - [Mattermost](/channels/mattermost): bot token + base URL
    - [Signal](/channels/signal): optional `signal-cli` install + account config
    - [iMessage](/channels/imessage): `imsg` CLI path + Messages DB access; use an SSH wrapper when the Gateway runs off-Mac
    - DM security: default is pairing. First DM sends a code; approve via
      `openclaw pairing approve <channel> <code>` or use allowlists.
  
**Daemon install**

- macOS: LaunchAgent
      - Requires logged-in user session; for headless, use a custom LaunchDaemon (not shipped).
    - Linux and Windows via WSL2: systemd user unit
      - Wizard attempts `loginctl enable-linger <user>` so gateway stays up after logout.
      - May prompt for sudo (writes `/var/lib/systemd/linger`); it tries without sudo first.
    - Native Windows: Scheduled Task first
      - If task creation is denied, OpenClaw falls back to a per-user Startup-folder login item and starts the gateway immediately.
      - Scheduled Tasks remain preferred because they provide better supervisor status.
    - Runtime selection: Node (recommended; required for WhatsApp and Telegram). Bun is not recommended.

  
**Health check**

- Starts gateway (if needed) and runs `openclaw health`.
    - `openclaw status --deep` adds the live gateway health probe to status output, including channel probes when supported.

  
**Skills**

- Reads available skills and checks requirements.
    - Lets you choose node manager: npm, pnpm, or bun.
    - Installs optional dependencies (some use Homebrew on macOS).

  
**Finish**

- Summary and next steps, including iOS, Android, and macOS app options.

  
> **Note:** If no GUI is detected, the wizard prints SSH port-forward instructions for the Control UI instead of opening a browser.
If Control UI assets are missing, the wizard attempts to build them; fallback is `pnpm ui:build` (auto-installs UI deps).


## Remote mode details

Remote mode configures this machine to connect to a gateway elsewhere.

> **Note:** Remote mode does not install or modify anything on the remote host.


What you set:

- Remote gateway URL (`ws://...`)
- Token if remote gateway auth is required (recommended)

> **Note:** - If gateway is loopback-only, use SSH tunneling or a tailnet.
- Discovery hints:
  - macOS: Bonjour (`dns-sd`)
  - Linux: Avahi (`avahi-browse`)



## Auth and model options

### Anthropic API key

Uses `ANTHROPIC_API_KEY` if present or prompts for a key, then saves it for daemon use.
  ### OpenAI Code subscription (OAuth)

Browser flow; paste `code#state`.

    Sets `agents.defaults.model` to `openai/gpt-5.5` through the Codex runtime when model is unset or already OpenAI-family.

  ### OpenAI Code subscription (device pairing)

Browser pairing flow with a short-lived device code.

    Sets `agents.defaults.model` to `openai/gpt-5.5` through the Codex runtime when model is unset or already OpenAI-family.

  ### OpenAI API key

Uses `OPENAI_API_KEY` if present or prompts for a key, then stores the credential in auth profiles.

    Sets `agents.defaults.model` to `openai/gpt-5.5` when model is unset, `openai/*`, or `openai-codex/*`.

  ### xAI (Grok) API key

Prompts for `XAI_API_KEY` and configures xAI as a model provider.
  ### OpenCode

Prompts for `OPENCODE_API_KEY` (or `OPENCODE_ZEN_API_KEY`) and lets you choose the Zen or Go catalog.
    Setup URL: [opencode.ai/auth](https://opencode.ai/auth).
  ### API key (generic)

Stores the key for you.
  ### Vercel AI Gateway

Prompts for `AI_GATEWAY_API_KEY`.
    More detail: [Vercel AI Gateway](/providers/vercel-ai-gateway).
  ### Cloudflare AI Gateway

Prompts for account ID, gateway ID, and `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    More detail: [Cloudflare AI Gateway](/providers/cloudflare-ai-gateway).
  ### MiniMax

Config is auto-written. Hosted default is `MiniMax-M2.7`; API-key setup uses
    `minimax/...`, and OAuth setup uses `minimax-portal/...`.
    More detail: [MiniMax](/providers/minimax).
  ### StepFun

Config is auto-written for StepFun standard or Step Plan on China or global endpoints.
    Standard currently includes `step-3.5-flash`, and Step Plan also includes `step-3.5-flash-2603`.
    More detail: [StepFun](/providers/stepfun).
  ### Synthetic (Anthropic-compatible)

Prompts for `SYNTHETIC_API_KEY`.
    More detail: [Synthetic](/providers/synthetic).
  ### Ollama (Cloud and local open models)

Prompts for `Cloud + Local`, `Cloud only`, or `Local only` first.
    `Cloud only` uses `OLLAMA_API_KEY` with `https://ollama.com`.
    The host-backed modes prompt for base URL (default `http://127.0.0.1:11434`), discover available models, and suggest defaults.
    `Cloud + Local` also checks whether that Ollama host is signed in for cloud access.
    More detail: [Ollama](/providers/ollama).
  ### Moonshot and Kimi Coding

Moonshot (Kimi K2) and Kimi Coding configs are auto-written.
    More detail: [Moonshot AI (Kimi + Kimi Coding)](/providers/moonshot).
  ### Custom provider

Works with OpenAI-compatible and Anthropic-compatible endpoints.

    Interactive onboarding supports the same API key storage choices as other provider API key flows:
    - **Paste API key now** (plaintext)
    - **Use secret reference** (env ref or configured provider ref, with preflight validation)

    Non-interactive flags:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key` (optional; falls back to `CUSTOM_API_KEY`)
    - `--custom-provider-id` (optional)
    - `--custom-compatibility <openai|anthropic>` (optional; default `openai`)
    - `--custom-image-input` / `--custom-text-input` (optional; override inferred model input capability)

  ### Skip

Leaves auth unconfigured.
  Model behavior:

- Pick default model from detected options, or enter provider and model manually.
- Custom-provider onboarding infers image support for common model IDs and asks only when the model name is unknown.
- When onboarding starts from a provider auth choice, the model picker prefers
  that provider automatically. For Volcengine and BytePlus, the same preference
  also matches their coding-plan variants (`volcengine-plan/*`,
  `byteplus-plan/*`).
- If that preferred-provider filter would be empty, the picker falls back to
  the full catalog instead of showing no models.
- Wizard runs a model check and warns if the configured model is unknown or missing auth.

Credential and profile paths:

- Auth profiles (API keys + OAuth): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Legacy OAuth import: `~/.openclaw/credentials/oauth.json`

Credential storage mode:

- Default onboarding behavior persists API keys as plaintext values in auth profiles.
- `--secret-input-mode ref` enables reference mode instead of plaintext key storage.
  In interactive setup, you can choose either:
  - environment variable ref (for example `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - configured provider ref (`file` or `exec`) with provider alias + id
- Interactive reference mode runs a fast preflight validation before saving.
  - Env refs: validates variable name + non-empty value in the current onboarding environment.
  - Provider refs: validates provider config and resolves the requested id.
  - If prefl
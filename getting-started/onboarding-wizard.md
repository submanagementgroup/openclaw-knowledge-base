---
domain: getting-started
topic: "OpenClaw Onboarding Wizard: openclaw onboard Interactive Setup"
type: procedure
keywords:
  - openclaw onboard
  - wizard
  - setup wizard
  - configure
  - interactive setup
  - daemon install
related:
  - getting-started/quickstart
  - gateway/gateway-runbook
  - gateway/configuration-overview
source:
  - start/wizard.md
  - start/wizard-cli-reference.md
  - start/onboarding.md
---

The `openclaw onboard` wizard is the recommended way to set up OpenClaw for the first time. It walks through provider credentials, channel connections, daemon installation, and workspace setup interactively.

## Running the Wizard

```bash
# Full interactive onboarding
openclaw onboard

# Onboard and install as a system daemon
openclaw onboard --install-daemon

# Config wizard only (re-run anytime)
openclaw configure
```

## What the Wizard Does

1. Checks Node version and system requirements
2. Prompts for an AI provider API key (OpenAI, Anthropic, Google, Ollama, etc.)
3. Sets a default model
4. Optionally connects channels (Telegram fastest, WhatsApp requires phone)
5. Optionally installs the Gateway as a system daemon (launchd/systemd)
6. Summarizes config written to `~/.openclaw/openclaw.json`

## Wizard CLI Reference (Automation)

The wizard can be driven non-interactively:
```bash
openclaw onboard --provider openai --api-key $OPENAI_API_KEY --no-daemon
```

CLI onboarding is the **recommended** way to set up OpenClaw on macOS,
Linux, or Windows (via WSL2; strongly recommended).
It configures a local Gateway or a remote Gateway connection, plus channels, skills,
and workspace defaults in one guided flow.

```bash
openclaw onboard
```

Fastest first chat: open the Control UI (no channel setup needed). Run
`openclaw dashboard` and chat in the browser. Docs: [Dashboard](/web/dashboard).

To reconfigure later:

```bash
openclaw configure
openclaw agents add <name>
```

`--json` does not imply non-interactive mode. For scripts, use `--non-interactive`.

CLI onboarding includes a web search step where you can pick a provider
such as Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search,
Ollama Web Search, Perplexity, SearXNG, or Tavily. Some providers require an
API key, while others are key-free. You can also configure this later with
`openclaw configure --section web`. Docs: [Web tools](/tools/web).

## QuickStart vs Advanced

Onboarding starts with **QuickStart** (defaults) vs **Advanced** (full control).

    - Local gateway (loopback)
    - Workspace default (or existing workspace)
    - Gateway port **18789**
    - Gateway auth **Token** (auto‑generated, even on loopback)
    - Tool policy default for new local setups: `tools.profile: "coding"` (existing explicit profile is preserved)
    - DM isolation default: local onboarding writes `session.dmScope: "per-channel-peer"` when unset. Details: [CLI Setup Reference](/start/wizard-cli-reference#outputs-and-internals)
    - Tailscale exposure **Off**
    - Telegram + WhatsApp DMs default to **allowlist** (you'll be prompted for your phone number)

    - Exposes every step (mode, workspace, gateway, channels, daemon, skills).

## What onboarding configures

**Local mode (default)** walks you through these steps:

1. **Model/Auth** — choose any supported provider/auth flow (API key, OAuth, or provider-specific manual auth), including Custom Provider
   (OpenA

This page is the full reference for `openclaw onboard`.
For the short guide, see [Onboarding (CLI)](/start/wizard).

## What the wizard does

Local mode (default) walks you through:

- Model and auth setup (OpenAI Code subscription OAuth, Anthropic Claude CLI or API key, plus MiniMax, GLM, Ollama, Moonshot, StepFun, and AI Gateway options)
- Workspace location and bootstrap files
- Gateway settings (port, bind, auth, tailscale)
- Channels and providers (Telegram, WhatsApp, Discord, Google Chat, Mattermost, Signal, BlueBubbles, and other bundled channel plugins)
- Daemon install (LaunchAgent, systemd user unit, or native Windows Scheduled Task with Startup-folder fallback)
- Health check
- Skills setup

Remote mode configures this machine to connect to a gateway elsewhere.
It does not install or modify anything on the remote host.

## Local flow details

    - If `~/.openclaw/openclaw.json` exists, choose Keep, Modify, or Reset.
    - Re-running the wizard does not wipe anything unless you explicitly choose Reset (or pass `--reset`).
    - CLI `--reset` defaults to `config+creds+sessions`; use `--reset-scope full` to also remove workspace.
    - If config is invalid or contains legacy keys, the wizard stops and asks you to run `openclaw doctor` before continuing.
    - Reset uses `trash` and offers scopes:
      - Config only
      - Config + credentials + sessions
      - Full reset (also removes workspace)

    - Full option matrix is in [Auth and model options](#auth-and-model-options).

    - Default `~/.openclaw/workspace` (configurable).
    - Seeds workspace files needed for first-run bootstrap ritual.
    - Workspace layout: [Agent workspace](/concepts/agent-workspace).

    - Prompts for port, bind, auth mode, and tailscale exposure.
    - Recommended: keep token auth enabled even for loopback so local WS clients must authenticate.
    - In token mode, interactive setup offers:
      - **Generate/store plaintext token** (default)
      - **Use SecretRef** (o

## Wizard CLI Automation Fields

Use `--non-interactive` to automate `openclaw onboard`.

`--json` does not imply non-interactive mode. Use `--non-interactive` (and `--workspace`) for scripts.

## Baseline non-interactive example

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --secret-input-mode plaintext \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-bootstrap \
  --skip-skills
```

Add `--json` for a machine-readable summary.

Use `--skip-bootstrap` when your automation pre-seeds workspace files and does not want onboarding to create the default bootstrap files.

Use `--secret-input-mode ref` to store env-backed refs in auth profiles instead of plaintext values.
Interactive selection between env refs and configured provider refs (`file` or `exec`) is available in the onboarding flow.

In non-interactive `ref` mode, provider env vars must be set in the process environment.
Passing inline key flags without the matching env var now fails fast.

Example:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

## Provider-specific examples

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice apiKey \
      --anthropic-api-key "$ANTHROPIC_API_KEY" \
      --gateway-port 18789 \
      --gateway-bind loopback
    ```

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice gemini-api-key \
      --gemini-api-key "$GEMINI_API_KEY" \
      --gateway-port 18789 \
      --gateway-bind loopback
    ```

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice zai-api-key \
      --zai-api-key "$ZAI_API_KEY" \
      --gateway-port 18789 \
      --gateway-bind loopback
    ```

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --a

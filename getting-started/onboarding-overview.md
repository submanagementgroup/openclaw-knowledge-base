---
domain: getting-started
topic: "Onboarding Overview: Choosing Between CLI Wizard and macOS App Setup"
type: concept
keywords:
  - onboarding overview
  - openclaw onboard
  - CLI onboarding
  - macOS app onboarding
  - non-interactive onboarding
  - custom provider onboarding
  - onboarding paths
  - onboarding comparison
source: start/onboarding-overview.md
related:
  - getting-started/onboarding-wizard
  - getting-started/quickstart
  - cli/setup-cmd
---

OpenClaw has two onboarding paths: CLI wizard (works everywhere) and macOS app onboarding (macOS only). Both configure model auth, the Gateway, and optional chat channels.

## Which Path Should I Use?

| | CLI onboarding | macOS app onboarding |
|--|----------------|---------------------|
| **Platforms** | macOS, Linux, Windows (native or WSL2) | macOS only |
| **Interface** | Terminal wizard | Guided UI in the app |
| **Best for** | Servers, headless, full control | Desktop Mac, visual setup |
| **Automation** | `--non-interactive` for scripts | Manual only |
| **Command** | `openclaw onboard` | Launch the app |

Most users should start with **CLI onboarding** — it works everywhere and gives the most control.

## What Onboarding Configures

Regardless of path, onboarding sets up:

1. **Model provider and auth** — API key, OAuth, or setup token for your chosen provider
2. **Workspace** — directory for agent files, bootstrap templates, and memory
3. **Gateway** — port, bind address, auth mode
4. **Channels** (optional) — built-in and bundled chat channels (BlueBubbles, Discord, Feishu, Google Chat, Mattermost, Microsoft Teams, Telegram, WhatsApp, and more)
5. **Daemon** (optional) — background service so the Gateway starts automatically

## CLI Onboarding

```bash
openclaw onboard

# Or install daemon in one step:
openclaw onboard --install-daemon
```

Full reference: [Onboarding (CLI)](/getting-started/onboarding-wizard) | Command docs: [`openclaw setup`](/cli/setup-cmd)

## macOS App Onboarding

Open the OpenClaw app. The first-run wizard walks through the same steps with a visual interface.

## Custom or Unlisted Providers

If your provider is not listed in onboarding, choose **Custom Provider** and enter:

- API compatibility mode (OpenAI-compatible, Anthropic-compatible, or auto-detect)
- Base URL and API key
- Model ID and optional alias

Multiple custom endpoints can coexist — each gets its own endpoint ID.

## Related

- [CLI onboarding wizard](/getting-started/onboarding-wizard)
- [Quickstart](/getting-started/quickstart)
- [CLI setup command](/cli/setup-cmd)

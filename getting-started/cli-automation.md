---
domain: getting-started
topic: "CLI Automation: Scripted Non-Interactive Onboarding with openclaw onboard"
type: procedure
keywords:
  - non-interactive onboarding
  - openclaw onboard non-interactive
  - scripted onboarding
  - CI onboarding
  - secret-input-mode ref
  - secret-input-mode plaintext
  - auth-choice
  - skip-bootstrap
  - openclaw agents add
  - custom provider non-interactive
  - anthropic non-interactive
  - gemini non-interactive
source: start/wizard-cli-automation.md
related:
  - getting-started/onboarding-wizard
  - getting-started/onboarding-overview
  - cli/setup-cmd
---

Use `--non-interactive` to automate `openclaw onboard` in scripts or CI. Note: `--json` does not imply non-interactive mode — use `--non-interactive` (and `--workspace`) explicitly for scripts.

## Baseline Non-Interactive Example

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

**Key flags:**

| Flag | Purpose |
|------|---------|
| `--skip-bootstrap` | Skip creating default workspace bootstrap files (use when pre-seeding workspace) |
| `--secret-input-mode plaintext` | Store API keys as plaintext in auth profiles |
| `--secret-input-mode ref` | Store env-backed refs instead of plaintext; provider env vars must be set in the process environment |

In non-interactive `ref` mode, passing inline key flags without the matching env var set fails fast.

**Using ref mode:**

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

## Provider-Specific Examples

### Anthropic

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

### Gemini

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

### Z.AI

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice zai-api-key \
  --zai-api-key "$ZAI_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

### Moonshot

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice moonshot-api-key \
  --moonshot-api-key "$MOONSHOT_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

### Ollama (local, key-free)

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice ollama \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk \
  --gateway-port 18789 \
  --gateway-bind loopback
```

### Custom Provider

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --custom-provider-id "my-custom" \
  --custom-compatibility anthropic \
  --custom-image-input \
  --gateway-port 18789 \
  --gateway-bind loopback
```

`--custom-api-key` is optional; if omitted, onboarding checks `CUSTOM_API_KEY`. Add `--custom-image-input` for vision-capable models with unknown model IDs, or `--custom-text-input` to force text-only metadata.

**Ref-mode custom provider:**

```bash
export CUSTOM_API_KEY="your-key"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --secret-input-mode ref \
  --custom-provider-id "my-custom" \
  --custom-compatibility anthropic \
  --custom-image-input \
  --gateway-port 18789 \
  --gateway-bind loopback
```

In ref mode, `apiKey` is stored as `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`.

### Vercel AI Gateway

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice ai-gateway-api-key \
  --ai-gateway-api-key "$AI_GATEWAY_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

### Cloudflare AI Gateway

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cloudflare-ai-gateway-api-key \
  --cloudflare-ai-gateway-account-id "your-account-id" \
  --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
  --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

### OpenCode

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice opencode-zen \
  --opencode-zen-api-key "$OPENCODE_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Swap to `--auth-choice opencode-go --opencode-go-api-key "$OPENCODE_API_KEY"` for the Go catalog.

## Add Another Agent

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.5 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

`openclaw agents add <name>` creates a separate agent with its own workspace, sessions, and auth profiles. Default workspaces follow `~/.openclaw/workspace-<agentId>`.

## Notes

- Anthropic setup-token remains available but OpenClaw now prefers Claude CLI reuse when available. For production, prefer an Anthropic API key.

## Related

- [Onboarding wizard](/getting-started/onboarding-wizard)
- [Onboarding overview](/getting-started/onboarding-overview)
- [CLI setup command](/cli/setup-cmd)

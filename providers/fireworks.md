---
domain: providers
topic: "Fireworks AI Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Fireworks AI
  - fireworks provider
  - AI provider
  - model config
  - API key
  - Fireworks
  - fast inference
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/fireworks.md
---

Fireworks AI provider setup, configuration, and model reference for OpenClaw.

[Fireworks](https://fireworks.ai) exposes open-weight and routed models through an OpenAI-compatible API. OpenClaw includes a bundled Fireworks provider plugin.

| Property      | Value                                                  |
| ------------- | ------------------------------------------------------ |
| Provider      | `fireworks`                                            |
| Auth          | `FIREWORKS_API_KEY`                                    |
| API           | OpenAI-compatible chat/completions                     |
| Base URL      | `https://api.fireworks.ai/inference/v1`                |
| Default model | `fireworks/accounts/fireworks/routers/kimi-k2p5-turbo` |

## Getting started

    ```bash
    openclaw onboard --auth-choice fireworks-api-key
    ```

    This stores your Fireworks key in OpenClaw config and sets the Fire Pass starter model as the default.

    ```bash
    openclaw models list --provider fireworks
    ```

## Non-interactive example

For scripted or CI setups, pass all values on the command line:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY" \
  --skip-health \
  --accept-risk
```

## Built-in catalog

| Model ref                                              | Name                        | Input      | Context | Max output | Notes                                                                                                                                               |
| ------------------------------------------------------ | --------------------------- | ---------- | ------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fireworks/accounts/fireworks/models/kimi-k2p6`        | Kimi K2.6                   | text,image | 262,144 | 262,144    | Latest Kimi model on Fireworks. Thinking is disabled for Fireworks K2.6 requests; route through Moonshot directly if you need Kimi thinking output. |
| `fireworks/accounts/fireworks/routers/kimi-k2p5-turbo` | Kimi K2.5 Turbo (Fire Pass) | text,image | 256,000 | 256,000    | Default bundled starter model on Fireworks                                                                                                          |

If Fireworks publishes a newer model such as a fresh Qwen or Gemma release, you can switch to it directly by using its Fireworks model id without waiting for a bundled catalog update.

## Custom Fireworks model ids

OpenClaw accepts dynamic Fireworks model ids too. Use the exact model or router id shown by Fireworks and prefix it with `fireworks/`.

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "fireworks/accounts/fireworks/routers/kimi-k2p5-turbo",
      },
    },
  },
}
```

    Every Fireworks model ref in OpenClaw starts with `fireworks/` followed by the exact id or router path from the Fireworks platform. For example:

    - Router model: `fireworks/accounts/fireworks/routers/kimi-k2p5-turbo`
    - Direct model: `fireworks/accounts/fireworks/models/<model-name>`

    OpenClaw strips the `fireworks/` prefix when building the API request and sends the remaining path to the Fireworks endpoint.

    If the Gateway runs outside your interactive shell, make sure `FIREWORKS_API_KEY` is available to that process too.

    A key sitting only in `~/.profile` will not help a launchd/systemd daemon unless that environment is imported there as well. Set the key in `~/.openclaw/.env` or via `env.shellEnv` to ensure the gateway process can read it.

## Related

    Choosing providers, model refs, and failover behavior.

    General troubleshooting and FAQ.

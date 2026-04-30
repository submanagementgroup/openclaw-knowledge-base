---
domain: cli
topic: "Infer CLI: openclaw infer for One-Shot Model Inference and API Testing"
type: reference
keywords:
  - openclaw infer
  - infer CLI
  - single-turn inference
  - model testing
  - API testing
  - one-shot inference
related:
  - cli/agent-cli
  - concepts/models
  - cli/cli-overview
source: cli/infer.md
---

The `openclaw infer` command runs a single-turn inference without a full agent session. Use it to test models, check API keys, and do quick one-shot tasks.

```bash
openclaw infer "explain the docker build cache"
openclaw infer "summarize this" --model anthropic/claude-opus-4-5
openclaw infer "what is 2+2" --json   # JSON output
```

`openclaw infer` is the canonical headless surface for provider-backed inference workflows.

It intentionally exposes capability families, not raw gateway RPC names and not raw agent tool ids.

## Turn infer into a skill

Copy and paste this to an agent:

```text
Read https://docs.openclaw.ai/cli/infer, then create a skill that routes my common workflows to `openclaw infer`.
Focus on model runs, image generation, video generation, audio transcription, TTS, web search, and embeddings.
```

A good infer-based skill should:

- map common user intents to the correct infer subcommand
- include a few canonical infer examples for the workflows it covers
- prefer `openclaw infer ...` in examples and suggestions
- avoid re-documenting the entire infer surface inside the skill body

Typical infer-focused skill coverage:

- `openclaw infer model run`
- `openclaw infer image generate`
- `openclaw infer audio transcribe`
- `openclaw infer tts convert`
- `openclaw infer web search`
- `openclaw infer embedding create`

## Why use infer

`openclaw infer` provides one consistent CLI for provider-backed inference tasks inside OpenClaw.

Benefits:

- Use the providers and models already configured in OpenClaw instead of wiring up one-off wrappers for each backend.
- Keep model, image, audio transcription, TTS, video, web, and embedding workflows under one command tree.
- Use a stable `--json` output shape for scripts, automation, and agent-driven workflows.
- Prefer a first-party OpenClaw surface when the task is fundamentally "run inference."
- Use the normal local path without requiring the gateway for most infer commands.

For end-to-end provider checks, prefer `openclaw infer ...` once lower-level
provider tests are green. It exercises the shipped CLI, config loading,
default-agent resolution, bundled plugin activation, runtime-dependency repair,
and the shared capability runtime before the provider request is made.

## Command tree

```text
 openclaw infer
  list
  inspect

  model
    run
    list
    inspect
    providers
    auth login
    auth logout
    auth status

  image
    generate
    edit
    describe
    describe-many
    providers

  audio
    transcribe
    providers

  tts
    convert
    voices
    providers
    status
    enable
    disable
    set-provider

  video
    generate
    describe
    providers

  web
    search
    fetch
    providers

  embedding
    create
    providers
```

## Common tasks

This table maps common inference tasks to the corresponding infer command.

| Task                         | Command                                                                                       | Notes                                                 |
| ---------------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Run a text/model prompt      | `openclaw infer model run --prompt "..." --json`                                              | Uses the normal local path by default                 |
| Run a model prompt on images | `openclaw infer model run --prompt "Describe this" --file ./image.png --model provider/model` | Repeat `--file` for multiple image inputs             |
| Generate an image            | `openclaw infer image generate --prompt "..." --json`                                         | Use `image edit` when starting from an existing file  |
| Describe an image file       | `openclaw infer image describe --file ./image.png --prompt "..." --json`                      | `--model` must be an image-capable `<provider/model>` |
| Transcribe audio             | `openclaw infer audio transcribe --file ./memo.m4a --json`                                    | `--model` must be `<provider/model>`                  |
| Synthesize speech            | `openclaw infer tts convert --text "..." --output ./speech.mp3 --json`                        | `tts status` is gateway-oriented                      |
| Generate a video             | `openclaw infer video generate --prompt "..." --json`                                         | Supports provider hints such as `--resolution`        |
| Describe a video file        | `openclaw infer video describe --file ./clip.mp4 --json`                                      | `--model` must be `<provider/model>`                  |
| Search the web               | `openclaw infer web search --query "..." --json`                                              |                                                       |
| Fetch a web page             | `openclaw infer web fetch --url https://example.com --json`                                   |                                                       |
| Create embeddings            | `openclaw infer embedding create --text "..." --json`                                         |                                                       |

## Behavior

- `openclaw infer ...` is the primary CLI surface for these workflows.
- Use `--json` when the output will be consumed by another command or script.
- Use `--provider` or `--model provider/model` when a specific backend is required.
- For `image describe`, `audio transcribe`, and `video describe`, `--model` must use the form `<provider/model>`.
- For `image describe`, an explicit `--model` runs that provider/model directly. The model must be image-capable in the model catalog or provider config. `codex/<model>` runs a bounded Codex app-server image-understanding turn; `openai-codex/<model>` uses the OpenAI Codex OAuth provider path.
- Stateless execution commands default to local.
- Gateway-managed state commands default to gateway.
- The normal local path does not require the gateway to be running.
- Local `model run` is a lean one-shot provider completion. It resolves the configured agent model and auth, but does not start a chat-agent turn, load tools, or open bundled MCP servers.
- `model run --file` accepts image files, detects their MIME type, and sends them with the supplied prompt to the selected model. Repeat `--file` for multiple images.
- `model run --file` rejects non-image inputs. Use `infer audio transcribe` for audio files and `infer video describe` for video files.
- `model run --gateway` exercises Gateway routing, saved auth, provider selection, and the embedded runtime, but still runs as a raw model probe: it sends the supplied prompt and any image attachments without prior session transcript, bootstrap/AGENTS context, context-engine assembly, tools, or bundled MCP servers.
- `model run --gateway --model <provider/model>` requires a trusted operator gateway credential because the request asks the Gateway to run a one-off provider/model override.

## Model

Use `model` for provider-backed text inference and model/provider inspection.

```bash
openclaw infer model run --prompt "Reply with exactly: smoke-ok" --json
openclaw infer model run --prompt "Summarize this changelog entry" --model openai/gpt-5.

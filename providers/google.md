---
domain: providers
topic: "Google Gemini Provider: Gemini Pro, Flash, Vertex AI, and Grounding Setup"
type: procedure
keywords:
  - Google
  - Gemini
  - gemini-2.5-pro
  - GOOGLE_API_KEY
  - Vertex AI
  - Google provider
  - multimodal
related:
  - providers/openai
  - providers/anthropic
  - concepts/models
source: providers/google.md
---

Google Gemini provider for OpenClaw. Supports Gemini Pro, Flash, and Ultra models with multimodal input, grounding, and code execution.

## Quick Setup

```json5
{
  agents: {
    defaults: {
      model: "google/gemini-2.5-pro",
    }
  }
}
```

Set `GOOGLE_API_KEY` or `GEMINI_API_KEY`. Vertex AI uses ADC (Application Default Credentials).

The Google plugin provides access to Gemini models through Google AI Studio, plus
image generation, media understanding (image/audio/video), text-to-speech, and web search via
Gemini Grounding.

- Provider: `google`
- Auth: `GEMINI_API_KEY` or `GOOGLE_API_KEY`
- API: Google Gemini API
- Runtime option: `agents.defaults.agentRuntime.id: "google-gemini-cli"`
  reuses Gemini CLI OAuth while keeping model refs canonical as `google/*`.

## Getting started

Choose your preferred auth method and follow the setup steps.

    **Best for:** standard Gemini API access through Google AI Studio.

        ```bash
        openclaw onboard --auth-choice gemini-api-key
        ```

        Or pass the key directly:

        ```bash
        openclaw onboard --non-interactive \
          --mode local \
          --auth-choice gemini-api-key \
          --gemini-api-key "$GEMINI_API_KEY"
        ```

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "google/gemini-3.1-pro-preview" },
            },
          },
        }
        ```

        ```bash
        openclaw models list --provider google
        ```

    The environment variables `GEMINI_API_KEY` and `GOOGLE_API_KEY` are both accepted. Use whichever you already have configured.

    **Best for:** reusing an existing Gemini CLI login via PKCE OAuth instead of a separate API key.

    The `google-gemini-cli` provider is an unofficial integration. Some users
    report account restrictions when using OAuth this way. Use at your own risk.

        The local `gemini` command must be available on `PATH`.

        ```bash
        # Homebrew
        brew install gemini-cli

        # or npm
        npm install -g @google/gemini-cli
        ```

        OpenClaw supports both Homebrew installs and global npm installs, including
        common Windows/npm layouts.

        ```bash
        openclaw models auth login --provider google-gemini-cli --set-default
        ```

        ```bash
        openclaw models list --provider google
        ```

    - Default model: `google/gemini-3.1-pro-preview`
    - Runtime: `google-gemini-cli`
    - Alias: `gemini-cli`

    Gemini 3.1 Pro's Gemini API model id is `gemini-3.1-pro-preview`. OpenClaw accepts the shorter `google/gemini-3.1-pro` as a convenience alias and normalizes it before provider calls.

    **Environment variables:**

    - `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
    - `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

    (Or the `GEMINI_CLI_*` variants.)

    If Gemini CLI OAuth requests fail after login, set `GOOGLE_CLOUD_PROJECT` or
    `GOOGLE_CLOUD_PROJECT_ID` on the gateway host and retry.

    If login fails before the browser flow starts, make sure the local `gemini`
    command is installed and on `PATH`.

    `google-gemini-cli/*` model refs are legacy compatibility aliases. New
    configs should use `google/*` model refs plus the `google-gemini-cli`
    runtime when they want local Gemini CLI execution.

## Capabilities

| Capability             | Supported                     |
| ---------------------- | ----------------------------- |
| Chat completions       | Yes                           |
| Image generation       | Yes                           |
| Music generation       | Yes                           |
| Text-to-speech         | Yes                           |
| Realtime voice         | Yes (Google Live API)         |
| Image understanding    | Yes                           |
| Audio transcription    | Yes                           |
| Video understanding    | Yes                           |
| Web search (Grounding) | Yes                           |
| Thinking/reasoning     | Yes (Gemini 2.5+ / Gemini 3+) |
| Gemma 4 models         | Yes                           |

Gemini 3 models use `thinkingLevel` rather than `thinkingBudget`. OpenClaw maps
Gemini 3, Gemini 3.1, and `gemini-*-latest` alias reasoning controls to
`thinkingLevel` so default/low-latency runs do not send disabled
`thinkingBudget` values.

`/think adaptive` keeps Google's dynamic thinking semantics instead of choosing
a fixed OpenClaw level. Gemini 3 and Gemini 3.1 omit a fixed `thinkingLevel` so
Google can choose the level; Gemini 2.5 sends Google's dynamic sentinel
`thinkingBudget: -1`.

Gemma 4 models (for example `gemma-4-26b-a4b-it`) support thinking mode. OpenClaw
rewrites `thinkingBudget` to a supported Google `thinkingLevel` for Gemma 4.
Setting thinking to `off` preserves thinking disabled instead of mapping to
`MINIMAL`.

## Image generation

The bundled `google` image-generation provider defaults to
`google/gemini-3.1-flash-image-preview`.

- Also supports `google/gemini-3-pro-image-preview`
- Generate: up to 4 images per request
- Edit mode: enabled, up to 5 input images
- Geometry controls: `size`, `aspectRatio`, and `resolution`

To use Google as the default image provider:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

See [Image Generation](/tools/image-generation) for shared tool parameters, provider selection, and failover behavior.

## Video generation

The bundled `google` plugin also registers video generation through the shared
`video_generate` tool.

- Default video model: `google/veo-3.1-fast-generate-preview`
- Modes: text-to-video, image-to-video, and single-video reference flows
- Supports `aspectRatio`, `resolution`, and `audio`
- Current duration clamp: **4 to 8 seconds**

To use Google as the default video provider:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

See [Video Generation](/tools/video-generation) for shared tool parameters, provider selection, and failover behavior.

## Music generation

The bundled `google` plugin also registers music generation through the shared
`music_generate` tool.

- Default music model: `google/lyria-3-clip-preview`
- Also supports `google/lyria-3-pro-preview`
- Prompt controls: `lyrics` and `instrumental`
- Output format: `mp3` by default, plus `wav` on `google/lyria-3-pro-preview`
- Reference inputs: up to 10 images
- Session-backed runs detach through the shared task/status flow, including `action: "status"`

To use Google as the default music provider:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

See [Music Generation](/tools/music-generation) for shared tool parameters, provider selection, and failover behavior.

## Text-to-speech

The bundled `google` speech provider uses the Gemini API TTS path with
`gemini-3.1-flash-tts-preview`.

- Default voice: `Kore`
- Auth: `messages.tts.providers.google.apiKey`, `models.providers.google.apiKey`, `GEMINI_API_KEY`, or `GOOGLE_API_KEY`
- Output: WAV for regular TTS attachments, Opus for voice-note targets, PCM for Talk/telephony
- Voice-note

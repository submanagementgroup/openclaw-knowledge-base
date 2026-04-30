---
domain: providers
topic: "XAI Grok Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - XAI Grok
  - xai provider
  - AI provider
  - model config
  - API key
  - xAI
  - Grok
  - grok-2
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/xai.md
---

XAI Grok provider setup, configuration, and model reference for OpenClaw.

OpenClaw ships a bundled `xai` provider plugin for Grok models.

## Getting started

    Create an API key in the [xAI console](https://console.x.ai/).

    Set `XAI_API_KEY`, or run:

    ```bash
    openclaw onboard --auth-choice xai-api-key
    ```

    ```json5
    {
      agents: { defaults: { model: { primary: "xai/grok-4" } } },
    }
    ```

OpenClaw uses the xAI Responses API as the bundled xAI transport. The same
`XAI_API_KEY` can also power Grok-backed `web_search`, first-class `x_search`,
and remote `code_execution`.
If you store an xAI key under `plugins.entries.xai.config.webSearch.apiKey`,
the bundled xAI model provider reuses that key as a fallback too.
`code_execution` tuning lives under `plugins.entries.xai.config.codeExecution`.

## Built-in catalog

OpenClaw includes these xAI model families out of the box:

| Family         | Model ids                                                                |
| -------------- | ------------------------------------------------------------------------ |
| Grok 3         | `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`               |
| Grok 4         | `grok-4`, `grok-4-0709`                                                  |
| Grok 4 Fast    | `grok-4-fast`, `grok-4-fast-non-reasoning`                               |
| Grok 4.1 Fast  | `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`                           |
| Grok 4.20 Beta | `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning` |
| Grok Code      | `grok-code-fast-1`                                                       |

The plugin also forward-resolves newer `grok-4*` and `grok-code-fast*` ids when
they follow the same API shape.

`grok-4-fast`, `grok-4-1-fast`, and the `grok-4.20-beta-*` variants are the
current image-capable Grok refs in the bundled catalog.

## OpenClaw feature coverage

The bundled plugin maps xAI's current public API surface onto OpenClaw's shared
provider and tool contracts. Capabilities that don't fit the shared contract
(for example streaming TTS and realtime voice) are not exposed — see the table
below.

| xAI capability             | OpenClaw surface                          | Status                                                              |
| -------------------------- | ----------------------------------------- | ------------------------------------------------------------------- |
| Chat / Responses           | `xai/<model>` model provider              | Yes                                                                 |
| Server-side web search     | `web_search` provider `grok`              | Yes                                                                 |
| Server-side X search       | `x_search` tool                           | Yes                                                                 |
| Server-side code execution | `code_execution` tool                     | Yes                                                                 |
| Images                     | `image_generate`                          | Yes                                                                 |
| Videos                     | `video_generate`                          | Yes                                                                 |
| Batch text-to-speech       | `messages.tts.provider: "xai"` / `tts`    | Yes                                                                 |
| Streaming TTS              | —                                         | Not exposed; OpenClaw's TTS contract returns complete audio buffers |
| Batch speech-to-text       | `tools.media.audio` / media understanding | Yes                                                                 |
| Streaming speech-to-text   | Voice Call `streaming.provider: "xai"`    | Yes                                                                 |
| Realtime voice             | —                                         | Not exposed yet; different session/WebSocket contract               |
| Files / batches            | Generic model API compatibility only      | Not a first-class OpenClaw tool                                     |

OpenClaw uses xAI's REST image/video/TTS/STT APIs for media generation,
speech, and batch transcription, xAI's streaming STT WebSocket for live
voice-call transcription, and the Responses API for model, search, and
code-execution tools. Features that need different OpenClaw contracts, such as
Realtime voice sessions, are documented here as upstream capabilities rather
than hidden plugin behavior.

### Fast-mode mappings

`/fast on` or `agents.defaults.models["xai/<model>"].params.fastMode: true`
rewrites native xAI requests as follows:

| Source model  | Fast-mode target   |
| ------------- | ------------------ |
| `grok-3`      | `grok-3-fast`      |
| `grok-3-mini` | `grok-3-mini-fast` |
| `grok-4`      | `grok-4-fast`      |
| `grok-4-0709` | `grok-4-fast`      |

### Legacy compatibility aliases

Legacy aliases still normalize to the canonical bundled ids:

| Legacy alias              | Canonical id                          |
| ------------------------- | ------------------------------------- |
| `grok-4-fast-reasoning`   | `grok-4-fast`                         |
| `grok-4-1-fast-reasoning` | `grok-4-1-fast`                       |
| `grok-4.20-reasoning`     | `grok-4.20-beta-latest-reasoning`     |
| `grok-4.20-non-reasoning` | `grok-4.20-beta-latest-non-reasoning` |

## Features

    The bundled `grok` web-search provider uses `XAI_API_KEY` too:

    ```bash
    openclaw config set tools.web.search.provider grok
    ```

    The bundled `xai` plugin registers video generation through the shared
    `video_generate` tool.

    - Default video model: `xai/grok-imagine-video`
    - Modes: text-to-video, image-to-video, reference-image generation, remote
      video edit, and remote video extension
    - Aspect ratios: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3`
    - Resolutions: `480P`, `720P`
    - Duration: 1-15 seconds for generation/image-to-video, 1-10 seconds when
      using `reference_image` roles, 2-10 seconds for extension
    - Reference-image generation: set `imageRoles` to `reference_image` for
      every supplied image; xAI accepts up to 7 such images

    Local video buffers are not accepted. Use remote `http(s)` URLs for
    video edit/extend inputs. Image-to-video accepts local image buffers because
    OpenClaw can encode those as data URLs for xAI.

    To use xAI as the default video provider:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "xai/grok-imagine-video",
          },
        },
      },
    }
    ```

    See [Video Generation](/tools/video-generation) for shared tool parameters,
    provider selection, and failover behavior.

    The bundled `xai` plugin registers image generation through the shared
    `image_generate` tool.

    - Default image model: `xai/grok-imagine-image`
    - Additional model: `xai/grok-imagine-image-pro`
    - Modes: text-to-image and reference-image edit
    - Reference inputs: one `image` or up to five `images`
    - Aspect ratios: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `2:3`, `3:2`
    - Resolutions: `1K`, `2K`
    - Count: up to 4 images

    OpenClaw asks xAI for `b64_json` image responses so generated media can be
    stored and delivered through the normal channel attachment path. Local
    reference images are converted to data URLs; remote `http(s)` references are
    passed through.

    To use xAI as the default image provider:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "xai/grok-imagine-image",
          },
        },
      },
    }
    ```

    xAI also documents `quality`, `mask`, `user`, and additional native ratios
    such as `1:2`, `2:1`, `9:20`, and `20:9`. OpenClaw forwards only the
    shared cross-provider image controls today; unsupported native-only knobs
    are i

---
domain: tools
topic: "Video Generation Tool: Runway and Provider Configuration"
type: procedure
keywords:
  - video generation
  - Runway
  - video tool
  - generate video
related:
  - tools/image-generation
  - providers/runway
  - tools/media-overview
source: tools/video-generation.md
---

Video generation tool for OpenClaw agents. Supports Runway and other video providers.

OpenClaw agents can generate videos from text prompts, reference images, or
existing videos. Sixteen provider backends are supported, each with
different model options, input modes, and feature sets. The agent picks the
right provider automatically based on your configuration and available API
keys.

The `video_generate` tool only appears when at least one video-generation
provider is available. If you do not see it in your agent tools, set a
provider API key or configure `agents.defaults.videoGenerationModel`.

OpenClaw treats video generation as three runtime modes:

- `generate` — text-to-video requests with no reference media.
- `imageToVideo` — request includes one or more reference images.
- `videoToVideo` — request includes one or more reference videos.

Providers can support any subset of those modes. The tool validates the
active mode before submission and reports supported modes in `action=list`.

## Quick start

    Set an API key for any supported provider:

    ```bash
    export GEMINI_API_KEY="your-key"
    ```

    ```bash
    openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
    ```

    > Generate a 5-second cinematic video of a friendly lobster surfing at sunset.

    The agent calls `video_generate` automatically. No tool allowlisting
    is needed.

## How async generation works

Video generation is asynchronous. When the agent calls `video_generate` in a
session:

1. OpenClaw submits the request to the provider and immediately returns a task id.
2. The provider processes the job in the background (typically 30 seconds to 5 minutes depending on the provider and resolution).
3. When the video is ready, OpenClaw wakes the same session with an internal completion event.
4. The agent posts the finished video back into the original conversation.

While a job is in flight, duplicate `video_generate` calls in the same
session return the current task status instead of starting another
generation. Use `openclaw tasks list` or `openclaw tasks show <taskId>` to
check progress from the CLI.

Outside of session-backed agent runs (for example, direct tool invocations),
the tool falls back to inline generation and returns the final media path
in the same turn.

Generated video files are saved under OpenClaw-managed media storage when
the provider returns bytes. The default generated-video save cap follows
the video media limit, and `agents.defaults.mediaMaxMb` raises it for
larger renders. When a provider also returns a hosted output URL, OpenClaw
can deliver that URL instead of failing the task if local persistence
rejects an oversized file.

### Task lifecycle

| State       | Meaning                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `queued`    | Task created, waiting for the provider to accept it.                                             |
| `running`   | Provider is processing (typically 30 seconds to 5 minutes depending on provider and resolution). |
| `succeeded` | Video ready; the agent wakes and posts it to the conversation.                                   |
| `failed`    | Provider error or timeout; the agent wakes with error details.                                   |

Check status from the CLI:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

If a video task is already `queued` or `running` for the current session,
`video_generate` returns the existing task status instead of starting a new
one. Use `action: "status"` to check explicitly without triggering a new
generation.

## Supported providers

| Provider              | Default model                   | Text | Image ref                                            | Video ref                                       | Auth                                     |
| --------------------- | ------------------------------- | :--: | ---------------------------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| Alibaba               | `wan2.6-t2v`                    |  ✓   | Yes (remote URL)                                     | Yes (remote URL)                                | `MODELSTUDIO_API_KEY`                    |
| BytePlus (1.0)        | `seedance-1-0-pro-250528`       |  ✓   | Up to 2 images (I2V models only; first + last frame) | —                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 1.5 | `seedance-1-5-pro-251215`       |  ✓   | Up to 2 images (first + last frame via role)         | —                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 2.0 | `dreamina-seedance-2-0-260128`  |  ✓   | Up to 9 reference images                             | Up to 3 videos                                  | `BYTEPLUS_API_KEY`                       |
| ComfyUI               | `workflow`                      |  ✓   | 1 image                                              | —                                               | `COMFY_API_KEY` or `COMFY_CLOUD_API_KEY` |
| DeepInfra             | `Pixverse/Pixverse-T2V`         |  ✓   | —                                                    | —                                               | `DEEPINFRA_API_KEY`                      |
| fal                   | `fal-ai/minimax/video-01-live`  |  ✓   | 1 image; up to 9 with Seedance reference-to-video    | Up to 3 videos with Seedance reference-to-video | `FAL_KEY`                                |
| Google                | `veo-3.1-fast-generate-preview` |  ✓   | 1 image                                              | 1 video                                         | `GEMINI_API_KEY`                         |
| MiniMax               | `MiniMax-Hailuo-2.3`            |  ✓   | 1 image                                              | —                                               | `MINIMAX_API_KEY` or MiniMax OAuth       |
| OpenAI                | `sora-2`                        |  ✓   | 1 image                                              | 1 video                                         | `OPENAI_API_KEY`                         |
| OpenRouter            | `google/veo-3.1-fast`           |  ✓   | Up to 4 images (first/last frame or references)      | —                                               | `OPENROUTER_API_KEY`                     |
| Qwen                  | `wan2.6-t2v`                    |  ✓   | Yes (remote URL)                                     | Yes (remote URL)                                | `QWEN_API_KEY`                           |
| Runway                | `gen4.5`                        |  ✓   | 1 image                                              | 1 video                                         | `RUNWAYML_API_SECRET`                    |
| Together              | `Wan-AI/Wan2.2-T2V-A14B`        |  ✓   | 1 image                                              | —                                               | `TOGETHER_API_KEY`                       |
| Vydra                 | `veo3`                          |  ✓   | 1 image (`kling`)                                    | —                                               | `VYDRA_API_KEY`                          |
| xAI                   | `grok-imagine-video`            |  ✓   | 1 first-frame image or up to 7 `reference_image`s    | 1 video                                         | `XAI_API_KEY`                            |

Some providers accept additional or alternate API key env vars. See
individual [provider pages](#related) for details.

Run `video_generate action=list` to inspect available providers, models, and
runtime modes at runtime.

### Capability matrix

The explicit mode contract used by `video_generate`, contract tests, and
the shared live sweep:

| Provider   | `generate` | `imageToVideo` | `videoToVideo` | Shared live lanes today                                                                                                                  |
| ---------- | :--------: | :------------: | :------------: | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba    |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` skipped because this provider needs remote `http(s)` video URLs                               |
| BytePlus   |     ✓      |       ✓        |       —        | `generate`, `imageToVideo`                                                                                                               |
| ComfyUI    |     ✓      |       ✓        |       —        | Not in the shared sweep; workflow-specific coverage lives with Comfy tests                                                               |
| DeepInfra  |     ✓      |       —

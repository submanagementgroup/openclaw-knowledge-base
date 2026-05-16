---
domain: tools
topic: "Video Generation Tool"
type: procedure
keywords:
  - video generation
  - AI video
  - video tool
  - video_generate
  - video providers
source: tools/video-generation.md
---

OpenClaw agents can generate videos from text prompts, reference images, or
existing videos. Sixteen provider backends are supported, each with
different model options, input modes, and feature sets. The agent picks the
right provider automatically based on your configuration and available API
keys.

> **Note:** The `video_generate` tool only appears when at least one video-generation
provider is available. If you do not see it in your agent tools, set a
provider API key or configure `agents.defaults.videoGenerationModel`.


OpenClaw treats video generation as three runtime modes:

- `generate` - text-to-video requests with no reference media.
- `imageToVideo` - request includes one or more reference images.
- `videoToVideo` - request includes one or more reference videos.

Providers can support any subset of those modes. The tool validates the
active mode before submission and reports supported modes in `action=list`.

## Quick start

**Configure auth**

Set an API key for any supported provider:

    ```bash
    export GEMINI_API_KEY="your-key"
    ```

  
**Pick a default model (optional)**

```bash
    openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
    ```
  
**Ask the agent**

> Generate a 5-second cinematic video of a friendly lobster surfing at sunset.

    The agent calls `video_generate` automatically. No tool allowlisting
    is needed.

  
## How async generation works

Video generation is asynchronous. When the agent calls `video_generate` in a
session:

1. OpenClaw submits the request to the provider and immediately returns a task id.
2. The provider processes the job in the background (typically 30 seconds to several minutes depending on the provider and resolution; slow queue-backed providers can run up to the configured timeout).
3. When the video is ready, OpenClaw wakes the same session with an internal completion event.
4. The agent tells the user and attaches the finished video. In group/channel
   chats that use message-tool-only visible delivery, the agent relays the
   result through the message tool instead of OpenClaw posting it directly.

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

| State       | Meaning                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------ |
| `queued`    | Task created, waiting for the provider to accept it.                                                   |
| `running`   | Provider is processing (typically 30 seconds to several minutes depending on provider and resolution). |
| `succeeded` | Video ready; the agent wakes and posts it to the conversation.                                         |
| `failed`    | Provider error or timeout; the agent wakes with error details.                                         |

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
| BytePlus (1.0)        | `seedance-1-0-pro-250528`       |  ✓   | Up to 2 images (I2V models only; first + last frame) | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 1.5 | `seedance-1-5-pro-251215`       |  ✓   | Up to 2 images (first + last frame via role)         | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 2.0 | `dreamina-seedance-2-0-260128`  |  ✓   | Up to 9 reference images                             | Up to 3 videos                                  | `BYTEPLUS_API_KEY`                       |
| ComfyUI               | `workflow`                      |  ✓   | 1 image                                              | -                                               | `COMFY_API_KEY` or `COMFY_CLOUD_API_KEY` |
| DeepInfra             | `Pixverse/Pixverse-T2V`         |  ✓   | -                                                    | -                                               | `DEEPINFRA_API_KEY`                      |
| fal                   | `fal-ai/minimax/video-01-live`  |  ✓   | 1 image; up to 9 with Seedance reference-to-video    | Up to 3 videos with Seedance reference-to-video | `FAL_KEY`                                |
| Google                | `veo-3.1-fast-generate-preview` |  ✓   | 1 image                                              | 1 video                                         | `GEMINI_API_KEY`                         |
| MiniMax               | `MiniMax-Hailuo-2.3`            |  ✓   | 1 image                                              | -                                               | `MINIMAX_API_KEY` or MiniMax OAuth       |
| OpenAI                | `sora-2`                        |  ✓   | 1 image                                              | 1 video                                         | `OPENAI_API_KEY`                         |
| OpenRouter            | `google/veo-3.1-fast`           |  ✓   | Up to 4 images (first/last frame or references)      | -                                               | `OPENROUTER_API_KEY`                     |
| Qwen                  | `wan2.6-t2v`                    |  ✓   | Yes (remote URL)                                     | Yes (remote URL)                                | `QWEN_API_KEY`                           |
| Runway                | `gen4.5`                        |  ✓   | 1 image                                              | 1 video                                         | `RUNWAYML_API_SECRET`                    |
| Together              | `Wan-AI/Wan2.2-T2V-A14B`        |  ✓   | 1 image                                              | -                                               | `TOGETHER_API_KEY`                       |
| Vydra                 | `veo3`                          |  ✓   | 1 image (`kling`)                                    | -                                               | `VYDRA_API_KEY`                          |
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
| BytePlus   |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                               |
| ComfyUI    |     ✓      |       ✓        |       -        | Not in the shared sweep; workflow-specific coverage lives with Comfy tests                                                               |
| DeepInfra  |     ✓      |       -        |       -        | `generate`; native DeepInfra video schemas are text-to-video in the bundled contract                                                     |
| fal        |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` only when using Seedance reference-to-video                                                   |
| Google     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; shared `videoToVideo` skipped because the current buffer-backed Gemini/Veo sweep does not accept that input  |
| MiniMax    |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                               |
| OpenAI     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; shared `videoToVideo` skipped because this org/input path currently needs provider-side inpaint/remix access |
| OpenRouter |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                               |
| Qwen       |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` skipped because this provider needs remote `http(s)` video URLs                               |
| Runway     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` runs only when the selected model is `runway/gen4_aleph`                                      |
| Together   |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                               |
| Vydra      |     ✓      |       ✓        |       -        | `generate`; shared `imageToVideo` skipped because bundled `veo3` is text-only and bundled `kling` requires a remote image URL            |
| xAI        |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` skipped because this provider currently needs a remote MP4 URL                                |

## Tool parameters

### Required


  Text description of the video to generate. Required for `action: "generate"`.


### Content inputs

Single reference image (path or URL).
Multiple reference images (up to 9).

Optional per-position role hints parallel to the combined image list.
Canonical values: `first_frame`, `last_frame`, `reference_image`.

Single reference video (path or URL).
Multiple reference videos (up to 4).

Optional per-position role hints parallel to the combined video list.
Canonical value: `reference_video`.


Single reference audio (path or URL). Used for background music or voice
reference when the provider supports audio inputs.

Multiple reference audios (up to 3).

Optional per-position role hints parallel to the combined audio list.
Canonical value: `reference_audio`.


> **Note:** Role hints are forwarded to the provider as-is. Canonical values come from
the `VideoGenerationAssetRole` union but providers may accept additional
role strings. `*Roles` arrays must not have more entries than the
corresponding reference list; off-by-one mistakes fail with a clear error.
Use an empty string to leave a slot unset. For xAI, set every image role to
`reference_image` to use its `reference_images` generation mode; omit the
role or use `first_frame` for single-image image-to-video.


### Style controls


  Aspect-ratio hint such as `1:1`, `16:9`, `9:16`, `adaptive`, or a provider-specific value. OpenClaw normalizes or ignores unsupported values per provider.

Resolution hint such as `480P`, `720P`, `768P`, `1080P`, `4K`, or a provider-specific value. OpenClaw normalizes or ignores unsupported values per provider.

  Target duration in seconds (rounded to nearest provider-supported value).

Size hint when the provider supports it.

  Enable generated audio in the output when supported. Distinct from `audioRef*` (inputs).

Toggle provider watermarking when supported.

`adaptive` is a provider-specific sentinel: it is forwarded as-is to
providers that declare `adaptive` in their capabilities (e.g. BytePlus
Seedance uses it to auto-detect the ratio from the input image
dimensions). Providers that do not declare it surface the value via
`details.ignoredOverrides` in the tool result so the drop is visible.

### Advanced


  `"status"` returns the current session task; `"list"` inspects providers.

Provider/model override (e.g. `runway/gen4.5`).
Output filename hint.
Optional provider operation timeout in milliseconds. When omitte
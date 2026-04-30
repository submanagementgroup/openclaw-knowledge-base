---
domain: tools
topic: "Image Generation Tool: DALL-E, fal.ai, ComfyUI Providers and Configuration"
type: procedure
keywords:
  - image generation
  - DALL-E
  - fal.ai
  - ComfyUI
  - image tool
  - generate images
related:
  - providers/fal
  - providers/comfy
  - providers/openai
  - tools/media-overview
source: tools/image-generation.md
---

Image generation tool for OpenClaw agents. Supports OpenAI DALL-E, fal.ai, ComfyUI, and other providers.

The `image_generate` tool lets the agent create and edit images using your
configured providers. Generated images are delivered automatically as media
attachments in the agent's reply.

The tool only appears when at least one image-generation provider is
available. If you do not see `image_generate` in your agent's tools,
configure `agents.defaults.imageGenerationModel`, set up a provider API key,
or sign in with OpenAI Codex OAuth.

## Quick start

    Set an API key for at least one provider (for example `OPENAI_API_KEY`,
    `GEMINI_API_KEY`, `OPENROUTER_API_KEY`) or sign in with OpenAI Codex OAuth.

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openai/gpt-image-2",
            timeoutMs: 180_000,
          },
        },
      },
    }
    ```

    Codex OAuth uses the same `openai/gpt-image-2` model ref. When an
    `openai-codex` OAuth profile is configured, OpenClaw routes image
    requests through that OAuth profile instead of first trying
    `OPENAI_API_KEY`. Explicit `models.providers.openai` config (API key,
    custom/Azure base URL) opts back into the direct OpenAI Images API
    route.

    _"Generate an image of a friendly robot mascot."_

    The agent calls `image_generate` automatically. No tool allow-listing
    needed — it is enabled by default when a provider is available.

For OpenAI-compatible LAN endpoints such as LocalAI, keep the custom
`models.providers.openai.baseUrl` and explicitly opt in with
`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true`. Private and
internal image endpoints remain blocked by default.

## Common routes

| Goal                                                 | Model ref                                          | Auth                                   |
| ---------------------------------------------------- | -------------------------------------------------- | -------------------------------------- |
| OpenAI image generation with API billing             | `openai/gpt-image-2`                               | `OPENAI_API_KEY`                       |
| OpenAI image generation with Codex subscription auth | `openai/gpt-image-2`                               | OpenAI Codex OAuth                     |
| OpenAI transparent-background PNG/WebP               | `openai/gpt-image-1.5`                             | `OPENAI_API_KEY` or OpenAI Codex OAuth |
| DeepInfra image generation                           | `deepinfra/black-forest-labs/FLUX-1-schnell`       | `DEEPINFRA_API_KEY`                    |
| OpenRouter image generation                          | `openrouter/google/gemini-3.1-flash-image-preview` | `OPENROUTER_API_KEY`                   |
| LiteLLM image generation                             | `litellm/gpt-image-2`                              | `LITELLM_API_KEY`                      |
| Google Gemini image generation                       | `google/gemini-3.1-flash-image-preview`            | `GEMINI_API_KEY` or `GOOGLE_API_KEY`   |

The same `image_generate` tool handles text-to-image and reference-image
editing. Use `image` for one reference or `images` for multiple references.
Provider-supported output hints such as `quality`, `outputFormat`, and
`background` are forwarded when available and reported as ignored when a
provider does not support them. Bundled transparent-background support is
OpenAI-specific; other providers may still preserve PNG alpha if their
backend emits it.

## Supported providers

| Provider   | Default model                           | Edit support                       | Auth                                                  |
| ---------- | --------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| ComfyUI    | `workflow`                              | Yes (1 image, workflow-configured) | `COMFY_API_KEY` or `COMFY_CLOUD_API_KEY` for cloud    |
| DeepInfra  | `black-forest-labs/FLUX-1-schnell`      | Yes (1 image)                      | `DEEPINFRA_API_KEY`                                   |
| fal        | `fal-ai/flux/dev`                       | Yes                                | `FAL_KEY`                                             |
| Google     | `gemini-3.1-flash-image-preview`        | Yes                                | `GEMINI_API_KEY` or `GOOGLE_API_KEY`                  |
| LiteLLM    | `gpt-image-2`                           | Yes (up to 5 input images)         | `LITELLM_API_KEY`                                     |
| MiniMax    | `image-01`                              | Yes (subject reference)            | `MINIMAX_API_KEY` or MiniMax OAuth (`minimax-portal`) |
| OpenAI     | `gpt-image-2`                           | Yes (up to 4 images)               | `OPENAI_API_KEY` or OpenAI Codex OAuth                |
| OpenRouter | `google/gemini-3.1-flash-image-preview` | Yes (up to 5 input images)         | `OPENROUTER_API_KEY`                                  |
| Vydra      | `grok-imagine`                          | No                                 | `VYDRA_API_KEY`                                       |
| xAI        | `grok-imagine-image`                    | Yes (up to 5 images)               | `XAI_API_KEY`                                         |

Use `action: "list"` to inspect available providers and models at runtime:

```text
/tool image_generate action=list
```

## Provider capabilities

| Capability            | ComfyUI            | DeepInfra | fal               | Google         | MiniMax               | OpenAI         | Vydra | xAI            |
| --------------------- | ------------------ | --------- | ----------------- | -------------- | --------------------- | -------------- | ----- | -------------- |
| Generate (max count)  | Workflow-defined   | 4         | 4                 | 4              | 9                     | 4              | 1     | 4              |
| Edit / reference      | 1 image (workflow) | 1 image   | 1 image           | Up to 5 images | 1 image (subject ref) | Up to 5 images | —     | Up to 5 images |
| Size control          | —                  | ✓         | ✓                 | ✓              | —                     | Up to 4K       | —     | —              |
| Aspect ratio          | —                  | —         | ✓ (generate only) | ✓              | ✓                     | —              | —     | ✓              |
| Resolution (1K/2K/4K) | —                  | —         | ✓                 | ✓              | —                     | —              | —     | 1K, 2K         |

## Tool parameters

<ParamField path="prompt" type="string" required>
  Image generation prompt. Required for `action: "generate"`.
</ParamField>
<ParamField path="action" type='"generate" | "list"' default="generate">
  Use `"list"` to inspect available providers and models at runtime.
</ParamField>
<ParamField path="model" type="string">
  Provider/model override (e.g. `openai/gpt-image-2`). Use
  `openai/gpt-image-1.5` for transparent OpenAI backgrounds.
</ParamField>
<ParamField path="image" type="string">
  Single reference image path or URL for edit mode.
</ParamField>
<ParamField path="images" type="string[]">
  Multiple reference images for edit mode (up to 5 on supporting providers).
</ParamField>
<ParamField path="size" type="string">
  Size hint: `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `3840x2160`.
</ParamField>
<ParamField path="aspectRatio" type="string">
  Aspect ratio: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`.
</ParamField>
<ParamField path="resolution" type='"1K" | "2K" | "4K"'>Resolution hint.</ParamField>
<ParamField path="quality" type='"low" | "medium" | "high" | "auto"'>
  Quality hint when the provider supports it.
</ParamField>
<ParamField path="outputFormat" type='"png" | "jpeg" | "webp"'>
  Output format hint when the provider supports it.
</ParamField>
<ParamField path="background" type='"transparent" | "opaque" | "auto"'>
  Backg

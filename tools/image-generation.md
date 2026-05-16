---
domain: tools
topic: "Image Generation Tool"
type: procedure
keywords:
  - image generation
  - AI images
  - image_generate
  - DALL-E
  - Stable Diffusion
  - image tool
source: tools/image-generation.md
---

The `image_generate` tool lets the agent create and edit images using your
configured providers. Generated images are delivered automatically as media
attachments in the agent's reply.

> **Note:** The tool only appears when at least one image-generation provider is
available. If you do not see `image_generate` in your agent's tools,
configure `agents.defaults.imageGenerationModel`, set up a provider API key,
or sign in with OpenAI Codex OAuth.


## Quick start

**Configure auth**

Set an API key for at least one provider (for example `OPENAI_API_KEY`,
    `GEMINI_API_KEY`, `OPENROUTER_API_KEY`) or sign in with OpenAI Codex OAuth.
  
**Pick a default model (optional)**

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

  
**Ask the agent**

_"Generate an image of a friendly robot mascot."_

    The agent calls `image_generate` automatically. No tool allow-listing
    needed - it is enabled by default when a provider is available.

  
> **Note:** For OpenAI-compatible LAN endpoints such as LocalAI, keep the custom
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
| fal        | `fal-ai/flux/dev`                       | Yes (model-specific limits)        | `FAL_KEY`                                             |
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

| Capability            | ComfyUI            | DeepInfra | fal                       | Google         | MiniMax               | OpenAI         | Vydra | xAI            |
| --------------------- | ------------------ | --------- | ------------------------- | -------------- | --------------------- | -------------- | ----- | -------------- |
| Generate (max count)  | Workflow-defined   | 4         | 4                         | 4              | 9                     | 4              | 1     | 4              |
| Edit / reference      | 1 image (workflow) | 1 image   | Flux: 1; GPT: 10; NB2: 14 | Up to 5 images | 1 image (subject ref) | Up to 5 images | -     | Up to 5 images |
| Size control          | -                  | ✓         | ✓                         | ✓              | -                     | Up to 4K       | -     | -              |
| Aspect ratio          | -                  | -         | ✓                         | ✓              | ✓                     | -              | -     | ✓              |
| Resolution (1K/2K/4K) | -                  | -         | ✓                         | ✓              | -                     | -              | -     | 1K, 2K         |

## Tool parameters


  Image generation prompt. Required for `action: "generate"`.


  Use `"list"` to inspect available providers and models at runtime.


  Provider/model override (e.g. `openai/gpt-image-2`). Use
  `openai/gpt-image-1.5` for transparent OpenAI backgrounds.


  Single reference image path or URL for edit mode.


  Multiple reference images for edit mode (up to 5 on supporting providers).


  Size hint: `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `3840x2160`.


  Aspect ratio: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`.

Resolution hint.

  Quality hint when the provider supports it.


  Output format hint when the provider supports it.


  Background hint when the provider supports it. Use `transparent` with
  `outputFormat: "png"` or `"webp"` for transparency-capable providers.

Number of images to generate (1-4).

  Optional provider request timeout in milliseconds. When Codex calls
  `image_generate` through dynamic tools, this per-call value still overrides
  the configured default and is capped at 600000 ms.

Output filename hint.

  OpenAI-only hints: `background`, `moderation`, `outputCompression`, and `user`.


> **Note:** Not all providers support all parameters. When a fallback provider supports a
nearby geometry option instead of the exact requested one, OpenClaw remaps to
the closest supported size, aspect ratio, or resolution before submission.
Unsupported output hints are dropped for providers that do not declare
support and reported in the tool result. Tool results report the applied
settings; `details.normalization` captures any requested-to-applied
translation.


## Configuration

### Model selection

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-2",
        timeoutMs: 180_000,
        fallbacks: [
          "openrouter/google/gemini-3.1-flash-image-preview",
          "google/gemini-3.1-flash-image-preview",
          "fal/fal-ai/flux/dev",
        ],
      },
    },
  },
}
```

### Provider selection order

OpenClaw tries providers in this order:

1. **`model` parameter** from the tool call (if the agent specifies one).
2. **`imageGenerationModel.primary`** from config.
3. **`imageGenerationModel.fallbacks`** in order.
4. **Auto-detection** - auth-backed provider defaults only:
   - current default provider first;
   - remaining registered image-generation providers in provider-id order.

If a provider fails (auth error, rate limit, etc.), the next configured
candidate is tried automatically. If all fail, the error includes details
from each attempt.

### Per-call model overrides are exact

A per-call `model` override tries only that provider/model and does
    not continue to configured primary/fallback or auto-detected providers.
  ### Auto-detection is auth-aware

A provider default only enters the candidate list when OpenClaw can
    actually authenticate that provider. Set
    `agents.defaults.mediaGenerationAutoProviderFallback: false` to use only
    explicit `model`, `primary`, and `fallbacks` entries.
  ### Timeouts

Set `agents.defaults.imageGenerationModel.timeoutMs` for slow image
    backends. A per-call `timeoutMs` tool parameter overrides the configured
    default. Codex dynamic-tool calls honor the same timeout budget, bounded
    by OpenClaw's 600000 ms dynamic-tool bridge maximum.
  ### Inspect at runtime

Use `action: "list"` to inspect the currently registered providers,
    their default models, and auth env-var hints.
  ### Image editing

OpenAI, OpenRouter, Google, DeepInfra, fal, MiniMax, ComfyUI, and xAI support editing
reference images. Pass a reference image path or URL:

```text
"Generate a watercolor version of this photo" + image: "/path/to/photo.jpg"
```

OpenAI, OpenRouter, Google, and xAI support up to 5 reference images via the
`images` parameter. fal supports 1 reference image for Flux image-to-image, up
to 10 for GPT Image 2 edits, and up to 14 for Nano Banana 2 edits. MiniMax and
ComfyUI support 1.

## Provider deep dives

### OpenAI gpt-image-2 (and gpt-image-1.5)

OpenAI image generation defaults to `openai/gpt-image-2`. If an
    `openai-codex` OAuth profile is configured, OpenClaw reuses the same
    OAuth profile used by Codex subscription chat models and sends the
    image request through the Codex Responses backend. Legacy Codex base
    URLs such as `https://chatgpt.com/backend-api` are canonicalized to
    `https://chatgpt.com/backend-api/codex` for image requests. OpenClaw
    does **not** silently fall back to `OPENAI_API_KEY` for that request -
    to force direct OpenAI Images API routing, configure
    `models.providers.openai` explicitly with an API key, custom base URL,
    or Azure endpoint.

    The `openai/gpt-image-1.5`, `openai/gpt-image-1`, and
    `openai/gpt-image-1-mini` models can still be selected explicitly. Use
    `gpt-image-1.5` for transparent-background PNG/WebP output; the current
    `gpt-image-2` API rejects `background: "transparent"`.

    `gpt-image-2` supports both text-to-image generation and
    reference-image editing through the same `image_generate` tool.
    OpenClaw forwards `prompt`, `count`, `size`, `quality`, `outputFormat`,
    and reference images to
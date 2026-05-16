---
domain: nodes
topic: "Media Nodes: Images, Media Understanding, Location"
type: reference
keywords:
  - images node
  - media understanding
  - location node
  - multimedia
  - media input
source: 
  - nodes/images.md
  - nodes/media-understanding.md
  - nodes/location-command.md
---

## Images Node

The WhatsApp channel runs via **Baileys Web**. This document captures the current media handling rules for send, gateway, and agent replies.

## Goals

- Send media with optional captions via `openclaw message send --media`.
- Allow auto-replies from the web inbox to include media alongside text.
- Keep per-type limits sane and predictable.

## CLI Surface

- `openclaw message send --media <path-or-url> [--message <caption>]`
  - `--media` optional; caption can be empty for media-only sends.
  - `--dry-run` prints the resolved payload; `--json` emits `{ channel, to, messageId, mediaUrl, caption }`.

## WhatsApp Web channel behavior

- Input: local file path **or** HTTP(S) URL.
- Flow: load into a Buffer, detect media kind, and build the correct payload:
  - **Images:** resize & recompress to JPEG (max side 2048px) targeting `channels.whatsapp.mediaMaxMb` (default: 50 MB).
  - **Audio/Voice/Video:** pass-through up to 16 MB; audio is sent as a voice note (`ptt: true`).
  - **Documents:** anything else, up to 100 MB, with filename preserved when available.
- WhatsApp GIF-style playback: send an MP4 with `gifPlayback: true` (CLI: `--gif-playback`) so mobile clients loop inline.
- MIME detection prefers magic bytes, then headers, then file extension.
- Caption comes from `--message` or `reply.text`; empty caption is allowed.
- Logging: non-verbose shows `↩️`/`✅`; verbose includes size and source path/URL.

## Auto-Reply Pipeline

- `getReplyFromConfig` returns `{ text?, mediaUrl?, mediaUrls? }`.
- When media is present, the web sender resolves local paths or URLs using the same pipeline as `openclaw message send`.
- Multiple media entries are sent sequentially if provided.

## Inbound media to commands (Pi)

- When inbound web messages include media, OpenClaw downloads to a temp file and exposes templating variables:
  - `{{MediaUrl}}` pseudo-URL for the inbound media.
  - `{{MediaPath}}` local temp path written before running the command.
- When a per-session Docker sandbox is enabled, inbound media is copied into the sandbox workspace and `MediaPath`/`MediaUrl` are rewritten to a relative path like `media/inbound/<filename>`.
- Media understanding (if configured via `tools.media.*` or shared `tools.media.models`) runs before templating and can insert `[Image]`, `[Audio]`, and `[Video]` blocks into `Body`.
  - Audio sets `{{Transcript}}` and uses the transcript for command parsing so slash commands still work.
  - Video and image descriptions preserve any caption text for command parsing.
  - If the active primary image model already supports vision natively, OpenClaw skips the `[Image]` summary block and passes the original image to the model instead.
- By default only the first matching image/audio/video attachment is processed; set `tools.media.<cap>.attachments` to process multiple attachments.

## Limits and errors

**Outbound send caps (WhatsApp web send)**

- Images: up to `channels.whatsapp.mediaMaxMb` (default: 50 MB) after recompression.
- Audio/voice/video: 16 MB cap; documents: 100 MB cap.
- Oversize or unreadable media → clear error in logs and the reply is skipped.

**Media understanding caps (transcription/description)**

- Image default: 10 MB (`tools.media.image.maxBytes`).
- Audio default: 20 MB (`tools.media.audio.maxBytes`).
- Video default: 50 MB (`tools.media.video.maxBytes`).
- Oversize media skips understanding, but replies still go through with the original body.

## Notes for Tests

- Cover send + reply flows for image/audio/document cases.
- Validate recompression for images (size bound) and voice-note flag for audio.
- Ensure multi-media replies fan out as sequential sends.

## Related

- [Camera capture](/nodes/camera)
- [Media understanding](/nodes/media-understanding)
- [Audio and voice notes](/nodes/audio)


## Media Understanding

OpenClaw can **summarize inbound media** (image/audio/video) before the reply pipeline runs. It auto-detects when local tools or provider keys are available, and can be disabled or customized. If understanding is off, models still receive the original files/URLs as usual.

Vendor-specific media behavior is registered by vendor plugins, while OpenClaw core owns the shared `tools.media` config, fallback order, and reply-pipeline integration.

## Goals

- Optional: pre-digest inbound media into short text for faster routing + better command parsing.
- Preserve original media delivery to the model (always).
- Support **provider APIs** and **CLI fallbacks**.
- Allow multiple models with ordered fallback (error/size/timeout).

## High-level behavior

**Collect attachments**

Collect inbound attachments (`MediaPaths`, `MediaUrls`, `MediaTypes`).
  
**Select per-capability**

For each enabled capability (image/audio/video), select attachments per policy (default: **first**).
  
**Choose model**

Choose the first eligible model entry (size + capability + auth).
  
**Fallback on failure**

If a model fails or the media is too large, **fall back to the next entry**.
  
**Apply success block**

On success:

    - `Body` becomes `[Image]`, `[Audio]`, or `[Video]` block.
    - Audio sets `{{Transcript}}`; command parsing uses caption text when present, otherwise the transcript.
    - Captions are preserved as `User text:` inside the block.

  
If understanding fails or is disabled, **the reply flow continues** with the original body + attachments.

## Config overview

`tools.media` supports **shared models** plus per-capability overrides:

### Top-level keys

- `tools.media.models`: shared model list (use `capabilities` to gate).
    - `tools.media.image` / `tools.media.audio` / `tools.media.video`:
      - defaults (`prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`)
      - provider overrides (`baseUrl`, `headers`, `providerOptions`)
      - Deepgram audio options via `tools.media.audio.providerOptions.deepgram`
      - audio transcript echo controls (`echoTranscript`, default `false`; `echoFormat`)
      - optional **per-capability `models` list** (preferred before shared models)
      - `attachments` policy (`mode`, `maxAttachments`, `prefer`)
      - `scope` (optional gating by channel/chatType/session key)
    - `tools.media.concurrency`: max concurrent capability runs (default **2**).

  ```json5
{
  tools: {
    media: {
      models: [
        /* shared list */
      ],
      image: {
        /* optional overrides */
      },
      audio: {
        /* optional overrides */
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
      video: {
        /* optional overrides */
      },
    },
  },
}
```

### Model entries

Each `models[]` entry can be **provider** or **CLI**:

**Provider entry:**

```json5
    {
      type: "provider", // default if omitted
      provider: "openai",
      model: "gpt-5.5",
      prompt: "Describe the image in <= 500 chars.",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"], // optional, used for multi-modal entries
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  
**CLI entry:**

```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    CLI templates can also use:

    - `{{MediaDir}}` (directory containing the media file)
    - `{{OutputDir}}` (scratch dir created for this run)
    - `{{OutputBase}}` (scratch file base path, no extension)

  
## Defaults and limits

Recommended defaults:

- `maxChars`: **500** for image/video (short, command-friendly)
- `maxChars`: **unset** for audio (full transcript unless you set a limit)
- `maxBytes`:
  - image: **10MB**
  - audio: **20MB**
  - video: **50MB**

### Rules

- If media exceeds `maxBytes`, that model is skipped and the **next model is tried**.
    - Audio files smaller than **1024 bytes** are treated as empty/corrupt and skipped before provider/CLI transcription; inbound reply context receives a deterministic placeholder transcript so the agent knows the note was too small.
    - If the model returns more than `maxChars`, output is trimmed.
    - `prompt` defaults to simple "Describe the {media}." plus the `maxChars` guidance (image/video only).
    - If the active primary image model already supports vision natively, OpenClaw skips the `[Image]` summary block and passes the original image into the model instead.
    - If a Gateway/WebChat primary model is text-only, image attachments are preserved as offloaded `media://inbound/*` refs so the image/PDF tools or configured image model can still inspect them instead of losing the attachment.
    - Explicit `openclaw infer image describe --model <provider/model>` requests are different: they run that image-capable provider/model directly, including Ollama refs such as `ollama/qwen2.5vl:7b`.
    - If `<capability>.enabled: true` but no models are configured, OpenClaw tries the **active reply model** when its provider supports the capability.

  ### Auto-detect media understanding (default)

If `tools.media.<capability>.enabled` is **not** set to `false` and you haven't configured models, OpenClaw auto-detects in this order and **stops at the first working option**:

**Active reply model**

Active reply model when its provider supports the capability.
  
**agents.defaults.imageModel**

`agents.defaults.imageModel` primary/fallback refs (image only).
    Prefer `provider/model` refs. Bare refs are qualified from configured image-capable provider model entries only when the match is unique.
  
**Local CLIs (audio only)**

Local CLIs (if installed):

    - `sherpa-onnx-offline` (requires `SHERPA_ONNX_MODEL_DIR` with encoder/decoder/joiner/tokens)
    - `whisper-cli` (`whisper-cpp`; uses `WHISPER_CPP_MODEL` or the bundled tiny model)
    - `whisper` (Python CLI; downloads models automatically)

  
**Gemini CLI**

`gemini` using `read_many_files`.
  
**Provider auth**

- Configured `models.providers.*` entries that support the capability are tried before the bundled fallback order.
    - Image-only config providers with an image-capable model auto-register for media understanding even when they are not a bundled vendor plugin.
    - Ollama image understanding is available when selected explicitly, for example through `agents.defaults.imageModel` or `openclaw infer image describe --model ollama/<vision-model>`.

    Bundled fallback order:

    - Audio: OpenAI → Groq → xAI → Deepgram → OpenRouter → Google → SenseAudio → ElevenLabs → Mistral
    - Image: OpenAI → Anthropic → Google → MiniMax → MiniMax Portal → Z.AI
    - Video: Google → Qwen → Moonshot

  
To disable auto-detection, set:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

> **Note:** Binary detection is best-effort across macOS/Linux/Windows; ensure the CLI is on `PATH` (we expand `~`), or set an explicit CLI model with a full command path.


### Proxy environment support (provider models)

When provider-based **audio** and **video** media understanding is enabled, OpenClaw honors standard outbound proxy environment variables for provider HTTP calls:

- `HTTPS_PROXY`
- `HTTP_PROXY`
- `ALL_PROXY`
- `https_proxy`
- `http_proxy`
- `all_proxy`

If no proxy env vars are set, media understanding uses direct egress. If the proxy value is malformed, OpenClaw logs a warning and falls back to direct fetch.

## Capabilities (optional)

If you set `capabilities`, the entry only runs for those media types. For shared lists, OpenClaw can infer defaults:

- `openai`, `anthropic`, `minimax`: **image**
- `minimax-portal`: **image**
- `moonshot`: **image + video**
- `openrouter`: **image + audio**
- `google` (Gemini API): **image + audio + video**
- `qwen`: **image + video**
- `mistral`: **audio**
- `zai`: **image**
- `groq`: **audio**
- `xai`: **audio**
- `deepgram`: **audio**
- Any `models.providers.<id>.models[]` catalog with an image-capable model: **image**

For CLI entries, **set `capabilities` explicitly** to avoid surprising matches. If you omit `capabilities`, the entry is eligible for the list it appears in.

## Provider support matrix (OpenClaw integrations)

| Capability | Provider integration                                                                                                         | Notes                                                                                                                                                                                                                                   |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Image      | OpenAI, OpenAI Codex OAuth, Codex app-server, OpenRouter, Anthropic, Google, MiniMax, Moonshot, Qwen, Z.AI, config providers | Vendor plugins register image support; `openai-codex/*` uses OAuth provider plumbing; `codex/*` uses a bounded Codex app-server turn; MiniMax and MiniMax OAuth both use `MiniMax-VL-01`; image-capable config providers auto-register. |
| Audio      | OpenAI, Groq, xAI, Deepgram, OpenRouter, Google, SenseAudio, ElevenLabs, Mistral                                             | Provider transcription (Whisper/Groq/xAI/Deepgram/OpenRouter STT/Gemini/SenseAudio/Scribe/Voxtral).                                                                                                                                     |
| Video      | Google, Qwen, Moonshot                                                                                                       | Provider video understanding via vendor plugins; Qwen video understanding uses the Standard DashScope endpoints.                                                                                                                        |

> **Note:** **MiniMax note**

- `minimax` and `minimax-portal` image understanding comes from the plugin-owned `MiniMax-VL-01` media provider.
- The bundled MiniMax text catalog still starts text-only; explicit `models.providers.minimax` entries materialize image-capable M2.7 chat refs.



## Model selection guidance

- Prefer the strongest latest-generation model available for each media capability when quality and safety matter.
- For tool-enabled agents handling untrusted inputs, avoid older/weaker media models.
- Keep at least one fallback per capability for availability (quality model + faster/cheaper model).
- CLI fallbacks (`whisper-cli`, `whisper`, `gemini`) are useful when provider APIs are unavailable.
- `parakeet-mlx` note: with `--output-dir`, OpenClaw reads `<output-dir>/<media-basename>.txt` when output format is `txt` (or unspecified); non-`txt` formats fall back to stdout.

## Attachment policy

Per-capability `attachments` controls which attachments are processed:


  Whether to process the first selected attachment or all of them.


  Cap the number processed.


  Selection preference among candidate attachments.


When `mode: "all"`, outputs are labeled `[Image 1/2]`, `[Audio 2/2]`, etc.

### File-attachment extraction behavior

- Extracted file text is wrapped as **untrusted external content** before it is appended to the media prompt.
    - The injected block uses explicit boundary markers like `<<>>` / `<<>>` and includes a `Source: External` metadata line.
    - This attachment-extraction path intentionally omits the long `SECURITY NOTICE:` banner to avoid bloating the media prompt; the boundary markers and metadata still remain.
    - If a file has no extractable text, OpenClaw injects `[No extractable text]`.
    - If a PDF falls back to rendered page images in this path, the media prompt keeps the placeholder `[PDF content rendered to images; images not forwarded to model]` because this attachment-extraction step forwards text blocks, not the rendered PDF images.

  ## Config examples

**Shared models + overrides:**

```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.5", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  
**Audio + video only:**

```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{MediaPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  
**Image-only:**

```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.5" },
              { provider: "anthropic", model: "claude-opus-4-6" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  
**Multi-modal single entry:**

```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  
## Status output

When media understanding runs, `/status` includes a short summary line:

```
📎 Media: image ok (openai/gpt-5.4) · audio skipped (maxBytes)
```

This shows per-capability outcomes and the chosen provider/model when applicable.

## Notes

- Understanding is **best-effort**. Errors do not block replies.
- Attachments are still passed to models even when understanding is disabled.
- Use `scope` to limit where understanding runs (e.g. only DMs).

## Related

- [Configuration](/gateway/configuration)
- [Image & media support](/nodes/images)


## Location Command

## TL;DR

- `location.get` is a node command (via `node.invoke`).
- Off by default.
- Android app settings use a selector: Off / While Using.
- Separate toggle: Precise Location.

## Why a selector (not just a switch)

OS permissions are multi-level. We can expose a selector in-app, but the OS still decides the actual grant.

- iOS/macOS may expose **While Using** or **Always** in system prompts/Settings.
- Android app currently supports foreground location only.
- Precise location is a separate grant (iOS 14+ "Precise", Android "fine" vs "coarse").

Selector in UI drives our requested mode; actual grant lives in OS settings.

## Settings model

Per node device:

- `location.enabledMode`: `off | whileUsing`
- `location.preciseEnabled`: bool

UI behavior:

- Selecting `whileUsing` requests foreground permission.
- If OS denies requested level, revert to the highest granted level and show status.

## Permissions mapping (node.permissions)

Optional. macOS node reports `location` via the permissions map; iOS/Android may omit it.

## Command: `location.get`

Called via `node.invoke`.

Params (suggested):

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

Response payload:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

Errors (stable codes):

- `LOCATION_DISABLED`: selector is off.
- `LOCATION_PERMISSION_REQUIRED`: permission missing for requested mode.
- `LOCATION_BACKGROUND_UNAVAILABLE`: app is backgrounded but only While Using allowed.
- `LOCATION_TIMEOUT`: no fix in time.
- `LOCATION_UNAVAILABLE`: system failure / no providers.

## Background behavior

- Android app denies `location.get` while backgrounded.
- Keep OpenClaw open when requesting location on Android.
- Other node platforms may differ.

## Model/tooling integration

- Tool surface: `nodes` tool adds `location_get` action (node required).
- CLI: `openclaw nodes location get --node <id>`.
- Agent guidelines: only call when user enabled location and understands the scope.

## UX copy (suggested)

- Off: "Location sharing is disabled."
- While Using: "Only when OpenClaw is open."
- Precise: "Use precise GPS location. Toggle off to share approximate location."

## Related

- [Channel location parsing](/channels/location)
- [Camera capture](/nodes/camera)
- [Talk mode](/nodes/talk)

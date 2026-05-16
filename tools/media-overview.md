---
domain: tools
topic: "Media Tools Overview: Image, Audio, Video"
type: concept
keywords:
  - media tools
  - image generation
  - audio
  - video generation
  - TTS
  - media overview
source: tools/media-overview.md
---

OpenClaw generates images, videos, and music, understands inbound media
(images, audio, video), and speaks replies aloud with text-to-speech. All
media capabilities are tool-driven: the agent decides when to use them based
on the conversation, and each tool only appears when at least one backing
provider is configured.

Live speech uses the Talk session contract instead of the one-shot media tool
path. Talk has three modes: provider-native `realtime`, local or streaming
`stt-tts`, and `transcription` for observe-only speech capture. Those modes
share provider catalogs, event envelopes, and cancellation semantics with
telephony, meetings, browser realtime, and native push-to-talk clients.

## Capabilities

**Image generation:** Create and edit images from text prompts or reference images via
    `image_generate`. Synchronous — completes inline with the reply.
  

**Video generation:** Text-to-video, image-to-video, and video-to-video via `video_generate`.
    Async — runs in the background and posts the result when ready.
  

**Music generation:** Generate music or audio tracks via `music_generate`. Async on shared
    providers; ComfyUI workflow path runs synchronously.
  

**Text-to-speech:** Convert outbound replies to spoken audio via the `tts` tool plus
    `messages.tts` config. Synchronous.
  

**Media understanding:** Summarize inbound images, audio, and video using vision-capable model
    providers and dedicated media-understanding plugins.
  

**Speech-to-text:** Transcribe inbound voice messages through batch STT or Voice Call
    streaming STT providers.
  

## Provider capability matrix

| Provider    | Image | Video | Music | TTS | STT | Realtime voice | Media understanding |
| ----------- | :---: | :---: | :---: | :-: | :-: | :------------: | :-----------------: |
| Alibaba     |       |   ✓   |       |     |     |                |                     |
| BytePlus    |       |   ✓   |       |     |     |                |                     |
| ComfyUI     |   ✓   |   ✓   |   ✓   |     |     |                |                     |
| DeepInfra   |   ✓   |   ✓   |       |  ✓  |  ✓  |                |          ✓          |
| Deepgram    |       |       |       |     |  ✓  |       ✓        |                     |
| ElevenLabs  |       |       |       |  ✓  |  ✓  |                |                     |
| fal         |   ✓   |   ✓   |       |     |     |                |                     |
| Google      |   ✓   |   ✓   |   ✓   |  ✓  |     |       ✓        |          ✓          |
| Gradium     |       |       |       |  ✓  |     |                |                     |
| Local CLI   |       |       |       |  ✓  |     |                |                     |
| Microsoft   |       |       |       |  ✓  |     |                |                     |
| MiniMax     |   ✓   |   ✓   |   ✓   |  ✓  |     |                |                     |
| Mistral     |       |       |       |     |  ✓  |                |                     |
| OpenAI      |   ✓   |   ✓   |       |  ✓  |  ✓  |       ✓        |          ✓          |
| OpenRouter  |   ✓   |   ✓   |       |  ✓  |  ✓  |                |          ✓          |
| Qwen        |       |   ✓   |       |     |     |                |                     |
| Runway      |       |   ✓   |       |     |     |                |                     |
| SenseAudio  |       |       |       |     |  ✓  |                |                     |
| Together    |       |   ✓   |       |     |     |                |                     |
| Vydra       |   ✓   |   ✓   |       |  ✓  |     |                |                     |
| xAI         |   ✓   |   ✓   |       |  ✓  |  ✓  |                |          ✓          |
| Xiaomi MiMo |   ✓   |       |       |  ✓  |     |                |          ✓          |

> **Note:** Media understanding uses any vision-capable or audio-capable model registered
in your provider config. The matrix above lists providers with dedicated
media-understanding support; most multimodal LLM providers (Anthropic, Google,
OpenAI, etc.) can also understand inbound media when configured as the active
reply model.


## Async vs synchronous

| Capability      | Mode         | Why                                                                                                  |
| --------------- | ------------ | ---------------------------------------------------------------------------------------------------- |
| Image           | Synchronous  | Provider responses return in seconds; completes inline with reply.                                   |
| Text-to-speech  | Synchronous  | Provider responses return in seconds; attached to the reply audio.                                   |
| Video           | Asynchronous | Provider processing takes 30 s to several minutes; slow queues can run up to the configured timeout. |
| Music (shared)  | Asynchronous | Same provider-processing characteristic as video.                                                    |
| Music (ComfyUI) | Synchronous  | Local workflow runs inline against the configured ComfyUI server.                                    |

For async tools, OpenClaw submits the request to the provider, returns a task
id immediately, and tracks the job in the task ledger. The agent continues
responding to other messages while the job runs. When the provider finishes,
OpenClaw wakes the agent with the generated media paths so it can tell the
user and, when required by source-delivery policy, relay the result through
the message tool. For message-tool-only group/channel routes, OpenClaw treats
missing message-tool delivery evidence as a failed completion attempt and sends
the generated media fallback directly to the original channel.

## Speech-to-text and Voice Call

Deepgram, DeepInfra, ElevenLabs, Mistral, OpenAI, OpenRouter, SenseAudio, and xAI can all transcribe
inbound audio through the batch `tools.media.audio` path when configured.
Channel plugins that preflight a voice note for mention gating or command
parsing mark the transcribed attachment on the inbound context, so the shared
media-understanding pass reuses that transcript instead of making a second
STT call for the same audio.

Deepgram, ElevenLabs, Mistral, OpenAI, and xAI also register Voice Call
streaming STT providers, so live phone audio can be forwarded to the selected
vendor without waiting for a completed recording.

For live user conversations, prefer [Talk mode](/nodes/talk). Batch audio
attachments stay on the media path; browser realtime, native push-to-talk,
telephony, and meeting audio should use Talk events and the session-scoped
catalogs returned by the Gateway.

## Provider mappings (how vendors split across surfaces)

### Google

Image, video, music, batch TTS, backend realtime voice, and
    media-understanding surfaces.
  ### OpenAI

Image, video, batch TTS, batch STT, Voice Call streaming STT, backend
    realtime voice, and memory-embedding surfaces.
  ### DeepInfra

Chat/model routing, image generation/editing, text-to-video, batch TTS,
    batch STT, image media understanding, and memory-embedding surfaces.
    DeepInfra-native rerank/classification/object-detection models are not
    registered until OpenClaw has dedicated provider contracts for those
    categories.
  ### xAI

Image, video, search, code-execution, batch TTS, batch STT, and Voice
    Call streaming STT. xAI Realtime voice is an upstream capability but is
    not registered in OpenClaw until the shared realtime-voice contract can
    represent it.
  ## Related

- [Image generation](/tools/image-generation)
- [Video generation](/tools/video-generation)
- [Music generation](/tools/music-generation)
- [Text-to-speech](/tools/tts)
- [Media understanding](/nodes/media-understanding)
- [Audio nodes](/nodes/audio)
- [Talk mode](/nodes/talk)

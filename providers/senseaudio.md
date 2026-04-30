---
domain: providers
topic: "SenseAudio Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - SenseAudio
  - senseaudio provider
  - AI provider
  - model config
  - API key
  - SenseAudio
  - audio
  - TTS
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/senseaudio.md
---

SenseAudio provider setup, configuration, and model reference for OpenClaw.

# SenseAudio

SenseAudio can transcribe inbound audio/voice-note attachments through
OpenClaw's shared `tools.media.audio` pipeline. OpenClaw posts multipart audio
to the OpenAI-compatible transcription endpoint and injects the returned text
as `{{Transcript}}` plus an `[Audio]` block.

| Detail        | Value                                            |
| ------------- | ------------------------------------------------ |
| Website       | [senseaudio.cn](https://senseaudio.cn)           |
| Docs          | [senseaudio.cn/docs](https://senseaudio.cn/docs) |
| Auth          | `SENSEAUDIO_API_KEY`                             |
| Default model | `senseaudio-asr-pro-1.5-260319`                  |
| Default URL   | `https://api.senseaudio.cn/v1`                   |

## Getting Started

    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```

    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "senseaudio", model: "senseaudio-asr-pro-1.5-260319" }],
          },
        },
      },
    }
    ```

    Send an audio message through any connected channel. OpenClaw uploads the
    audio to SenseAudio and uses the transcript in the reply pipeline.

## Options

| Option     | Path                                  | Description                         |
| ---------- | ------------------------------------- | ----------------------------------- |
| `model`    | `tools.media.audio.models[].model`    | SenseAudio ASR model id             |
| `language` | `tools.media.audio.models[].language` | Optional language hint              |
| `prompt`   | `tools.media.audio.prompt`            | Optional transcription prompt       |
| `baseUrl`  | `tools.media.audio.baseUrl` or model  | Override the OpenAI-compatible base |
| `headers`  | `tools.media.audio.request.headers`   | Extra request headers               |

SenseAudio is batch STT only in OpenClaw. Voice Call realtime transcription
continues to use providers with streaming STT support.

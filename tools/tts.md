---
domain: tools
topic: "Text-to-Speech Tool: TTS Providers, Personas, Voice Config, and Auto-TTS"
type: procedure
keywords:
  - TTS
  - text to speech
  - tts tool
  - ElevenLabs
  - OpenAI TTS
  - Azure TTS
  - voice
  - persona
related:
  - providers/elevenlabs
  - providers/azure-speech
  - nodes/audio
source: tools/tts.md
---

Text-to-speech (TTS) in OpenClaw converts agent text replies to audio. Multiple providers supported: OpenAI TTS, ElevenLabs, Azure Speech, and others.

## Quick Start

```json5
{
  agents: {
    defaults: {
      tts: {
        enabled: true,
        provider: "openai",   // or "elevenlabs", "azure", "deepgram"
      }
    }
  }
}
```

## TTS Providers and Configuration

OpenClaw can convert outbound replies into audio across **14 speech providers**
and deliver native voice messages on Feishu, Matrix, Telegram, and WhatsApp,
audio attachments everywhere else, and PCM/Ulaw streams for telephony and Talk.

## Quick start

    OpenAI and ElevenLabs are the most reliable hosted options. Microsoft and
    Local CLI work without an API key. See the [provider matrix](#supported-providers)
    for the full list.

    Export the env var for your provider (for example `OPENAI_API_KEY`,
    `ELEVENLABS_API_KEY`). Microsoft and Local CLI need no key.

    Set `messages.tts.auto: "always"` and `messages.tts.provider`:

    ```json5
    {
      messages: {
        tts: {
          auto: "always",
          provider: "elevenlabs",
        },
      },
    }
    ```

    `/tts status` shows the current state. `/tts audio Hello from OpenClaw`
    sends a one-off audio reply.

Auto-TTS is **off** by default. When `messages.tts.provider` is unset,
OpenClaw picks the first configured provider in registry auto-select order.

## Supported providers

| Provider          | Auth                                                                                                             | Notes                                                                   |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **Azure Speech**  | `AZURE_SPEECH_KEY` + `AZURE_SPEECH_REGION` (also `AZURE_SPEECH_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`)          | Native Ogg/Opus voice-note output and telephony.                        |
| **DeepInfra**     | `DEEPINFRA_API_KEY`                                                                                              | OpenAI-compatible TTS. Defaults to `hexgrad/Kokoro-82M`.                |
| **ElevenLabs**    | `ELEVENLABS_API_KEY` or `XI_API_KEY`                                                                             | Voice cloning, multilingual, deterministic via `seed`.                  |
| **Google Gemini** | `GEMINI_API_KEY` or `GOOGLE_API_KEY`                                                                             | Gemini API TTS; persona-aware via `promptTemplate: "audio-profile-v1"`. |
| **Gradium**       | `GRADIUM_API_KEY`                                                                                                | Voice-note and telephony output.                                        |
| **Inworld**       | `INWORLD_API_KEY`                                                                                                | Streaming TTS API. Native Opus voice-note and PCM telephony.            |
| **Local CLI**     | none                                                                                                             | Runs a configured local TTS command.                                    |
| **Microsoft**     | none                                                                                                             | Public Edge neural TTS via `node-edge-tts`. Best-effort, no SLA.        |
| **MiniMax**       | `MINIMAX_API_KEY` (or Token Plan: `MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`)      | T2A v2 API. Defaults to `speech-2.8-hd`.                                |
| **OpenAI**        | `OPENAI_API_KEY`                                                                                                 | Also used for auto-summary; supports persona `instructions`.            |
| **OpenRouter**    | `OPENROUTER_API_KEY` (can reuse `models.providers.openrouter.apiKey`)                                            | Default model `hexgrad/kokoro-82m`.                                     |
| **Volcengine**    | `VOLCENGINE_TTS_API_KEY` or `BYTEPLUS_SEED_SPEECH_API_KEY` (legacy AppID/token: `VOLCENGINE_TTS_APPID`/`_TOKEN`) | BytePlus Seed Speech HTTP API.                                          |
| **Vydra**         | `VYDRA_API_KEY`                                                                                                  | Shared image, video, and speech provider.                               |
| **xAI**           | `XAI_API_KEY`                                                                                                    | xAI batch TTS. Native Opus voice-note is **not** supported.             |
| **Xiaomi MiMo**   | `XIAOMI_API_KEY`                                                                                                 | MiMo TTS through Xiaomi chat completions.                               |

If multiple providers are configured, the selected one is used first and the
others are fallback options. Auto-summary uses `summaryModel` (or
`agents.defaults.model.primary`), so that provider must also be authenticated
if you keep summaries enabled.

The bundled **Microsoft** provider uses Microsoft Edge's online neural TTS
service via `node-edge-tts`. It is a public web service without a published
SLA or quota — treat it as best-effort. The legacy provider id `edge` is
normalized to `microsoft` and `openclaw doctor --fix` rewrites persisted
config; new configs should always use `microsoft`.

## Configuration

TTS config lives under `messages.tts` in `~/.openclaw/openclaw.json`. Pick a
preset and adapt the provider block:

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "azure-speech",
      providers: {
        "azure-speech": {
          apiKey: "${AZURE_SPEECH_KEY}",
          region: "eastus",
          voice: "en-US-JennyNeural",
          lang: "en-US",
          outputFormat: "audio-24khz-48kbitrate-mono-mp3",
          voiceNoteOutputFormat: "ogg-24khz-16bit-mono-opus",
        },
      },
    },
  },
}
```

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
      providers: {
        elevenlabs: {
          apiKey: "${ELEVENLABS_API_KEY}",

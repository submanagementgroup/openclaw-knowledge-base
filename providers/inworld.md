---
domain: providers
topic: "InWorld Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - InWorld
  - inworld provider
  - AI provider
  - model config
  - API key
  - InWorld
  - game AI
  - character AI
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/inworld.md
---

InWorld provider setup, configuration, and model reference for OpenClaw.

Inworld is a streaming text-to-speech (TTS) provider. In OpenClaw it
synthesizes outbound reply audio (MP3 by default, OGG_OPUS for voice notes)
and PCM audio for telephony channels such as Voice Call.

OpenClaw posts to Inworld's streaming TTS endpoint, concatenates the
returned base64 audio chunks into a single buffer, and hands the result to
the standard reply-audio pipeline.

| Detail        | Value                                                       |
| ------------- | ----------------------------------------------------------- |
| Website       | [inworld.ai](https://inworld.ai)                            |
| Docs          | [docs.inworld.ai/tts/tts](https://docs.inworld.ai/tts/tts)  |
| Auth          | `INWORLD_API_KEY` (HTTP Basic, Base64 dashboard credential) |
| Default voice | `Sarah`                                                     |
| Default model | `inworld-tts-1.5-max`                                       |

## Getting started

    Copy the credential from your Inworld dashboard (Workspace > API Keys)
    and set it as an env var. The value is sent verbatim as the HTTP Basic
    credential, so do not Base64-encode it again or convert it to a bearer
    token.

    ```
    INWORLD_API_KEY=<base64-credential-from-dashboard>
    ```

    ```json5
    {
      messages: {
        tts: {
          auto: "always",
          provider: "inworld",
          providers: {
            inworld: {
              voiceId: "Sarah",
              modelId: "inworld-tts-1.5-max",
            },
          },
        },
      },
    }
    ```

    Send a reply through any connected channel. OpenClaw synthesizes the
    audio with Inworld and delivers it as MP3 (or OGG_OPUS when the channel
    expects a voice note).

## Configuration options

| Option        | Path                                         | Description                                                       |
| ------------- | -------------------------------------------- | ----------------------------------------------------------------- |
| `apiKey`      | `messages.tts.providers.inworld.apiKey`      | Base64 dashboard credential. Falls back to `INWORLD_API_KEY`.     |
| `baseUrl`     | `messages.tts.providers.inworld.baseUrl`     | Override Inworld API base URL (default `https://api.inworld.ai`). |
| `voiceId`     | `messages.tts.providers.inworld.voiceId`     | Voice identifier (default `Sarah`).                               |
| `modelId`     | `messages.tts.providers.inworld.modelId`     | TTS model id (default `inworld-tts-1.5-max`).                     |
| `temperature` | `messages.tts.providers.inworld.temperature` | Sampling temperature `0..2` (optional).                           |

## Notes

    Inworld uses HTTP Basic auth with a single Base64-encoded credential
    string. Copy it verbatim from the Inworld dashboard. The provider sends
    it as `Authorization: Basic <apiKey>` without any further encoding, so
    do not Base64-encode it yourself and do not pass a bearer-style token.
    See [TTS auth notes](/tools/tts#inworld-primary) for the same callout.

    Supported model ids: `inworld-tts-1.5-max` (default),
    `inworld-tts-1.5-mini`, `inworld-tts-1-max`, `inworld-tts-1`.

    Replies use MP3 by default. When the channel target is `voice-note`
    OpenClaw asks Inworld for `OGG_OPUS` so the audio plays as a native
    voice bubble. Telephony synthesis uses raw `PCM` at 22050 Hz to feed
    the telephony bridge.

    Override the API host with `messages.tts.providers.inworld.baseUrl`.
    Trailing slashes are stripped before requests are sent.

## Related

    TTS overview, providers, and `messages.tts` config.

    Full config reference including `messages.tts` settings.

    All bundled OpenClaw providers.

    Common issues and debugging steps.

---
domain: providers
topic: "Gradium Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Gradium
  - gradium provider
  - AI provider
  - model config
  - API key
  - Gradium
  - provider
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/gradium.md
---

Gradium provider setup, configuration, and model reference for OpenClaw.

Gradium is a bundled text-to-speech provider for OpenClaw. It can generate normal audio replies, voice-note-compatible Opus output, and 8 kHz u-law audio for telephony surfaces.

## Setup

Create a Gradium API key, then expose it to OpenClaw:

```bash
export GRADIUM_API_KEY="gsk_..."
```

You can also store the key in config under `messages.tts.providers.gradium.apiKey`.

## Config

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "gradium",
      providers: {
        gradium: {
          voiceId: "YTpq7expH9539ERJ",
          // apiKey: "${GRADIUM_API_KEY}",
          // baseUrl: "https://api.gradium.ai",
        },
      },
    },
  },
}
```

## Voices

| Name      | Voice ID           |
| --------- | ------------------ |
| Emma      | `YTpq7expH9539ERJ` |
| Kent      | `LFZvm12tW_z0xfGo` |
| Tiffany   | `Eu9iL_CYe8N-Gkx_` |
| Christina | `2H4HY2CBNyJHBCrP` |
| Sydney    | `jtEKaLYNn6iif5PR` |
| John      | `KWJiFWu2O9nMPYcR` |
| Arthur    | `3jUdJyOi9pgbxBTK` |

Default voice: Emma.

## Output

- Audio-file replies use WAV.
- Voice-note replies use Opus and are marked voice-compatible.
- Telephony synthesis uses `ulaw_8000` at 8 kHz.

## Related

- [Text-to-Speech](/tools/tts)
- [Media Overview](/tools/media-overview)

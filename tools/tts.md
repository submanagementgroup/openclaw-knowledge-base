---
domain: tools
topic: "Text-to-Speech (TTS) Tool"
type: procedure
keywords:
  - TTS
  - text-to-speech
  - voice synthesis
  - speech output
  - tts config
  - tts provider
source: tools/tts.md
---

OpenClaw can convert outbound replies into audio across **14 speech providers**
and deliver native voice messages on Feishu, Matrix, Telegram, and WhatsApp,
audio attachments everywhere else, and PCM/Ulaw streams for telephony and Talk.

TTS is the speech-output half of Talk's `stt-tts` mode. Provider-native
`realtime` Talk sessions synthesize speech inside the realtime provider instead
of calling this TTS path, while `transcription` sessions do not synthesize an
assistant voice response.

## Quick start

**Pick a provider**

OpenAI and ElevenLabs are the most reliable hosted options. Microsoft and
    Local CLI work without an API key. See the [provider matrix](#supported-providers)
    for the full list.
  
**Set the API key**

Export the env var for your provider (for example `OPENAI_API_KEY`,
    `ELEVENLABS_API_KEY`). Microsoft and Local CLI need no key.
  
**Enable in config**

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

  
**Try it in chat**

`/tts status` shows the current state. `/tts audio Hello from OpenClaw`
    sends a one-off audio reply.
  
> **Note:** Auto-TTS is **off** by default. When `messages.tts.provider` is unset,
OpenClaw picks the first configured provider in registry auto-select order.
The built-in `tts` agent tool is explicit-intent only: ordinary chat stays
text unless the user asks for audio, uses `/tts`, or enables Auto-TTS/directive
speech.


## Supported providers

| Provider          | Auth                                                                                                             | Notes                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Azure Speech**  | `AZURE_SPEECH_KEY` + `AZURE_SPEECH_REGION` (also `AZURE_SPEECH_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`)          | Native Ogg/Opus voice-note output and telephony.                                            |
| **DeepInfra**     | `DEEPINFRA_API_KEY`                                                                                              | OpenAI-compatible TTS. Defaults to `hexgrad/Kokoro-82M`.                                    |
| **ElevenLabs**    | `ELEVENLABS_API_KEY` or `XI_API_KEY`                                                                             | Voice cloning, multilingual, deterministic via `seed`; streamed for Discord voice playback. |
| **Google Gemini** | `GEMINI_API_KEY` or `GOOGLE_API_KEY`                                                                             | Gemini API batch TTS; persona-aware via `promptTemplate: "audio-profile-v1"`.               |
| **Gradium**       | `GRADIUM_API_KEY`                                                                                                | Voice-note and telephony output.                                                            |
| **Inworld**       | `INWORLD_API_KEY`                                                                                                | Streaming TTS API. Native Opus voice-note and PCM telephony.                                |
| **Local CLI**     | none                                                                                                             | Runs a configured local TTS command.                                                        |
| **Microsoft**     | none                                                                                                             | Public Edge neural TTS via `node-edge-tts`. Best-effort, no SLA.                            |
| **MiniMax**       | `MINIMAX_API_KEY` (or Token Plan: `MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`)      | T2A v2 API. Defaults to `speech-2.8-hd`.                                                    |
| **OpenAI**        | `OPENAI_API_KEY`                                                                                                 | Also used for auto-summary; supports persona `instructions`.                                |
| **OpenRouter**    | `OPENROUTER_API_KEY` (can reuse `models.providers.openrouter.apiKey`)                                            | Default model `hexgrad/kokoro-82m`.                                                         |
| **Volcengine**    | `VOLCENGINE_TTS_API_KEY` or `BYTEPLUS_SEED_SPEECH_API_KEY` (legacy AppID/token: `VOLCENGINE_TTS_APPID`/`_TOKEN`) | BytePlus Seed Speech HTTP API.                                                              |
| **Vydra**         | `VYDRA_API_KEY`                                                                                                  | Shared image, video, and speech provider.                                                   |
| **xAI**           | `XAI_API_KEY`                                                                                                    | xAI batch TTS. Native Opus voice-note is **not** supported.                                 |
| **Xiaomi MiMo**   | `XIAOMI_API_KEY`                                                                                                 | MiMo TTS through Xiaomi chat completions.                                                   |

If multiple providers are configured, the selected one is used first and the
others are fallback options. Auto-summary uses `summaryModel` (or
`agents.defaults.model.primary`), so that provider must also be authenticated
if you keep summaries enabled.

> **Note:** The bundled **Microsoft** provider uses Microsoft Edge's online neural TTS
service via `node-edge-tts`. It is a public web service without a published
SLA or quota — treat it as best-effort. The legacy provider id `edge` is
normalized to `microsoft` and `openclaw doctor --fix` rewrites persisted
config; new configs should always use `microsoft`.


## Configuration

TTS config lives under `messages.tts` in `~/.openclaw/openclaw.json`. Pick a
preset and adapt the provider block:

**Azure Speech:**

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
  
**ElevenLabs:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
      providers: {
        elevenlabs: {
          apiKey: "${ELEVENLABS_API_KEY}",
          model: "eleven_multilingual_v2",
          voiceId: "EXAVITQu4vr4xnSDxMaL",
        },
      },
    },
  },
}
```
  
**Google Gemini:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "google",
      providers: {
        google: {
          apiKey: "${GEMINI_API_KEY}",
          model: "gemini-3.1-flash-tts-preview",
          voiceName: "Kore",
          // Optional natural-language style prompts:
          // audioProfile: "Speak in a calm, podcast-host tone.",
          // speakerName: "Alex",
        },
      },
    },
  },
}
```
  
**Gradium:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "gradium",
      providers: {
        gradium: {
          apiKey: "${GRADIUM_API_KEY}",
          voiceId: "YTpq7expH9539ERJ",
        },
      },
    },
  },
}
```
  
**Inworld:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "inworld",
      providers: {
        inworld: {
          apiKey: "${INWORLD_API_KEY}",
          modelId: "inworld-tts-1.5-max",
          voiceId: "Sarah",
          temperature: 0.7,
        },
      },
    },
  },
}
```
  
**Local CLI:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "tts-local-cli",
      providers: {
        "tts-local-cli": {
          command: "say",
          args: ["-o", "{{OutputPath}}", "{{Text}}"],
          outputFormat: "wav",
          timeoutMs: 120000,
        },
      },
    },
  },
}
```
  
**Microsoft (no key):**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "microsoft",
      providers: {
        microsoft: {
          enabled: true,
          voice: "en-US-MichelleNeural",
          lang: "en-US",
          outputFormat: "audio-24khz-48kbitrate-mono-mp3",
          rate: "+0%",
          pitch: "+0%",
        },
      },
    },
  },
}
```
  
**MiniMax:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "minimax",
      providers: {
        minimax: {
          apiKey: "${MINIMAX_API_KEY}",
          model: "speech-2.8-hd",
          voiceId: "English_expressive_narrator",
          speed: 1.0,
          vol: 1.0,
          pitch: 0,
        },
      },
    },
  },
}
```
  
**OpenAI + ElevenLabs:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openai",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: { enabled: true },
      providers: {
        openai: {
          apiKey: "${OPENAI_API_KEY}",
          model: "gpt-4o-mini-tts",
          voice: "alloy",
        },
        elevenlabs: {
          apiKey: "${ELEVENLABS_API_KEY}",
          model: "eleven_multilingual_v2",
          voiceId: "EXAVITQu4vr4xnSDxMaL",
          voiceSettings: { stability: 0.5, similarityBoost: 0.75, style: 0.0, useSpeakerBoost: true, speed: 1.0 },
          applyTextNormalization: "auto",
          languageCode: "en",
        },
      },
    },
  },
}
```
  
**OpenRouter:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openrouter",
      providers: {
        openrouter: {
          apiKey: "${OPENROUTER_API_KEY}",
          model: "hexgrad/kokoro-82m",
          voice: "af_alloy",
          responseFormat: "mp3",
        },
      },
    },
  },
}
```
  
**Volcengine:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "volcengine",
      providers: {
        volcengine: {
          apiKey: "${VOLCENGINE_TTS_API_KEY}",
          resourceId: "seed-tts-1.0",
          voice: "en_female_anna_mars_bigtts",
        },
      },
    },
  },
}
```
  
**xAI:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "xai",
      providers: {
        xai: {
          apiKey: "${XAI_API_KEY}",
          voiceId: "eve",
          language: "en",
          responseFormat: "mp3",
        },
      },
    },
  },
}
```
  
**Xiaomi MiMo:**

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "xiaomi",
      providers: {
        xiaomi: {
          apiKey: "${XIAOMI_API_KEY}",
          model: "mimo-v2.5-tts",
          voice: "mimo_default",
          format: "mp3",
        },
      },
    },
  },
}
```
  
### Per-agent voice overrides

Use `agents.list[].tts` when one agent should speak with a different provider,
voice, model, persona, or auto-TTS mode. The agent block deep-merges over
`messages.tts`, so provider credentials can stay in the global provider config:

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
      providers: {
        elevenlabs: { apiKey: "${ELEVENLABS_API_KEY}", model: "eleven_multilingual_v2" },
      },
    },
  },
  agents: {
    list: [
      {
        id: "reader",
        tts: {
          providers: {
            elevenlabs: { voiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
      },
    ],
  },
}
```

To pin a per-agent persona, set `agents.list[].tts.persona` alongside provider
config — it overrides the global `messages.tts.persona` for that agent only.

Precedence order for automatic replies, `/tts audio`, `/tts status`, and the
`tts` agent tool:

1. `messages.tts`
2. active `agents.list[].tts`
3. channel override, when the channel supports `channels.<channel>.tts`
4. account override, when the channel passes `channels.<channel>.accounts.<id>.tts`
5. local `/tts` preferences for this host
6. inline `[[tts:...]]` directives when [model overrides](#model-driven-directives) are enabled

Channel and account overrides use the same shape as `messages.tts` and
deep-merge over the earlier layers, so shared provider credentials can stay in
`messages.tts` while a channel or bot account changes only voice, model, persona,
or auto mode:

```json5
{
  messages: {
    tts: {
      provider: "openai",
      providers: {
        openai: { apiKey: "${OPENAI_API_KEY}", model: "gpt-4o-mini-tts" },
      },
    },
  },
  channels: {
    feishu: {
      accounts: {
        english: {
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

## Personas

A **persona** is a stable spoken identity that can be applied deterministically
across providers. It can prefer one provider, define provider-neutral prompt
intent, and carry provider-specific bindings for voices, models, prompt
templates, seeds, and voice settings.

### Minimal persona

```json5
{
  messages: {
    tts: {
      auto: "always",
      persona: "narrator",
      personas: {
        narrator: {
          label: "Narrator",
          provider: "elevenlabs",
          providers: {
            elevenlabs: { voiceId: "EXAVITQu4vr4xnSDxMaL", modelId: "eleven_multilingual_v2" },
          },
        },
      },
    },
  },
}
```

### Full persona (provider-neutral prompt)

```json5
{
  messages: {
    tts: {
      auto: "always",
      persona: "alfred",
      personas: {
        alfred: {
          label: "Alfred",
          description: "Dry, warm British butler narrator.",
          provider: "google",
          fallbackPolic
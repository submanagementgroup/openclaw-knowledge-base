---
domain: tools
topic: "TTS Advanced Configuration and Providers"
type: reference
keywords:
  - TTS
  - TTS providers
  - TTS config advanced
  - ElevenLabs TTS
  - Azure speech
source: tools/tts.md
---

y: "preserve-persona",
          prompt: {
            profile: "A brilliant British butler. Dry, witty, warm, charming, emotionally expressive, never generic.",
            scene: "A quiet late-night study. Close-mic narration for a trusted operator.",
            sampleContext: "The speaker is answering a private technical request with concise confidence and dry warmth.",
            style: "Refined, understated, lightly amused.",
            accent: "British English.",
            pacing: "Measured, with short dramatic pauses.",
            constraints: ["Do not read configuration values aloud.", "Do not explain the persona."],
          },
          providers: {
            google: {
              model: "gemini-3.1-flash-tts-preview",
              voiceName: "Algieba",
              promptTemplate: "audio-profile-v1",
            },
            openai: { model: "gpt-4o-mini-tts", voice: "cedar" },
            elevenlabs: {
              voiceId: "voice_id",
              modelId: "eleven_multilingual_v2",
              seed: 42,
              voiceSettings: {
                stability: 0.65,
                similarityBoost: 0.8,
                style: 0.25,
                useSpeakerBoost: true,
                speed: 0.95,
              },
            },
          },
        },
      },
    },
  },
}
```

### Persona resolution

The active persona is selected deterministically:

1. `/tts persona <id>` local preference, if set.
2. `messages.tts.persona`, if set.
3. No persona.

Provider selection runs explicit-first:

1. Direct overrides (CLI, gateway, Talk, allowed TTS directives).
2. `/tts provider <id>` local preference.
3. Active persona's `provider`.
4. `messages.tts.provider`.
5. Registry auto-select.

For each provider attempt, OpenClaw merges configs in this order:

1. `messages.tts.providers.<id>`
2. `messages.tts.personas.<persona>.providers.<id>`
3. Trusted request overrides
4. Allowed model-emitted TTS directive overrides

### How providers use persona prompts

Persona prompt fields (`profile`, `scene`, `sampleContext`, `style`, `accent`,
`pacing`, `constraints`) are **provider-neutral**. Each provider decides how
to use them:

### Google Gemini

Wraps persona prompt fields in a Gemini TTS prompt structure **only when**
    the effective Google provider config sets `promptTemplate: "audio-profile-v1"`
    or `personaPrompt`. The older `audioProfile` and `speakerName` fields are
    still prepended as Google-specific prompt text. Inline audio tags such as
    `[whispers]` or `[laughs]` inside a `[[tts:text]]` block are preserved
    inside the Gemini transcript; OpenClaw does not generate these tags.
  ### OpenAI

Maps persona prompt fields to the request `instructions` field **only when**
    no explicit OpenAI `instructions` is configured. Explicit `instructions`
    always wins.
  ### Other providers

Use only the provider-specific persona bindings under
    `personas.<id>.providers.<provider>`. Persona prompt fields are ignored
    unless the provider implements its own persona-prompt mapping.
  ### Fallback policy

`fallbackPolicy` controls behavior when a persona has **no binding** for the
attempted provider:

| Policy              | Behavior                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `preserve-persona`  | **Default.** Provider-neutral prompt fields stay available; the provider may use them or ignore them.                                            |
| `provider-defaults` | Persona is omitted from prompt preparation for that attempt; the provider uses its neutral defaults while fallback to other providers continues. |
| `fail`              | Skip that provider attempt with `reasonCode: "not_configured"` and `personaBinding: "missing"`. Fallback providers are still tried.              |

The whole TTS request only fails when **every** attempted provider is skipped
or fails.

Talk session provider selection is session-scoped. A Talk client should choose
provider ids, model ids, voice ids, and locales from `talk.catalog` and pass
them through the Talk session or handoff request. Opening a voice session should
not mutate `messages.tts` or global Talk provider defaults.

## Model-driven directives

By default, the assistant **can** emit `[[tts:...]]` directives to override
voice, model, or speed for a single reply, plus an optional
`[[tts:text]]...[[/tts:text]]` block for expressive cues that should appear in
audio only:

```text
Here you go.

[[tts:voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](laughs) Read the song once more.[[/tts:text]]
```

When `messages.tts.auto` is `"tagged"`, **directives are required** to trigger
audio. Streaming block delivery strips directives from visible text before the
channel sees them, even when split across adjacent blocks.

`provider=...` is ignored unless `modelOverrides.allowProvider: true`. When a
reply declares `provider=...`, the other keys in that directive are parsed
only by that provider; unsupported keys are stripped and reported as TTS
directive warnings.

**Available directive keys:**

- `provider` (registered provider id; requires `allowProvider: true`)
- `voice` / `voiceName` / `voice_name` / `google_voice` / `voiceId`
- `model` / `google_model`
- `stability`, `similarityBoost`, `style`, `speed`, `useSpeakerBoost`
- `vol` / `volume` (MiniMax volume, 0–10)
- `pitch` (MiniMax integer pitch, −12 to 12; fractional values are truncated)
- `emotion` (Volcengine emotion tag)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

**Disable model overrides entirely:**

```json5
{ messages: { tts: { modelOverrides: { enabled: false } } } }
```

**Allow provider switching while keeping other knobs configurable:**

```json5
{ messages: { tts: { modelOverrides: { enabled: true, allowProvider: true, allowSeed: false } } } }
```

## Slash commands

Single command `/tts`. On Discord, OpenClaw also registers `/voice` because
`/tts` is a built-in Discord command — text `/tts ...` still works.

```text
/tts off | on | status
/tts chat on | off | default
/tts latest
/tts provider <id>
/tts persona <id> | off
/tts limit <chars>
/tts summary off
/tts audio <text>
```

> **Note:** Commands require an authorized sender (allowlist/owner rules apply) and either
`commands.text` or native command registration must be enabled.


Behavior notes:

- `/tts on` writes the local TTS preference to `always`; `/tts off` writes it to `off`.
- `/tts chat on|off|default` writes a session-scoped auto-TTS override for the current chat.
- `/tts persona <id>` writes the local persona preference; `/tts persona off` clears it.
- `/tts latest` reads the latest assistant reply from the current session transcript and sends it as audio once. It stores only a hash of that reply on the session entry to suppress duplicate voice sends.
- `/tts audio` generates a one-off audio reply (does **not** toggle TTS on).
- `limit` and `summary` are stored in **local prefs**, not the main config.
- `/tts status` includes fallback diagnostics for the latest attempt — `Fallback: <primary> -> <used>`, `Attempts: ...`, and per-attempt detail (`provider:outcome(reasonCode) latency`).
- `/status` shows the active TTS mode plus configured provider, model, voice, and sanitized custom endpoint metadata when TTS is enabled.

## Per-user preferences

Slash commands write local overrides to `prefsPath`. The default is
`~/.openclaw/settings/tts.json`; override with the `OPENCLAW_TTS_PREFS` env var
or `messages.tts.prefsPath`.

| Stored field | Effect                                       |
| ------------ | -------------------------------------------- |
| `auto`       | Local auto-TTS override (`always`, `off`, …) |
| `provider`   | Local primary provider override              |
| `persona`    | Local persona override                       |
| `maxLength`  | Summary threshold (default `1500` chars)     |
| `summarize`  | Summary toggle (default `true`)              |

These override the effective config from `messages.tts` plus the active
`agents.list[].tts` block for that host.

## Output formats (fixed)

TTS voice delivery is channel-capability driven. Channel plugins advertise
whether voice-style TTS should ask providers for a native `voice-note` target or
keep normal `audio-file` synthesis and only mark compatible output for voice
delivery.

- **Voice-note capable channels**: voice-note replies prefer Opus (`opus_48000_64` from ElevenLabs, `opus` from OpenAI).
  - 48kHz / 64kbps is a good voice message tradeoff.
- **Feishu / WhatsApp**: when a voice-note reply is produced as MP3/WebM/WAV/M4A
  or another likely audio file, the channel plugin transcodes it to 48kHz
  Ogg/Opus with `ffmpeg` before sending the native voice message. WhatsApp sends
  the result through the Baileys `audio` payload with `ptt: true` and
  `audio/ogg; codecs=opus`. If conversion fails, Feishu receives the original
  file as an attachment; WhatsApp send fails rather than posting an incompatible
  PTT payload.
- **Other channels**: MP3 (`mp3_44100_128` from ElevenLabs, `mp3` from OpenAI).
  - 44.1kHz / 128kbps is the default balance for speech clarity.
- **MiniMax**: MP3 (`speech-2.8-hd` model, 32kHz sample rate) for normal audio attachments. For channel-advertised voice-note targets, OpenClaw transcodes the MiniMax MP3 to 48kHz Opus with `ffmpeg` before delivery when the channel advertises transcoding.
- **Xiaomi MiMo**: MP3 by default, or WAV when configured. For channel-advertised voice-note targets, OpenClaw transcodes Xiaomi output to 48kHz Opus with `ffmpeg` before delivery when the channel advertises transcoding.
- **Local CLI**: uses the configured `outputFormat`. Voice-note targets are
  converted to Ogg/Opus and telephony output is converted to raw 16 kHz mono PCM
  with `ffmpeg`.
- **Google Gemini**: Gemini API TTS returns raw 24kHz PCM. OpenClaw wraps it as WAV for audio attachments, transcodes it to 48kHz Opus for voice-note targets, and returns PCM directly for Talk/telephony.
- **Gradium**: WAV for audio attachments, Opus for voice-note targets, and `ulaw_8000` at 8 kHz for telephony.
- **Inworld**: MP3 for normal audio attachments, native `OGG_OPUS` for voice-note targets, and raw `PCM` at 22050 Hz for Talk/telephony.
- **xAI**: MP3 by default; `responseFormat` may be `mp3`, `wav`, `pcm`, `mulaw`, or `alaw`. OpenClaw uses xAI's batch REST TTS endpoint and returns a complete audio attachment; xAI's streaming TTS WebSocket is not used by this provider path. Native Opus voice-note format is not supported by this path.
- **Microsoft**: uses `microsoft.outputFormat` (default `audio-24khz-48kbitrate-mono-mp3`).
  - The bundled transport accepts an `outputFormat`, but not all formats are available from the service.
  - Output format values follow Microsoft Speech output formats (including Ogg/WebM Opus).
  - Telegram `sendVoice` accepts OGG/MP3/M4A; use OpenAI/ElevenLabs if you need
    guaranteed Opus voice messages.
  - If the configured Microsoft output format fails, OpenClaw retries with MP3.

OpenAI/ElevenLabs output formats are fixed per channel (see above).

## Auto-TTS behavior

When `messages.tts.auto` is enabled, OpenClaw:

- Skips TTS if the reply already contains media or a `MEDIA:` directive.
- Skips very short replies (under 10 chars).
- Summarizes long replies when summaries are enabled, using
  `summaryModel` (or `agents.defaults.model.primary`).
- Attaches the generated audio to the reply.
- In `mode: "final"`, still sends audio-only TTS for streamed final replies
  after the text stream completes; the generated media goes through the same
  channel media normalization as normal reply attachments.

If the reply exceeds `maxLength` and summary is off (or no API key for the
summary model), audio is skipped and the normal text reply is sent.

```text
Reply -> TTS enabled?
  no  -> send text
  yes -> has media / MEDIA: / short?
          yes -> send text
          no  -> length > limit?
                   no  -> TTS -> attach audio
                   yes -> summary enabled?
                            no  -> send text
                            yes -> summarize -> TTS -> attach audio
```

## Output formats by channel

| Target                                | Format                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Feishu / Matrix / Telegram / WhatsApp | Voice-note replies prefer **Opus** (`opus_48000_64` from ElevenLabs, `opus` from OpenAI). 48 kHz / 64 kbps balances clarity and size. |
| Other channels                        | **MP3** (`mp3_44100_128` from ElevenLabs, `mp3` from OpenAI). 44.1 kHz / 128 kbps default for speech.                                 |
| Talk / telephony                      | Provider-native **PCM** (Inworld 22050 Hz, Google 24 kHz), or `ulaw_8000` from Gradium for telephony.                                 |

Per-provider notes:

- **Feishu / WhatsApp transcoding:** When a voice-note reply lands as MP3/WebM/WAV/M4A, the channel plugin transcodes to 48 kHz Ogg/Opus with `ffmpeg`. WhatsApp sends through Baileys with `ptt: true` and `audio/ogg; codecs=opus`. If conversion fails: Feishu falls back to attaching the original file; WhatsApp send fails rather than posting an incompatible PTT payload.
- **MiniMax / Xiaomi MiMo:** Default MP3 (32 kHz for MiniMax `speech-2.8-hd`); transcoded to 48 kHz Opus for voice-note targets via `ffmpeg`.
- **Local CLI:** Uses configured `outputFormat`. Voice-note targets are converted to Ogg/Opus and telephony output to r
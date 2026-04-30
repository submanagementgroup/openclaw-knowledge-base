---
domain: plugins
topic: "Voice Call Plugin: Real-Time Voice Conversations with OpenClaw Agents"
type: procedure
keywords:
  - voice call
  - voice conversation
  - real-time voice
  - voice plugin
  - phone call
related:
  - tools/tts
  - nodes/audio
source: plugins/voice-call.md
---

Voice call plugin for OpenClaw: enables real-time voice conversations with the agent.

Voice calls for OpenClaw via a plugin. Supports outbound notifications,
multi-turn conversations, full-duplex realtime voice, streaming
transcription, and inbound calls with allowlist policies.

**Current providers:** `twilio` (Programmable Voice + Media Streams),
`telnyx` (Call Control v2), `plivo` (Voice API + XML transfer + GetInput
speech), `mock` (dev/no network).

The Voice Call plugin runs **inside the Gateway process**. If you use a
remote Gateway, install and configure the plugin on the machine running
the Gateway, then restart the Gateway to load it.

## Quick start

        ```bash
        openclaw plugins install @openclaw/voice-call
        ```

        ```bash
        PLUGIN_SRC=./path/to/local/voice-call-plugin
        openclaw plugins install "$PLUGIN_SRC"
        cd "$PLUGIN_SRC" && pnpm install
        ```

    If npm reports the OpenClaw-owned package as deprecated, that package version
    is from an older external package train; use a current packaged OpenClaw
    build or the local folder path until a newer npm package is published.

    Restart the Gateway afterwards so the plugin loads.

    Set config under `plugins.entries.voice-call.config` (see
    [Configuration](#configuration) below for the full shape). At minimum:
    `provider`, provider credentials, `fromNumber`, and a publicly
    reachable webhook URL.

    ```bash
    openclaw voicecall setup
    ```

    The default output is readable in chat logs and terminals. It checks
    plugin enablement, provider credentials, webhook exposure, and that
    only one audio mode (`streaming` or `realtime`) is active. Use
    `--json` for scripts.

    ```bash
    openclaw voicecall smoke
    openclaw voicecall smoke --to "+15555550123"
    ```

    Both are dry runs by default. Add `--yes` to actually place a short
    outbound notify call:

    ```bash
    openclaw voicecall smoke --to "+15555550123" --yes
    ```

For Twilio, Telnyx, and Plivo, setup must resolve to a **public webhook URL**.
If `publicUrl`, the tunnel URL, the Tailscale URL, or the serve fallback
resolves to loopback or private network space, setup fails instead of
starting a provider that cannot receive carrier webhooks.

## Configuration

If `enabled: true` but the selected provider is missing credentials,
Gateway startup logs a setup-incomplete warning with the missing keys and
skips starting the runtime. Commands, RPC calls, and agent tools still
return the exact missing provider configuration when used.

Voice-call credentials accept SecretRefs. `plugins.entries.voice-call.config.twilio.authToken` and `plugins.entries.voice-call.config.tts.providers.*.apiKey` resolve through the standard SecretRef surface; see [SecretRef credential surface](/reference/secretref-credential-surface).

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio", // or "telnyx" | "plivo" | "mock"
          fromNumber: "+15550001234", // or TWILIO_FROM_NUMBER for Twilio
          toNumber: "+15550005678",

          twilio: {
            accountSid: "ACxxxxxxxx",
            authToken: "...",
          },
          telnyx: {
            apiKey: "...",
            connectionId: "...",
            // Telnyx webhook public key from the Mission Control Portal
            // (Base64; can also be set via TELNYX_PUBLIC_KEY).
            publicKey: "...",
          },
          plivo: {
            authId: "MAxxxxxxxxxxxxxxxxxxxx",
            authToken: "...",
          },

          // Webhook server
          serve: {
            port: 3334,
            path: "/voice/webhook",
          },

          // Webhook security (recommended for tunnels/proxies)
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
            trustedProxyIPs: ["100.64.0.1"],
          },

          // Public exposure (pick one)
          // publicUrl: "https://example.ngrok.app/voice/webhook",
          // tunnel: { provider: "ngrok" },
          // tailscale: { mode: "funnel", path: "/voice/webhook" },

          outbound: {
            defaultMode: "notify", // notify | conversation
          },

          streaming: { enabled: true /* see Streaming transcription */ },
          realtime: { enabled: false /* see Realtime voice */ },
        },
      },
    },
  },
}
```

    - Twilio, Telnyx, and Plivo all require a **publicly reachable** webhook URL.
    - `mock` is a local dev provider (no network calls).
    - Telnyx requires `telnyx.publicKey` (or `TELNYX_PUBLIC_KEY`) unless `skipSignatureVerification` is true.
    - `skipSignatureVerification` is for local testing only.
    - On ngrok free tier, set `publicUrl` to the exact ngrok URL; signature verification is always enforced.
    - `tunnel.allowNgrokFreeTierLoopbackBypass: true` allows Twilio webhooks with invalid signatures **only** when `tunnel.provider="ngrok"` and `serve.bind` is loopback (ngrok local agent). Local dev only.
    - Ngrok free-tier URLs can change or add interstitial behaviour; if `publicUrl` drifts, Twilio signatures fail. Production: prefer a stable domain or a Tailscale funnel.

    - `streaming.preStartTimeoutMs` closes sockets that never send a valid `start` frame.
    - `streaming.maxPendingConnections` caps total unauthenticated pre-start sockets.
    - `streaming.maxPendingConnectionsPerIp` caps unauthenticated pre-start sockets per source IP.
    - `streaming.maxConnections` caps total open media stream sockets (pending + active).

    Older configs using `provider: "log"`, `twilio.from`, or legacy
    `streaming.*` OpenAI keys are rewritten by `openclaw doctor --fix`.
    Runtime fallback still accepts the old voice-call keys for now, but
    the rewrite path is `openclaw doctor --fix` and the compat shim is
    temporary.

    Auto-migrated streaming keys:

    - `streaming.sttProvider` → `streaming.provider`
    - `streaming.openaiApiKey` → `streaming.providers.openai.apiKey`
    - `streaming.sttModel` → `streaming.providers.openai.model`
    - `streaming.silenceDurationMs` → `streaming.providers.openai.silenceDurationMs`
    - `streaming.vadThreshold` → `streaming.providers.openai.vadThreshold`

## Realtime voice conversations

`realtime` selects a full-duplex realtime voice provider for live call
audio. It is separate from `streaming`, which only forwards audio to
realtime transcription providers.

`realtime.enabled` cannot be combined with `streaming.enabled`. Pick one
audio mode per call.

Current runtime behaviour:

- `realtime.enabled` is supported for Twilio Media Streams.
- `realtime.provider` is optional. If unset, Voice Call uses the first registered realtime voice provider.
- Bundled realtime voice providers: Google Gemini Live (`google`) and OpenAI (`openai`), registered by their provider plugins.
- Provider-owned raw config lives under `realtime.providers.<providerId>`.
- Voice Call exposes the shared `openclaw_agent_consult` realtime tool by default. The realtime model can call it when the caller asks for deeper reasoning, current information, or normal OpenClaw tools.
- If `realtime.provider` points at an unregistered provider, or no realtime voice provider is registered at all, Voice Call logs a warning and skips realtime media instead of failing the whole plugin.
- Consult session keys reuse the existing voice session when available, then fall back to the caller/callee phone number so follow-up consult calls keep context during the call.

### Tool policy

`realtime.toolPolicy` controls the consult run:

| Policy           | Behavior                                                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | Expose the consult tool and limit the regular agent to `read`, `web_search`, `web_fetch`, `x_search`, `memory_search`, and `memory_get`. |
| `owner`          | Expose the consult tool and let the regular agent use the normal agent tool policy.                                                      |
| `none`           | Do not expose the consult tool. Custom `realtime.tools` are still passed through to the realtime provider.                               |

### Realtime provider examples

    Defaults: API key from `realtime.providers.google.apiKey`,
    `GEMINI_API_KEY`, or `GOOGLE_GENERATIVE_AI_API_KEY`; model
    `gemini-2.5-flash-native-audio-preview-12-2025`; voice `Kore`.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              provider: "twilio",
              inboundPolicy: "allowlist",
              allowFrom: ["+15550005678"],
              realtime: {
                enabled: true,
                provider: "google",
                instructions: "Speak briefly. Call openclaw_agent_consult before using deeper tools.",

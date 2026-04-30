---
domain: plugins
topic: "Google Meet Plugin: Joining Calls, Transcription, and Meeting Participation"
type: procedure
keywords:
  - Google Meet
  - video calls
  - meeting plugin
  - Google Meet plugin
  - call transcription
related:
  - plugins/voice-call
  - tools/tts
source: plugins/google-meet.md
---

Google Meet plugin for OpenClaw: join Google Meet calls, transcribe, and participate in video meetings.

Google Meet participant support for OpenClaw — the plugin is explicit by design:

- It only joins an explicit `https://meet.google.com/...` URL.
- It can create a new Meet space through the Google Meet API, then join the
  returned URL.
- `realtime` voice is the default mode.
- Realtime voice can call back into the full OpenClaw agent when deeper
  reasoning or tools are needed.
- Agents choose the join behavior with `mode`: use `realtime` for live
  listen/talk-back, or `transcribe` to join/control the browser without the
  realtime voice bridge.
- Auth starts as personal Google OAuth or an already signed-in Chrome profile.
- There is no automatic consent announcement.
- The default Chrome audio backend is `BlackHole 2ch`.
- Chrome can run locally or on a paired node host.
- Twilio accepts a dial-in number plus optional PIN or DTMF sequence.
- The CLI command is `googlemeet`; `meet` is reserved for broader agent
  teleconference workflows.

## Quick start

Install the local audio dependencies and configure a backend realtime voice
provider. OpenAI is the default; Google Gemini Live also works with
`realtime.provider: "google"`:

```bash
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# or
export GEMINI_API_KEY=...
```

`blackhole-2ch` installs the `BlackHole 2ch` virtual audio device. Homebrew's
installer requires a reboot before macOS exposes the device:

```bash
sudo reboot
```

After reboot, verify both pieces:

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Enable the plugin:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

Check setup:

```bash
openclaw googlemeet setup
```

The setup output is meant to be agent-readable and mode-aware. It reports Chrome
profile, node pinning, and, for realtime Chrome joins, the BlackHole/SoX audio
bridge and delayed realtime intro checks. For observe-only joins, check the same
transport with `--mode transcribe`; that mode skips realtime audio prerequisites
because it does not listen through or speak through the bridge:

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

When Twilio delegation is configured, setup also reports whether the
`voice-call` plugin and Twilio credentials are ready. Treat any `ok: false`
check as a blocker for the checked transport and mode before asking an agent to
join. Use `openclaw googlemeet setup --json` for scripts or machine-readable
output. Use `--transport chrome`, `--transport chrome-node`, or `--transport twilio`
to preflight a specific transport before an agent tries it.

Join a meeting:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

Or let an agent join through the `google_meet` tool:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "realtime"
}
```

Create a new meeting and join it:

```bash
openclaw googlemeet create --transport chrome-node --mode realtime
```

Create only the URL without joining:

```bash
openclaw googlemeet create --no-join
```

`googlemeet create` has two paths:

- API create: used when Google Meet OAuth credentials are configured. This is
  the most deterministic path and does not depend on browser UI state.
- Browser fallback: used when OAuth credentials are absent. OpenClaw uses the
  pinned Chrome node, opens `https://meet.google.com/new`, waits for Google to
  redirect to a real meeting-code URL, then returns that URL. This path requires
  the OpenClaw Chrome profile on the node to already be signed in to Google.
  Browser automation handles Meet's own first-run microphone prompt; that prompt
  is not treated as a Google login failure.
  Join and create flows also try to reuse an existing Meet tab before opening a
  new one. Matching ignores harmless URL query strings such as `authuser`, so an
  agent retry should focus the already-open meeting instead of creating a second
  Chrome tab.

The command/tool output includes a `source` field (`api` or `browser`) so agents
can explain which path was used. `create` joins the new meeting by default and
returns `joined: true` plus the join session. To only mint the URL, use
`create --no-join` on the CLI or pass `"join": false` to the tool.

Or tell an agent: "Create a Google Meet, join it with realtime voice, and send
me the link." The agent should call `google_meet` with `action: "create"` and
then share the returned `meetingUri`.

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "realtime"
}
```

For an observe-only/browser-control join, set `"mode": "transcribe"`. That does
not start the duplex realtime model bridge, does not require BlackHole or SoX,
and will not talk back into the meeting. Chrome joins in this mode also avoid
OpenClaw's microphone/camera permission grant and avoid the Meet **Use
microphone** path. If Meet shows an audio-choice interstitial, automation tries
the no-microphone path and otherwise reports a manual action instead of opening
the local microphone.

During realtime sessions, `google_meet` status includes browser and audio bridge
health such as `inCall`, `manualActionRequired`, `providerConnected`,
`realtimeReady`, `audioInputActive`, `audioOutputActive`, last input/output
timestamps, byte counters, and bridge closed state. If a safe Meet page prompt
appears, browser automation handles it when it can. Login, host admission, and
browser/OS permission prompts are reported as manual action with a reason and
message for the agent to relay.

Local Chrome joins through the signed-in OpenClaw browser profile. Realtime mode
requires `BlackHole 2ch` for the microphone/speaker path used by OpenClaw. For
clean duplex audio, use separate virtual devices or a Loopback-style graph; a
single BlackHole device is enough for a first smoke test but can echo.

### Local gateway + Parallels Chrome

You do **not** need a full OpenClaw Gateway or model API key inside a macOS VM
just to make the VM own Chrome. Run the Gateway and agent locally, then run a
node host in the VM. Enable the bundled plugin on the VM once so the node
advertises the Chrome command:

What runs where:

- Gateway host: OpenClaw Gateway, agent workspace, model/API keys, realtime
  provider, and the Google Meet plugin config.
- Parallels macOS VM: OpenClaw CLI/node host, Google Chrome, SoX, BlackHole 2ch,
  and a Chrome profile signed in to Google.
- Not needed in the VM: Gateway service, agent config, OpenAI/GPT key, or model
  provider setup.

Install the VM dependencies:

```bash
brew install blackhole-2ch sox
```

Reboot the VM after installing BlackHole so macOS exposes `BlackHole 2ch`:

```bash
sudo reboot
```

After reboot, verify the VM can see the audio device and SoX commands:

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Install or update OpenClaw in the VM, then enable the bundled plugin there:

```bash
openclaw plugins enable google-meet
```

Start the node host in the VM:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

If `<gateway-host>` is a LAN IP and you are not using TLS, the node refuses the
plaintext WebSocket unless you opt in for that trusted private network:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Use the same environment variable when installing the node as a LaunchAgent:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

`OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` is process environment, not an
`openclaw.json` setting. `openclaw node install` stores it in the LaunchAgent
environment when it is present on the install command.

Approve the node from the Gateway host:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Confirm the Gateway sees the node and that it advertises both `googlemeet.chrome`
and browser capability/`browser.proxy`:

```bash
openclaw nodes status
```

Route Meet through that node on the Gateway host:

```json5
{
  gateway: {
    nodes: {
      allowCommands: ["googlemeet.chrome", "browser.proxy"],
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

Now join normally from the Gateway host:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

or ask the agent to use the `google_meet` tool with `transport: "chrome-node"`.

For a one-command smoke test that creates or reuses a session, speaks a known
phra

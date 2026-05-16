---
domain: plugins
topic: "Google Meet Plugin"
type: integration
keywords:
  - Google Meet
  - google meet plugin
  - video call
  - meeting recording
  - meet transcription
source: plugins/google-meet.md
---

Google Meet participant support for OpenClaw — the plugin is explicit by design:

- It only joins an explicit `https://meet.google.com/...` URL.
- It can create a new Meet space through the Google Meet API, then join the
  returned URL.
- `agent` is the default talk-back mode: realtime transcription listens, the
  configured OpenClaw agent answers, and regular OpenClaw TTS speaks into Meet.
- `bidi` remains available as the fallback direct realtime voice model mode.
- Agents choose the join behavior with `mode`: use `agent` for live
  listen/talk-back, `bidi` for direct realtime voice fallback, or `transcribe`
  to join/control the browser without the talk-back bridge.
- Auth starts as personal Google OAuth or an already signed-in Chrome profile.
- There is no automatic consent announcement.
- The default Chrome audio backend is `BlackHole 2ch`.
- Chrome can run locally or on a paired node host.
- Twilio accepts a dial-in number plus optional PIN or DTMF sequence; it
  cannot dial a Meet URL directly.
- The CLI command is `googlemeet`; `meet` is reserved for broader agent
  teleconference workflows.

## Quick start

Install the local audio dependencies and configure a realtime transcription
provider plus regular OpenClaw TTS. OpenAI is the default transcription
provider; Google Gemini Live also works as a separate `bidi` voice fallback with
`realtime.voiceProvider: "google"`:

```bash
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# only needed when realtime.voiceProvider is "google" for bidi mode
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
`voice-call` plugin, Twilio credentials, and public webhook exposure are ready.
Treat any `ok: false` check as a blocker for the checked transport and mode
before asking an agent to join. Use `openclaw googlemeet setup --json` for
scripts or machine-readable output. Use `--transport chrome`,
`--transport chrome-node`, or `--transport twilio` to preflight a specific
transport before an agent tries it.

For Twilio, always preflight the transport explicitly when the default transport
is Chrome:

```bash
openclaw googlemeet setup --transport twilio
```

That catches missing `voice-call` wiring, Twilio credentials, or unreachable
webhook exposure before the agent tries to dial the meeting.

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
  "mode": "agent"
}
```

The agent-facing `google_meet` tool stays available on non-macOS hosts for
artifact, calendar, setup, transcribe, Twilio, and `chrome-node` flows. Local
Chrome talk-back actions are blocked there because the bundled Chrome audio path
currently depends on macOS `BlackHole 2ch`. On Linux, use `mode: "transcribe"`,
Twilio dial-in, or a macOS `chrome-node` host for Chrome talk-back
participation.

Create a new meeting and join it:

```bash
openclaw googlemeet create --transport chrome-node --mode agent
```

For API-created rooms, use Google Meet `SpaceConfig.accessType` when you want
the room's no-knock policy to be explicit instead of inherited from the Google
account defaults:

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

`OPEN` lets anyone with the Meet URL join without knocking. `TRUSTED` lets the
host organization's trusted users, invited external users, and dial-in users
join without knocking. `RESTRICTED` limits no-knock entry to invitees. These
settings only apply to the official Google Meet API creation path, so OAuth
credentials must be configured.

If you authenticated Google Meet before this option was available, rerun
`openclaw googlemeet auth login --json` after adding the
`meetings.space.settings` scope to your Google OAuth consent screen.

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

Or tell an agent: "Create a Google Meet, join it with the agent talk-back mode,
and send me the link." The agent should call `google_meet` with
`action: "create"` and then share the returned `meetingUri`.

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent"
}
```

For an observe-only/browser-control join, set `"mode": "transcribe"`. That does
not start the duplex realtime voice bridge, does not require BlackHole or SoX,
and will not talk back into the meeting. Chrome joins in this mode also avoid
OpenClaw's microphone/camera permission grant and avoid the Meet **Use
microphone** path. If Meet shows an audio-choice interstitial, automation tries
the no-microphone path and otherwise reports a manual action instead of opening
the local microphone. In transcribe mode, managed Chrome transports also install
a best-effort Meet caption observer. `googlemeet status --json` and
`googlemeet doctor` surface `captioning`, `captionsEnabledAttempted`,
`transcriptLines`, `lastCaptionAt`, `lastCaptionSpeaker`, `lastCaptionText`,
and a short `recentTranscript` tail so operators can tell whether the browser
joined the call and whether Meet captions are producing text.
Use `openclaw googlemeet test-listen <meet-url> --transport chrome-node` when
you need a yes/no probe: it joins in transcribe mode, waits for fresh caption or
transcript movement, and returns `listenVerified`, `listenTimedOut`, manual
action fields, and the latest caption health.

During realtime sessions, `google_meet` status includes browser and audio bridge
health such as `inCall`, `manualActionRequired`, `providerConnected`,
`realtimeReady`, `audioInputActive`, `audioOutputActive`, last input/output
timestamps, byte counters, and bridge closed state. If a safe Meet page prompt
appears, browser automation handles it when it can. Login, host admission, and
browser/OS permission prompts are reported as manual action with a reason and
message for the agent to relay. Managed Chrome sessions only emit the intro or
test phrase after browser health reports `inCall: true`; otherwise status reports
`speechReady: false` and the speech attempt is blocked instead of pretending the
agent spoke into the meeting.

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
phrase, and prints session health:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

During realtime join, OpenClaw browser automation fills the guest name, clicks
Join/Ask to join, and accepts Meet's first-run "Use microphone" choice when that
prompt appears. During observe-only join or browser-only meeting creation, it
continues past the same prompt without microphone when that choice is available.
If the browser profile is not signed in, Meet is waiting for host admission,
Chrome needs microphone/camera permission for a realtime join, or Meet is stuck
on a prompt automation could not resolve, the join/test-speech result reports
`manualActionRequired: true` with `manualActionReason` and
`manualActionMessage`. Agents should stop retrying the join, report that exact
message plus the current `browserUrl`/`browserTitle`, and retry only after the
manual browser action is complete.

If `chromeNode.node` is omitted, OpenClaw auto-selects only when exactly one
connected node advertises both `googlemeet.chrome` and browser control. If
several capable nodes are connected, set `chromeNode.node` to the node id,
display name, or remote IP.

Common failure checks:

- `Configured Google Meet node ... is not usable: offline`: the pinned node is
  known to the Gateway but unavailable. Agents should treat that node as
  diagnostic state, not as a usable Chrome host, and report the setup blocker
  instead of falling back to another transport unless the user asked for that.
- `No connected Google Meet-capable node`: start `openclaw node run` in the VM,
  approve pairing, and make sure `openclaw plugins enable google-meet` and
  `openclaw plugins enable browser` were run in the VM. Also confirm the
  Gateway host allows both node commands with
  `gateway.nodes.allowCommands: ["googlemeet.chrome", "browser.proxy"]`.
- `BlackHole 2ch audio device not found`: install `blackhole-2ch` on the host
  being checked and reboot before using local Chrome audio.
- `BlackHole 2ch audio device not found on the node`: install `blackhole-2ch`
  in the VM and reboot the VM.
- Chrome opens but cannot join: sign in to the browser profile inside the VM, or
  keep `chrome.guestName` set for gue
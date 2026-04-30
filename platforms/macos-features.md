---
domain: platforms
topic: "macOS Features: Canvas, Peekaboo Screen Capture, Voice Overlay, and Voice Wake"
type: reference
keywords:
  - Canvas
  - Peekaboo
  - voice overlay
  - voice wake
  - screen capture
  - macOS features
related:
  - platforms/macos-app
  - nodes/audio
source:
  - platforms/mac/canvas.md
  - platforms/mac/peekaboo.md
  - platforms/mac/voice-overlay.md
---

macOS-specific features: Canvas for screen sharing, Peekaboo for screen capture, voice overlay, and voice wake.

## Canvas (Screen Sharing)

The macOS app embeds an agent‑controlled **Canvas panel** using `WKWebView`. It
is a lightweight visual workspace for HTML/CSS/JS, A2UI, and small interactive
UI surfaces.

## Where Canvas lives

Canvas state is stored under Application Support:

- `~/Library/Application Support/OpenClaw/canvas/<session>/...`

The Canvas panel serves those files via a **custom URL scheme**:

- `openclaw-canvas://<session>/<path>`

Examples:

- `openclaw-canvas://main/` → `<canvasRoot>/main/index.html`
- `openclaw-canvas://main/assets/app.css` → `<canvasRoot>/main/assets/app.css`
- `openclaw-canvas://main/widgets/todo/` → `<canvasRoot>/main/widgets/todo/index.html`

If no `index.html` exists at the root, the app shows a **built‑in scaffold page**.

## Panel behavior

- Borderless, resizable panel anchored near the menu bar (or mouse cursor).
- Remembers size/position per session.
- Auto‑reloads when local canvas files change.
- Only one Canvas panel is visible at a time (session is switched as needed).

Canvas can be disabled from Settings → **Allow Canvas**. When disabled, canvas
node commands return `CANVAS_DISABLED`.

## Agent API surface

Canvas is exposed via the **Gateway WebSocket**, so the agent can:

- show/hide the panel
- navigate to a path or URL
- evaluate JavaScript
- capture a snapshot image

CLI examples:

```bash
openclaw nodes canvas present --node <id>
openclaw nodes canvas navigate --node <id> --url "/"
openclaw nodes canvas eval --node <id> --js "document.title"
openclaw nodes canvas snapshot --node <id>
```

Notes:

- `canvas.navigate` accepts **local canvas paths**, `http(s)` URLs, and `file://` URLs.
- If you pass `"/"`, the Canvas shows the local scaffold or `index.html`.

## A2UI in Canvas

A2UI is hosted by the Gateway canvas host and rendered inside the Canvas panel.
When the Gateway advertises a Canvas host, the macOS app auto‑navigates to the
A2UI host page on first open.

Default A2UI host URL:

```
http://<gateway-host>:18789/__openclaw__/a2ui/
```

### A2UI commands (v0.8)

Canvas currently accepts **A2UI v0.8** server→client messages:

- `beginRendering`
- `surfaceUpdate`
- `dataModelUpdate`
- `deleteSurface`

`createSurface` (v0.9) is not supported.

CLI example:

```bash
cat > /tmp/a2ui-v0.8.jsonl <<'EOFA2'
{"surfaceUpdate":{"surfaceId":"main","components":[{"id":"root","component":{"Column":{"children":{"explicitList":["title","content"]}}}},{"id":"title","component":{"Text":{"text":{"literalString":"Canvas (A2UI v0.8)"},"usageHint":"h1"}}},{"id":"content","component":{"Text":{"text":{"literalString":"If you can read this, A2UI push works."},"usageHint":"body"}}}]}}
{"beginRendering":{"surfaceId":"main","root":"root"}}
EOFA2

openclaw nodes canvas a2ui push --jsonl /tmp/a2ui-v0.8.jsonl --node <id>
```

Quick smoke:

```bash
openclaw nodes canvas a2ui push --node <id> --text "Hello from A2UI"
```

## Triggering agent runs from Canvas

Canvas can trigger new agent runs via deep links:

- `openclaw://agent?...`

Example (in JS):

```js
window.location.href = "openclaw://agent?message=Review%20this%20design";
```

The app prompts for confirmation unless a valid key is provided.

## Security notes

- Canvas scheme blocks directory traversal; files must live under the session root.
- Local Canvas content uses a custom scheme (no loopback server required).
- External `http(s)` URLs are allowed only when explicitly navigated.

## Related

- [macOS app](/platforms/macos)
- [WebChat](/web/webchat)

## Peekaboo (Screen Capture)

OpenClaw can host **PeekabooBridge** as a local, permission‑aware UI automation
broker. This lets the `peekaboo` CLI drive UI automation while reusing the
macOS app’s TCC permissions.

## What this is (and is not)

- **Host**: OpenClaw.app can act as a PeekabooBridge host.
- **Client**: use the `peekaboo` CLI (no separate `openclaw ui ...` surface).
- **UI**: visual overlays stay in Peekaboo.app; OpenClaw is a thin broker host.

## Relationship to Computer Use

OpenClaw has three desktop-control paths, and they intentionally stay separate:

- **PeekabooBridge host**: OpenClaw.app can host the local PeekabooBridge socket.
  The `peekaboo` CLI remains the client and uses OpenClaw.app's macOS
  permissions for Peekaboo automation primitives such as screenshots, clicks,
  menus, dialogs, Dock actions, and window management.
- **Codex Computer Use**: the bundled `codex` plugin prepares Codex app-server,
  verifies that Codex's `computer-use` MCP server is available, and then lets
  Codex own native desktop-control tool calls during Codex-mode turns. OpenClaw
  does not proxy those actions through PeekabooBridge.
- **Direct `cua-driver` MCP**: OpenClaw can register TryCua's upstream
  `cua-driver mcp` server as a normal MCP server. That gives agents the CUA
  driver's own schemas and pid/window/element-index workflow without routing
  through the Codex marketplace or the PeekabooBridge socket.

Use Peekaboo when you want the broad macOS automation surface and OpenClaw.app's
permission-aware bridge host. Use Codex Computer Use when a Codex-mode agent
should rely on Codex's native computer-use plugin. Use direct `cua-driver mcp`
when you want the CUA driver exposed to any OpenClaw-managed runtime as a normal
MCP server.

## Enable the bridge

In the macOS app:

- Settings → **Enable Peekaboo Bridge**

When enabled, OpenClaw starts a local UNIX socket server. If disabled, the host
is stopped and `peekaboo` will fall back to other available hosts.

## Client discovery order

Peekaboo clients typically try hosts in this order:

1. Peekaboo.app (full UX)
2. Claude.app (if installed)
3. OpenClaw.app (thin broker)

Use `peekaboo bridge status --verbose` to see which host is active and which
socket path is in use. You can override with:

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## Security & permissions

- The bridge validates **caller code signatures**; an allowlist of TeamIDs is
  enforced (Peekaboo host TeamID + OpenClaw app TeamID).
- Requests time out after ~10 seconds.
- If required permissions are missing, the bridge returns a clear error message
  rather than launching System Settings.

## Snapshot behavior (automation)

Snapshots are stored in memory and expire automatically after a short window.
If you need longer retention, re‑capture from the client.

## Troubleshooting

- If `peekaboo` reports “bridge client is not authorized”, ensure the client is
  properly signed or run the host with `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1`
  in **debug** mode only.
- If no hosts are found, open one of the host apps (Peekaboo.app or OpenClaw.app)
  and confirm permissions are granted.

## Related

- [macOS app](/platforms/macos)
- [macOS permissions](/platforms/mac/permissions)

## Voice Overlay

# Voice Overlay Lifecycle (macOS)

Audience: macOS app contributors. Goal: keep the voice overlay predictable when wake-word and push-to-talk overlap.

## Current intent

- If the overlay is already visible from wake-word and the user presses the hotkey, the hotkey session _adopts_ the existing text instead of resetting it. The overlay stays up while the hotkey is held. When the user releases: send if there is trimmed text, otherwise dismiss.
- Wake-word alone still auto-sends on silence; push-to-talk sends immediately on release.

## Implemented (Dec 9, 2025)

- Overlay sessions now carry a token per capture (wake-word or push-to-talk). Partial/final/send/dismiss/level updates are dropped when the token doesn’t match, avoiding stale callbacks.
- Push-to-talk adopts any visible overlay text as a prefix (so pressing the hotkey while the wake overlay is up keeps the text and appends new speech). It waits up to 1.5s for a final transcript before falling back to the current text.
- Chime/overlay logging is emitted at `info` in categories `voicewake.overlay`, `voicewake.ptt`, and `voicewake.chime` (session start, partial, final, send, dismiss, chime reason).

## Next steps

1. **VoiceSessionCoordinator (actor)**
   - Owns exactly one `VoiceSession` at a time.
   - API (token-based): `beginWakeCapture`, `beginPushToTalk`, `updatePartial`, `endCapture`, `cancel`, `applyCooldown`.
   - Drops callbacks that carry stale tokens (prevents old recognizers from reopening the overlay).
2. **VoiceSession (model)**
   - Fields: `token`, `source` (wakeWord|pushToTalk), committed/volatile text, chime flags, timers (auto-send, idle), `overlayMode` (display|editing|sending), cooldown deadline.
3. **Overlay binding**
   - `VoiceSessionPublisher` (`ObservableObject`) mirrors the active session into SwiftUI.
   - `VoiceWakeOverlayView` renders only via the publisher; it never mutates global singletons directly.
   - Overlay user actions (`sendNow`, `dismiss`, `edit`) call back into the coordinator with the session token.
4. **Unified send path**
   - On `endCapture`: if trimmed text is empty → dismiss; else `performSend(session:)` (plays send chime once, forwards, dismisses).
   - Push-to-talk: no delay; wake-word: optional delay for auto-send.
   - Apply a short cooldown to the wake runtime after push-to-talk finishes so wake-word doesn’t immediately retrigger.
5. **Logging**
   - Coordinator emits `.info` logs in subsystem `ai.openclaw`, categories `voicewake.overlay` and `voicewake.chime`.
   - Key events: `session_started`, `adopted_by_push_to_talk`, `partial`, `finalized`, `send`, `dismiss`, `cancel`, `cooldown`.

## Debugging checklist

- Stream logs while reproducing a sticky overlay:

  ```bash
  sudo log stream --predicate 'subsystem == "ai.openclaw" AND category CONTAINS "voicewake"' --level info --style compact
  ```

- Verify only one active session token; stale callbacks should be dropped by the coordinator.
- Ensure push-to-talk release always calls `endCapture` with the active token; if text is empty, expect `dismiss` without chime or send.

## Migration steps (suggested)

1. Add `VoiceSessionCoordinator`, `VoiceSession`, and `VoiceSessionPublisher`.
2. Refactor `VoiceWakeRuntime` to create/update/end sessions instead of touching `VoiceWakeOverlayController` directly.
3. Refactor `VoicePushToTalk` to adopt existing sessions and call `endCapture` on release; apply runtime cooldown.
4. Wire `VoiceWakeOverlayController` to the publisher; remove direct calls from runtime/PTT.
5. Add integration tests for session adoption, cooldown, and empty-text dismissal.

## Related

- [macOS app](/platforms/macos)
- [Voice wake (macOS)](/platforms/mac/voicewake)
- [Talk mode](/nodes/talk)

## Voice Wake

# Voice Wake & Push-to-Talk

## Modes

- **Wake-word mode** (default): always-on Speech recognizer waits for trigger tokens (`swabbleTriggerWords`). On match it starts capture, shows the overlay with partial text, and auto-sends after silence.
- **Push-to-talk (Right Option hold)**: hold the right Option key to capture immediately—no trigger needed. The overlay appears while held; releasing finalizes and forwards after a short delay so you can tweak text.

## Runtime behavior (wake-word)

- Speech recognizer lives in `VoiceWakeRuntime`.
- Trigger only fires when there’s a **meaningful pause** between the wake word and the next word (~0.55s gap). The overlay/chime can start on the pause even before the command begins.
- Silence windows: 2.0s when speech is flowing, 5.0s if only the trigger was heard.
- Hard stop: 120s to prevent runaway sessions.
- Debounce between sessions: 350ms.
- Overlay is driven via `VoiceWakeOverlayController` with committed/volatile coloring.
- After send, recognizer restarts cleanly to listen for the next trigger.

## Lifecycle invariants

- If Voice Wake is enabled and permissions are granted, the wake-word recognizer should be listening (except during an explicit push-to-talk capture).
- Overlay visibility (including manual dismiss via the X button) must never prevent the recognizer from resuming.

## Sticky overlay failure mode (previous)

Previously, if the overlay got stuck visible and you manually closed it, Voice Wake could appear “dead” because the runtime’s restart attempt could be blocked by overlay visibility and no subsequent restart was scheduled.

Hardening:

- Wake runtime restart is no longer blocked by overlay visibility.
- Overlay dismiss completion triggers a `VoiceWakeRuntime.refresh(...)` via `VoiceSessionCoordinator`, so manual X-dismiss always resumes listening.

## Push-to-talk specifics

- Hotkey detection uses a global `.flagsChanged` monitor for **right Option** (`keyCode 61` + `.option`). We only observe events (no swallowing).
- Capture pipeline lives in `VoicePushToTalk`: starts Speech immediately, streams partials to the overlay, and calls `VoiceWakeForwarder` on release.
- When push-to-talk starts we pause the wake-word runtime to avoid dueling audio taps; it restarts automatically after release.
- Permissions: requires Microphone + Speech; seeing events needs Accessibility/Input Monitoring approval.
- External keyboards: some may not expose right Option as expected—offer a fallback shortcut if users report misses.

## User-facing settings

- **Voice Wake** toggle: enables wake-word runtime.
- **Hold Cmd+Fn to talk**: enables the push-to-talk monitor. Disabled on macOS < 26.
- Language & mic pickers, live level meter, trigger-word table, tester (local-only; does not forward).
- Mic picker preserves the last selection if a device disconnects, shows a disconnected hint, and temporarily falls back to the system default until it returns.
- **Sounds**: chimes on trigger detect and on send; defaults to the macOS “Glass” system sound. You can pick any `NSSound`-loadable file (e.g. MP3/WAV/AIFF) for each event or choose **No Sound**.

## Forwarding behavior

- When Voice Wake is enabled, transcripts are forwarded to the active gateway/agent (the same local vs remote mode used by the rest of the mac app).
- Replies are delivered to the **last-used main provider** (WhatsApp/Telegram/Discord/WebChat). If delivery fails, the error is logged and the run is still visible via WebChat/session logs.

## Forwarding payload

- `VoiceWakeForwarder.prefixedTranscript(_:)` prepends the machine hint before sending. Shared between wake-word and push-to-talk paths.

## Quick verification

- Toggle push-to-talk on, hold Cmd+Fn, speak, release: overlay should show partials then send.
- While holding, menu-bar ears should stay enlarged (uses `triggerVoiceEars(ttl:nil)`); they drop after release.

## Related

- [Voice wake](/nodes/voicewake)
- [Voice overlay](/platforms/mac/voice-overlay)
- [macOS app](/platforms/macos)

## WebChat (macOS)

The macOS menu bar app embeds the WebChat UI as a native SwiftUI view. It
connects to the Gateway and defaults to the **main session** for the selected
agent (with a session switcher for other sessions).

- **Local mode**: connects directly to the local Gateway WebSocket.
- **Remote mode**: forwards the Gateway control port over SSH and uses that
  tunnel as the data plane.

## Launch & debugging

- Manual: Lobster menu → “Open Chat”.
- Auto‑open for testing:

  ```bash
  dist/OpenClaw.app/Contents/MacOS/OpenClaw --webchat
  ```

- Logs: `./scripts/clawlog.sh` (subsystem `ai.openclaw`, category `WebChatSwiftUI`).

## How it is wired

- Data plane: Gateway WS methods `chat.history`, `chat.send`, `chat.abort`,
  `chat.inject` and events `chat`, `agent`, `presence`, `tick`, `health`.
- `chat.history` returns display-normalized transcript rows: inline directive
  tags are stripped from visible text, plain-text tool-call XML payloads
  (including `<tool_call>...</tool_call>`,
  `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`,
  `<function_calls>...</function_calls>`, and truncated tool-call blocks) and
  leaked ASCII/full-width model control tokens are stripped, pure
  silent-token assistant rows such as exact `NO_REPLY` / `no_reply` are
  omitted, and oversized rows can be replaced with placeholders.
- Session: defaults to the primary session (`main`, or `global` when scope is
  global). The UI can switch between sessions.
- Onboarding uses a dedicated session to keep first‑run setup separate.

## Security surface

- Remote mode forwards only the Gateway WebSocket control port over SSH.

## Known limitations

- The UI is optimized for chat sessions (not a full browser sandbox).

## Related

- [WebChat](/web/webchat)
- [macOS app](/platforms/macos)

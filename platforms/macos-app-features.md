---
domain: platforms
topic: "macOS App Features: Menu Bar Icon, Logging, Remote Control, Skills UI, and Voice Wake"
type: reference
keywords:
  - macOS menu bar icon
  - macOS logging
  - macOS remote control
  - macOS skills settings
  - macOS voice wake
  - push to talk
  - SSH tunnel macOS
  - rolling diagnostics log
  - unified logging privacy
  - voice wake forwarding
  - ai.openclaw plist
source:
  - platforms/mac/icon.md
  - platforms/mac/logging.md
  - platforms/mac/remote.md
  - platforms/mac/skills.md
  - platforms/mac/voicewake.md
related:
  - platforms/macos-gateway-lifecycle
  - platforms/macos-app
  - nodes/voicewake
---

macOS-specific app features: menu bar icon states, logging configuration, remote control over SSH, the Skills settings UI, and Voice Wake and push-to-talk pipeline.

## Menu Bar Icon States

| State | Behavior |
|-------|----------|
| **Idle** | Normal icon animation (blink, occasional wiggle) |
| **Paused** | `appearsDisabled`; no motion |
| **Voice wake triggered** | Ears scale up 1.9× with ear holes; drop after 1s silence |
| **Agent working** | Faster leg wiggle and slight offset while work is in-flight |

**Wiring:**
- Voice wake: `AppState.triggerVoiceEars(ttl: nil)` on trigger; `stopVoiceEars()` after 1s silence.
- Agent activity: `AppStateStore.shared.setWorking(true/false)` around work spans. Keep spans short and reset in `defer` blocks to avoid stuck animations.

Icon is 18×18 pt (36×36 px Retina). Keep TTLs under 10s so the icon returns to baseline quickly. No external CLI/broker toggle for ears/working — keep signals internal to avoid accidental flapping.

## Logging

### Rolling Diagnostics File Log

OpenClaw can write a local rotating JSONL file log. Off by default.

- Enable: **Debug pane → Logs → App logging → "Write rolling diagnostics log (JSONL)"**
- Location: `~/Library/Logs/OpenClaw/diagnostics.jsonl` (auto-rotates; old files suffixed `.1`, `.2`, …)
- Verbosity: **Debug pane → Logs → App logging → Verbosity**

Treat the file as sensitive; review before sharing.

### Unified Logging Privacy (macOS)

Unified logging redacts payloads by default. To enable private data logging for `ai.openclaw`, install a subsystem plist as root:

```bash
# Write the plist and install atomically
sudo install -m 644 -o root /tmp/ai.openclaw.plist \
  /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

The plist content (write to `/tmp/ai.openclaw.plist` first):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
```

No reboot required; logd notices quickly. Only new log lines include private payloads.

**Disable after debugging:**

```bash
sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
sudo log config --reload
```

This surface can include phone numbers and message bodies — keep the plist only while actively needed.

## Remote Control (macOS App to Remote Host)

The macOS app can control a remote OpenClaw gateway over SSH. Configure in **Settings → General → OpenClaw runs**.

**Modes:**

| Mode | Description |
|------|-------------|
| Local (this Mac) | Everything runs locally. No SSH. |
| Remote over SSH | Commands execute on the remote host; app opens `ssh -N -L ...` tunnel. |
| Remote direct (ws/wss) | No SSH tunnel; app connects to gateway URL directly (Tailscale Serve or HTTPS proxy). |

**Remote setup:**
1. Settings → General → Remote over SSH → set SSH target (`user@host`, optional `:port`).
2. Choose Transport: SSH tunnel or Direct (ws/wss).
3. Click **Test remote** — success means `openclaw status --json` ran correctly on the remote.

**Prerequisites on remote host:**
1. Node + pnpm installed; `openclaw` CLI on PATH for non-interactive shells.
2. SSH key auth configured. Tailscale IPs recommended for stable off-LAN reachability.

**Troubleshooting:**
- `exit 127 / not found`: `openclaw` not on PATH for non-login shells. Symlink into `/usr/local/bin` or `/opt/homebrew/bin`.
- Web Chat stuck: confirm gateway is running on remote and forwarded port matches gateway WS port.
- Node IP shows 127.0.0.1: expected with SSH tunnel. Switch to Direct (ws/wss) to see real client IP.
- Dashboard works but Mac capabilities are offline: the app's operator/control connection is healthy, but the companion node connection is not connected or is missing its command surface. Open the menu bar device section and check whether the Mac is `paired · disconnected`. For `wss://*.ts.net` Tailscale Serve endpoints, the app detects stale legacy TLS leaf pins after certificate rotation, clears the stale pin when macOS trusts the new certificate, and retries automatically. If the certificate is not system-trusted or the host is not a Tailscale Serve name, review the certificate or switch to Remote over SSH.
- Voice Wake: trigger phrases are forwarded automatically in remote mode; no separate forwarder needed.

**Security:** Prefer loopback binds on remote host and connect via SSH or Tailscale. SSH tunneling uses strict host-key checking; trust the host key first.

## Skills Settings UI

The macOS app surfaces skills via the gateway — it does not parse skills locally.

**Data source:** `skills.status` (gateway) returns all skills plus eligibility, missing requirements, and allowlist blocks.

**Install preference:** Homebrew first (when `skills.install.preferBrew` is enabled and `brew` exists), then `uv`, then configured node manager, then `go` or `download`.

**Env/API keys:** Stored in `~/.openclaw/openclaw.json` under `skills.entries.<skillKey>`. Updated via `skills.update` (patches `enabled`, `apiKey`, and `env`).

**Remote mode:** Install and config updates happen on the gateway host, not the local Mac.

## Voice Wake and Push-to-Talk

### Wake-Word Mode (Default)

Always-on Speech recognizer waits for trigger tokens. On match: starts capture, shows overlay with partial text, auto-sends after silence.

- Silence windows: 2.0s while speech is flowing; 5.0s if only the trigger was heard
- Hard stop: 120s (prevents runaway sessions)
- Debounce between sessions: 350ms

### Push-to-Talk (Right Option Hold)

Hold Right Option key to capture immediately. Overlay appears while held; release finalizes.

- Pauses wake-word runtime during PTT to avoid dueling audio taps; restarts automatically after release.
- Requires Microphone + Speech permissions (Accessibility/Input Monitoring for key monitoring).
- External keyboards: some may not expose right Option as expected.

### Lifecycle Invariants

- If Voice Wake is enabled and permissions granted, the wake-word recognizer is always listening (except during PTT capture).
- Overlay manual dismiss triggers `VoiceWakeRuntime.refresh(...)` — the X button always resumes listening.

### Forwarding

Transcripts are forwarded to the active gateway/agent (same local vs remote mode). Replies are delivered to the **last-used main provider** (WhatsApp/Telegram/Discord/WebChat).

### Quick Verification

Toggle push-to-talk on, hold Right Option, speak, release: overlay should show partials then send. While holding, menu-bar ears should stay enlarged (1.9×), dropping after release.

## Related

- [macOS gateway lifecycle](/platforms/macos-gateway-lifecycle)
- [macOS app](/platforms/macos-app)
- [Voice wake (nodes)](/nodes/voicewake)

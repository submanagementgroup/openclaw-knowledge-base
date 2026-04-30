---
domain: cli
topic: "openclaw voicecall: Voice-Call Plugin CLI Commands"
type: reference
keywords:
  - openclaw voicecall
  - voice call plugin
  - voicecall setup
  - voicecall smoke
  - voicecall dtmf
  - twilio
  - telnyx
  - plivo
  - webhook expose
source: cli/voicecall.md
related:
  - cli/cli-overview
  - plugins/voice-call
---

`openclaw voicecall` is a plugin-provided CLI command that appears only when the voice-call plugin is installed and enabled. It provides setup verification, call initiation, DTMF control, and webhook exposure.

## Common Commands

```bash
openclaw voicecall setup
openclaw voicecall smoke
openclaw voicecall status --call-id <id>
openclaw voicecall call --to "+15555550123" --message "Hello" --mode notify
openclaw voicecall continue --call-id <id> --message "Any questions?"
openclaw voicecall dtmf --call-id <id> --digits "ww123456#"
openclaw voicecall end --call-id <id>
```

## Setup Verification

`setup` prints readiness checks. Use `--json` for scripts:

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

For external providers (`twilio`, `telnyx`, `plivo`), setup must resolve a public webhook URL from `publicUrl`, a tunnel, or Tailscale exposure. A loopback/private address is rejected because carriers cannot reach it.

## Smoke Testing

`smoke` runs readiness checks without placing a real call, unless `--to` and `--yes` are both provided:

```bash
openclaw voicecall smoke --to "+15555550123"        # dry run
openclaw voicecall smoke --to "+15555550123" --yes  # live notify call
```

## Exposing Webhooks via Tailscale

```bash
openclaw voicecall expose --mode serve    # Tailscale Serve (preferred)
openclaw voicecall expose --mode funnel   # Tailscale Funnel (public)
openclaw voicecall expose --mode off      # disable exposure
```

Prefer Tailscale Serve over Funnel when possible to limit exposure.

## Related

- [CLI overview](/cli)
- [Voice call plugin](/plugins/voice-call)

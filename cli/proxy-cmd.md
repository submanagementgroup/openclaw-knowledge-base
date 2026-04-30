---
domain: cli
topic: "openclaw proxy: Local Debug Proxy and Traffic Capture Inspector"
type: reference
keywords:
  - openclaw proxy
  - debug proxy
  - traffic capture
  - proxy sessions
  - proxy blobs
  - double-sends
  - retry-storms
  - ws-duplicate-frames
source: cli/proxy.md
related:
  - cli/cli-overview
  - gateway/troubleshooting
---

`openclaw proxy` runs a local explicit debug proxy for transport-level investigation of OpenClaw traffic. It can start a local proxy, capture sessions, query traffic patterns, read blobs, and purge data.

## Commands

```bash
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

## Query Presets

`openclaw proxy query --preset <name>` accepts these diagnostic presets:

- `double-sends` — detect duplicate request sending
- `retry-storms` — detect excessive retry patterns
- `cache-busting` — detect cache bypass patterns
- `ws-duplicate-frames` — detect duplicate WebSocket frames
- `missing-ack` — detect missing acknowledgements
- `error-bursts` — detect error rate spikes

## Notes

- `start` defaults to `127.0.0.1` unless `--host` is set.
- `run` starts a local debug proxy and then runs the command after `--`.
- Captures are local debugging data; use `openclaw proxy purge` when finished.
- This command is for transport-level debugging only, not for production use.

## Related

- [CLI overview](/cli)
- [Gateway troubleshooting](/gateway/troubleshooting)

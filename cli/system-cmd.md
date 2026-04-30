---
domain: cli
topic: "openclaw system: System Events, Heartbeat Control, and Presence"
type: reference
keywords:
  - openclaw system
  - system event
  - heartbeat enable
  - heartbeat disable
  - system presence
  - gateway RPC
  - enqueue event
source: cli/system.md
related:
  - cli/cli-overview
  - gateway/heartbeat
---

`openclaw system` provides system-level helpers for the Gateway: enqueue system events, control heartbeats, and view presence. All subcommands use Gateway RPC.

## Common Commands

```bash
# Enqueue a system event immediately
openclaw system event --text "Check for urgent follow-ups" --mode now

# Enqueue for the next scheduled heartbeat tick
openclaw system event --text "Check for urgent follow-ups"

# Control heartbeats
openclaw system heartbeat enable
openclaw system heartbeat disable
openclaw system heartbeat last

# View current system presence
openclaw system presence
```

## `system event`

Enqueue a system event on the **main** session. The next heartbeat will inject it as a `System:` line in the prompt.

Flags:

- `--text <text>`: required system event text
- `--mode <mode>`: `now` (trigger immediately) or `next-heartbeat` (default)
- `--json`: machine-readable output
- `--url`, `--token`, `--timeout`, `--expect-final`: shared Gateway RPC flags

## `system heartbeat`

Control heartbeat operation:

- `last`: show the last heartbeat event
- `enable`: turn heartbeats on (use if they were disabled)
- `disable`: pause heartbeats

## `system presence`

List current system presence entries the Gateway knows about (nodes, instances, and status lines).

## Shared Gateway RPC Flags

All `system` subcommands accept:

- `--url <url>`: Gateway WebSocket URL
- `--token <token>`: Gateway auth token
- `--timeout <ms>`: request timeout
- `--expect-final`: wait for a final response

## Notes

- Requires a running Gateway reachable by current config (local or remote).
- System events are ephemeral and not persisted across restarts.

## Related

- [CLI overview](/cli)
- [Heartbeat](/gateway/heartbeat)

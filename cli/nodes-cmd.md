---
domain: cli
topic: "openclaw nodes: Manage Paired Nodes, Pairing, and Node Capabilities"
type: reference
keywords:
  - openclaw nodes
  - nodes list
  - nodes pending
  - nodes approve
  - nodes reject
  - nodes invoke
  - node pairing
  - node capabilities
  - camera node
  - canvas node
  - screen node
source: cli/nodes.md
related:
  - cli/cli-overview
  - nodes/nodes-overview
  - tools/exec-approvals
---

`openclaw nodes` manages paired nodes (devices) and invokes node capabilities such as camera, canvas, screen, and location. Nodes are iOS, Android, or macOS devices paired to the gateway.

## Common Commands

```bash
# List all paired nodes
openclaw nodes list
openclaw nodes list --connected
openclaw nodes list --last-connected 24h

# Manage pairing
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>

# Check status
openclaw nodes status
openclaw nodes status --connected

# Invoke a node command
openclaw nodes invoke --node <id|name|ip> --command <command> --params <json>
```

## Pairing Approval Notes

- `openclaw nodes pending` only needs pairing scope.
- `gateway.nodes.pairing.autoApproveCidrs` can skip the pending step for trusted CIDR blocks — off by default.
- `openclaw nodes approve <requestId>` inherits extra scope requirements from the pending request:
  - commandless request: pairing only
  - non-exec node commands: pairing + write
  - `system.run` / `system.run.prepare` / `system.which`: pairing + admin

## Invoke Flags

- `--params <json>`: JSON object string (default `{}`)
- `--invoke-timeout <ms>`: node invoke timeout (default `15000`)
- `--idempotency-key <key>`: optional idempotency key

`system.run` and `system.run.prepare` are blocked in `nodes invoke`; use the `exec` tool with `host=node` for shell execution on nodes.

## Filtering

- `--connected`: only show currently-connected nodes
- `--last-connected <duration>`: filter to nodes that connected within a duration (e.g. `24h`, `7d`)

## Related

- [Nodes overview](/nodes/nodes-overview)
- [Exec approvals](/tools/exec-approvals)
- [Node troubleshooting](/nodes/troubleshooting)

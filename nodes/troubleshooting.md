---
domain: nodes
topic: "Node Troubleshooting: Pairing, Permissions, Foreground Requirements, and Error Codes"
type: troubleshooting
keywords:
  - node troubleshooting
  - NODE_BACKGROUND_UNAVAILABLE
  - CAMERA_DISABLED
  - SYSTEM_RUN_DENIED
  - LOCATION_PERMISSION_REQUIRED
  - node pairing vs approvals
  - exec approvals node
  - canvas foreground
  - screen foreground
  - camera permission node
source: nodes/troubleshooting.md
related:
  - nodes/nodes-overview
  - nodes/location-command
  - tools/exec-approvals
  - gateway/troubleshooting
---

Use this page when a node is visible in status but node tools fail. The main issue categories are: foreground requirements, OS permissions, pairing vs approval distinction, and gateway node command policy.

## Command Ladder

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe

# Node-specific checks
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

**Healthy signals:**
- Node is connected and paired for role `node`
- `nodes describe` includes the capability you are calling
- Exec approvals show expected mode/allowlist

## Foreground Requirements

`canvas.*`, `camera.*`, and `screen.*` are **foreground only** on iOS/Android nodes.

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

If you see `NODE_BACKGROUND_UNAVAILABLE`, bring the node app to the foreground and retry.

## Permissions Matrix

| Capability | iOS | Android | macOS node | Typical failure |
|-----------|-----|---------|------------|----------------|
| `camera.snap`, `camera.clip` | Camera (+ mic for clip) | Camera (+ mic for clip) | Camera (+ mic for clip) | `*_PERMISSION_REQUIRED` |
| `screen.record` | Screen Recording (+ mic optional) | Screen capture prompt | Screen Recording | `*_PERMISSION_REQUIRED` |
| `location.get` | While Using or Always | Foreground/Background location | Location permission | `LOCATION_PERMISSION_REQUIRED` |
| `system.run` | n/a (node host path) | n/a (node host path) | Exec approvals required | `SYSTEM_RUN_DENIED` |

## Pairing vs Approvals: Three Separate Gates

1. **Device pairing**: can this node connect to the gateway?
2. **Gateway node command policy**: is the RPC command ID allowed by `gateway.nodes.allowCommands` / `denyCommands`?
3. **Exec approvals**: can this node run a specific shell command locally?

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

- If pairing is missing: approve the node device first.
- If `nodes describe` is missing a command: check the gateway node command policy and whether the node declared that command on connect.
- If pairing is fine but `system.run` fails: fix exec approvals/allowlist on that node.

For `host=node` runs, the gateway binds execution to the prepared canonical `systemRunPlan`. If a caller mutates command/cwd or session metadata before the approved run is forwarded, the gateway rejects it as an approval mismatch.

## Common Node Error Codes

| Code | Meaning and Fix |
|------|-----------------|
| `NODE_BACKGROUND_UNAVAILABLE` | App is backgrounded; bring it to foreground and retry |
| `CAMERA_DISABLED` | Camera toggle disabled in node settings |
| `*_PERMISSION_REQUIRED` | OS permission missing/denied; grant in system Settings |
| `LOCATION_DISABLED` | Location mode is off in node settings |
| `LOCATION_PERMISSION_REQUIRED` | Requested location mode not granted by OS |
| `LOCATION_BACKGROUND_UNAVAILABLE` | App is backgrounded with only While Using permission |
| `SYSTEM_RUN_DENIED: approval required` | Exec request needs explicit approval |
| `SYSTEM_RUN_DENIED: allowlist miss` | Command blocked by allowlist mode (on Windows node hosts, `cmd.exe /c ...` forms are treated as allowlist misses in allowlist mode unless explicitly approved) |

## Fast Recovery Loop

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

If still stuck:
1. Re-approve device pairing
2. Re-open node app (bring to foreground)
3. Re-grant OS permissions in system Settings
4. Recreate/adjust exec approval policy

## Related

- [Nodes overview](/nodes/nodes-overview)
- [Location command](/nodes/location-command)
- [Exec approvals](/tools/exec-approvals)
- [Gateway troubleshooting](/gateway/troubleshooting)

---
domain: cli
topic: "openclaw daemon: Legacy Alias for Gateway Service Management"
type: reference
keywords:
  - openclaw daemon
  - daemon status
  - daemon install
  - daemon start
  - daemon stop
  - daemon restart
  - legacy gateway alias
  - launchd
  - systemd
source: cli/daemon.md
related:
  - cli/cli-overview
  - cli/gateway-cli
  - gateway/gateway-runbook
---

`openclaw daemon` is a legacy alias for Gateway service management commands. It maps to the same service control surface as `openclaw gateway` service commands. Prefer `openclaw gateway` for new scripts.

## Usage

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## Subcommands

- `status`: show service install state and probe Gateway health
- `install`: install service (`launchd`/`systemd`/`schtasks`)
- `uninstall`: remove service
- `start`: start service
- `stop`: stop service
- `restart`: restart service

## Common Options

- `status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
- `install`: `--port`, `--runtime <node|bun>`, `--token`, `--force`, `--json`
- lifecycle (`uninstall|start|stop|restart`): `--json`

## Security Notes for Install

- `status --deep` adds a system-level service scan; warns if multiple gateway-like services are found.
- On macOS, `install` keeps LaunchAgent plists owner-only and loads environment values through an owner-only file and wrapper.
- If token auth requires a token and `gateway.auth.token` is SecretRef-managed, `install` validates the SecretRef is resolvable but does not persist the resolved token into service environment metadata.
- If both `gateway.auth.token` and `gateway.auth.password` are configured and `gateway.auth.mode` is unset, install is blocked until mode is set explicitly.
- Token-drift checks on Linux include both `Environment=` and `EnvironmentFile=` unit sources.

## Prefer Modern Commands

Use [`openclaw gateway`](/cli/gateway-cli) for current docs and examples. `openclaw daemon` is kept for backwards compatibility only.

## Related

- [CLI overview](/cli)
- [Gateway CLI](/cli/gateway-cli)
- [Gateway runbook](/gateway/gateway-runbook)

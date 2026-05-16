---
domain: gateway
topic: "Running Multiple Gateways"
type: procedure
keywords:
  - multiple gateways
  - gateway instances
  - parallel gateways
  - gateway lock
  - port conflicts
source: ['gateway/multiple-gateways.md', 'gateway/gateway-lock.md']
---

Most setups should use one Gateway because a single Gateway can handle multiple messaging connections and agents. If you need stronger isolation or redundancy (e.g., a rescue bot), run separate Gateways with isolated profiles/ports.

## Best recommended setup

For most users, the simplest rescue-bot setup is:

- keep the main bot on the default profile
- run the rescue bot on `--profile rescue`
- use a completely separate Telegram bot for the rescue account
- keep the rescue bot on a different base port such as `19789`

This keeps the rescue bot isolated from the main bot so it can debug or apply
config changes if the primary bot is down. Leave at least 20 ports between
base ports so the derived browser/canvas/CDP ports never collide.

## Rescue-Bot Quickstart

Use this as the default path unless you have a strong reason to do something
else:

```bash
# Rescue bot (separate Telegram bot, separate profile, port 19789)
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

If your main bot is already running, that is usually all you need.

During `openclaw --profile rescue onboard`:

- use the separate Telegram bot token
- keep the `rescue` profile
- use a base port at least 20 higher than the main bot
- accept the default rescue workspace unless you already manage one yourself

If onboarding already installed the rescue service for you, the final
`gateway install` is not needed.

## Why this works

The rescue bot stays independent because it has its own:

- profile/config
- state directory
- workspace
- base port (plus derived ports)
- Telegram bot token

For most setups, use a completely separate Telegram bot for the rescue profile:

- easy to keep operator-only
- separate bot token and identity
- independent from the main bot's channel/app install
- simple DM-based recovery path when the main bot is broken

## What `--profile rescue onboard` Changes

`openclaw --profile rescue onboard` uses the normal onboarding flow, but it
writes everything into a separate profile.

In practice, that means the rescue bot gets its own:

- config file
- state directory
- workspace (by default `~/.openclaw/workspace-rescue`)
- managed service name

The prompts are otherwise the same as normal onboarding.

## General multi-gateway setup

The rescue-bot layout above is the easiest default, but the same isolation
pattern works for any pair or group of Gateways on one host.

For a more general setup, give each extra Gateway its own named profile and its
own base port:

```bash
# main (default profile)
openclaw setup
openclaw gateway --port 18789

# extra gateway
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

If you want both Gateways to use named profiles, that also works:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

Services follow the same pattern:

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

Use the rescue-bot quickstart when you want a fallback operator lane. Use the
general profile pattern when you want multiple long-lived Gateways for
different channels, tenants, workspaces, or operational roles.

## Isolation checklist

Keep these unique per Gateway instance:

- `OPENCLAW_CONFIG_PATH` — per-instance config file
- `OPENCLAW_STATE_DIR` — per-instance sessions, creds, caches
- `agents.defaults.workspace` — per-instance workspace root
- `gateway.port` (or `--port`) — unique per instance
- derived browser/canvas/CDP ports

If these are shared, you will hit config races and port conflicts.

## Port mapping (derived)

Base port = `gateway.port` (or `OPENCLAW_GATEWAY_PORT` / `--port`).

- browser control service port = base + 2 (loopback only)
- canvas host is served on the Gateway HTTP server (same port as `gateway.port`)
- Browser profile CDP ports auto-allocate from `browser.controlPort + 9 .. + 108`

If you override any of these in config or env, you must keep them unique per instance.

## Browser/CDP notes (common footgun)

- Do **not** pin `browser.cdpUrl` to the same values on multiple instances.
- Each instance needs its own browser control port and CDP range (derived from its gateway port).
- If you need explicit CDP ports, set `browser.profiles.<name>.cdpPort` per instance.
- Remote Chrome: use `browser.profiles.<name>.cdpUrl` (per profile, per instance).

## Manual env example

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## Quick checks

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

Interpretation:

- `gateway status --deep` helps catch stale launchd/systemd/schtasks services from older installs.
- `gateway probe` warning text such as `multiple reachable gateways detected` is expected only when you intentionally run more than one isolated gateway.

## Related

- [Gateway runbook](/gateway)
- [Gateway lock](/gateway/gateway-lock)
- [Configuration](/gateway/configuration)


## Gateway Lock

## Why

- Ensure only one gateway instance runs per base port on the same host; additional gateways must use isolated profiles and unique ports.
- Survive crashes/SIGKILL without leaving stale lock files.
- Fail fast with a clear error when the control port is already occupied.

## Mechanism

- The gateway first acquires a per-config lock file under the state lock directory and probes the configured port for an existing listener.
- If the recorded lock owner is gone, the port is free, or the lock is stale, startup reclaims the lock and continues.
- The gateway then binds the HTTP/WebSocket listener (default `ws://127.0.0.1:18789`) using an exclusive TCP listener.
- If the bind fails with `EADDRINUSE`, startup throws `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- On shutdown the gateway closes the HTTP/WebSocket server and removes the lock file.

## Error surface

- If another process holds the port, startup throws `GatewayLockError("another gateway instance is already listening on ws://127.0.0.1:<port>")`.
- Other bind failures surface as `GatewayLockError("failed to bind gateway socket on ws://127.0.0.1:<port>: …")`.

## Operational notes

- If the port is occupied by _another_ process, the error is the same; free the port or choose another with `openclaw gateway --port <port>`.
- Under a service supervisor, a new gateway process that sees an existing healthy `/healthz` responder leaves that process in control. On systemd, the duplicate starter exits with code 78 so the default `RestartPreventExitStatus=78` stops `Restart=always` from looping on a lock or `EADDRINUSE` conflict. If the existing process never becomes healthy, retries are bounded and startup fails with a clear lock error instead of looping forever.
- The macOS app still maintains its own lightweight PID guard before spawning the gateway; the runtime lock is enforced by the lock file plus HTTP/WebSocket bind.

## Related

- [Multiple Gateways](/gateway/multiple-gateways) — running multiple instances with unique ports
- [Troubleshooting](/gateway/troubleshooting) — diagnosing `EADDRINUSE` and port conflicts

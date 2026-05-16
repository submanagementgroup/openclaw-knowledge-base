---
domain: cli
topic: "CLI Commands (cont): doctor, docs, flows..."
type: reference
keywords:
  - doctor
  - docs
  - flows
  - health
  - hooks
  - CLI commands
  - openclaw CLI
source: 
  - cli/doctor.md
  - cli/docs.md
  - cli/flows.md
  - cli/health.md
  - cli/hooks.md
  - cli/logs.md
  - cli/memory.md
  - cli/node.md
---
## Update hook packs

```bash
openclaw plugins update <id>
openclaw plugins update --all
```

Update tracked npm-based hook packs through the unified plugins updater.

`openclaw hooks update` still works as a compatibility alias, but it prints a
deprecation warning and forwards to `openclaw plugins update`.

**Options:**

- `--all`: Update all tracked hook packs
- `--dry-run`: Show what would change without writing

When a stored integrity hash exists and the fetched artifact hash changes,
OpenClaw prints a warning and asks for confirmation before proceeding. Use
global `--yes` to bypass prompts in CI/non-interactive runs.

## Bundled hooks

### session-memory

Saves session context to memory when you issue `/new` or `/reset`.

**Enable:**

```bash
openclaw hooks enable session-memory
```

**Output:** `~/.openclaw/workspace/memory/YYYY-MM-DD-HHMM.md` by default. Set `hooks.internal.entries.session-memory.llmSlug: true` for model-generated filename slugs.

**See:** [session-memory documentation](/automation/hooks#session-memory)

### bootstrap-extra-files

Injects additional bootstrap files (for example monorepo-local `AGENTS.md` / `TOOLS.md`) during `agent:bootstrap`.

**Enable:**

```bash
openclaw hooks enable bootstrap-extra-files
```

**See:** [bootstrap-extra-files documentation](/automation/hooks#bootstrap-extra-files)

### command-logger

Logs all command events to a centralized audit file.

**Enable:**

```bash
openclaw hooks enable command-logger
```

**Output:** `~/.openclaw/logs/commands.log`

**View logs:**

```bash
# Recent commands
tail -n 20 ~/.openclaw/logs/commands.log

# Pretty-print
cat ~/.openclaw/logs/commands.log | jq .

# Filter by action
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .
```

**See:** [command-logger documentation](/automation/hooks#command-logger)

### boot-md

Runs `BOOT.md` when the gateway starts (after channels start).

**Events**: `gateway:startup`

**Enable**:

```bash
openclaw hooks enable boot-md
```

**See:** [boot-md documentation](/automation/hooks#boot-md)

## Related

- [CLI reference](/cli)
- [Automation hooks](/automation/hooks)


## openclaw logs Command

# `openclaw logs`

Tail Gateway file logs over RPC (works in remote mode).

Related:

- Logging overview: [Logging](/logging)
- Gateway CLI: [gateway](/cli/gateway)

## Options

- `--limit <n>`: maximum number of log lines to return (default `200`)
- `--max-bytes <n>`: maximum bytes to read from the log file (default `250000`)
- `--follow`: follow the log stream
- `--interval <ms>`: polling interval while following (default `1000`)
- `--json`: emit line-delimited JSON events
- `--plain`: plain text output without styled formatting
- `--no-color`: disable ANSI colors
- `--local-time`: render timestamps in your local timezone

## Shared Gateway RPC options

`openclaw logs` also accepts the standard Gateway client flags:

- `--url <url>`: Gateway WebSocket URL
- `--token <token>`: Gateway token
- `--timeout <ms>`: timeout in ms (default `30000`)
- `--expect-final`: wait for a final response when the Gateway call is agent-backed

When you pass `--url`, the CLI does not auto-apply config or environment credentials. Include `--token` explicitly if the target Gateway requires auth.

## Examples

```bash
openclaw logs
openclaw logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --limit 500
openclaw logs --local-time
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

## Notes

- Use `--local-time` to render timestamps in your local timezone.
- If the implicit local loopback Gateway asks for pairing, closes during connect, or times out before `logs.tail` answers, `openclaw logs` falls back to the configured Gateway file log automatically. Explicit `--url` targets do not use this fallback.
- When using `--follow`, transient gateway disconnects (WebSocket close, timeout, connection drop) trigger automatic reconnection with exponential backoff (up to 8 retries, capped at 30 s between attempts). A warning is printed to stderr on each retry, and a `[logs] gateway reconnected` notice is printed once a poll succeeds. In `--json` mode both the retry warning and the reconnect transition are emitted as `{"type":"notice"}` records on stderr. Non-recoverable errors (auth failure, bad configuration) still exit immediately.

## Related

- [CLI reference](/cli)
- [Gateway logging](/gateway/logging)


## openclaw memory Commands

# `openclaw memory`

Manage semantic memory indexing and search.
Provided by the active memory plugin (default: `memory-core`; set `plugins.slots.memory = "none"` to disable).

Related:

- Memory concept: [Memory](/concepts/memory)
- Memory wiki: [Memory Wiki](/plugins/memory-wiki)
- Wiki CLI: [wiki](/cli/wiki)
- Plugins: [Plugins](/tools/plugin)

## Examples

```bash
openclaw memory status
openclaw memory status --deep
openclaw memory status --fix
openclaw memory index --force
openclaw memory search "meeting notes"
openclaw memory search --query "deployment" --max-results 20
openclaw memory promote --limit 10 --min-score 0.75
openclaw memory promote --apply
openclaw memory promote --json --min-recall-count 0 --min-unique-queries 0
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
openclaw memory rem-harness
openclaw memory rem-harness --json
openclaw memory status --json
openclaw memory status --deep --index
openclaw memory status --deep --index --verbose
openclaw memory status --agent main
openclaw memory index --agent main --verbose
```

## Options

`memory status` and `memory index`:

- `--agent <id>`: scope to a single agent. Without it, these commands run for each configured agent; if no agent list is configured, they fall back to the default agent.
- `--verbose`: emit detailed logs during probes and indexing.

`memory status`:

- `--deep`: probe local vector-store readiness, embedding-provider readiness, and semantic vector-search readiness. Plain `memory status` stays fast and does not run live embedding or provider discovery work; unknown vector-store or semantic-vector state means it was not probed in that command. QMD lexical `searchMode: "search"` skips semantic vector probes and embedding maintenance even with `--deep`.
- `--index`: run a reindex if the store is dirty (implies `--deep`).
- `--fix`: repair stale recall locks and normalize promotion metadata.
- `--json`: print JSON output.

If `memory status` shows `Dreaming status: blocked`, the managed dreaming cron is enabled but the heartbeat that drives it is not firing for the default agent. See [Dreaming never runs](/concepts/dreaming#dreaming-never-runs-status-shows-blocked) for the two common causes.

`memory index`:

- `--force`: force a full reindex.

`memory search`:

- Query input: pass either positional `[query]` or `--query <text>`.
- If both are provided, `--query` wins.
- If neither is provided, the command exits with an error.
- `--agent <id>`: scope to a single agent (default: the default agent).
- `--max-results <n>`: limit the number of results returned.
- `--min-score <n>`: filter out low-score matches.
- `--json`: print JSON results.

`memory promote`:

Preview and apply short-term memory promotions.

```bash
openclaw memory promote [--apply] [--limit <n>] [--include-promoted]
```

- `--apply` -- write promotions to `MEMORY.md` (default: preview only).
- `--limit <n>` -- cap the number of candidates shown.
- `--include-promoted` -- include entries already promoted in previous cycles.

Full options:

- Ranks short-term candidates from `memory/YYYY-MM-DD.md` using weighted promotion signals (`frequency`, `relevance`, `query diversity`, `recency`, `consolidation`, `conceptual richness`).
- Uses short-term signals from both memory recalls and daily-ingestion passes, plus light/REM phase reinforcement signals.
- When dreaming is enabled, `memory-core` auto-manages one cron job that runs a full sweep (`light -> REM -> deep`) in the background (no manual `openclaw cron add` required).
- `--agent <id>`: scope to a single agent (default: the default agent).
- `--limit <n>`: max candidates to return/apply.
- `--min-score <n>`: minimum weighted promotion score.
- `--min-recall-count <n>`: minimum recall count required for a candidate.
- `--min-unique-queries <n>`: minimum distinct query count required for a candidate.
- `--apply`: append selected candidates into `MEMORY.md` and mark them promoted.
- `--include-promoted`: include already promoted candidates in output.
- `--json`: print JSON output.

`memory promote-explain`:

Explain a specific promotion candidate and its score breakdown.

```bash
openclaw memory promote-explain <selector> [--agent <id>] [--include-promoted] [--json]
```

- `<selector>`: candidate key, path fragment, or snippet fragment to look up.
- `--agent <id>`: scope to a single agent (default: the default agent).
- `--include-promoted`: include already promoted candidates.
- `--json`: print JSON output.

`memory rem-harness`:

Preview REM reflections, candidate truths, and deep promotion output without writing anything.

```bash
openclaw memory rem-harness [--agent <id>] [--include-promoted] [--json]
```

- `--agent <id>`: scope to a single agent (default: the default agent).
- `--include-promoted`: include already promoted deep candidates.
- `--json`: print JSON output.

## Dreaming

Dreaming is the background memory consolidation system with three cooperative
phases: **light** (sort/stage short-term material), **deep** (promote durable
facts into `MEMORY.md`), and **REM** (reflect and surface themes).

- Enable with `plugins.entries.memory-core.config.dreaming.enabled: true`.
- Toggle from chat with `/dreaming on|off` (or inspect with `/dreaming status`).
- Dreaming runs on one managed sweep schedule (`dreaming.frequency`) and executes phases in order: light, REM, deep.
- Only the deep phase writes durable memory to `MEMORY.md`.
- Human-readable phase output and diary entries are written to `DREAMS.md` (or existing `dreams.md`), with optional per-phase reports in `memory/dreaming/<phase>/YYYY-MM-DD.md`.
- Ranking uses weighted signals: recall frequency, retrieval relevance, query diversity, temporal recency, cross-day consolidation, and derived concept richness.
- Promotion re-reads the live daily note before writing to `MEMORY.md`, so edited or deleted short-term snippets do not get promoted from stale recall-store snapshots.
- Scheduled and manual `memory promote` runs share the same deep phase defaults unless you pass CLI threshold overrides.
- Automatic runs fan out across configured memory workspaces.

Default scheduling:

- **Sweep cadence**: `dreaming.frequency = 0 3 * * *`
- **Deep thresholds**: `minScore=0.8`, `minRecallCount=3`, `minUniqueQueries=3`, `recencyHalfLifeDays=14`, `maxAgeDays=30`

Example:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Notes:

- `memory index --verbose` prints per-phase details (provider, model, sources, batch activity).
- `memory status` includes any extra paths configured via `memorySearch.extraPaths`.
- If effectively active memory remote API key fields are configured as SecretRefs, the command resolves those values from the active gateway snapshot. If gateway is unavailable, the command fails fast.
- Gateway version skew note: this command path requires a gateway that supports `secrets.resolve`; older gateways return an unknown-method error.
- Tune scheduled sweep cadence with `dreaming.frequency`. Deep promotion policy is otherwise internal; use CLI flags on `memory promote` when you need one-off manual overrides.
- `memory rem-harness --path <file-or-dir> --grounded` previews grounded `What Happened`, `Reflections`, and `Possible Lasting Updates` from historical daily notes without writing anything.
- `memory rem-backfill --path <file-or-dir>` writes reversible grounded diary entries into `DREAMS.md` for UI review.
- `memory rem-backfill --path <file-or-dir> --stage-short-term` also seeds grounded durable candidates into the live short-term promotion store so the normal deep phase can rank them.
- `memory rem-backfill --rollback` removes previously written grounded diary entries, and `memory rem-backfill --rollback-short-term` removes previously staged grounded short-term candidates.
- See [Dreaming](/concepts/dreaming) for full phase descriptions and configuration reference.

## Related

- [CLI reference](/cli)
- [Memory overview](/concepts/memory)


## openclaw node Command

# `openclaw node`

Run a **headless node host** that connects to the Gateway WebSocket and exposes
`system.run` / `system.which` on this machine.

## Why use a node host?

Use a node host when you want agents to **run commands on other machines** in your
network without installing a full macOS companion app there.

Common use cases:

- Run commands on remote Linux/Windows boxes (build servers, lab machines, NAS).
- Keep exec **sandboxed** on the gateway, but delegate approved runs to other hosts.
- Provide a lightweight, headless execution target for automation or CI nodes.

Execution is still guarded by **exec approvals** and per-agent allowlists on the
node host, so you can keep command access scoped and explicit.

## Browser proxy (zero-config)

Node hosts automatically advertise a browser proxy if `browser.enabled` is not
disabled on the node. This lets the agent use browser automation on that node
without extra configuration.

By default, the proxy exposes the node's normal browser profile surface. If you
set `nodeHost.browserProxy.allowProfiles`, the proxy becomes restrictive:
non-allowlisted profile targeting is rejected, and persistent profile
create/delete routes are blocked through the proxy.

Disable it on the node if needed:

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## Run (foreground)

```bash
openclaw node run --host <gateway-host> --port 18789
```

Options:

- `--host <host>`: Gateway WebSocket host (default: `127.0.0.1`)
- `--port <port>`: Gateway WebSocket port (default: `18789`)
- `--tls`: Use TLS for the gateway connection
- `--tls-fingerprint <sha256>`: Expected TLS certificate fingerprint (sha256)
- `--node-id <id>`: Override node id (clears pairing token)
- `--display-name <name>`: Override the node display name

## Gateway auth for node host

`openclaw node run` and `openclaw node install` resolve gateway auth from config/env (no `--token`/`--password` flags on node commands):

- `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` are checked first.
- Then local config fallback: `gateway.auth.token` / `gateway.auth.password`.
- In local mode, node host intentionally does not inherit `gateway.remote.token` / `gateway.remote.password`.
- If `gateway.auth.token` / `gateway.auth.password` is explicitly configured via SecretRef and unresolved, node auth resolution fails closed (no remote fallback masking).
- In `gateway.mode=remote`, remote client fields (`gateway.remote.token` / `gateway.remote.password`) are also eligible per remote precedence rules.
- Node host auth resolution only honors `OPENCLAW_GATEWAY_*` env vars.

For a node connecting to a non-loopback `ws://` Gateway on a trusted private
network, set `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`. Without it, node startup
fails closed and asks you to use `wss://`, an SSH tunnel, or Tailscale.
This is a process-environment opt-in, not an `openclaw.json` config key.
`openclaw node install` persists it into the supervised node service when it is
present in the install command environment.

## Service (background)

Install a headless node host as a user service.

```bash
openclaw node install --host <gateway-host> --port 18789
```

Options:

- `--host <host>`: Gateway WebSocket host (default: `127.0.0.1`)
- `--port <port>`: Gateway WebSocket port (default: `18789`)
- `--tls`: Use TLS for the gateway connection
- `--tls-fingerprint <sha256>`: Expected TLS certificate fingerprint (sha256)
- `--node-id <id>`: Override node id (clears pairing token)
- `--display-name <name>`: Override the node display name
- `--runtime <runtime>`: Service runtime (`node` or `bun`)
- `--force`: Reinstall/overwrite if already installed

Manage the service:

```bash
openclaw node status
openclaw node start
openclaw node stop
openclaw node restart
openclaw node uninstall
```

Use `openclaw node run` for a foreground node host (no service).

Service commands accept `--json` for machine-readable output.

The node host retries Gateway restart and network closes in-process. If the
Gateway reports a terminal token/password/bootstrap auth pause, the node host
logs the close detail and exits non-zero so launchd/systemd can restart it with
fresh config and credentials. Pairing-required pauses stay in the foreground
flow so the pending request can be approved.

## Pairing

The first connection creates a pending device pairing request (`role: node`) on the Gateway.
Approve it via:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

On tightly controlled node networks, the Gateway operator can explicitly opt in
to auto-approving first-time node pairing from trusted CIDRs:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

This is disabled by default. It only applies to fresh `role: node` pairing with
no requested scopes. Operator/browser clients, Control UI, WebChat, and role,
scope, metadata, or public-key upgrades still require manual approval.

If the node retries pairing with changed auth details (role/scopes/public key),
the previous pending request is superseded and a new `requestId` is created.
Run `openclaw devices list` again before approval.

The node host stores its node id, token, display name, and gateway connection info in
`~/.openclaw/node.json`.

## Exec approvals

`system.run` is gated by local exec approvals:

- `~/.openclaw/exec-approvals.json`
- [Exec approvals](/tools/exec-approvals)
- `openclaw approvals --node <id|name|ip>` (edit from the Gateway)

For approved async node exec, OpenClaw prepares a canonical `systemRunPlan`
before prompting. The later approved `system.run` forward reuses that stored
plan, so edits to command/cwd/session fields after the approval request was
created are rejected instead of changing what the node executes.

## Related

- [CLI reference](/cli)
- [Nodes](/nodes)

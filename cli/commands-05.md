---
domain: cli
topic: "CLI Commands: sessions, setup, skills..."
type: reference
keywords:
  - sessions
  - setup
  - skills
  - status
  - system
  - CLI commands
  - openclaw CLI
source: 
  - cli/sessions.md
  - cli/setup.md
  - cli/skills.md
  - cli/status.md
  - cli/system.md
  - cli/tasks.md
  - cli/tui.md
  - cli/uninstall.md
---

## openclaw sessions Command

# `openclaw sessions`

List stored conversation sessions.

Session lists are not channel/provider liveness checks. They show persisted
conversation rows from session stores. A quiet Discord, Slack, Telegram, or
other channel can reconnect successfully without creating a new session row
until a message is processed. Use `openclaw channels status --probe`,
`openclaw status --deep`, or `openclaw health --verbose` when you need live
channel connectivity.

`openclaw sessions` and Gateway `sessions.list` responses are bounded by
default so large long-lived stores cannot monopolize the CLI process or Gateway
event loop. The CLI returns the newest 100 sessions by default; pass
`--limit <n>` for a smaller/larger window or `--limit all` when you intentionally
need the full store. JSON responses include `totalCount`, `limitApplied`, and
`hasMore` when callers need to show that more rows exist.

RPC clients can pass `configuredAgentsOnly: true` to keep the broad combined
discovery source but return only rows for agents currently present in config.
Control UI uses that mode by default so deleted or disk-only agent stores do
not reappear in the Sessions view.

```bash
openclaw sessions
openclaw sessions --agent work
openclaw sessions --all-agents
openclaw sessions --active 120
openclaw sessions --limit 25
openclaw sessions --verbose
openclaw sessions --json
```

Scope selection:

- default: configured default agent store
- `--verbose`: verbose logging
- `--agent <id>`: one configured agent store
- `--all-agents`: aggregate all configured agent stores
- `--store <path>`: explicit store path (cannot be combined with `--agent` or `--all-agents`)
- `--limit <n|all>`: max rows to output (default `100`; `all` restores full output)

Export a trajectory bundle for a stored session:

```bash
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --workspace .
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --output bug-123 --json
```

This is the command path used by the `/export-trajectory` slash command after
the owner approves the exec request. The output directory is always resolved
inside `.openclaw/trajectory-exports/` under the selected workspace.

`openclaw sessions --all-agents` reads configured agent stores. Gateway and ACP
session discovery are broader: they also include disk-only stores found under
the default `agents/` root or a templated `session.store` root. Those
discovered stores must resolve to regular `sessions.json` files inside the
agent root; symlinks and out-of-root paths are skipped.

JSON examples:

`openclaw sessions --all-agents --json`:

```json
{
  "path": null,
  "stores": [
    { "agentId": "main", "path": "/home/user/.openclaw/agents/main/sessions/sessions.json" },
    { "agentId": "work", "path": "/home/user/.openclaw/agents/work/sessions/sessions.json" }
  ],
  "allAgents": true,
  "count": 2,
  "totalCount": 2,
  "limitApplied": 100,
  "hasMore": false,
  "activeMinutes": null,
  "sessions": [
    { "agentId": "main", "key": "agent:main:main", "model": "gpt-5" },
    { "agentId": "work", "key": "agent:work:main", "model": "claude-opus-4-6" }
  ]
}
```

## Cleanup maintenance

Run maintenance now (instead of waiting for the next write cycle):

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --agent work --dry-run
openclaw sessions cleanup --all-agents --dry-run
openclaw sessions cleanup --enforce
openclaw sessions cleanup --enforce --active-key "agent:main:telegram:direct:123"
openclaw sessions cleanup --dry-run --fix-dm-scope
openclaw sessions cleanup --json
```

`openclaw sessions cleanup` uses `session.maintenance` settings from config:

- Scope note: `openclaw sessions cleanup` maintains session stores, transcripts, and trajectory sidecars. It does not prune cron run logs (`cron/runs/<jobId>.jsonl`), which are managed by `cron.runLog.maxBytes` and `cron.runLog.keepLines` in [Cron configuration](/automation/cron-jobs#configuration) and explained in [Cron maintenance](/automation/cron-jobs#maintenance).
- Cleanup also prunes unreferenced primary transcripts, compaction checkpoints, and trajectory sidecars older than `session.maintenance.pruneAfter`; files still referenced by `sessions.json` are preserved.

- `--dry-run`: preview how many entries would be pruned/capped without writing.
  - In text mode, dry-run prints a per-session action table (`Action`, `Key`, `Age`, `Model`, `Flags`) so you can see what would be kept vs removed.
- `--enforce`: apply maintenance even when `session.maintenance.mode` is `warn`.
- `--fix-missing`: remove entries whose transcript files are missing, even if they would not normally age/count out yet.
- `--fix-dm-scope`: when `session.dmScope` is `main`, retire stale peer-keyed direct-DM rows left behind by earlier `per-peer`, `per-channel-peer`, or `per-account-channel-peer` routing. Use `--dry-run` first; applying the cleanup removes those rows from `sessions.json` and preserves their transcripts as deleted archives.
- `--active-key <key>`: protect a specific active key from disk-budget eviction. Durable external conversation pointers, such as group sessions and thread-scoped chat sessions, are also kept by age/count/disk-budget maintenance.
- `--agent <id>`: run cleanup for one configured agent store.
- `--all-agents`: run cleanup for all configured agent stores.
- `--store <path>`: run against a specific `sessions.json` file.
- `--json`: print a JSON summary. With `--all-agents`, output includes one summary per store.

When a Gateway is reachable, non-dry-run cleanup for configured agent stores is
sent through the Gateway so it shares the same session-store writer as runtime
traffic. Use `--store <path>` for explicit offline repair of a store file.

`openclaw sessions cleanup --all-agents --dry-run --json`:

```json
{
  "allAgents": true,
  "mode": "warn",
  "dryRun": true,
  "stores": [
    {
      "agentId": "main",
      "storePath": "/home/user/.openclaw/agents/main/sessions/sessions.json",
      "beforeCount": 120,
      "afterCount": 80,
      "missing": 0,
      "dmScopeRetired": 0,
      "pruned": 40,
      "capped": 0
    },
    {
      "agentId": "work",
      "storePath": "/home/user/.openclaw/agents/work/sessions/sessions.json",
      "beforeCount": 18,
      "afterCount": 18,
      "missing": 0,
      "dmScopeRetired": 0,
      "pruned": 0,
      "capped": 0
    }
  ]
}
```

Related:

- Session config: [Configuration reference](/gateway/config-agents#session)

## Related

- [CLI reference](/cli)
- [Session management](/concepts/session)


## openclaw setup Command

# `openclaw setup`

Initialize the baseline config and agent workspace. With any onboarding flag present, also runs the wizard.

> **Note:** `openclaw setup` is for mutable config installs. In Nix mode (`OPENCLAW_NIX_MODE=1`) OpenClaw refuses setup writes because the config file is managed by Nix. Use the first-party [nix-openclaw Quick Start](https://github.com/openclaw/nix-openclaw#quick-start) or the equivalent source config for another Nix package.


## Options

| Flag                       | Description                                                                                         |
| -------------------------- | --------------------------------------------------------------------------------------------------- |
| `--workspace <dir>`        | Agent workspace directory (default `~/.openclaw/workspace`; stored as `agents.defaults.workspace`). |
| `--wizard`                 | Run interactive onboarding.                                                                         |
| `--non-interactive`        | Run onboarding without prompts.                                                                     |
| `--mode <mode>`            | Onboarding mode: `local` or `remote`.                                                               |
| `--import-from <provider>` | Migration provider to run during onboarding.                                                        |
| `--import-source <path>`   | Source agent home for `--import-from`.                                                              |
| `--import-secrets`         | Import supported secrets during onboarding migration.                                               |
| `--remote-url <url>`       | Remote Gateway WebSocket URL.                                                                       |
| `--remote-token <token>`   | Remote Gateway token (optional).                                                                    |

### Wizard auto-trigger

`openclaw setup` runs the wizard when any of these flags are explicitly present, even without `--wizard`:

`--wizard`, `--non-interactive`, `--mode`, `--import-from`, `--import-source`, `--import-secrets`, `--remote-url`, `--remote-token`.

## Examples

```bash
openclaw setup
openclaw setup --workspace ~/.openclaw/workspace
openclaw setup --wizard
openclaw setup --wizard --import-from hermes --import-source ~/.hermes
openclaw setup --non-interactive --mode remote --remote-url wss://gateway-host:18789 --remote-token <token>
```

## Notes

- Plain `openclaw setup` initializes config and workspace without running the full onboarding flow.
- After plain setup, run `openclaw onboard` for the full guided journey, `openclaw configure` for targeted changes, or `openclaw channels add` to add channel accounts.
- If Hermes state is detected, interactive onboarding can offer migration automatically. Import onboarding requires a fresh setup; use [Migrate](/cli/migrate) for dry-run plans, backups, and overwrite mode outside onboarding.

## Related

- [CLI reference](/cli)
- [Onboarding (CLI)](/start/wizard)
- [Getting started](/start/getting-started)
- [Install overview](/install)


## openclaw skills Command

# `openclaw skills`

Inspect local skills and install/update skills from ClawHub.

Related:

- Skills system: [Skills](/tools/skills)
- Skills config: [Skills config](/tools/skills-config)
- ClawHub installs: [ClawHub](/clawhub/cli)

## Commands

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install <slug>
openclaw skills install <slug> --version <version>
openclaw skills install <slug> --force
openclaw skills install <slug> --agent <id>
openclaw skills update <slug>
openclaw skills update --all
openclaw skills update --all --agent <id>
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills list --agent <id>
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills info <name> --agent <id>
openclaw skills check
openclaw skills check --agent <id>
openclaw skills check --json
```

`search`/`install`/`update` use ClawHub directly and install into the active
workspace `skills/` directory. `list`/`info`/`check` still inspect the local
skills visible to the current workspace and config. Workspace-backed commands
resolve the target workspace from `--agent <id>`, then the current working
directory when it is inside a configured agent workspace, then the default
agent.

This CLI `install` command downloads skill folders from ClawHub. Gateway-backed
skill dependency installs triggered from onboarding or Skills settings use the
separate `skills.install` request path instead.

Notes:

- `search [query...]` accepts an optional query; omit it to browse the default
  ClawHub search feed.
- `search --limit <n>` caps returned results.
- `install --force` overwrites an existing workspace skill folder for the same
  slug.
- `--agent <id>` targets one configured agent workspace and overrides current
  working directory inference.
- `update --all` only updates tracked ClawHub installs in the active workspace.
- `check --agent <id>` checks the selected agent's workspace and reports which
  ready skills are actually visible to that agent's prompt or command surface.
- `list` is the default action when no subcommand is provided.
- `list`, `info`, and `check` write their rendered output to stdout. With
  `--json`, that means the machine-readable payload stays on stdout for pipes
  and scripts.

## Related

- [CLI reference](/cli)
- [Skills](/tools/skills)


## openclaw status Command

Diagnostics for channels + sessions.

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --usage
```

Notes:

- `--deep` runs live probes (WhatsApp Web + Telegram + Discord + Slack + Signal).
- Plain `openclaw status` stays on the fast read-only path and marks memory as `not checked` instead of unavailable when it skips memory inspection. Heavy security audit, plugin compatibility, and memory-vector probes are left to `openclaw status --all`, `openclaw status --deep`, `openclaw security audit`, and `openclaw memory status --deep`.
- `status --json --all` reports memory details from the active memory plugin runtime selected by `plugins.slots.memory`. Custom memory plugins can leave built-in `agents.defaults.memorySearch.enabled` disabled and still report their own files, chunks, vector, and FTS state.
- `--usage` prints normalized provider usage windows as `X% left`.
- Session status output separates `Execution:` from `Runtime:`. `Execution` is the sandbox path (`direct`, `docker/*`), while `Runtime` tells you whether the session is using `OpenClaw Pi Default`, `OpenAI Codex`, a CLI backend, or an ACP backend such as `codex (acp/acpx)`. See [Agent runtimes](/concepts/agent-runtimes) for the provider/model/runtime distinction.
- MiniMax's raw `usage_percent` / `usagePercent` fields are remaining quota, so OpenClaw inverts them before display; count-based fields win when present. `model_remains` responses prefer the chat-model entry, derive the window label from timestamps when needed, and include the model name in the plan label.
- When the current session snapshot is sparse, `/status` can backfill token and cache counters from the most recent transcript usage log. Existing nonzero live values still win over transcript fallback values.
- `/status` includes compact Gateway process uptime and host system uptime.
- Transcript fallback can also recover the active runtime model label when the live session entry is missing it. If that transcript model differs from the selected model, status resolves the context window against the recovered runtime model instead of the selected one.
- For prompt-size accounting, transcript fallback prefers the larger prompt-oriented total when session metadata is missing or smaller, so custom-provider sessions do not collapse to `0` token displays.
- Output includes per-agent session stores when multiple agents are configured.
- Overview includes Gateway + node host service install/runtime status when available.
- Overview includes update channel + git SHA (for source checkouts).
- Update info surfaces in the Overview; if an update is available, status prints a hint to run `openclaw update` (see [Updating](/install/updating)).
- Model pricing refresh failures are shown as optional pricing warnings. They do
  not mean the Gateway or channels are unhealthy.
- Read-only status surfaces (`status`, `status --json`, `status --all`) resolve supported SecretRefs for their targeted config paths when possible.
- If a supported channel SecretRef is configured but unavailable in the current command path, status stays read-only and reports degraded output instead of crashing. Human output shows warnings such as "configured token unavailable in this command path", and JSON output includes `secretDiagnostics`.
- When command-local SecretRef resolution succeeds, status prefers the resolved snapshot and clears transient "secret unavailable" channel markers from the final output.
- `status --all` includes a Secrets overview row and a diagnosis section that summarizes secret diagnostics (truncated for readability) without stopping report generation.

## Related

- [CLI reference](/cli)
- [Doctor](/gateway/doctor)


## openclaw system Command

# `openclaw system`

System-level helpers for the Gateway: enqueue system events, control heartbeats,
and view presence.

All `system` subcommands use Gateway RPC and accept the shared client flags:

- `--url <url>`
- `--token <token>`
- `--timeout <ms>`
- `--expect-final`

## Common commands

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
openclaw system event --text "Check for urgent follow-ups" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

Enqueue a system event on the **main** session by default. The next heartbeat
will inject it as a `System:` line in the prompt. Use `--mode now` to trigger
the heartbeat immediately; `next-heartbeat` waits for the next scheduled tick.

Pass `--session-key` to target a specific session (for example to relay an
async-task completion back to the channel that started it).

> **Timing exception with `--session-key`:** when `--session-key` is supplied,
> `--mode next-heartbeat` collapses to an immediate targeted wake instead of
> waiting for the next scheduled tick. Targeted wakes use heartbeat intent
> `immediate` so they bypass the runner's not-due gate that would otherwise
> defer (and effectively drop) an `event`-intent wake. If you want delayed
> delivery, omit `--session-key` so the event lands on the main session and
> rides the next regular heartbeat.

Flags:

- `--text <text>`: required system event text.
- `--mode <mode>`: `now` or `next-heartbeat` (default).
- `--session-key <sessionKey>`: optional; target a specific agent session
  instead of the agent's main session. Keys that do not belong to the
  resolved agent fall back to the agent's main session.
- `--json`: machine-readable output.
- `--url`, `--token`, `--timeout`, `--expect-final`: shared Gateway RPC flags.

## `system heartbeat last|enable|disable`

Heartbeat controls:

- `last`: show the last heartbeat event.
- `enable`: turn heartbeats back on (use this if they were disabled).
- `disable`: pause heartbeats.

Flags:

- `--json`: machine-readable output.
- `--url`, `--token`, `--timeout`, `--expect-final`: shared Gateway RPC flags.

## `system presence`

List the current system presence entries the Gateway knows about (nodes,
instances, and similar status lines).

Flags:

- `--json`: machine-readable output.
- `--url`, `--token`, `--timeout`, `--expect-final`: shared Gateway RPC flags.

## Notes

- Requires a running Gateway reachable by your current config (local or remote).
- System events are ephemeral and not persisted across restarts.

## Related

- [CLI reference](/cli)


## openclaw tasks Command

Inspect durable background tasks and Task Flow state. With no subcommand,
`openclaw tasks` is equivalent to `openclaw tasks list`.

See [Background Tasks](/automation/tasks) for the lifecycle and delivery model.

## Usage

```bash
openclaw tasks
openclaw tasks list
openclaw tasks list --runtime acp
openclaw tasks list --status running
openclaw tasks show <lookup>
openclaw tasks notify <lookup> state_changes
openclaw tasks cancel <lookup>
openclaw tasks audit
openclaw tasks maintenance
openclaw tasks maintenance --apply
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

## Root Options

- `--json`: output JSON.
- `--runtime <name>`: filter by kind: `subagent`, `acp`, `cron`, or `cli`.
- `--status <name>`: filter by status: `queued`, `running`, `succeeded`, `failed`, `timed_out`, `cancelled`, or `lost`.

## Subcommands

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

Lists tracked background tasks newest first.

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

Shows one task by task ID, run ID, or session key.

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

Changes the notification policy for a running task.

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

Cancels a running background task.

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

Surfaces stale, lost, delivery-failed, or otherwise inconsistent task and Task Flow records. Lost tasks retained until `cleanupAfter` are warnings; expired or unstamped lost tasks are errors.

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

Previews or applies task and Task Flow reconciliation, cleanup stamping, pruning,
and stale cron run session registry cleanup.
For cron tasks, reconciliation uses persisted run logs/job state before marking an
old active task `lost`, so completed cron runs do not become false audit errors
just because the in-memory Gateway runtime state is gone. Offline CLI audit is
not authoritative for the Gateway's process-local cron active-job set. CLI tasks
with a run id/source id are marked `lost` when their live Gateway run context is
gone, even if an old child-session row remains.
When applied, maintenance also prunes `cron:<jobId>:run:<uuid>` session registry
rows older than 7 days while preserving currently running cron jobs and leaving
non-cron session rows untouched.

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

Inspects or cancels durable Task Flow state under the task ledger.

## Related

- [CLI reference](/cli)
- [Background tasks](/automation/tasks)


## openclaw tui (Terminal UI)

# `openclaw tui`

Open the terminal UI connected to the Gateway, or run it in local embedded
mode.

Related:

- TUI guide: [TUI](/web/tui)

## Options

| Flag                  | Default                                   | Description                                                                        |
| --------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------- |
| `--local`             | `false`                                   | Run against the local embedded agent runtime instead of a Gateway.                 |
| `--url <url>`         | `gateway.remote.url` from config          | Gateway WebSocket URL.                                                             |
| `--token <token>`     | (none)                                    | Gateway token if required.                                                         |
| `--password <pass>`   | (none)                                    | Gateway password if required.                                                      |
| `--session <key>`     | `main` (or `global` when scope is global) | Session key. Inside an agent workspace it auto-selects that agent unless prefixed. |
| `--deliver`           | `false`                                   | Deliver assistant replies through configured channels.                             |
| `--thinking <level>`  | (model default)                           | Thinking level override.                                                           |
| `--message <text>`    | (none)                                    | Send an initial message after connecting.                                          |
| `--timeout-ms <ms>`   | `agents.defaults.timeoutSeconds`          | Agent timeout. Invalid values log a warning and are ignored.                       |
| `--history-limit <n>` | `200`                                     | History entries to load on attach.                                                 |

Aliases: `openclaw chat` and `openclaw terminal` invoke the same command with `--local` implied.

Notes:

- `chat` and `terminal` are aliases for `openclaw tui --local`.
- `--local` cannot be combined with `--url`, `--token`, or `--password`.
- `tui` resolves configured gateway auth SecretRefs for token/password auth when possible (`env`/`file`/`exec` providers).
- When launched from inside a configured agent workspace directory, TUI auto-selects that agent for the session key default (unless `--session` is explicitly `agent:<id>:...`).
- Local mode uses the embedded agent runtime directly. Most local tools work, but Gateway-only features are unavailable.
- Local mode adds `/auth [provider]` inside the TUI command surface.
- Plugin approval gates still apply in local mode. Tools that require approval prompt for a decision in the terminal; nothing is silently auto-approved because the Gateway is not involved.

## Examples

```bash
openclaw chat
openclaw tui --local
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
openclaw chat --message "Compare my config to the docs and tell me what to fix"
# when run inside an agent workspace, infers that agent automatically
openclaw tui --session bugfix
```

## Config repair loop

Use local mode when the current config already validates and you want the
embedded agent to inspect it, compare it against the docs, and help repair it
from the same terminal:

If `openclaw config validate` is already failing, use `openclaw configure` or
`openclaw doctor --fix` first. `openclaw chat` does not bypass the invalid-
config guard.

```bash
openclaw chat
```

Then inside the TUI:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

Apply targeted fixes with `openclaw config set` or `openclaw configure`, then
rerun `openclaw config validate`. See [TUI](/web/tui) and [Config](/cli/config).

## Related

- [CLI reference](/cli)
- [TUI](/web/tui)


## openclaw uninstall Command

# `openclaw uninstall`

Uninstall the gateway service + local data (CLI remains).

Options:

- `--service`: remove the gateway service
- `--state`: remove state and config
- `--workspace`: remove workspace directories
- `--app`: remove the macOS app
- `--all`: remove service, state, workspace, and app
- `--yes`: skip confirmation prompts
- `--non-interactive`: disable prompts; requires `--yes`
- `--dry-run`: print actions without removing files

Examples:

```bash
openclaw backup create
openclaw uninstall
openclaw uninstall --service --yes --non-interactive
openclaw uninstall --state --workspace --yes --non-interactive
openclaw uninstall --all --yes
openclaw uninstall --dry-run
```

Notes:

- Run `openclaw backup create` first if you want a restorable snapshot before removing state or workspaces.
- `--all` is shorthand for removing service, state, workspace, and app together.
- `--non-interactive` requires `--yes`.

## Related

- [CLI reference](/cli)
- [Uninstall](/install/uninstall)

---
domain: cli
topic: "Automation CLI: cron, hooks, tasks, and wiki Commands"
type: reference
keywords:
  - openclaw cron
  - openclaw hooks
  - openclaw tasks
  - openclaw wiki
  - automation CLI
related:
  - automation/cron-jobs
  - automation/hooks
  - cli/cli-overview
source:
  - cli/cron.md
  - cli/hooks.md
  - cli/tasks.md
  - cli/wiki.md
---

CLI commands for automation: cron, hooks, tasks, and wiki.

## Cron CLI (openclaw cron)

# `openclaw cron`

Manage cron jobs for the Gateway scheduler.

Run `openclaw cron --help` for the full command surface. See [Cron jobs](/automation/cron-jobs) for the conceptual guide.

## Sessions

`--session` accepts `main`, `isolated`, `current`, or `session:<id>`.

    - `main` binds to the agent's main session.
    - `isolated` creates a fresh transcript and session id for each run.
    - `current` binds to the active session at creation time.
    - `session:<id>` pins to an explicit persistent session key.

    Isolated runs reset ambient conversation context. Channel and group routing, send/queue policy, elevation, origin, and ACP runtime binding are reset for the new run. Safe preferences and explicit user-selected model or auth overrides can carry across runs.

## Delivery

`openclaw cron list` and `openclaw cron show <job-id>` preview the resolved delivery route. For `channel: "last"`, the preview shows whether the route resolved from the main or current session, or will fail closed.

Isolated `cron add` jobs default to `--announce` delivery. Use `--no-deliver` to keep output internal. `--deliver` remains as a deprecated alias for `--announce`.

### Delivery ownership

Isolated cron chat delivery is shared between the agent and the runner:

- The agent can send directly using the `message` tool when a chat route is available.
- `announce` fallback-delivers the final reply only when the agent did not send directly to the resolved target.
- `webhook` posts the finished payload to a URL.
- `none` disables runner fallback delivery.

`--announce` is runner fallback delivery for the final reply. `--no-deliver` disables that fallback but does not remove the agent's `message` tool when a chat route is available.

Reminders created from an active chat preserve the live chat delivery target for fallback announce delivery. Internal session keys may be lowercase; do not use them as a source of truth for case-sensitive provider IDs such as Matrix room IDs.

### Failure delivery

Failure notifications resolve in this order:

1. `delivery.failureDestination` on the job.
2. Global `cron.failureDestination`.
3. The job's primary announce target (when no explicit failure destination is set).

Main-session jobs may only use `delivery.failureDestination` when primary delivery mode is `webhook`. Isolated jobs accept it in all modes.

Note: isolated cron runs treat run-level agent failures as job errors even when
no reply payload is produced, so model/provider failures still increment error
counters and trigger failure notifications.

## Scheduling

### One-shot jobs

`--at <datetime>` schedules a one-shot run. Offset-less datetimes are treated as UTC unless you also pass `--tz <iana>`, which interprets the wall-clock time in the given timezone.

One-shot jobs delete after success by default. Use `--keep-after-run` to preserve them.

### Recurring jobs

Recurring jobs use exponential retry backoff after consecutive errors: 30s, 1m, 5m, 15m, 60m. The schedul

## Hooks CLI (openclaw hooks)

# `openclaw hooks`

Manage agent hooks (event-driven automations for commands like `/new`, `/reset`, and gateway startup).

Running `openclaw hooks` with no subcommand is equivalent to `openclaw hooks list`.

Related:

- Hooks: [Hooks](/automation/hooks)
- Plugin hooks: [Plugin hooks](/plugins/hooks)

## List all hooks

```bash
openclaw hooks list
```

List all discovered hooks from workspace, managed, extra, and bundled directories.
Gateway startup does not load internal hook handlers until at least one internal hook is configured.

**Options:**

- `--eligible`: Show only eligible hooks (requirements met)
- `--json`: Output as JSON
- `-v, --verbose`: Show detailed information including missing requirements

**Example output:**

```
Hooks (4/4 ready)

Ready:
  🚀 boot-md ✓ - Run BOOT.md on gateway startup
  📎 bootstrap-extra-files ✓ - Inject extra workspace bootstrap files during agent bootstrap
  📝 command-logger ✓ - Log all command events to a centralized audit file
  💾 session-memory ✓ - Save session context to memory when /new or /reset command is issued
```

**Example (verbose):**

```bash
openclaw hooks list --verbose
```

Shows missing requirements for ineligible hooks.

**Example (JSON):**

```bash
openclaw hooks list --json
```

Returns structured JSON for programmatic use.

## Get hook information

```bash
openclaw hooks info <name>
```

Show detailed information about a specific hook.

**Arguments:**

- `<name>`: Hook name or hook key (e.g., `session-memory`)

**Options:**

- `--json`: Output as JSON

**Example:**

```bash
openclaw hooks info session-memory
```

**Output:**

```
💾 session-memory ✓ Ready

Save session context to memory when /new or /reset command is issued

Details:
  Source: openclaw-bundled
  Path: /path/to/openclaw/hooks/bundled/session-memory/HOOK.md
  Handler: /path/to/openclaw/hooks/bundled/session-memory/handler.ts
  Homepage: https://docs.openclaw.ai/automation/hooks#session-memory
  Events: command:new, command:reset

Requirements:
  Config: ✓ workspace.dir
```

## Check hooks eligibility

```bash
openclaw hooks check
```

Show summary of hook eligibility status (how many are ready vs. not ready).

**Options:**

- `--json`: Output as JSON

**Example output:**

```
Hooks Status

Total hooks: 4
Ready: 4
Not ready: 0
```

## Enable a Hook

```bash
openclaw hooks enable <name>
```

Enable a specific hook by adding it to your config (`~/.openclaw/openclaw.json` by default).

**Note:** Workspace hooks are disabled by default until enabled here or in config. Hooks managed by plugins show `plugin:<id>` in `openclaw hooks list` and can’t be enabled/disabled here. Enable/disable the plugin instead.

**Arguments:**

- `<name>`: Hook name (e.g., `session-memory`)

**Example:**

```bash
openclaw hooks enable session-memory
```

**Output:**

```
✓ Enabled hook: 💾 session-memory
```

**What it does:**

- Checks if hook exists and is eligible
- Updates `hooks.internal.entries.<name>.enabled = true` in your config
- Saves config

## Tasks CLI (openclaw tasks)

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

Previews or applies task and Task Flow reconciliation, cleanup stamping, and pruning.
For cron tasks, reconciliation uses persisted run logs/job state before marking an
old active task `lost`, so completed cron runs do not become false audit errors

## Wiki CLI (openclaw wiki)

# `openclaw wiki`

Inspect and maintain the `memory-wiki` vault.

Provided by the bundled `memory-wiki` plugin.

Related:

- [Memory Wiki plugin](/plugins/memory-wiki)
- [Memory Overview](/concepts/memory)
- [CLI: memory](/cli/memory)

## What it is for

Use `openclaw wiki` when you want a compiled knowledge vault with:

- wiki-native search and page reads
- provenance-rich syntheses
- contradiction and freshness reports
- bridge imports from the active memory plugin
- optional Obsidian CLI helpers

## Common commands

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "who should I ask about Teams?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha Summary" \
  --body "Short synthesis body" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Still active?"

openclaw wiki bridge import
openclaw wiki unsafe-local import

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## Commands

### `wiki status`

Inspect current vault mode, health, and Obsidian CLI availability.

Use this first when you are unsure whether the vault is initialized, bridge mode
is healthy, or Obsidian integration is available.

When bridge mode is active and configured to read memory artifacts, this command
queries the running Gateway so it sees the same active memory plugin context as
agent/runtime memory.

### `wiki doctor`

Run wiki health checks and surface configuration or vault problems.

When bridge mode is active and configured to read memory artifacts, this command
queries the running Gateway before building the report. Disabled bridge imports
and bridge configs that do not read memory artifacts remain local/offline.

Typical issues include:

- bridge mode enabled without public memory artifacts
- invalid or missing vault layout
- missing external Obsidian CLI when Obsidian mode is expected

### `wiki init`

Create the wiki vault layout and starter pages.

This initializes the root structure, including top-level indexes and cache
directories.

### `wiki ingest <path-or-url>`

Import content into the wiki source layer.

Notes:

- URL ingest is controlled by `ingest.allowUrlIngest`
- imported source pages keep provenance in frontmatter
- auto-compile can run after ingest when enabled

### `wiki compile`

Rebuild indexes, related blocks, dashboards, and compiled digests.

This writes stable machine-facing artifacts under:

- `.openclaw-wiki/cache/agent-digest.json`
- `.openclaw-wiki/cache/claims.jsonl`

If `render.createDashboards` is enabled, compile also refreshes report pages.

### `wiki lint`

Lint the vault and report:

---
domain: cli
topic: "CLI Commands: doctor, docs, flows..."
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

## openclaw doctor Command

# `openclaw doctor`

Health checks + quick fixes for the gateway and channels.

Related:

- Troubleshooting: [Troubleshooting](/gateway/troubleshooting)
- Security audit: [Security](/gateway/security)

## Examples

```bash
openclaw doctor
openclaw doctor --repair
openclaw doctor --deep
openclaw doctor --repair --non-interactive
openclaw doctor --generate-gateway-token
```

For channel-specific permissions, use the channel probes instead of `doctor`:

```bash
openclaw channels capabilities --channel discord --target channel:<channel-id>
openclaw channels status --probe
```

The targeted Discord capabilities probe reports the bot's effective channel permissions; the status probe audits configured Discord channels and voice auto-join targets.

## Options

- `--no-workspace-suggestions`: disable workspace memory/search suggestions
- `--yes`: accept defaults without prompting
- `--repair`: apply recommended non-service repairs without prompting; gateway service installs and rewrites still require interactive confirmation or explicit gateway commands
- `--fix`: alias for `--repair`
- `--force`: apply aggressive repairs, including overwriting custom service config when needed
- `--non-interactive`: run without prompts; safe migrations and non-service repairs only
- `--generate-gateway-token`: generate and configure a gateway token
- `--deep`: scan system services for extra gateway installs and report recent Gateway supervisor restart handoffs

Notes:

- In Nix mode (`OPENCLAW_NIX_MODE=1`), read-only doctor checks still work, but `doctor --fix`, `doctor --repair`, `doctor --yes`, and `doctor --generate-gateway-token` are disabled because `openclaw.json` is immutable. Edit the Nix source for this install instead; for nix-openclaw, use the agent-first [Quick Start](https://github.com/openclaw/nix-openclaw#quick-start).
- Interactive prompts (like keychain/OAuth fixes) only run when stdin is a TTY and `--non-interactive` is **not** set. Headless runs (cron, Telegram, no terminal) will skip prompts.
- Performance: non-interactive `doctor` runs skip eager plugin loading so headless health checks stay fast. Interactive sessions still fully load plugins when a check needs their contribution.
- `--fix` (alias for `--repair`) writes a backup to `~/.openclaw/openclaw.json.bak` and drops unknown config keys, listing each removal.
- `doctor --fix --non-interactive` reports missing or stale gateway service definitions but does not install or rewrite them outside update repair mode. Run `openclaw gateway install` for a missing service, or `openclaw gateway install --force` when you intentionally want to replace the launcher.
- State integrity checks now detect orphan transcript files in the sessions directory. Archiving them as `.deleted.<timestamp>` requires an interactive confirmation; `--fix`, `--yes`, and headless runs leave them in place.
- Doctor also scans `~/.openclaw/cron/jobs.json` (or `cron.store`) for legacy cron job shapes and can rewrite them in place before the scheduler has to auto-normalize them at runtime.
- On Linux, doctor warns when the user's crontab still runs legacy `~/.openclaw/bin/ensure-whatsapp.sh`; that script is no longer maintained and can log false WhatsApp gateway outages when cron lacks the systemd user-bus environment.
- When WhatsApp is enabled, doctor checks for a degraded Gateway event loop with local `openclaw-tui` clients still running. `doctor --fix` stops only verified local TUI clients so WhatsApp replies are not queued behind stale TUI refresh loops.
- Doctor rewrites legacy `openai-codex/*` model refs to canonical `openai/*` refs across primary models, fallbacks, heartbeat/subagent/compaction overrides, hooks, channel model overrides, and stale session route pins. `--fix` moves Codex intent onto provider/model-scoped `agentRuntime.id: "codex"` entries, preserves session auth-profile pins such as `openai-codex:...`, removes stale whole-agent/session runtime pins, and keeps repaired OpenAI agent refs on Codex auth routing instead of direct OpenAI API-key auth.
- Doctor cleans legacy plugin dependency staging state created by older OpenClaw versions and relinks the host `openclaw` package for managed npm plugins that declare it as a peer dependency. It also repairs missing downloadable plugins that are referenced by config, such as `plugins.entries`, configured channels, configured provider/search settings, or configured agent runtimes. During package updates, doctor skips package-manager plugin repair until the package swap is complete; rerun `openclaw doctor --fix` afterward if a configured plugin still needs recovery. If the download fails, doctor reports the install error and preserves the configured plugin entry for the next repair attempt.
- Doctor repairs stale plugin config by removing missing plugin ids from `plugins.allow`/`plugins.deny`/`plugins.entries`, plus matching dangling channel config, heartbeat targets, and channel model overrides when plugin discovery is healthy.
- Doctor quarantines invalid plugin config by disabling the affected `plugins.entries.<id>` entry and removing its invalid `config` payload. Gateway startup already skips only that bad plugin so other plugins and channels can keep running.
- Set `OPENCLAW_SERVICE_REPAIR_POLICY=external` when another supervisor owns the gateway lifecycle. Doctor still reports gateway/service health and applies non-service repairs, but skips service install/start/restart/bootstrap and legacy service cleanup.
- On Linux, doctor ignores inactive extra gateway-like systemd units and does not rewrite command/entrypoint metadata for a running systemd gateway service during repair. Stop the service first or use `openclaw gateway install --force` when you intentionally want to replace the active launcher.
- Doctor auto-migrates legacy flat Talk config (`talk.voiceId`, `talk.modelId`, and friends) into `talk.provider` + `talk.providers.<provider>`.
- Repeat `doctor --fix` runs no longer report/apply Talk normalization when the only difference is object key order.
- Doctor includes a memory-search readiness check and can recommend `openclaw configure --section model` when embedding credentials are missing.
- Doctor warns when no command owner is configured. The command owner is the human operator account allowed to run owner-only commands and approve dangerous actions. DM pairing only lets someone talk to the bot; if you approved a sender before first-owner bootstrap existed, set `commands.ownerAllowFrom` explicitly.
- Doctor warns when Codex-mode agents are configured and personal Codex CLI assets exist in the operator's Codex home. Local Codex app-server launches use isolated per-agent homes, so use `openclaw migrate codex --dry-run` to inventory assets that should be promoted deliberately.
- Doctor removes retired `plugins.entries.codex.config.codexDynamicToolsProfile`; Codex app-server always keeps Codex-native workspace tools native.
- Doctor warns when skills allowed for the default agent are unavailable in the current runtime environment because bins, env vars, config, or OS requirements are missing. `doctor --fix` can disable those unavailable skills with `skills.entries.<skill>.enabled=false`; install/configure the missing requirement instead when you want to keep the skill active.
- If sandbox mode is enabled but Docker is unavailable, doctor reports a high-signal warning with remediation (`install Docker` or `openclaw config set agents.defaults.sandbox.mode off`).
- If legacy sandbox registry files (`~/.openclaw/sandbox/containers.json` or `~/.openclaw/sandbox/browsers.json`) are present, doctor reports them; `openclaw doctor --fix` migrates valid entries into sharded registry directories and quarantines invalid legacy files.
- If `gateway.auth.token`/`gateway.auth.password` are SecretRef-managed and unavailable in the current command path, doctor reports a read-only warning and does not write plaintext fallback credentials.
- If channel SecretRef inspection fails in a fix path, doctor continues and reports a warning instead of exiting early.
- After state-directory migrations, doctor warns when enabled default Telegram or Discord accounts depend on env fallback and `TELEGRAM_BOT_TOKEN` or `DISCORD_BOT_TOKEN` is unavailable to the doctor process.
- Telegram `allowFrom` username auto-resolution (`doctor --fix`) requires a resolvable Telegram token in the current command path. If token inspection is unavailable, doctor reports a warning and skips auto-resolution for that pass.

## macOS: `launchctl` env overrides

If you previously ran `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...` (or `...PASSWORD`), that value overrides your config file and can cause persistent "unauthorized" errors.

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```

## Related

- [CLI reference](/cli)
- [Gateway doctor](/gateway/doctor)


## openclaw docs Command

# `openclaw docs`

Search the live OpenClaw docs index from the terminal. The command shells out to the public Mintlify-hosted docs MCP search endpoint at `https://docs.openclaw.ai/mcp.SearchOpenClaw` and renders the results in your terminal.

## Usage

```bash
openclaw docs                       # print docs entrypoint and example search
openclaw docs <query...>            # search the live docs index
```

Arguments:

| Argument     | Description                                                                        |
| ------------ | ---------------------------------------------------------------------------------- |
| `[query...]` | Free-form search query. Multi-word queries are joined with spaces and sent as one. |

## Examples

```bash
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

With no query, `openclaw docs` prints the docs entrypoint URL plus a sample search command instead of running a search.

## How it works

`openclaw docs` invokes the `mcporter` CLI to call the docs search MCP tool, then parses the `Title: / Link: / Content:` blocks from the tool output into a list of results.

To resolve `mcporter`, OpenClaw checks in order:

1. `mcporter` on `PATH` (used directly if present).
2. `pnpm dlx mcporter ...` if `pnpm` is installed.
3. `npx -y mcporter ...` if `npx` is installed.

If none are available, the command fails with a hint to install `pnpm` (`npm install -g pnpm`).

The search call uses a fixed 30 second timeout. Result snippets are truncated to ~220 characters per entry.

## Output

In a rich (TTY) terminal, results render as a heading followed by a bullet list. Each bullet shows the page title, the linked docs URL, and a short snippet on the next line. Empty results print "No results.".

In non-rich output (piped, `--no-color`, scripts), the same data renders as Markdown:

```markdown
# Docs search: <query>

- [Title](https://docs.openclaw.ai/...) - snippet
- [Title](https://docs.openclaw.ai/...) - snippet
```

## Exit codes

| Code | Meaning                                             |
| ---- | --------------------------------------------------- |
| `0`  | Search succeeded (including zero-result responses). |
| `1`  | The MCP tool call failed; stderr is printed inline. |

## Related

- [CLI reference](/cli)
- [Live docs](https://docs.openclaw.ai)


## openclaw flows Command

# `openclaw tasks flow`

There is no top-level `openclaw flows` command. Durable TaskFlow inspection lives under `openclaw tasks flow`.

## Subcommands

```bash
openclaw tasks flow list   [--json] [--status <name>]
openclaw tasks flow show   <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

| Subcommand | Description                | Arguments / options                                                                   |
| ---------- | -------------------------- | ------------------------------------------------------------------------------------- |
| `list`     | List tracked TaskFlows.    | `--json` machine-readable output; `--status <name>` filter (see status values below). |
| `show`     | Show one TaskFlow.         | `<lookup>` flow id or owner key; `--json` machine-readable output.                    |
| `cancel`   | Cancel a running TaskFlow. | `<lookup>` flow id or owner key.                                                      |

`<lookup>` accepts either a flow id (returned by `list` / `show`) or the flow's owner key (the stable identifier the owning subsystem uses to track the flow).

### Status filter values

`--status` on `list` accepts one of:

`queued`, `running`, `waiting`, `blocked`, `succeeded`, `failed`, `cancelled`, `lost`

## Examples

```bash
openclaw tasks flow list
openclaw tasks flow list --status running
openclaw tasks flow list --json
openclaw tasks flow show flow_abc123
openclaw tasks flow show flow_abc123 --json
openclaw tasks flow cancel flow_abc123
```

For full TaskFlow concepts and authoring see [TaskFlow](/automation/taskflow). For the parent `tasks` command see [tasks CLI reference](/cli/tasks).

## Related

- [CLI reference](/cli)
- [Automation](/automation)
- [TaskFlow](/automation/taskflow)


## openclaw health Command

# `openclaw health`

Fetch health from the running Gateway.

## Options

| Flag             | Default | Description                                                        |
| ---------------- | ------- | ------------------------------------------------------------------ |
| `--json`         | `false` | Print machine-readable JSON instead of text.                       |
| `--timeout <ms>` | `10000` | Connection timeout in milliseconds.                                |
| `--verbose`      | `false` | Verbose logging. Forces a live probe and expands per-agent output. |
| `--debug`        | `false` | Alias for `--verbose`.                                             |

Examples:

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

Notes:

- Default `openclaw health` asks the running gateway for its health snapshot. When the
  gateway already has a fresh cached snapshot, it can return that cached payload and
  refresh in the background.
- `--verbose` forces a live probe, prints gateway connection details, and expands the
  human-readable output across all configured accounts and agents.
- Output includes per-agent session stores when multiple agents are configured.

## Related

- [CLI reference](/cli)
- [Gateway health](/gateway/health)


## openclaw hooks Command

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

**Note:** Workspace hooks are disabled by default until enabled here or in config. Hooks managed by plugins show `plugin:<id>` in `openclaw hooks list` and can't be enabled/disabled here. Enable/disable the plugin instead.

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
- Saves config to disk

If the hook came from `<workspace>/hooks/`, this opt-in step is required before
the Gateway will load it.

**After enabling:**

- Restart the gateway so hooks reload (menu bar app restart on macOS, or restart your gateway process in dev).

## Disable a Hook

```bash
openclaw hooks disable <name>
```

Disable a specific hook by updating your config.

**Arguments:**

- `<name>`: Hook name (e.g., `command-logger`)

**Example:**

```bash
openclaw hooks disable command-logger
```

**Output:**

```
⏸ Disabled hook: 📝 command-logger
```

**After disabling:**

- Restart the gateway so hooks reload

## Notes

- `openclaw hooks list --json`, `info --json`, and `check --json` write structured JSON directly to stdout.
- Plugin-managed hooks cannot be enabled or disabled here; enable or disable the owning plugin instead.

## Install hook packs

```bash
openclaw plugins install <package>        # npm by default
openclaw plugins install npm:<package>    # npm only
openclaw plugins install <package> --pin  # pin version
openclaw plugins install <path>           # local path
```

Install hook packs through the unified plugins installer.

`openclaw hooks install` still works as a compatibility alias, but it prints a
deprecation warning and forwards to `openclaw plugins install`.

Npm specs are **registry-only** (package name + optional **exact version** or
**dist-tag**). Git/URL/file specs and semver ranges are rejected. Dependency
installs run project-local with `--ignore-scripts` for safety, even when your
shell has global npm install settings.

Bare specs and `@latest` stay on the stable track. If npm resolves either of
those to a prerelease, OpenClaw stops and asks you to opt in explicitly with a
prerelease tag such as `@beta`/`@rc` or an exact prerelease version.

**What it does:**

- Copies the hook pack into `~/.openclaw/hooks/<id>`
- Enables the installed hooks in `hooks.internal.entries.*`
- Records the install under `hooks.internal.installs`

**Options:**

- `-l, --link`: Link a local directory instead of copying (adds it to `hooks.internal.load.extraDirs`)
- `--pin`: Record npm installs as exact resolved `name@version` in `hooks.internal.installs`

**Supported archives:** `.zip`, `.tgz`, `.tar.gz`, `.tar`

**Examples:**

```bash
# Local directory
openclaw plugins install ./my-hook-pack

# Local archive
openclaw plugins install ./my-hook-pack.zip

# NPM package
openclaw plugins install @openclaw/my-hook-pack

# Link a local directory without copying
openclaw plugins install -l ./my-hook-pack
```

Linked hook packs are treated as managed hooks from an operator-configured
directory, not as workspace hooks.

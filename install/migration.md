---
domain: install
topic: "Migration: Moving to OpenClaw from Claude, Hermes, and Other Platforms"
type: procedure
keywords:
  - migration
  - migrating
  - migrate from Claude
  - migrate from Hermes
  - import sessions
related:
  - install/installation-methods
  - install/updating
source:
  - install/migrating.md
  - install/migrating-claude.md
  - install/migrating-hermes.md
---

Migrating to OpenClaw from other platforms.

## General Migration

OpenClaw supports three migration paths: importing from another agent system, moving an existing install to a new machine, and upgrading a plugin in place.

## Import from another agent system

Use the bundled migration providers to bring instructions, MCP servers, skills, model config, and (opt-in) API keys into OpenClaw. Plans are previewed before any change, secrets are redacted in reports, and apply is backed by a verified backup.

    Import Claude Code and Claude Desktop state, including `CLAUDE.md`, MCP servers, skills, and project commands.

    Import Hermes config, providers, MCP servers, memory, skills, and supported `.env` keys.

The CLI entry point is [`openclaw migrate`](/cli/migrate). Onboarding can also offer migration when it detects a known source (`openclaw onboard --flow import`).

## Move OpenClaw to a new machine

Copy the **state directory** (`~/.openclaw/` by default) and your **workspace** to preserve:

- **Config** — `openclaw.json` and all gateway settings.
- **Auth** — per-agent `auth-profiles.json` (API keys plus OAuth), plus any channel or provider state under `credentials/`.
- **Sessions** — conversation history and agent state.
- **Channel state** — WhatsApp login, Telegram session, and similar.
- **Workspace files** — `MEMORY.md`, `USER.md`, skills, and prompts.

Run `openclaw status` on the old machine to confirm your state directory path. Custom profiles use `~/.openclaw-<profile>/` or a path set via `OPENCLAW_STATE_DIR`.

### Migration steps

    On the **old** machine, stop the gateway so files are not changing mid-copy, then archive:

    ```bash
    openclaw gateway stop
    cd ~
    tar -czf openclaw-state.tgz .openclaw
    ```

    If you use multiple profiles (for example `~/.openclaw-work`), archive each separately.

    [Install](/install) the CLI (and Node if needed) on the new machine. It is fine if onboarding creates a fresh `~/.openclaw/`. You will overwrite it next.

    Transfer the archive via `scp`, `rsync -a`, or an external drive, then extract:

    ```bash
    cd ~
    tar -xzf openclaw-state.tgz
    ```

    Ensure hidden directories were included and file ownership matches the user that will run the gateway.

    On the new machine, run [Doctor](/gateway/doctor) to apply config migrations and repair services:

    ```bash
    openclaw doctor
    openclaw gateway restart
    openclaw status
    ```

### Common pitfalls

    If the old gateway used `--profile` or `OPENCLAW_STATE_DIR` and the new one does not, channels will appear logged out and sessions will be empty. Launch the gateway with the **same** profile or state-dir you migrated, then rerun `openclaw doctor`.

    The config file alone is not enough. Model auth profiles live under `agents/<agentId>/agent/auth-profiles.json`, and channel and provider state lives under `credentials/`. Always migrate the **entire** state directory.

    If you copied as root or switched users, the gateway may fail to read credentials. Ensure the state d

## Migrating from Claude (Anthropic)

OpenClaw imports local Claude state through the bundled Claude migration provider. The provider previews every item before changing state, redacts secrets in plans and reports, and creates a verified backup before apply.

Onboarding imports require a fresh OpenClaw setup. If you already have local OpenClaw state, reset config, credentials, sessions, and the workspace first, or use `openclaw migrate` directly with `--overwrite` after reviewing the plan.

## Two ways to import

    The wizard offers Claude when it detects local Claude state.

    ```bash
    openclaw onboard --flow import
    ```

    Or point at a specific source:

    ```bash
    openclaw onboard --import-from claude --import-source ~/.claude
    ```

    Use `openclaw migrate` for scripted or repeatable runs. See [`openclaw migrate`](/cli/migrate) for the full reference.

    ```bash
    openclaw migrate claude --dry-run
    openclaw migrate apply claude --yes
    ```

    Add `--from <path>` to import a specific Claude Code home or project root.

## What gets imported

    - Project `CLAUDE.md` and `.claude/CLAUDE.md` content is copied or appended into the OpenClaw agent workspace `AGENTS.md`.
    - User `~/.claude/CLAUDE.md` content is appended into workspace `USER.md`.

    MCP server definitions are imported from project `.mcp.json`, Claude Code `~/.claude.json`, and Claude Desktop `claude_desktop_config.json` when present.

    - Claude skills with a `SKILL.md` file are copied into the OpenClaw workspace skills directory.
    - Claude command Markdown files under `.claude/commands/` or `~/.claude/commands/` are converted into OpenClaw skills with `disable-model-invocation: true`.

## What stays archive-only

The provider copies these into the migration report for manual review, but does **not** load them into live OpenClaw config:

- Claude hooks
- Claude permissions and broad tool allowlists
- Claude environment defaults
- `CLAUDE.local.md`
- `.claude/rules/`
- Claude subagents under `.claude/agents/` or `~/.claude/agents/`
- Claude Code caches, plans, and project history directories
- Claude Desktop extensions and OS-stored credentials

OpenClaw refuses to execute hooks, trust permission allowlists, or decode opaque OAuth and Desktop credential state automatically. Move what you need by hand after reviewing the archive.

## Source selection

Without `--from`, OpenClaw inspects the default Claude Code home at `~/.claude`, the sampled Claude Code `~/.claude.json` state file, and the Claude Desktop MCP config on macOS.

When `--from` points at a project root, OpenClaw imports only that project's Claude files such as `CLAUDE.md`, `.claude/settings.json`, `.claude/commands/`, `.claude/skills/`, and `.mcp.json`. It does not read your global Claude home during a project-root import.

## Recommended flow

    ```bash
    openclaw migrate claude --dry-run
    ```

    The plan lists everything that will change, including conflicts, skipped items, and sensitive values redacted from

## Migrating from Hermes

OpenClaw imports Hermes state through a bundled migration provider. The provider previews everything before changing state, redacts secrets in plans and reports, and creates a verified backup before apply.

Imports require a fresh OpenClaw setup. If you already have local OpenClaw state, reset config, credentials, sessions, and the workspace first, or use `openclaw migrate` directly with `--overwrite` after reviewing the plan.

## Two ways to import

    The fastest path. The wizard detects Hermes at `~/.hermes` and shows a preview before applying.

    ```bash
    openclaw onboard --flow import
    ```

    Or point at a specific source:

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

    Use `openclaw migrate` for scripted or repeatable runs. See [`openclaw migrate`](/cli/migrate) for the full reference.

    ```bash
    openclaw migrate hermes --dry-run    # preview only
    openclaw migrate apply hermes --yes  # apply with confirmation skipped
    ```

    Add `--from <path>` when Hermes lives outside `~/.hermes`.

## What gets imported

    - Default model selection from Hermes `config.yaml`.
    - Configured model providers and custom OpenAI-compatible endpoints from `providers` and `custom_providers`.

    MCP server definitions from `mcp_servers` or `mcp.servers`.

    - `SOUL.md` and `AGENTS.md` are copied into the OpenClaw agent workspace.
    - `memories/MEMORY.md` and `memories/USER.md` are **appended** to the matching OpenClaw memory files instead of overwriting them.

    Memory config defaults for OpenClaw file memory. External memory providers such as Honcho are recorded as archive or manual-review items so you can move them deliberately.

    Skills with a `SKILL.md` file under `skills/<name>/` are copied, along with per-skill config values from `skills.config`.

    Set `--include-secrets` to import supported `.env` keys: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY`, `GOOGLE_API_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `DEEPSEEK_API_KEY`. Without the flag, secrets are never copied.

## What stays archive-only

The provider copies these into the migration report directory for manual review, but does **not** load them into live OpenClaw config or credentials:

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `auth.json`
- `state.db`

OpenClaw refuses to execute or trust this state automatically because the formats and trust assumptions can drift between systems. Move what you need by hand after reviewing the archive.

## Recommended flow

    ```bash
    openclaw migrate hermes --dry-run
    ```

    The plan lists everything that will change, including conflicts, skipped items, and any sensitive items. Plan output redacts nested secret-looking keys.

    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw creates and verifies a backup before applying. If you need API keys imported, add `--include-secrets`.

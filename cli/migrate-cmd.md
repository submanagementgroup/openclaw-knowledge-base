---
domain: cli
topic: "openclaw migrate: Import State from Another Agent System (Hermes, Claude)"
type: procedure
keywords:
  - openclaw migrate
  - migrate from hermes
  - migrate from claude
  - migration provider
  - migration dry-run
  - migration plan
  - migration apply
  - import secrets
  - backup migration
source: cli/migrate.md
related:
  - cli/cli-overview
  - install/migration
  - getting-started/onboarding-wizard
---

`openclaw migrate` imports state from another agent system through a plugin-owned migration provider. Bundled providers cover Claude and Hermes; third-party plugins can register additional providers.

## Commands

```bash
# List available migration providers
openclaw migrate list

# Dry-run (build plan, no changes)
openclaw migrate claude --dry-run
openclaw migrate hermes --dry-run

# Apply migration
openclaw migrate apply claude --yes
openclaw migrate apply hermes --yes
openclaw migrate apply hermes --include-secrets --yes

# Via onboarding
openclaw onboard --flow import
openclaw onboard --import-from claude --import-source ~/.claude
openclaw onboard --import-from hermes --import-source ~/.hermes
```

## Key Flags

| Flag | Description |
|------|-------------|
| `--dry-run` | Build the plan and exit without changing state |
| `--from <path>` | Override the source state directory |
| `--include-secrets` | Import supported credentials (off by default) |
| `--overwrite` | Allow apply to replace existing targets when conflicts exist |
| `--yes` | Skip confirmation prompt (required in non-interactive mode) |
| `--no-backup` | Skip pre-apply backup (requires `--force`) |
| `--force` | Required alongside `--no-backup` when apply would refuse |
| `--json` | Print plan or apply result as JSON |

## Safety Model

`openclaw migrate` is preview-first:

1. **Preview before apply**: always shows an itemized plan before changes. JSON plans and apply output redact nested secret-looking keys (API keys, tokens, auth headers, cookies, passwords).
2. **Backups**: apply creates and verifies an OpenClaw backup before applying. Skip with `--no-backup --force`.
3. **Conflicts**: apply refuses when the plan has conflicts. Review the plan, then rerun with `--overwrite` if replacing existing targets is intentional.
4. **Secrets**: never imported by default. Use `--include-secrets` to import supported credentials.

## Claude Provider

Detects Claude Code state at `~/.claude` by default. Use `--from <path>` to specify a different Claude Code home.

**What Claude imports:**
- Project `CLAUDE.md` and `.claude/CLAUDE.md` into the OpenClaw agent workspace
- User `~/.claude/CLAUDE.md` appended to workspace `USER.md`
- MCP server definitions from `.mcp.json`, `~/.claude.json`, and Claude Desktop config
- Claude skill directories that include `SKILL.md`
- Claude command Markdown files converted into OpenClaw skills

## Hermes Provider

Detects state at `~/.hermes` by default.

**What Hermes imports:**
- Default model configuration from `config.yaml`
- Model providers and custom OpenAI-compatible endpoints
- MCP server definitions
- `SOUL.md` and `AGENTS.md` into the workspace
- `memories/MEMORY.md` and `memories/USER.md` appended to memory files
- Skills that include a `SKILL.md` file
- Supported API keys from `.env` (only with `--include-secrets`)

**Supported `.env` keys:** `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY`, `GOOGLE_API_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `DEEPSEEK_API_KEY`

## After Applying

```bash
openclaw doctor
```

## Related

- [Migration guide](/install/migration)
- [Onboarding wizard](/getting-started/onboarding-wizard)
- [Gateway health](/gateway/health-diagnostics-logging)

---
domain: cli
topic: "OpenClaw CLI Overview: All Commands and Quick Reference"
type: reference
keywords:
  - CLI
  - openclaw CLI
  - command line
  - CLI reference
  - openclaw commands
related:
  - gateway/gateway-runbook
  - getting-started/quickstart
source: cli/index.md
---

OpenClaw CLI reference. The `openclaw` command provides subcommands for all Gateway operations, agent runs, channel management, and configuration.

## Quick Reference

```bash
openclaw --help              # all commands
openclaw <command> --help    # command-specific help
openclaw gateway             # gateway management
openclaw agent               # run an agent turn
openclaw channels            # channel management
openclaw plugins             # plugin management
openclaw cron                # cron job management
openclaw config              # configuration
openclaw memory              # memory search
openclaw models              # list models
openclaw logs                # view logs
openclaw status              # system status
openclaw doctor              # diagnostics
openclaw onboard             # setup wizard
```

`openclaw` is the main CLI entry point. Each core command has either a
dedicated reference page or is documented with the command it aliases; this
index lists the commands, the global flags, and the output styling rules that
apply across the CLI.

## Command pages

| Area                 | Commands                                                                                                                                                                                                                                  |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Setup and onboarding | [`crestodian`](/cli/crestodian) · [`setup`](/cli/setup) · [`onboard`](/cli/onboard) · [`configure`](/cli/configure) · [`config`](/cli/config) · [`completion`](/cli/completion) · [`doctor`](/cli/doctor) · [`dashboard`](/cli/dashboard) |
| Reset and uninstall  | [`backup`](/cli/backup) · [`reset`](/cli/reset) · [`uninstall`](/cli/uninstall) · [`update`](/cli/update)                                                                                                                                 |
| Messaging and agents | [`message`](/cli/message) · [`agent`](/cli/agent) · [`agents`](/cli/agents) · [`acp`](/cli/acp) · [`mcp`](/cli/mcp)                                                                                                                       |
| Health and sessions  | [`status`](/cli/status) · [`health`](/cli/health) · [`sessions`](/cli/sessions)                                                                                                                                                           |
| Gateway and logs     | [`gateway`](/cli/gateway) · [`logs`](/cli/logs) · [`system`](/cli/system)                                                                                                                                                                 |
| Models and inference | [`models`](/cli/models) · [`infer`](/cli/infer) · `capability` (alias for [`infer`](/cli/infer)) · [`memory`](/cli/memory) · [`commitments`](/cli/commitments) · [`wiki`](/cli/wiki)                                                      |
| Network and nodes    | [`directory`](/cli/directory) · [`nodes`](/cli/nodes) · [`devices`](/cli/devices) · [`node`](/cli/node)                                                                                                                                   |
| Runtime and sandbox  | [`approvals`](/cli/approvals) · `exec-policy` (see [`approvals`](/cli/approvals)) · [`sandbox`](/cli/sandbox) · [`tui`](/cli/tui) · `chat`/`terminal` (aliases for [`tui --local`](/cli/tui)) · [`browser`](/cli/browser)                 |
| Automation           | [`cron`](/cli/cron) · [`tasks`](/cli/tasks) · [`hooks`](/cli/hooks) · [`webhooks`](/cli/webhooks)                                                                                                                                         |
| Discovery and docs   | [`dns`](/cli/dns) · [`docs`](/cli/docs)                                                                                                                                                                                                   |
| Pairing and channels | [`pairing`](/cli/pairing) · [`qr`](/cli/qr) · [`channels`](/cli/channels)                                                                                                                                                                 |
| Security and plugins | [`security`](/cli/security) · [`secrets`](/cli/secrets) · [`skills`](/cli/skills) · [`plugins`](/cli/plugins) · [`proxy`](/cli/proxy)                                                                                                     |
| Legacy aliases       | [`daemon`](/cli/daemon) (gateway service) · [`clawbot`](/cli/clawbot) (namespace)                                                                                                                                                         |
| Plugins (optional)   | [`voicecall`](/cli/voicecall) (if installed)                                                                                                                                                                                              |

## Global flags

| Flag                    | Purpose                                                               |
| ----------------------- | --------------------------------------------------------------------- |
| `--dev`                 | Isolate state under `~/.openclaw-dev` and shift default ports         |
| `--profile <name>`      | Isolate state under `~/.openclaw-<name>`                              |
| `--container <name>`    | Target a named container for execution                                |
| `--no-color`            | Disable ANSI

---
domain: cli
topic: "Crestodian CLI: Auth Profile and Credential Management"
type: reference
keywords:
  - crestodian
  - auth profiles
  - credential management
  - openclaw crestodian
related:
  - concepts/oauth-auth-profiles
  - gateway/secrets
  - cli/cli-overview
source: cli/crestodian.md
---

Crestodian CLI for managing auth profiles and credentials.

# `openclaw crestodian`

Crestodian is OpenClaw's local setup, repair, and configuration helper. It is
designed to stay reachable when the normal agent path is broken.

Running `openclaw` with no command starts Crestodian in an interactive terminal.
Running `openclaw crestodian` starts the same helper explicitly.

## What Crestodian shows

On startup, interactive Crestodian opens the same TUI shell used by
`openclaw tui`, with a Crestodian chat backend. The chat log starts with a short
greeting:

- when to start Crestodian
- the model or deterministic planner path Crestodian is actually using
- config validity and the default agent
- Gateway reachability from the first startup probe
- the next debug action Crestodian can take

It does not dump secrets or load plugin CLI commands just to start. The TUI
still provides the normal header, chat log, status line, footer, autocomplete,
and editor controls.

Use `status` for the detailed inventory with config path, docs/source paths,
local CLI probes, API-key presence, agents, model, and Gateway details.

Crestodian uses the same OpenClaw reference discovery as regular agents. In a Git checkout,
it points itself at local `docs/` and the local source tree. In an npm package install, it
uses the bundled package docs and links to
[https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw), with explicit
guidance to review source whenever the docs are not enough.

## Examples

```bash
openclaw
openclaw crestodian
openclaw crestodian --json
openclaw crestodian --message "models"
openclaw crestodian --message "validate config"
openclaw crestodian --message "setup workspace ~/Projects/work model openai/gpt-5.5" --yes
openclaw crestodian --message "set default model openai/gpt-5.5" --yes
openclaw onboard --modern
```

Inside the Crestodian TUI:

```text
status
health
doctor
doctor fix
validate config
setup
setup workspace ~/Projects/work model openai/gpt-5.5
config set gateway.port 19001
config set-ref gateway.auth.token env OPENCLAW_GATEWAY_TOKEN
gateway status
restart gateway
agents
create agent work workspace ~/Projects/work
models
set default model openai/gpt-5.5
talk to work agent
talk to agent for ~/Projects/work
audit
quit
```

## Safe startup

Crestodian's startup path is deliberately small. It can run when:

- `openclaw.json` is missing
- `openclaw.json` is invalid
- the Gateway is down
- plugin command registration is unavailable
- no agent has been configured yet

`openclaw --help` and `openclaw --version` still use the normal fast paths.
Noninteractive `openclaw` exits with a short message instead of printing root
help, because the no-command product is Crestodian.

## Operations and approval

Crestodian uses typed operations instead of editing config ad hoc.

Read-only operations can run immediately:

- show overview
- list agents
- show model/backend status
- run status or health checks
- check Gateway reachability
- run doctor without interactive fixes
- validate config
- show the audit-log path

Persistent operations require conversational approval in interactive mode unless
you pass `--yes` for a direct command:

- write config
- run `config set`
- set supported SecretRef values through `config set-ref`
- run setup/onboarding bootstrap
- change the default model
- start, stop, or restart the Gateway
- create agents
- run doctor repairs that rewrite config or state

Applied writes are recorded in:

```text
~/.openclaw/audit/crestodian.jsonl
```

Discovery is not audited. Only applied operations and writes are logged.

`openclaw onboard --modern` starts Crestodian as the modern onboarding preview.
Plain `openclaw onboard` still runs classic onboarding.

## Setup bootstrap

`setup` is the chat-first onboarding bootstrap. It writes only through typed
config operations and asks for approval first.

```text
setup
setup workspace ~/Projects/work
setup workspace ~/Projects/work model openai/gpt-5.5
```

When no model is configured, setup selects the first usable backend in this
order and tells you what it chose:

- existing explicit model, if already configured
- `OPENAI_API_KEY` -> `openai/gpt-5.5`
- `ANTHROPIC_API_KEY` -> `anthropic/claude-opus-4-7`
- Claude Code CLI -> `claude-cli/claude-opus-4-7`
- Codex CLI -> `codex-cli/gpt-5.5`

If none are available, setup still writes the default workspace and leaves the
model unset. Install or log into Codex/Claude Code, or expose
`OPENAI_API_KEY`/`ANTHROPIC_API_KEY`, then run setup again.

## Model-Assisted Planner

Crestodian always starts in deterministic mode. For fuzzy commands that the
deterministic parser does not understand, local Crestodian can make one bounded
planner turn through OpenClaw's normal runtime paths. It first uses the
configured OpenClaw model. If no configured model is usable yet, it can fall
back to local runtimes already present on the machine:

- Claude Code CLI: `claude-cli/claude-opus-4-7`
- Codex app-server harness: `openai/gpt-5.5` with `agentRuntime.id: "codex"`
- Codex CLI: `codex-cli/gpt-5.5`

The model-assisted planner cannot mutate config directly. It must translate the
request into one of Crestodian's typed commands, then the normal approval and
audit rules apply. Crestodian prints the model it used and the interpreted
command before it runs anything. Configless fallback planner turns are
temporary, tool-disabled where the runtime supports it, and use a temporary
workspace/session.

Message-channel rescue mode does not use the model-assisted planner. Remote
rescue stays deterministic so a broken or compromised normal agent path cannot
be used as a config editor.

## Switching to an agent

Use a natural-language selector to leave Crestodian and open the normal TUI:

```text
talk to agent
talk to work agent
switch to main agent
```

`openclaw tui`, `openclaw chat`, and `openclaw terminal` still open the normal
agent TUI directly. They do not start Crestodian.

After switching into the normal TUI, use `/crestodian` to return to Crestodian.
You can include a follow-up request:

```text
/crestodian
/crestodian restart gateway
```

Agent switches inside the TUI leave a breadcrumb that `/crestodian` is available.

## Message rescue mode

Message rescue mode is the message-channel entrypoint for Crestodian. It is for
the case where your normal agent is dead, but a trusted channel such as WhatsApp
still receives commands.

Supported text command:

- `/crestodian <request>`

Operator flow:

```text
You, in a trusted owner DM: /crestodian status
OpenClaw: Crestodian rescue mode. Gateway reachable: no. Config valid: no.
You: /crestodian restart gateway
OpenClaw: Plan: restart the Gateway. Reply /crestodian yes to apply.
You: /crestodian yes
OpenClaw: Applied. Audit entry written.
```

Agent creation can also be queued from the local prompt or rescue mode:

```text
create agent work workspace ~/Projects/work model openai/gpt-5.5
/crestodian create agent work workspace ~/Projects/work
```

Remote rescue mode is an admin surface. It must be tr

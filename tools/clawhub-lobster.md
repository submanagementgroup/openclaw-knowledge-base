---
domain: tools
topic: "ClawHub and Lobster: Tool Discovery and Multi-Agent Sandbox Tool Framework"
type: reference
keywords:
  - ClawHub
  - Lobster
  - tool discovery
  - multi-agent sandbox
  - sandbox tools
related:
  - tools/tools-overview
  - tools/acp-agents
source:
  - tools/clawhub.md
  - tools/lobster.md
  - tools/multi-agent-sandbox-tools.md
---

ClawHub and Lobster: tool discovery and the Lobster multi-agent sandbox tool framework.

## ClawHub

ClawHub is the public registry for **OpenClaw skills and plugins**.

- Use native `openclaw` commands to search, install, and update skills, and to install plugins from ClawHub.
- Use the separate `clawhub` CLI for registry auth, publish, delete/undelete, and sync workflows.

Site: [clawhub.ai](https://clawhub.ai)

## Quick start

    ```bash
    openclaw skills search "calendar"
    ```

    ```bash
    openclaw skills install <skill-slug>
    ```

    Start a new OpenClaw session — it picks up the new skill.

    For registry-authenticated workflows (publish, sync, manage), install
    the separate `clawhub` CLI:

    ```bash
    npm i -g clawhub
    # or
    pnpm add -g clawhub
    ```

## Native OpenClaw flows

    ```bash
    openclaw skills search "calendar"
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Native `openclaw` commands install into your active workspace and
    persist source metadata so later `update` calls can stay on ClawHub.

    ```bash
    openclaw plugins install clawhub:<package>
    openclaw plugins update --all
    ```

    Bare npm-safe plugin specs are also tried against ClawHub before npm:

    ```bash
    openclaw plugins install openclaw-codex-app-server
    ```

    Use `npm:<package>` when you want npm-only resolution without a
    ClawHub lookup:

    ```bash
    openclaw plugins install npm:openclaw-codex-app-server
    ```

    Plugin installs validate advertised `pluginApi` and
    `minGatewayVersion` compatibility before archive install runs, so
    incompatible hosts fail closed early instead of partially installing
    the package.

`openclaw plugins install clawhub:...` only accepts installable plugin
families. If a ClawHub package is actually a skill, OpenClaw stops and
points you at `openclaw skills install <slug>` instead.

Anonymous ClawHub plugin installs also fail closed for private packages.
Community or other non-official channels can still install, but OpenClaw
warns so operators can review source and verification before enabling
them.

## What ClawHub is

- A public registry for OpenClaw skills and plugins.
- A versioned store of skill bundles and metadata.
- A discovery surface for search, tags, and usage signals.

A typical skill is a versioned bundle of files that includes:

- A `SKILL.md` file with the primary description and usage.
- Optional configs, scripts, or supporting files used by the skill.
- Metadata such as tags, summary, and install requirements.

ClawHub uses metadata to power discovery and safely expose skill
capabilities. The registry tracks usage signals (stars, downloads) to
improve ranking and visibility. Each publish creates a new semver
version, and the registry keeps version history so users can audit
changes.

## Workspace and skill loading

The separate `clawhub` CLI also installs skills into `./skills` under
your current working directory. If an OpenClaw workspace is configured,
`clawhub` falls back to that workspace unless you override `--workdir`
(or `CLAWHUB_WORKDIR`). OpenClaw loads workspace skills from
`<workspace>/skills` and picks them up in the **next** session.

If you already use `~/.openclaw/skills` or bundled skills, workspace
skills take precedence. For more detail on how skills are loaded,
shared, and gated, see [Skills](/tools/skills).

## Service features

| Feature                  | Notes                                                               |
| ------------------------ | ------------------------------------------------------------------- |
| Public browsing          | Skills and their `SKILL.md` content are publicly viewable.          |
| Search                   | Embedding-powered (vector search), not just keywords.               |
| Versioning               | Semver, changelogs, and tags (including `latest`).                  |
| Downloads                | Zip per version.                                                    |
| Stars and comments       | Community feedback.     

## Lobster (Multi-Agent Sandbox Tools)

Lobster is a workflow shell that lets OpenClaw run multi-step tool sequences as a single, deterministic operation with explicit approval checkpoints.

Lobster is one authoring layer above detached background work. For flow orchestration above individual tasks, see [Task Flow](/automation/taskflow) (`openclaw tasks flow`). For the task activity ledger, see [`openclaw tasks`](/automation/tasks).

## Hook

Your assistant can build the tools that manage itself. Ask for a workflow, and 30 minutes later you have a CLI plus pipelines that run as one call. Lobster is the missing piece: deterministic pipelines, explicit approvals, and resumable state.

## Why

Today, complex workflows require many back-and-forth tool calls. Each call costs tokens, and the LLM has to orchestrate every step. Lobster moves that orchestration into a typed runtime:

- **One call instead of many**: OpenClaw runs one Lobster tool call and gets a structured result.
- **Approvals built in**: Side effects (send email, post comment) halt the workflow until explicitly approved.
- **Resumable**: Halted workflows return a token; approve and resume without re-running everything.

## Why a DSL instead of plain programs?

Lobster is intentionally small. The goal is not "a new language," it's a predictable, AI-friendly pipeline spec with first-class approvals and resume tokens.

- **Approve/resume is built in**: A normal program can prompt a human, but it can’t _pause and resume_ with a durable token without you inventing that runtime yourself.
- **Determinism + auditability**: Pipelines are data, so they’re easy to log, diff, replay, and review.
- **Constrained surface for AI**: A tiny grammar + JSON piping reduces “creative” code paths and makes validation realistic.
- **Safety policy baked in**: Timeouts, output caps, sandbox checks, and allowlists are enforced by the runtime, not each script.
- **Still programmable**: Each step can call any CLI or script. If you want JS/TS, generate `.lobster` files from code.

## How it works

OpenClaw runs Lobster workflows **in-process** using an embedded runner. No external CLI subprocess is spawned; the workflow engine executes inside the gateway process and returns a JSON envelope directly.
If the pipeline pauses for approval, the tool returns a `resumeToken` so you can continue later.

## Pattern: small CLI + JSON pipes + approvals

Build tiny commands that speak JSON, then chain them into a single Lobster call. (Example command names below — swap in your own.)

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Apply changes?'",
  "timeoutMs": 30000
}
```

If the pipeline requests approval, resume with the token:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

AI triggers the workflow; Lobster executes the steps. Approval gates keep side effects explicit and auditable.

Example: map input items into tool calls:

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## JSON-only LLM steps (llm-task)

For workflows that need a **structured LLM step**, enable the optional
`llm-task` plugin tool and call it from Lobster. This keeps the workflow
deterministic while still letting you classify/summarize/draft with a model.

Enable the tool:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "allow": ["llm-task"] }
      }
    ]
  }
}
```

Use it in a pipeline:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Given the input email, return intent and draft.",
  "th

## Multi-Agent Sandbox Tools

Each agent in a multi-agent setup can override the global sandbox and tool policy. This page covers per-agent configuration, precedence rules, and examples.

    Backends and modes — full sandbox reference.

    Debug "why is this blocked?"

    Elevated exec for trusted senders.

Auth is scoped by agent: each agent has its own `agentDir` auth store at `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`. Never reuse `agentDir` across agents. Agents can read through to the default/main agent's auth profiles when they do not have a local profile, but OAuth refresh tokens are not cloned into secondary agent stores. If you copy credentials manually, copy only portable static `api_key` or `token` profiles.

---

## Configuration examples

    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "name": "Personal Assistant",
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "family",
            "name": "Family Bot",
            "workspace": "~/.openclaw/workspace-family",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read"],
              "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"]
            }
          }
        ]
      },
      "bindings": [
        {
          "agentId": "family",
          "match": {
            "provider": "whatsapp",
            "accountId": "*",
            "peer": {
              "kind": "group",
              "id": "120363424282127706@g.us"
            }
          }
        }
      ]
    }
    ```

    **Result:**

    - `main` agent: runs on host, full tool access.
    - `family` agent: runs in Docker (one container per agent), only `read` tool.

    ```json
    {
      "agents": {
        "list": [
          {
            "id": "personal",
            "workspace": "~/.openclaw/workspace-personal",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "work",
            "workspace": "~/.openclaw/workspace-work",
            "sandbox": {
              "mode": "all",
              "scope": "shared",
              "workspaceRoot": "/tmp/work-sandboxes"
            },
            "tools": {
              "allow": ["read", "write", "apply_patch", "exec"],
              "deny": ["browser", "gateway", "discord"]
            }
          }
        ]
      }
    }
    ```

    ```json
    {
      "tools": { "profile": "coding" },
      "agents": {
        "list": [
          {
            "id": "support",
            "tools": { "profile": "messaging", "allow": ["slack"] }
          }
        ]
      }
    }
    ```

    **Result:**

    - default agents get coding tools.
    - `support` agent is messaging-only (+ Slack tool).

    ```json
    {
      "agents": {
        "defaults": {
          "sandbox": {

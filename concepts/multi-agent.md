---
domain: concepts
topic: "Multi-Agent Routing: Isolated Agents, Binding Rules, Workspace Isolation, Per-Agent Config"
type: concept
keywords:
  - multi-agent
  - agent routing
  - binding rules
  - workspace isolation
  - agents.list
  - per-agent config
  - multiple agents
related:
  - concepts/agent-overview
  - gateway/config-agents-reference
  - concepts/session
source: concepts/multi-agent.md
---

OpenClaw supports multi-agent routing: multiple isolated agent instances with separate workspaces and sessions. Route different users or use cases to different agents using binding rules.

## Multi-Agent Setup

Run multiple _isolated_ agents — each with its own workspace, state directory (`agentDir`), and session history — plus multiple channel accounts (e.g. two WhatsApps) in one running Gateway. Inbound messages are routed to the right agent through bindings.

An **agent** here is the full per-persona scope: workspace files, auth profiles, model registry, and session store. `agentDir` is the on-disk state directory that holds this per-agent config at `~/.openclaw/agents/<agentId>/`. A **binding** maps a channel account (e.g. a Slack workspace or a WhatsApp number) to one of those agents.

## What is "one agent"?

An **agent** is a fully scoped brain with its own:

- **Workspace** (files, AGENTS.md/SOUL.md/USER.md, local notes, persona rules).
- **State directory** (`agentDir`) for auth profiles, model registry, and per-agent config.
- **Session store** (chat history + routing state) under `~/.openclaw/agents/<agentId>/sessions`.

Auth profiles are **per-agent**. Each agent reads from its own:

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

`sessions_history` is the safer cross-session recall path here too: it returns a bounded, sanitized view, not a raw transcript dump. Assistant recall strips thinking tags, `<relevant-memories>` scaffolding, plain-text tool-call XML payloads (including `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>`, and truncated tool-call blocks), downgraded tool-call scaffolding, leaked ASCII/full-width model control tokens, and malformed MiniMax tool-call XML before redaction/truncation.

Never reuse `agentDir` across agents (it causes auth/session collisions). Agents
can read through to the default/main agent's auth profiles when they do not have
a local profile, but OpenClaw does not clone OAuth refresh tokens into the
secondary agent store. If you want an independent OAuth account, sign in from
that agent; if you copy credentials manually, copy only portable static
`api_key` or `token` profiles.

Skills are loaded from each agent workspace plus shared roots such as `~/.openclaw/skills`, then filtered by the effective agent skill allowlist when configured. Use `agents.defaults.skills` for a shared baseline and `agents.list[].skills` for per-agent replacement. See [Skills: per-agent vs shared](/tools/skills#per-agent-vs-shared-skills) and [Skills: agent skill allowlists](/tools/skills#agent-skill-allowlists).

The Gateway can host **one agent** (default) or **many agents** side-by-side.

**Workspace note:** each agent's workspace is the **default cwd**, not a hard sandbox. Relative paths resolve inside the workspace, but absolute paths can reach other host locations unless sandboxing is enabled. See [Sandboxing](/gateway/sandboxing).

## Paths (quick map)

- Config: `~/.openclaw/openclaw.json` (or `OPENCLAW_CONFIG_PATH`)
- State dir: `~/.openclaw` (or `OPENCLAW_STATE_DIR`)
- Workspace: `~/.openclaw/workspace` (or `~/.openclaw/workspace-<agentId>`)
- Agent dir: `~/.openclaw/agents/<agentId>/agent` (or `agents.list[].agentDir`)
- Sessions: `~/.openclaw/agents/<agentId>/sessions`

### Single-agent mode (default)

If you do nothing, OpenClaw runs a single agent:

- `agentId` defaults to **`main`**.
- Sessions are keyed as `agent:main:<mainKey>`.
- Workspace defaults to `~/.openclaw/workspace` (or `~/.openclaw/workspace-<profile>` when `OPENCLAW_PROFILE` is set).
- State defaults to `~/.openclaw/agents/main/agent`.

## Agent helper

Use the agent wizard to add a new isolated agent:

```bash
openclaw agents add work
```

Then add `bindings` (or let the wizard do it) to route inbound messages.

Verify with:

```bash
openclaw agents list --bindings
```

## Quick start

    Use the wizard or create workspaces manually:

    ```bash
    openclaw agents add coding
    openclaw agents add social
    ```

    Each agent gets its own workspace with `SOUL.md`, `AGENTS.md`, and optional `USER.md`, plus a dedicated `agentDir` and session store under `~/.openclaw/agents/<agentId>`.

    Create one account per agent on your preferred channels:

    - Discord: one bot per agent, enable Message Content Intent, copy each token.
    - Telegram: one bot per agent via BotFather, copy each token.
    - WhatsApp: link each phone number per account.

    ```bash
    openclaw channels login --channel whatsapp --account work
    ```

    See channel guides: [Discord](/channels/discord), [Telegram](/channels/telegram), [WhatsApp](/channels/whatsapp).

    Add agents under `agents.list`, channel accounts under `channels.<channel>.accounts`, and connect them with `bindings` (examples below).

    ```bash
    openclaw gateway restart
    openclaw agents list --bindings
    openclaw channels status --probe
    ```

## Multiple agents = multiple people, multiple personalities

With **multiple agents**, each `agentId` becomes a **fully isolated persona**:

- **Different phone numbers/accounts** (per channel `accountId`).
- **Different personalities** (per-agent workspace files like `AGENTS.md` and `SOUL.md`).
- **Separate auth + sessions** (no cross-talk unless explicitly enabled).

This lets **multiple people** share one Gateway server while keeping their AI "brains" and data isolated.

## Cross-agent QMD memory search

If one agent should search another agent's QMD session transcripts, add extra collections under `agents.list[].memorySearch.qmd.extraCollections`. Use `agents.defaults.memorySearch.qmd.extraCollections` only when every agent should inherit the same shared transcript collections.

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspaces/main",
      memorySearch: {
        qmd: {
          extraCollections: [{ path: "~/agents/family/sessions", name: "family-sessions" }],
        },
      },
    },
    list: [
      {
        id: "main",
        workspace: "~/workspaces/main",
        memorySearch: {
          qmd: {
            extraCollections: [{ path: "notes" }], // resolves inside workspace -> collection named "notes-main"
          },
        },
      },
      { id: "family", workspace: "~/workspaces/family" },
    ],
  },
  memory: {
    backend: "qmd",
    qmd: { includeDefaultMemory: false },
  },
}
```

The extra collection path can be shared across agents, but the collection name stays explicit when the path is outside the agent workspace. Paths inside the workspace remain agent-scoped so each agent keeps its own transcript search set.

## One WhatsApp number, multiple people (DM split)

You can route **different WhatsApp DMs** to different agents while staying on **one WhatsApp account**. Match on sender E.164 (like `+15551234567`) with `peer.kind: "direct"`. Replies still come from the same WhatsApp number (no per-agent sender identity).

Direct chats collapse to the agent's **main session key**, so true isolation requires **one agent per person**.

Example:

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

Notes:

- DM access control is **global per WhatsApp account** (pairing/allowlist), not per agent.
- For shared groups, bind the group to one agent or use [Broadcast groups](/channels/broadcast-groups).

## Routing rules (how messages pick an agent)

Bindings are **deterministic** and **most-specific wins**:

    Exact DM/group/channel id.

    Thread inheritance.

    Discord role routing.

    Discord.

    Slack.

    Per-account fallback.

    `accountId: "*"`.

    Fallback to `agents.list[].default`, else first list entry, default: `main`.

    - If multiple bindings match in the same tier, the first one in config order wins.
    - If a binding sets multiple match fields (for example `peer` + `guildId`), all specified fields are required (`AND` semantics).

    - A binding that omits `accountId` matches the default account only.
    - Use `accountId: "*"` for a channel-wide fallback across all accounts.
    - If you later add the same binding for the same agent with an explicit account id, OpenClaw upgrades the existing channel-only binding to account-scoped instead of duplicating it.

## Multiple accounts / phone numbers

Channels that support **multiple accounts** (e.g. WhatsApp) use `accountId` to identify each login. Each `accountId` can be routed to a different agent, so one server can host multiple phone numbers without mixing sessions.

If you want a channel-wide default account when `accountId` is omitted, set `channels.<channel>.defaultAccount` (optional). When unset, OpenClaw fall

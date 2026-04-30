---
domain: gateway
topic: "Configuration Reference: agents.defaults, Workspace, Model, Heartbeat, and Per-Agent Overrides"
type: reference
keywords:
  - agents.defaults
  - agents.list
  - workspace
  - model config
  - heartbeat config
  - compaction
  - contextLimits
  - sandbox config
related:
  - gateway/configuration-overview
  - gateway/config-channels-reference
  - gateway/configuration-examples
source:
  - gateway/config-agents.md
  - gateway/configuration-reference.md
---

The `agents` top-level key controls agent defaults and per-agent overrides. All fields nest under `agents.defaults` (applies to all agents) or `agents.list[]` (per-agent overrides).

## Agent Defaults (`agents.defaults`)

Key fields:

| Field | Default | Description |
|-------|---------|-------------|
| `workspace` | `~/.openclaw/workspace` | Agent workspace root directory |
| `model` | (first available) | Default model identifier |
| `agentRuntime` | `embedded-pi` | Runtime mode |
| `heartbeat.every` | `30m` | Heartbeat cadence |
| `compaction.enabled` | true | Auto-compact long sessions |
| `sandbox` | off | Sandbox mode for tool execution |
| `timeoutSeconds` | 172800 | Max agent run time (48h) |

## Context Budget Fields

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        maxContextTokens: 200000,   // hard cap on total context
        compactionReserve: 20000,   // tokens held back for compaction output
      },
      bootstrapMaxChars: 50000,     // max chars for a single bootstrap file
      bootstrapTotalMaxChars: 200000, // total bootstrap context limit
    }
  }
}
```

## Per-Agent Overrides (`agents.list`)

```json5
{
  agents: {
    list: [
      {
        id: "research",
        workspace: "~/.openclaw/workspace-research",
        model: "claude-opus-4-5",
        // any agents.defaults field can be overridden here
      }
    ]
  }
}
```

Agent-scoped configuration keys under `agents.*`, `multiAgent.*`, `session.*`,
`messages.*`, and `talk.*`. For channels, tools, gateway runtime, and other
top-level keys, see [Configuration reference](/gateway/configuration-reference).

## Agent defaults

### `agents.defaults.workspace`

Default: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

Optional repository root shown in the system prompt's Runtime line. If unset, OpenClaw auto-detects by walking upward from the workspace.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Optional default skill allowlist for agents that do not set
`agents.list[].skills`.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // inherits github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults
      { id: "locked-down", skills: [] }, // no skills
    ],
  },
}
```

- Omit `agents.defaults.skills` for unrestricted skills by default.
- Omit `agents.list[].skills` to inherit the defaults.
- Set `agents.list[].skills: []` for no skills.
- A non-empty `agents.list[].skills` list is the final set for that agent; it
  does not merge with defaults.

### `agents.defaults.skipBootstrap`

Disables automatic creation of workspace bootstrap files (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

Controls when workspace bootstrap files are injected into the system prompt. Default: `"always"`.

- `"continuation-skip"`: safe continuation turns (after a completed assistant response) skip workspace bootstrap re-injection, reducing prompt size. Heartbeat runs and post-compaction retries still rebuild context.
- `"never"`: disable workspace bootstrap and context-file injection on every turn. Use this only for agents that fully own their prompt lifecycle (custom context engines, native runtimes that build their own context, or specialized bootstrap-free workflows). Heartbeat and compaction-recovery turns also skip injection.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

Max characters per workspace bootstrap file before truncation. Default: `12000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 12000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

Max total characters injected across all workspace bootstrap files. Default: `60000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Controls agent-visible warning text when bootstrap context is truncated.
Default: `"once"`.

- `"off"`: never inject warning text into the system prompt.
- `"once"`: inject warning once per unique truncation signature (recommended).
- `"always"`: inject warning on every run when truncation exists.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### Context budget ownership map

OpenClaw has multiple high-volume prompt/context budgets, and they are
intentionally split by subsystem instead of all flowing through one generic
knob.

- `agents.defaults.bootstrapMaxChars` /
  `agents.defaults.bootstrapTotalMaxChars`:
  normal workspace bootstrap injection.
- `agents.defaults.startupContext.*`:
  one-shot reset/startup model-run prelude, including recent daily
  `memory/*.md` files. Bare chat `/new` and `/reset` commands are
  acknowledged without invoking the model.
- `skills.limits.*`:
  the compact skills list injected into the system prompt.
- `agents.defaults.contextLimits.*`:
  bounded runtime excerpts and injected runtime-owned blocks.
- `memory.qmd.limits.*`:
  indexed memory-search snippet and injection sizing.

Use the matching per-agent override only when one agent needs a different
budget:

- `agents.list[].skillsLimits.maxSkillsPromptChars`
- `agents.list[].contextLimits.*`

#### `agents.defaults.startupContext`

Controls the first-turn startup prelude injected on reset/startup model runs.
Bare chat `/new` and `/reset` commands acknowledge the reset without invoking
the model, so they do not load this prelude.

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

Shared defaults for bounded runtime context surfaces.

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        memoryGetDefaultLines: 120,
        toolResultMaxChars: 16000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}

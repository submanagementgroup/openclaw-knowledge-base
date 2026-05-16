---
domain: concepts
topic: "Active Memory Configuration Reference"
type: reference
keywords:
  - active-memory config
  - config.agents
  - config.timeoutMs
  - config.toolsAllow
  - setupGraceTimeoutMs
  - persistTranscripts
  - circuit breaker
source: concepts/active-memory.md
---

ply continues without memory context.
`toolsAllow` only accepts concrete memory tool names. Wildcards, `group:*`
entries, and core agent tools such as `read`, `exec`, `message`, and
`web_search` are ignored before the hidden memory sub-agent starts.

Default-behavior note: Active Memory no longer includes `memory_recall` in the
memory-core default allowlist. Existing `memory-lancedb` setups keep working
when `plugins.slots.memory` is set to `memory-lancedb`. Explicit `toolsAllow`
always overrides the automatic default.

### Built-in memory-core

The default setup does not need an explicit `toolsAllow`:

```json5
{
  plugins: {
    entries: {
      "active-memory": {
        enabled: true,
        config: {
          agents: ["main"],
          // Default: ["memory_search", "memory_get"]
        },
      },
    },
  },
}
```

### LanceDB memory

The bundled `memory-lancedb` plugin exposes `memory_recall`. Selecting the
memory slot is enough for Active Memory to use that recall tool:

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small",
          },
        },
      },
      "active-memory": {
        enabled: true,
        config: {
          agents: ["main"],
          promptAppend: "Use memory_recall for long-term user preferences, past decisions, and previously discussed topics. If recall finds nothing useful, return NONE.",
        },
      },
    },
  },
}
```

### Lossless Claw

Lossless Claw is a context-engine plugin with its own recall tools. Install and
configure it as a context engine first; see [Context engine](/concepts/context-engine).
Then let Active Memory use the Lossless Claw recall tools:

```json5
{
  plugins: {
    entries: {
      "lossless-claw": {
        enabled: true,
      },
      "active-memory": {
        enabled: true,
        config: {
          agents: ["main"],
          toolsAllow: ["lcm_grep", "lcm_describe", "lcm_expand_query"],
          promptAppend: "Use lcm_grep first for compacted conversation recall. Use lcm_describe to inspect a specific summary. Use lcm_expand_query only when the latest user message needs exact details that may have been compacted away. Return NONE if the retrieved context is not clearly useful.",
        },
      },
    },
  },
}
```

Do not include `lcm_expand` in `toolsAllow` for the main Active Memory sub-agent.
Lossless Claw uses that as a lower-level delegated expansion tool.

## Advanced escape hatches

These options are intentionally not part of the recommended setup.

`config.thinking` can override the blocking memory sub-agent thinking level:

```json5
thinking: "medium"
```

Default:

```json5
thinking: "off"
```

Do not enable this by default. Active Memory runs in the reply path, so extra
thinking time directly increases user-visible latency.

`config.promptAppend` adds extra operator instructions after the default Active
Memory prompt and before the conversation context:

```json5
promptAppend: "Prefer stable long-term preferences over one-off events."
```

Use `promptAppend` with custom `toolsAllow` when a non-core memory plugin needs
provider-specific tool order or query-shaping instructions.

`config.promptOverride` replaces the default Active Memory prompt. OpenClaw
still appends the conversation context afterward:

```json5
promptOverride: "You are a memory search agent. Return NONE or one compact user fact."
```

Prompt customization is not recommended unless you are deliberately testing a
different recall contract. The default prompt is tuned to return either `NONE`
or compact user-fact context for the main model.

## Transcript persistence

Active memory blocking memory sub-agent runs create a real `session.jsonl`
transcript during the blocking memory sub-agent call.

By default, that transcript is temporary:

- it is written to a temp directory
- it is used only for the blocking memory sub-agent run
- it is deleted immediately after the run finishes

If you want to keep those blocking memory sub-agent transcripts on disk for debugging or
inspection, turn persistence on explicitly:

```json5
{
  plugins: {
    entries: {
      "active-memory": {
        enabled: true,
        config: {
          agents: ["main"],
          persistTranscripts: true,
          transcriptDir: "active-memory",
        },
      },
    },
  },
}
```

When enabled, active memory stores transcripts in a separate directory under the
target agent's sessions folder, not in the main user conversation transcript
path.

The default layout is conceptually:

```text
agents/<agent>/sessions/active-memory/<blocking-memory-sub-agent-session-id>.jsonl
```

You can change the relative subdirectory with `config.transcriptDir`.

Use this carefully:

- blocking memory sub-agent transcripts can accumulate quickly on busy sessions
- `full` query mode can duplicate a lot of conversation context
- these transcripts contain hidden prompt context and recalled memories

## Configuration

All active memory configuration lives under:

```text
plugins.entries.active-memory
```

The most important fields are:

| Key                          | Type                                                                                                 | Meaning                                                                                                                                                                                                                                                  |
| ---------------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                    | `boolean`                                                                                            | Enables the plugin itself                                                                                                                                                                                                                                |
| `config.agents`              | `string[]`                                                                                           | Agent ids that may use active memory                                                                                                                                                                                                                     |
| `config.model`               | `string`                                                                                             | Optional blocking memory sub-agent model ref; when unset, active memory uses the current session model                                                                                                                                                   |
| `config.allowedChatTypes`    | `("direct" \| "group" \| "channel")[]`                                                               | Session types that may run Active Memory; defaults to direct-message style sessions                                                                                                                                                                      |
| `config.allowedChatIds`      | `string[]`                                                                                           | Optional per-conversation allowlist applied after `allowedChatTypes`; non-empty lists fail closed                                                                                                                                                        |
| `config.deniedChatIds`       | `string[]`                                                                                           | Optional per-conversation denylist that overrides allowed session types and allowed ids                                                                                                                                                                  |
| `config.queryMode`           | `"message" \| "recent" \| "full"`                                                                    | Controls how much conversation the blocking memory sub-agent sees                                                                                                                                                                                        |
| `config.promptStyle`         | `"balanced" \| "strict" \| "contextual" \| "recall-heavy" \| "precision-heavy" \| "preference-only"` | Controls how eager or strict the blocking memory sub-agent is when deciding whether to return memory                                                                                                                                                     |
| `config.toolsAllow`          | `string[]`                                                                                           | Concrete memory tool names the blocking memory sub-agent may call; defaults to `["memory_search", "memory_get"]`, or `["memory_recall"]` when `plugins.slots.memory` is `memory-lancedb`; wildcards, `group:*` entries, and core agent tools are ignored |
| `config.thinking`            | `"off" \| "minimal" \| "low" \| "medium" \| "high" \| "xhigh" \| "adaptive" \| "max"`                | Advanced thinking override for the blocking memory sub-agent; default `off` for speed                                                                                                                                                                    |
| `config.promptOverride`      | `string`                                                                                             | Advanced full prompt replacement; not recommended for normal use                                                                                                                                                                                         |
| `config.promptAppend`        | `string`                                                                                             | Advanced extra instructions appended to the default or overridden prompt                                                                                                                                                                                 |
| `config.timeoutMs`           | `number`                                                                                             | Hard timeout for the blocking memory sub-agent, capped at 120000 ms                                                                                                                                                                                      |
| `config.setupGraceTimeoutMs` | `number`                                                                                             | Advanced extra setup budget before the recall timeout expires; defaults to 0 and is capped at 30000 ms. See [Cold-start grace](#cold-start-grace) for v2026.4.x upgrade guidance                                                                         |
| `config.maxSummaryChars`     | `number`                                                                                             | Maximum total characters allowed in the active-memory summary                                                                                                                                                                                            |
| `config.logging`             | `boolean`                                                                                            | Emits active memory logs while tuning                                                                                                                                                                                                                    |
| `config.persistTranscripts`  | `boolean`                                                                                            | Keeps blocking memory sub-agent transcripts on disk instead of deleting temp files                                                                                                                                                                       |
| `config.transcriptDir`       | `string`                                                                                             | Relative blocking memory sub-agent transcript directory under the agent sessions folder                                                                                                                                                                  |

Useful tuning fields:

| Key                                | Type     | Meaning                                                                                                                                                           |
| ---------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `config.maxSummaryChars`           | `number` | Maximum total characters allowed in the active-memory summary                                                                                                     |
| `config.recentUserTurns`           | `number` | Prior user turns to include when `queryMode` is `recent`                                                                                                          |
| `config.recentAssistantTurns`      | `number` | Prior assistant turns to include when `queryMode` is `recent`                             
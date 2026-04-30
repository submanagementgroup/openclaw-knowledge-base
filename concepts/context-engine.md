---
domain: concepts
topic: "Context Engine: Prompt Assembly, Token Budget, Bootstrap Files, and System Prompt Construction"
type: concept
keywords:
  - context
  - context engine
  - prompt assembly
  - token budget
  - system prompt
  - bootstrap context
  - contextLimits
related:
  - concepts/system-prompt
  - concepts/compaction
  - concepts/agent-workspace
source:
  - concepts/context.md
  - concepts/context-engine.md
---

The context engine assembles the full prompt sent to the model: system prompt + bootstrap files + conversation history + skill prompts. Understanding context helps you tune token usage and model behavior.

## Context Assembly

“Context” is **everything OpenClaw sends to the model for a run**. It is bounded by the model’s **context window** (token limit).

Beginner mental model:

- **System prompt** (OpenClaw-built): rules, tools, skills list, time/runtime, and injected workspace files.
- **Conversation history**: your messages + the assistant’s messages for this session.
- **Tool calls/results + attachments**: command output, file reads, images/audio, etc.

Context is _not the same thing_ as “memory”: memory can be stored on disk and reloaded later; context is what’s inside the model’s current window.

## Quick start (inspect context)

- `/status` → quick “how full is my window?” view + session settings.
- `/context list` → what’s injected + rough sizes (per file + totals).
- `/context detail` → deeper breakdown: per-file, per-tool schema sizes, per-skill entry sizes, and system prompt size.
- `/usage tokens` → append per-reply usage footer to normal replies.
- `/compact` → summarize older history into a compact entry to free window space.

See also: [Slash commands](/tools/slash-commands), [Token use & costs](/reference/token-use), [Compaction](/concepts/compaction).

## Example output

Values vary by model, provider, tool policy, and what’s in your workspace.

### `/context list`

```
🧠 Context breakdown
Workspace: <workspaceDir>
Bootstrap max/file: 12,000 chars
Sandbox: mode=non-main sandboxed=false
System prompt (run): 38,412 chars (~9,603 tok) (Project Context 23,901 chars (~5,976 tok))

Injected workspace files:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Skills list (system prompt text): 2,184 chars (~546 tok) (12 skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Tool list (system prompt text): 1,032 chars (~258 tok)
Tool schemas (JSON): 31,988 chars (~7,997 tok) (counts toward context; not shown as text)
Tools: (same as above)

Session tokens (cached): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Context breakdown (detailed)
…
Top skills (prompt entry size):
- frontend-design: 412 chars (~103 tok)
- oracle: 401 chars (~101 tok)
… (+10 more skills)

Top tools (schema size):
- browser: 9,812 chars (~2,453 tok)
- exec: 6,240 chars (~1,560 tok)
… (+N more tools)
```

## What counts toward the context window

Everything the model receives counts, including:

- System prompt (all sections).
- Conversation history.
- Tool calls + tool results.
- Attachments/transcripts (images/audio/files).
- Compaction summaries and pruning artifacts.
- Provider “wrappers” or hidden

## Context Engine

A **context engine** controls how OpenClaw builds model context for each run: which messages to include, how to summarize older history, and how to manage context across subagent boundaries.

OpenClaw ships with a built-in `legacy` engine and uses it by default — most users never need to change this. Install and select a plugin engine only when you want different assembly, compaction, or cross-session recall behavior.

## Quick start

    ```bash
    openclaw doctor
    # or inspect config directly:
    cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
    ```

    Context engine plugins are installed like any other OpenClaw plugin.

        ```bash
        openclaw plugins install @martian-engineering/lossless-claw
        ```

        ```bash
        openclaw plugins install -l ./my-context-engine
        ```

    ```json5
    // openclaw.json
    {
      plugins: {
        slots: {
          contextEngine: "lossless-claw", // must match the plugin's registered engine id
        },
        entries: {
          "lossless-claw": {
            enabled: true,
            // Plugin-specific config goes here (see the plugin's docs)
          },
        },
      },
    }
    ```

    Restart the gateway after installing and configuring.

    Set `contextEngine` to `"legacy"` (or remove the key entirely — `"legacy"` is the default).

## How it works

Every time OpenClaw runs a model prompt, the context engine participates at four lifecycle points:

    Called when a new message is added to the session. The engine can store or index the message in its own data store.

    Called before each model run. The engine returns an ordered set of messages (and an optional `systemPromptAddition`) that fit within the token budget.

    Called when the context window is full, or when the user runs `/compact`. The engine summarizes older history to free space.

    Called after a run completes. The engine can persist state, trigger background compaction, or update indexes.

For the bundled non-ACP Codex harness, OpenClaw applies the same lifecycle by projecting assembled context into Codex developer instructions and the current turn prompt. Codex still owns its native thread history and native compactor.

### Subagent lifecycle (optional)

OpenClaw calls two optional subagent lifecycle hooks:

<ParamField path="prepareSubagentSpawn" type="method">
  Prepare shared context state before a child run starts. The hook receives parent/child session keys, `contextMode` (`isolated` or `fork`), available transcript ids/files, and optional TTL. If it returns a rollback handle, OpenClaw calls it when spawn fails after preparation succeeds.
</ParamField>
<ParamField path="onSubagentEnded" type="method">
  Clean up when a subagent session completes or is swept.
</ParamField>

### System prompt addition

The `assemble` method can return a `systemPromptAddition` string. OpenClaw prepends this to the system prompt for the run. This lets engines inject dynamic recall guidance, retrieval instructions, or context-aware hints without requiring static workspace files.

## The legacy engine

The built-in `legacy` engine preserves OpenClaw's original behavior:

- **Ingest**: no-op (the session manager handles message persistence directly).
- **Assemble**: pass-through (the existing sanitize → validate → limit pipeline in the runtime handles context assembly).
- **Compact**: delegates to the built-in summarization compaction, which creates a single summary of older messages and keeps recent messages intact.
- **After turn**: no-op.

The legacy engine does not register tools or provide a `systemPromptAddition`.

When no `plugins.slots.contextEngine` is set (or it's set to `"legacy"`), this engine is used automatically.

## Plugin engines

A plugin can register a context engine using the plugin API:

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", (ctx) => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // Store the message in your data store
      return { ingested: true };
    },

    async assemble({ sessionId, messages, tokenBudget, availableTools, citationsMode }) {
      // Return messages that fit the budget
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // Summarize older context
      return { ok: true, compacted: true };
    },
  }));
}
```

The factory `ctx` includes optional `config`, `agentDir`, and `workspaceDir`
values so plugins can initialize per-agent or per-workspace state before the
first lifecycle

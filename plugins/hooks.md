---
domain: plugins
topic: "Plugin Hooks: Lifecycle Callbacks for Agent Events, Tool Calls, and Message Handling"
type: reference
keywords:
  - plugin hooks
  - lifecycle hooks
  - before_tool_call
  - message_sending
  - agent hooks
  - plugin callbacks
related:
  - plugins/plugin-architecture
  - automation/hooks
source: plugins/hooks.md
---

Plugin hooks are lifecycle callbacks that let plugins intercept agent and gateway events. Hooks can modify behavior, block actions, or inject context.

Plugin hooks are in-process extension points for OpenClaw plugins. Use them
when a plugin needs to inspect or change agent runs, tool calls, message flow,
session lifecycle, subagent routing, installs, or Gateway startup.

Use [internal hooks](/automation/hooks) instead when you want a small
operator-installed `HOOK.md` script for command and Gateway events such as
`/new`, `/reset`, `/stop`, `agent:bootstrap`, or `gateway:startup`.

## Quick start

Register typed plugin hooks with `api.on(...)` from your plugin entry:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "tool-preflight",
  name: "Tool Preflight",
  register(api) {
    api.on(
      "before_tool_call",
      async (event) => {
        if (event.toolName !== "web_search") {
          return;
        }

        return {
          requireApproval: {
            title: "Run web search",
            description: `Allow search query: ${String(event.params.query ?? "")}`,
            severity: "info",
            timeoutMs: 60_000,
            timeoutBehavior: "deny",
          },
        };
      },
      { priority: 50 },
    );
  },
});
```

Hook handlers run sequentially in descending `priority`. Same-priority hooks
keep registration order.

`api.on(name, handler, opts?)` accepts:

- `priority` — handler ordering (higher runs first).
- `timeoutMs` — optional per-hook budget. When set, the hook runner aborts that
  handler after the budget elapses and continues with the next one, instead of
  letting slow setup or recall work consume the caller's configured model
  timeout. Omit it to use the default observation/decision timeout that the
  hook runner applies generically.

Each hook receives `event.context.pluginConfig`, the resolved config for the
plugin that registered that handler. Use it for hook decisions that need
current plugin options; OpenClaw injects it per handler without mutating the
shared event object seen by other plugins.

## Hook catalog

Hooks are grouped by the surface they extend. Names in **bold** accept a
decision result (block, cancel, override, or require approval); all others are
observation-only.

**Agent turn**

- `before_model_resolve` — override provider or model before session messages load
- `agent_turn_prepare` — consume queued plugin turn injections and add same-turn context before prompt hooks
- `before_prompt_build` — add dynamic context or system-prompt text before the model call
- `before_agent_start` — compatibility-only combined phase; prefer the two hooks above
- **`before_agent_reply`** — short-circuit the model turn with a synthetic reply or silence
- **`before_agent_finalize`** — inspect the natural final answer and request one more model pass
- `agent_end` — observe final messages, success state, and run duration
- `heartbeat_prompt_contribution` — add heartbeat-only context for background monitor and lifecycle plugins

**Conversation observation**

- `model_call_started` / `model_call_ended` — observe sanitized provider/model call metadata, timing, outcome, and bounded request-id hashes without prompt or response content
- `llm_input` — observe provider input (system prompt, prompt, history)
- `llm_output` — observe provider output

**Tools**

- **`before_tool_call`** — rewrite tool params, block execution, or require approval
- `after_tool_call` — observe tool results, errors, and duration
- **`tool_result_persist`** — rewrite the assistant message produced from a tool result
- **`before_message_write`** — inspect or block an in-progress message write (rare)

**Messages and delivery**

- **`inbound_claim`** — claim an inbound message before agent routing (synthetic replies)
- `message_received` — observe inbound content, sender, thread, and metadata
- **`message_sending`** — rewrite outbound content or cancel delivery
- `message_sent` — observe outbound delivery success or failure
- **`before_dispatch`** — inspect or rewrite an outbound dispatch before channel handoff
- **`reply_dispatch`** — participate in the final reply-dispatch pipeline

**Sessions and compaction**

- `session_start` / `session_end` — track session lifecycle boundaries
- `before_compaction` / `after_compaction` — observe or annotate compaction cycles
- `before_reset` — observe session-reset events (`/reset`, programmatic resets)

**Subagents**

- `subagent_spawning` / `subagent_delivery_target` / `subagent_spawned` / `subagent_ended` — coordinate subagent routing and completion delivery

**Lifecycle**

- `gateway_start` / `gateway_stop` — start or stop plugin-owned services with the Gateway
- `cron_changed` — observe gateway-owned cron lifecycle changes (added, updated, removed, started, finished, scheduled)
- **`before_install`** — inspect skill or plugin install scans and optionally block

## Tool call policy

`before_tool_call` receives:

- `event.toolName`
- `event.params`
- optional `event.runId`
- optional `event.toolCallId`
- context fields such as `ctx.agentId`, `ctx.sessionKey`, `ctx.sessionId`,
  `ctx.runId`, `ctx.jobId` (set on cron-driven runs), and diagnostic `ctx.trace`

It can return:

```typescript
type BeforeToolCallResult = {
  params?: Record<string, unknown>;
  block?: boolean;
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    timeoutBehavior?: "allow" | "deny";
    pluginId?: string;
    onResolution?: (
      decision: "allow-once" | "allow-always" | "deny" | "timeout" | "cancelled",
    ) => Promise<void> | void;
  };
};
```

Rules:

- `block: true` is terminal and skips lower-priority handlers.
- `block: false` is treated as no decision.
- `params` rewrites the tool parameters for execution.
- `requireApproval` pauses the agent run and asks the user through plugin
  approvals. The `/approve` command can approve both exec and plugin approvals.
- A lower-priority `block: true` can still block after a higher-priority hook
  requested approval.
- `onResolution` receives the resolved approval decision — `allow-once`,
  `allow-always`, `deny`, `timeout`, or `cancelled`.

Bundled plugins that need host-level policy can register trusted tool policies
with `api.registerTrustedToolPolicy(...)`. These run before ordinary
`before_tool_call` hooks and before external plugin decisions. Use them only
for host-trusted gates such as workspace policy, budget enforcement, or
reserved workflow safety. External plugins should use normal `before_tool_call`
hooks.

### Tool result persistence

Tool results can include structured `details` for UI rendering, diagnostics,
media routing, or plugin-owned metadata. Treat `details` as runtime metadata,
not prompt content:

- OpenClaw strips `toolResult.details` before provider replay and compaction
  input so metadata does not become model context.
- Persisted session entries keep only bounded `details`. Oversized details are
  replaced with a compact summary and `persistedDetailsTruncated: true`.
- `tool_result_persist` and `before_message_write` run before the final
  persistence cap. Hooks should still keep returned `details` small and avoid
  placing prompt-relevant text only in `details`; put model-visible tool output
  in `content`.

## Prompt and model hooks

Use the phase-specific hooks for new plugins:

- `before_model_resolve`: receives only the current prompt and attachment
  metadata. Return `providerOverride` or `modelOverride`.
- `agent_turn_prepare`: receives the current prompt, prepared session messages,
  and any exactly-once queued injections drained for this session. Return
  `prependContext` or `appendContext`.
- `before_prompt_build`: receives the current prompt and session messages.
  Return `prependContext`, `appendContext`, `systemPrompt`,
  `prependSystemContext`, or `appendSystemContext`.
- `heartbeat_prompt_contribution`: runs only for heartbeat turns and returns
  `prependContext` or `appendContext`. It is intended for background monitors
  that need to summarize current state without changing user-initiated turns.

`before_agent_start` remains for compatibility. Prefer the explicit hooks above
so your plugin does not depend on a legacy combined phase.

`before_agent_start` and `agent_end` include `event.runId` when OpenClaw can
identify the active run. The same value is also available on `ctx.runId`.
Cron-driven runs also expose `ctx.jobId` (the originating cron job id) so
plugin hooks can scope metrics, side effects, or state to a specific scheduled
job.

`agent_end` is an observation hook and runs fire-and-forget after the turn. The
hook runner applies a 30 second timeout so a wedged plugin or embedding
endpoint cannot leave the hook promise pending forever. A timeout is logged and
OpenClaw continues; it does not cancel plugin-owned network work unless the
plugin also uses its own abort signal.

Use `model_call_started` and `model_call_ended` for provider-call telemetry
that should not rece

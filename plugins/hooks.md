---
domain: plugins
topic: "Plugin Hooks System"
type: reference
keywords:
  - hooks
  - plugin hooks
  - hook types
  - onStart
  - onMessage
  - hook configuration
source: plugins/hooks.md
---

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

- `priority` - handler ordering (higher runs first).
- `timeoutMs` - optional per-hook budget. When set, the hook runner aborts that
  handler after the budget elapses and continues with the next one, instead of
  letting slow setup or recall work consume the caller's configured model
  timeout. Omit it to use the default observation/decision timeout that the
  hook runner applies generically.

Operators can also set hook budgets without patching plugin code:

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "timeoutMs": 30000,
          "timeouts": {
            "before_prompt_build": 90000,
            "agent_end": 60000
          }
        }
      }
    }
  }
}
```

`hooks.timeouts.<hookName>` overrides `hooks.timeoutMs`, which overrides the
plugin-authored `api.on(..., { timeoutMs })` value. Each configured value must
be a positive integer no greater than 600000 milliseconds. Prefer per-hook
overrides for known slow hooks so one plugin does not get a longer budget
everywhere.

Each hook receives `event.context.pluginConfig`, the resolved config for the
plugin that registered that handler. Use it for hook decisions that need
current plugin options; OpenClaw injects it per handler without mutating the
shared event object seen by other plugins.

## Hook catalog

Hooks are grouped by the surface they extend. Names in **bold** accept a
decision result (block, cancel, override, or require approval); all others are
observation-only.

**Agent turn**

- `before_model_resolve` - override provider or model before session messages load
- `agent_turn_prepare` - consume queued plugin turn injections and add same-turn context before prompt hooks
- `before_prompt_build` - add dynamic context or system-prompt text before the model call
- `before_agent_start` - compatibility-only combined phase; prefer the two hooks above
- **`before_agent_run`** - inspect the final prompt and session messages before model submission and optionally block the run
- **`before_agent_reply`** - short-circuit the model turn with a synthetic reply or silence
- **`before_agent_finalize`** - inspect the natural final answer and request one more model pass
- `agent_end` - observe final messages, success state, and run duration
- `heartbeat_prompt_contribution` - add heartbeat-only context for background monitor and lifecycle plugins

**Conversation observation**

- `model_call_started` / `model_call_ended` - observe sanitized provider/model call metadata, timing, outcome, and bounded request-id hashes without prompt or response content
- `llm_input` - observe provider input (system prompt, prompt, history)
- `llm_output` - observe provider output

**Tools**

- **`before_tool_call`** - rewrite tool params, block execution, or require approval
- `after_tool_call` - observe tool results, errors, and duration
- **`tool_result_persist`** - rewrite the assistant message produced from a tool result
- **`before_message_write`** - inspect or block an in-progress message write (rare)

**Messages and delivery**

- **`inbound_claim`** - claim an inbound message before agent routing (synthetic replies)
- `message_received` - observe inbound content, sender, thread, and metadata
- **`message_sending`** - rewrite outbound content or cancel delivery
- `message_sent` - observe outbound delivery success or failure
- **`before_dispatch`** - inspect or rewrite an outbound dispatch before channel handoff
- **`reply_dispatch`** - participate in the final reply-dispatch pipeline

**Sessions and compaction**

- `session_start` / `session_end` - track session lifecycle boundaries. The event's `reason` is one of `new`, `reset`, `idle`, `daily`, `compaction`, `deleted`, `shutdown`, `restart`, or `unknown`. The `shutdown` and `restart` values fire from the gateway shutdown finalizer when the process is stopped or restarted while sessions are still active, so downstream plugins (such as memory or transcript stores) can finalize ghost rows that would otherwise be left in an open state across restarts. The finalizer is bounded so a slow plugin cannot block SIGTERM/SIGINT.
- `before_compaction` / `after_compaction` - observe or annotate compaction cycles
- `before_reset` - observe session-reset events (`/reset`, programmatic resets)

**Subagents**

- `subagent_spawning` / `subagent_delivery_target` / `subagent_spawned` / `subagent_ended` - coordinate subagent routing and completion delivery

**Lifecycle**

- `gateway_start` / `gateway_stop` - start or stop plugin-owned services with the Gateway
- `cron_changed` - observe gateway-owned cron lifecycle changes (added, updated, removed, started, finished, scheduled)
- **`before_install`** - inspect skill or plugin install scans and optionally block

## Tool call policy

`before_tool_call` receives:

- `event.toolName`
- `event.params`
- optional `event.derivedPaths`, containing best-effort host-derived target path
  hints for well-known tool envelopes such as `apply_patch`; when present,
  these paths may be incomplete or may over-approximate what the tool will
  actually touch (for example, with malformed or partial inputs)
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
- `onResolution` receives the resolved approval decision - `allow-once`,
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

`before_agent_run` runs after prompt construction and before any model input,
including prompt-local image loading and `llm_input` observation. It receives
the current user input as `prompt`, plus loaded session history in `messages`
and the active system prompt. Return `{ outcome: "block", reason, message? }`
to stop the run before the model can read the prompt. `reason` is internal;
`message` is the user-facing replacement. The only supported outcomes are
`pass` and `block`; unsupported decision shapes fail closed.

When a run is blocked, OpenClaw stores only the replacement text in
`message.content` plus non-sensitive block metadata such as the blocking plugin
id and timestamp. The original user text is not retained in transcript or future
context. Internal block reasons are treated as sensitive and excluded from
transcript, history, broadcast, log, and diagnostics payloads. Observability
should use sanitized fields such as blocker id, outcome, timestamp, or a safe
category.

`before_agent_start` and `agent_end` include `event.runId` when OpenClaw can
identify the active run. The same value is also available on `ctx.runId`.
Cron-driven runs also expose `ctx.jobId` (the originating cron job id) so
plugin hooks can scope metrics, side effects, or state to a specific scheduled
job.

For channel-originated runs, `ctx.messageProvider` is the provider surface such
as `discord` or `telegram`, while `ctx.channelId` is the conversation target
identifier when OpenClaw can derive one from the session key or delivery
metadata.

`agent_end` is an observation hook and runs fire-and-forget after the turn. The
hook runner applies a 30 second timeout so a wedged plugin or embedding
endpoint cannot leave the hook promise pending forever. A timeout is logged and
OpenClaw continues; it does not cancel plugin-owned network work unless the
plugin also uses its own abort signal.

Use `model_call_started` and `model_call_ended` for provider-call telemetry
that should not receive raw prompts, history, responses, headers, request
bodies, or provider request IDs. These hooks include stable metadata such as
`runId`, `callId`, `provider`, `model`, optional `api`/`transport`, termina
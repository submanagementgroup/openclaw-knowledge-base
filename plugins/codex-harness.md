---
domain: plugins
topic: "Codex Harness Plugin: OpenClaw Integration with OpenAI Codex Agent Runtime"
type: reference
keywords:
  - Codex harness
  - OpenAI Codex
  - Codex plugin
  - coding agent harness
  - Codex integration
related:
  - plugins/sdk-agent-harness
  - providers/openai
source: plugins/codex-harness.md
---

The Codex harness plugin adapts OpenClaw to run with OpenAI's Codex coding agent runtime.

The bundled `codex` plugin lets OpenClaw run embedded agent turns through the
Codex app-server instead of the built-in PI harness.

Use this when you want Codex to own the low-level agent session: model
discovery, native thread resume, native compaction, and app-server execution.
OpenClaw still owns chat channels, session files, model selection, tools,
approvals, media delivery, and the visible transcript mirror.

If you are trying to orient yourself, start with
[Agent runtimes](/concepts/agent-runtimes). The short version is:
`openai/gpt-5.5` is the model ref, `codex` is the runtime, and Telegram,
Discord, Slack, or another channel remains the communication surface.

## What this plugin changes

The bundled `codex` plugin contributes several separate capabilities:

| Capability                        | How you use it                                      | What it does                                                                  |
| --------------------------------- | --------------------------------------------------- | ----------------------------------------------------------------------------- |
| Native embedded runtime           | `agentRuntime.id: "codex"`                          | Runs OpenClaw embedded agent turns through Codex app-server.                  |
| Native chat-control commands      | `/codex bind`, `/codex resume`, `/codex steer`, ... | Binds and controls Codex app-server threads from a messaging conversation.    |
| Codex app-server provider/catalog | `codex` internals, surfaced through the harness     | Lets the runtime discover and validate app-server models.                     |
| Codex media-understanding path    | `codex/*` image-model compatibility paths           | Runs bounded Codex app-server turns for supported image understanding models. |
| Native hook relay                 | Plugin hooks around Codex-native events             | Lets OpenClaw observe/block supported Codex-native tool/finalization events.  |

Enabling the plugin makes those capabilities available. It does **not**:

- start using Codex for every OpenAI model
- convert `openai-codex/*` model refs into the native runtime
- make ACP/acpx the default Codex path
- hot-switch existing sessions that already recorded a PI runtime
- replace OpenClaw channel delivery, session files, auth-profile storage, or
  message routing

The same plugin also owns the native `/codex` chat-control command surface. If
the plugin is enabled and the user asks to bind, resume, steer, stop, or inspect
Codex threads from chat, agents should prefer `/codex ...` over ACP. ACP remains
the explicit fallback when the user asks for ACP/acpx or is testing the ACP
Codex adapter.

Native Codex turns keep OpenClaw plugin hooks as the public compatibility layer.
These are in-process OpenClaw hooks, not Codex `hooks.json` command hooks:

- `before_prompt_build`
- `before_compaction`, `after_compaction`
- `llm_input`, `llm_output`
- `before_tool_call`, `after_tool_call`
- `before_message_write` for mirrored transcript records
- `before_agent_finalize` through Codex `Stop` relay
- `agent_end`

Plugins can also register runtime-neutral tool-result middleware to rewrite
OpenClaw dynamic tool results after OpenClaw executes the tool and before the
result is returned to Codex. This is separate from the public
`tool_result_persist` plugin hook, which transforms OpenClaw-owned transcript
tool-result writes.

For the plugin hook semantics themselves, see [Plugin hooks](/plugins/hooks)
and [Plugin guard behavior](/tools/plugin).

The harness is off by default. New configs should keep OpenAI model refs
canonical as `openai/gpt-*` and explicitly force
`agentRuntime.id: "codex"` or `OPENCLAW_AGENT_RUNTIME=codex` when they
want native app-server execution. Legacy `codex/*` model refs still auto-select
the harness for compatibility, but runtime-backed legacy provider prefixes are
not shown as normal model/provider choices.

If the `codex` plugin is enabled but the primary model is still
`openai-codex/*`, `openclaw doctor` warns instead of changing the route. That is
intentional: `openai-codex/*` remains the PI Codex OAuth/subscription path, and
native app-server execution stays an explicit runtime choice.

## Route map

Use this table before changing config:

| Desired behavior                            | Model ref                  | Runtime config                         | Plugin requirement          | Expected status label          |
| ------------------------------------------- | -------------------------- | -------------------------------------- | --------------------------- | ------------------------------ |
| OpenAI API through normal OpenClaw runner   | `openai/gpt-*`             | omitted or `runtime: "pi"`             | OpenAI provider             | `Runtime: OpenClaw Pi Default` |
| Codex OAuth/subscription through PI         | `openai-codex/gpt-*`       | omitted or `runtime: "pi"`             | OpenAI Codex OAuth provider | `Runtime: OpenClaw Pi Default` |
| Native Codex app-server embedded turns      | `openai/gpt-*`             | `agentRuntime.id: "codex"`             | `codex` plugin              | `Runtime: OpenAI Codex`        |
| Mixed providers with conservative auto mode | provider-specific refs     | `agentRuntime.id: "auto"`              | Optional plugin runtimes    | Depends on selected runtime    |
| Explicit Codex ACP adapter session          | ACP prompt/model dependent | `sessions_spawn` with `runtime: "acp"` | healthy `acpx` backend      | ACP task/session status        |

The important split is provider versus runtime:

- `openai-codex/*` answers "which provider/auth route should PI use?"
- `agentRuntime.id: "codex"` answers "which loop should execute this
  embedded turn?"
- `/codex ...` answers "which native Codex conversation should this chat bind
  or control?"
- ACP answers "which external harness process should acpx launch?"

## Pick the right model prefix

OpenAI-family routes are prefix-specific. Use `openai-codex/*` when you want
Codex OAuth through PI; use `openai/*` when you want direct OpenAI API access or
when you are forcing the native Codex app-server harness:

| Model ref                                     | Runtime path                                 | Use when                                                                  |
| --------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------- |
| `openai/gpt-5.4`                              | OpenAI provider through OpenClaw/PI plumbing | You want current direct OpenAI Platform API access with `OPENAI_API_KEY`. |
| `openai-codex/gpt-5.5`                        | OpenAI Codex OAuth through OpenClaw/PI       | You want ChatGPT/Codex subscription auth with the default PI runner.      |
| `openai/gpt-5.5` + `agentRuntime.id: "codex"` | Codex app-server harness                     | You want native Codex app-server execution for the embedded agent turn.   |

GPT-5.5 is currently subscription/OAuth-only in OpenClaw. Use
`openai-codex/gpt-5.5` for PI OAuth, or `openai/gpt-5.5` with the Codex
app-server harness. Direct API-key access for `openai/gpt-5.5` is supported
once OpenAI enables GPT-5.5 on the public API.

Legacy `codex/gpt-*` refs remain accepted as compatibility aliases. Doctor
compatibility migration rewrites legacy primary runtime refs to canonical model
refs and records the runtime policy separately, while fallback-only legacy refs
are left unchanged because runtime is configured for the whole agent container.
New PI Codex OAuth configs should use `openai-codex/gpt-*`; new native
app-server harness configs should use `openai/gpt-*` plus
`agentRuntime.id: "codex"`.

`agents.defaults.imageModel` follows the same prefix split. Use
`openai-codex/gpt-*` when image understanding should run through the OpenAI
Codex OAuth provider path. Use `codex/gpt-*` when image understanding should run
through a bounded Codex app-server turn. The Codex app-server model must
advertise image input support; text-only Codex models fail before the media turn
starts.

Use `/status` to confirm the effective harness for the current session. If the
selection is surprising, enable debug logging for the `agents/harness` subsystem
and inspect the gateway's structured `agent harness selected` record. It
includes the selected harness id, selection reason, runtime/fallback policy, and,
in `auto` mode, each plugin candidate's support result.

### What doctor warnings mean

`openclaw doctor` warns when all of these are true:

- the bundled `codex` plugin is enabled or allowed
- an agent's primary model is `openai-codex/*`
- that agent's effective runtime is not `codex`

That warning exists because users often expect "Codex plugin enabled" to imply
"native Codex app-server runtime." OpenClaw does not make that leap. The warning
means:

- **No change is required** if you intended ChatGPT/Codex OAuth through PI.
-

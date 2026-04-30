---
domain: concepts
topic: "Agent Runtime Modes: embedded-pi, Delegate Architecture, and CLI Backends"
type: concept
keywords:
  - agent runtime
  - embedded-pi
  - runtime modes
  - agentRuntime
  - delegate architecture
  - CLI backends
related:
  - concepts/delegate-architecture
  - gateway/config-agents-reference
  - concepts/agent-loop
source: concepts/agent-runtimes.md
---

OpenClaw supports multiple agent runtime modes. The default is `embedded-pi` (Pi agent embedded in the Gateway process). Other runtimes add isolation or alternative execution models.

## Runtime Modes

An **agent runtime** is the component that owns one prepared model loop: it
receives the prompt, drives model output, handles native tool calls, and returns
the finished turn to OpenClaw.

Runtimes are easy to confuse with providers because both show up near model
configuration. They are different layers:

| Layer         | Examples                              | What it means                                                       |
| ------------- | ------------------------------------- | ------------------------------------------------------------------- |
| Provider      | `openai`, `anthropic`, `openai-codex` | How OpenClaw authenticates, discovers models, and names model refs. |
| Model         | `gpt-5.5`, `claude-opus-4-6`          | The model selected for the agent turn.                              |
| Agent runtime | `pi`, `codex`, `claude-cli`           | The low level loop or backend that executes the prepared turn.      |
| Channel       | Telegram, Discord, Slack, WhatsApp    | Where messages enter and leave OpenClaw.                            |

You will also see the word **harness** in code. A harness is the implementation
that provides an agent runtime. For example, the bundled Codex harness
implements the `codex` runtime. Public config uses `agentRuntime.id`; `openclaw
doctor --fix` rewrites older runtime-policy keys to that shape.

There are two runtime families:

- **Embedded harnesses** run inside OpenClaw's prepared agent loop. Today this
  is the built-in `pi` runtime plus registered plugin harnesses such as
  `codex`.
- **CLI backends** run a local CLI process while keeping the model ref
  canonical. For example, `anthropic/claude-opus-4-7` with
  `agentRuntime.id: "claude-cli"` means "select the Anthropic model, execute
  through Claude CLI." `claude-cli` is not an embedded harness id and must not
  be passed to AgentHarness selection.

## Three things named Codex

Most confusion comes from three different surfaces sharing the Codex name:

| Surface                                              | OpenClaw name/config                 | What it does                                                                                        |
| ---------------------------------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------- |
| Codex OAuth provider route                           | `openai-codex/*` model refs          | Uses ChatGPT/Codex subscription OAuth through the normal OpenClaw PI runner.                        |
| Native Codex app-server runtime                      | `agentRuntime.id: "codex"`           | Runs the embedded agent turn through the bundled Codex app-server harness.                          |
| Codex ACP adapter                                    | `runtime: "acp"`, `agentId: "codex"` | Runs Codex through the external ACP/acpx control plane. Use only when ACP/acpx is explicitly asked. |
| Native Codex chat-control command set                | `/codex ...`                         | Binds, resumes, steers, stops, and inspects Codex app-server threads from chat.                     |
| OpenAI Platform API route for GPT/Codex-style models | `openai/*` model refs                | Uses OpenAI API-key auth unless a runtime override, such as `runtime: "codex"`, runs the turn.      |

Those surfaces are intentionally independent. Enabling the `codex` plugin makes
the native app-server features available; it does not rewrite
`openai-codex/*` into `openai/*`, does not change existing sessions, and does
not make ACP the Codex default. Selecting `openai-codex/*` means "use the Codex
OAuth provider route" unless you separately force a runtime.

The common Codex setup uses the `openai` provider with the `codex` runtime:

```json5
{
  agents: {
    defaults: {
      model: "openai/gpt-5.5",
      agentRuntime: {
        id: "codex",
      },
    },
  },
}
```

That means OpenClaw selects an OpenAI model ref, then asks the Codex app-server
runtime to run the embedded agent turn. It does not mean the channel, model
provider catalog, or OpenClaw session store becomes Codex.

When the bundled `codex` plugin is enabled, natural-language Codex control
should use the native `/codex` command surface (`/codex bind`, `/codex threads`,
`/codex resume`, `/codex steer`, `/codex stop`) instead of ACP. Use ACP for
Codex only when the user explicitly asks for ACP/acpx or is testing the ACP
adapter path. Claude Code, Gemini CLI, OpenCode, Cursor, and similar external
harnesses still use ACP.

This is the agent-facing decision tree:

1. If the user asks for **Codex bind/control/thread/resume/steer/stop**, use the
   native `/codex` command surface when the bundled `codex` plugin is enabled.
2. If the user asks for **Codex as the embedded runtime**, use
   `openai/<model>` with `agentRuntime.id: "codex"`.
3. If the user asks for **Codex OAuth/subscription auth on the normal OpenClaw
   runner**, use `openai-codex/<model>` and leave the runtime as PI.
4. If the user explicitly says **ACP**, **acpx**, or **Codex ACP adapter**, use
   ACP with `runtime: "acp"` and `agentId: "codex"`.
5. If the request is for **Claude Code, Gemini CLI, OpenCode, Cursor, Droid, or
   another external harness**, use ACP/acpx, not the native sub-agent runtime.

| You mean...                             | Use...                                       |
| --------------------------------------- | -------------------------------------------- |
| Codex app-server chat/thread control    | `/codex ...` from the bundled `codex` plugin |
| Codex app-server embedded agent runtime | `agentRuntime.id: "codex"`                   |
| OpenAI Codex OAuth on the PI runner     | `openai-codex/*` model refs                  |
| Claude Code or other external harness   | ACP/acpx                                     |

For the OpenAI-family prefix split, see [OpenAI](/providers/openai) and
[Model providers](/concepts/model-providers). For the Codex runtime support
contract, see [Codex harness](/plugins/codex-harness#v1-support-contract).

## Runtime ownership

Different runtimes own different amounts of the loop.

| Surface                     | OpenClaw PI embedded                    | Codex app-server                                                            |
| --------------------------- | --------------------------------------- | --------------------------------------------------------------------------- |
| Model loop owner            | OpenClaw through the PI embedded runner | Codex app-server                                                            |
| Canonical thread state      | OpenClaw transcript                     | Codex thread, plus OpenClaw transcript mirror                               |
| OpenClaw dynamic tools      | Native OpenClaw tool loop               | Bridged through the Codex adapter                                           |
| Native shell and file tools | PI/OpenC

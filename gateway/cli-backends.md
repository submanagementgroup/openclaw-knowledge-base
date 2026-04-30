---
domain: gateway
topic: "CLI Backends: Local AI CLI Fallback with MCP Loopback Bridge"
type: reference
keywords:
  - CLI backends
  - codex-cli
  - claude-cli
  - google-gemini-cli
  - cliBackends
  - local AI fallback
  - MCP loopback bridge
  - bundleMcp
  - claude-stdio
  - JSONL streaming
  - model fallback
  - sessionArg
  - modelAliases
source: gateway/cli-backends.md
related:
  - gateway/local-models
  - gateway/configuration-overview
  - tools/acp-agents
---

OpenClaw can run local AI CLIs as a text-only fallback when API providers are down, rate-limited, or temporarily misbehaving. CLI backends do not receive OpenClaw tool calls directly, but backends with `bundleMcp: true` can receive gateway tools via a loopback MCP bridge.

CLI backends are a **safety net** rather than a primary path. For a full harness runtime with ACP session controls, background tasks, and persistent external coding sessions, use [ACP Agents](/tools/acp-agents) instead.

## Quick Start (Codex CLI, No Config Required)

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.5
```

If the gateway runs under launchd/systemd and PATH is minimal, add just the command path:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

If you explicitly reference a bundled CLI backend in config, OpenClaw auto-loads the owning bundled plugin.

## Using as a Fallback

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.5"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.5": {},
      },
    },
  },
}
```

If you use `agents.defaults.models` (allowlist), include your CLI backend models there too.

## Configuration Overview

All CLI backends live under `agents.defaults.cliBackends`. Each entry is keyed by a **provider id** (e.g. `codex-cli`). The provider id becomes the left side of model refs: `<provider>/<model>`.

### Full Config Example

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",        // "json", "jsonl", or "text"
          input: "arg",          // "arg" or "stdin"
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing", // "always", "existing", or "none"
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## How It Works

1. **Selects a backend** based on the provider prefix (`codex-cli/...`).
2. **Builds a system prompt** using the same OpenClaw prompt + workspace context.
3. **Executes the CLI** with a session id (if supported) so history stays consistent.
4. **Parses output** (JSON, JSONL, or plain text) and returns the final text.
5. **Persists session ids** per backend, so follow-ups reuse the same CLI session.

## Bundled Backends

### `codex-cli` (Bundled OpenAI Plugin Default)

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","-c","sandbox_mode=\"workspace-write\"","--skip-git-repo-check"]`
- `output: "jsonl"`, `resumeOutput: "text"`, `modelArg: "--model"`, `imageArg: "--image"`, `sessionMode: "existing"`

OpenClaw writes the assembled system prompt to a temporary file via Codex's `model_instructions_file` config override (`-c model_instructions_file="..."`).

### `claude-cli` (Bundled Anthropic Plugin)

`claude-cli` defaults to `liveSession: "claude-stdio"`, `output: "jsonl"`, and `input: "stdin"`. Follow-up turns reuse the live Claude process. If the gateway restarts or the idle process exits, OpenClaw resumes from the stored Claude session id (verified against an existing readable project transcript before resume).

**Prerequisites:**

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

OpenClaw maps exec policy to Claude's noninteractive permission mode: when effective policy is YOLO (`tools.exec.security: "full"` and `tools.exec.ask: "off"`), OpenClaw adds `--permission-mode bypassPermissions`.

OpenClaw passes skills two ways: the compact catalog in the appended system prompt, and a temporary Claude Code plugin dir with eligible skills filtered for that agent/session.

### `google-gemini-cli` (Bundled Google Plugin)

- `command: "gemini"`, `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`, `imagePathScope: "workspace"`, `modelArg: "--model"`, `sessionMode: "existing"`
- Reply text is read from the `response` JSON field; usage falls back to `stats`

Prerequisite: `gemini` on PATH (`brew install gemini-cli` or `npm install -g @google/gemini-cli`).

## Sessions

- `sessionMode: "always"`: always send a session id (new UUID if none stored).
- `sessionMode: "existing"`: only send a session id if one was stored before.
- `sessionMode: "none"`: never send a session id.
- For CLIs with a resume subcommand: use `resumeArgs` (replaces `args`) and optionally `resumeOutput`.
- Stored CLI sessions are provider-owned. `/reset` and explicit `session.reset` policies cut them; the implicit daily session reset does not.

## Fallback Prelude from claude-cli Sessions

When a `claude-cli` attempt fails over to a non-CLI candidate in `agents.defaults.model.fallbacks`, OpenClaw seeds the next attempt with a context prelude harvested from Claude Code's local JSONL transcript at `~/.claude/projects/`. The prelude prefers the latest `/compact` summary, then appends the most recent post-boundary turns up to a char budget. Tool blocks are coalesced to compact `(tool call: name)` and `(tool result: …)` hints.

## Bundle MCP Overlays

CLI backends with `bundleMcp: true` receive an MCP config overlay:

- `claude-cli`: generated strict MCP config file
- `codex-cli`: inline `mcp_servers` overrides with per-server approval mode so MCP calls cannot stall on local approval prompts
- `google-gemini-cli`: generated Gemini system settings file

OpenClaw spawns a loopback HTTP MCP server, authenticates with a per-session token (`OPENCLAW_MCP_TOKEN`), and scopes tool access to the current session, account, and channel context. Session-scoped bundled MCP runtimes are cached for `mcp.sessionIdleTtlMs` milliseconds (default 10 minutes).

## Images (Pass-Through)

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw writes base64 images to temp files. If `imageArg` is set, those paths are passed as CLI args. If `imageArg` is missing, OpenClaw appends file paths to the prompt (path injection).

## Inputs and Outputs

- `output: "json"`: parse JSON and extract text + session id
- `output: "jsonl"`: parse JSONL streams (e.g. Codex CLI `--json`)
- `output: "text"`: treat stdout as the final response
- `input: "arg"` (default): pass prompt as last CLI arg
- `input: "stdin"`: send prompt via stdin; also used when prompt exceeds `maxPromptArgChars`

For CLIs emitting Claude Code stream-json compatible JSONL, set `jsonlDialect: "claude-stream-json"`.

## Text Transforms (Plugin-Owned)

Plugins can declare bidirectional text transforms without replacing a provider:

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input` rewrites the system prompt and user prompt. `output` rewrites streamed assistant deltas before channel delivery.

## Limitations

- No direct OpenClaw tool calls — use `bundleMcp: true` for gateway tool access.
- Streaming is backend-specific.
- Codex CLI sessions resume via text output (less structured than initial `--json` run).

## Troubleshooting

- **CLI not found**: set `command` to a full path.
- **Wrong model name**: use `modelAliases` to map `provider/model` → CLI model.
- **No session continuity**: ensure `sessionArg` is set and `sessionMode` is not `none`.
- **Images ignored**: set `imageArg` and verify CLI supports file paths.

## Related

- [Local models](/gateway/local-models)
- [Gateway runbook](/gateway/gateway-runbook)
- [ACP Agents](/tools/acp-agents)

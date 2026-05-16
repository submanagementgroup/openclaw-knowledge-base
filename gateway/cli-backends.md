---
domain: gateway
topic: "CLI Backends Configuration"
type: reference
keywords:
  - cli backends
  - claude-cli
  - codex-cli
  - claude-code
  - cli backend config
  - cliBackends
source: gateway/cli-backends.md
---

OpenClaw can run **local AI CLIs** as a **text-only fallback** when API providers are down,
rate-limited, or temporarily misbehaving. This is intentionally conservative:

- **OpenClaw tools are not injected directly**, but backends with `bundleMcp: true`
  can receive gateway tools via a loopback MCP bridge.
- **JSONL streaming** for CLIs that support it.
- **Sessions are supported** (so follow-up turns stay coherent).
- **Images can be passed through** if the CLI accepts image paths.

This is designed as a **safety net** rather than a primary path. Use it when you
want "always works" text responses without relying on external APIs.

If you want a full harness runtime with ACP session controls, background tasks,
thread/conversation binding, and persistent external coding sessions, use
[ACP Agents](/tools/acp-agents) instead. CLI backends are not ACP.

> **Note:** Building a new backend plugin? Use
  [CLI backend plugins](/plugins/cli-backend-plugins). This page is for users
  configuring and operating an already registered backend.


## Beginner-friendly quick start

You can use Codex CLI **without any config** (the bundled OpenAI plugin
registers a default backend):

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.5
```

If your gateway runs under launchd/systemd and PATH is minimal, add just the
command path:

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

That's it. No keys, no extra auth config needed beyond the CLI itself.

If you use a bundled CLI backend as the **primary message provider** on a
gateway host, OpenClaw now auto-loads the owning bundled plugin when your config
explicitly references that backend in a model ref or under
`agents.defaults.cliBackends`.

## Using it as a fallback

Add a CLI backend to your fallback list so it only runs when primary models fail:

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

Notes:

- If you use `agents.defaults.models` (allowlist), you must include your CLI backend models there too.
- If the primary provider fails (auth, rate limits, timeouts), OpenClaw will
  try the CLI backend next.

## Configuration overview

All CLI backends live under:

```
agents.defaults.cliBackends
```

Each entry is keyed by a **provider id** (e.g. `codex-cli`, `my-cli`).
The provider id becomes the left side of your model ref:

```
<provider>/<model>
```

### Example configuration

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          // For CLIs with a dedicated prompt-file flag:
          // systemPromptFileArg: "--system-file",
          // Codex-style CLIs can point at a prompt file instead:
          // systemPromptFileConfigArg: "-c",
          // systemPromptFileConfigKey: "model_instructions_file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          // Opt in only if this backend may reseed safe invalidated sessions
          // from bounded raw OpenClaw transcript history before compaction.
          reseedFromRawTranscriptWhenUncompacted: true,
          serialize: true,
        },
      },
    },
  },
}
```

## How it works

1. **Selects a backend** based on the provider prefix (`codex-cli/...`).
2. **Builds a system prompt** using the same OpenClaw prompt + workspace context.
3. **Executes the CLI** with a session id (if supported) so history stays consistent.
   The bundled `claude-cli` backend keeps a Claude stdio process alive per
   OpenClaw session and sends follow-up turns over stream-json stdin.
4. **Parses output** (JSON or plain text) and returns the final text.
5. **Persists session ids** per backend, so follow-ups reuse the same CLI session.

> **Note:** The bundled Anthropic `claude-cli` backend is supported again. Anthropic staff
told us OpenClaw-style Claude CLI usage is allowed again, so OpenClaw treats
`claude -p` usage as sanctioned for this integration unless Anthropic publishes
a new policy.


The bundled OpenAI `codex-cli` backend passes OpenClaw's system prompt through
Codex's `model_instructions_file` config override (`-c
model_instructions_file="..."`). Codex does not expose a Claude-style
`--append-system-prompt` flag, so OpenClaw writes the assembled prompt to a
temporary file for each fresh Codex CLI session.

The bundled Anthropic `claude-cli` backend receives the OpenClaw skills snapshot
two ways: the compact OpenClaw skills catalog in the appended system prompt, and
a temporary Claude Code plugin passed with `--plugin-dir`. The plugin contains
only the eligible skills for that agent/session, so Claude Code's native skill
resolver sees the same filtered set that OpenClaw would otherwise advertise in
the prompt. Skill env/API key overrides are still applied by OpenClaw to the
child process environment for the run.

Claude CLI also has its own noninteractive permission mode. OpenClaw maps that
to the existing exec policy instead of adding Claude-specific config: when the
effective requested exec policy is YOLO (`tools.exec.security: "full"` and
`tools.exec.ask: "off"`), OpenClaw adds `--permission-mode bypassPermissions`.
Per-agent `agents.list[].tools.exec` settings override global `tools.exec` for
that agent. To force a different Claude mode, set explicit raw backend args
such as `--permission-mode default` or `--permission-mode acceptEdits` under
`agents.defaults.cliBackends.claude-cli.args` and matching `resumeArgs`.

The bundled Anthropic `claude-cli` backend also maps OpenClaw `/think` levels
to Claude Code's native `--effort` flag for non-off levels. `minimal` and
`low` map to `low`, `adaptive` and `medium` map to `medium`, and `high`,
`xhigh`, and `max` map directly. Other CLI backends need their owning plugin to
declare an equivalent argv mapper before `/think` can affect the spawned CLI.

Before OpenClaw can use the bundled `claude-cli` backend, Claude Code itself
must already be logged in on the same host:

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Use `agents.defaults.cliBackends.claude-cli.command` only when the `claude`
binary is not already on `PATH`.

## Sessions

- If the CLI supports sessions, set `sessionArg` (e.g. `--session-id`) or
  `sessionArgs` (placeholder `{sessionId}`) when the ID needs to be inserted
  into multiple flags.
- If the CLI uses a **resume subcommand** with different flags, set
  `resumeArgs` (replaces `args` when resuming) and optionally `resumeOutput`
  (for non-JSON resumes).
- `sessionMode`:
  - `always`: always send a session id (new UUID if none stored).
  - `existing`: only send a session id if one was stored before.
  - `none`: never send a session id.
- `claude-cli` defaults to `liveSession: "claude-stdio"`, `output: "jsonl"`,
  and `input: "stdin"` so follow-up turns reuse the live Claude process while
  it is active. Warm stdio is the default now, including for custom configs
  that omit transport fields. If the Gateway restarts or the idle process
  exits, OpenClaw resumes from the stored Claude session id. Stored session
  ids are verified against an existing readable project transcript before
  resume, so phantom bindings are cleared with `reason=transcript-missing`
  instead of silently starting a fresh Claude CLI session under `--resume`.
- Claude live sessions keep bounded JSONL output guards. Defaults allow up to
  8 MiB and 20,000 raw JSONL lines per turn. Tool-heavy Claude turns can raise
  them per backend with
  `agents.defaults.cliBackends.claude-cli.reliability.outputLimits.maxTurnRawChars`
  and `maxTurnLines`; OpenClaw clamps those settings to 64 MiB and 100,000
  lines.
- Stored CLI sessions are provider-owned continuity. The implicit daily session
  reset does not cut them; `/reset` and explicit `session.reset` policies still
  do.
- Fresh CLI sessions normally reseed only from OpenClaw's compaction summary
  plus post-compaction tail. To recover short sessions that are invalidated
  before compaction, a backend can opt in with
  `reseedFromRawTranscriptWhenUncompacted: true`. OpenClaw still keeps raw
  transcript reseed bounded and limits it to safe invalidations such as missing
  CLI transcripts, system-prompt/MCP changes, or session-expired retry; auth
  profile or credential-epoch changes never reseed raw transcript history.

Serialization notes:

- `serialize: true` keeps same-lane runs ordered.
- Most CLIs serialize on one provider lane.
- OpenClaw drops stored CLI session reuse when the selected auth identity changes,
  including a changed auth profile id, static API key, static token, or OAuth
  account identity when the CLI exposes one. OAuth access and refresh token
  rotation does not cut the stored CLI session. If a CLI does not expose a
  stable OAuth account id, OpenClaw lets that CLI enforce resume permissions.

## Fallback prelude from claude-cli sessions

When a `claude-cli` attempt fails over to a non-CLI candidate in
[`agents.defaults.model.fallbacks`](/concepts/model-failover), OpenClaw seeds
the next attempt with a context prelude harvested from Claude Code's local
JSONL transcript at `~/.claude/projects/`. Without this seed, the fallback
provider would start cold because OpenClaw's own session transcript is empty
for `claude-cli` runs.

- The prelude prefers the latest `/compact` summary or `compact_boundary`
  marker, then appends the most recent post-boundary turns up to a char
  budget. Pre-boundary turns are dropped because the summary already represents
  them.
- Tool blocks are coalesced to compact `(tool call: name)` and
  `(tool result: …)` hints to keep the prompt budget honest. The summary is
  labeled `(truncated)` if it overflows.
- Same-provider `claude-cli` to `claude-cli` fallbacks rely on Claude's own
  `--resume` and skip the prelude.
- The seed reuses the existing Claude session-file path validation, so
  arbitrary paths cannot be read.

## Images (pass-through)

If your CLI accepts image paths, set `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw will write base64 images to temp files. If `imageArg` is set, those
paths are passed as CLI args. If `imageArg` is missing, OpenClaw appends the
file paths to the prompt (path injection), which is enough for CLIs that auto-
load local files from plain paths.

## Inputs / outputs

- `output: "json"` (default) tries to parse JSON and extract text + session id.
- For Gemini CLI JSON output, OpenClaw reads reply text from `response` and
  usage from `stats` when `usage` is missing or empty.
- `output: "jsonl"` parses JSONL streams (for example Codex CLI `--json`) and extracts the final agent message plus session
  identifiers when present.
- `output: "text"` treats stdout as the final response.

Input modes:

- `input: "arg"` (default) passes the prompt as the last CLI arg.
- `input: "stdin"` sends the prompt via stdin.
- If the prompt is very long and `maxPromptArgChars` is set, stdin is used.

## Defaults (plugin-owned)

The bundled OpenAI plugin also registers a default for `codex-cli`:

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--s
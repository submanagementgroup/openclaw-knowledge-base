---
domain: tools
topic: "Tokenjuice: Compact Noisy Exec and Bash Tool Results"
type: concept
keywords:
  - tokenjuice
  - compact tool results
  - exec tool output
  - bash tool output
  - token compression
  - tool result middleware
  - openclaw plugins
  - tokenjuice enable
  - tokenjuice disable
source: tools/tokenjuice.md
related:
  - tools/exec
  - concepts/context-engine
---

`tokenjuice` is an optional bundled plugin that compacts noisy `exec` and `bash` tool results after the command runs. It changes the returned `tool_result` content — not the command itself, not exit codes, not shell input. It is opt-in and disabled by default.

## Enable the Plugin

```bash
# Fast path
openclaw config set plugins.entries.tokenjuice.enabled true

# Or via plugins command
openclaw plugins enable tokenjuice
```

No separate install step is needed. OpenClaw already ships the plugin.

Config file equivalent:

```json5
{
  plugins: {
    entries: {
      tokenjuice: {
        enabled: true,
      },
    },
  },
}
```

## What Tokenjuice Changes

- Compacts noisy `exec` and `bash` results before they are fed back into the session.
- Preserves exact file-content reads and other commands that should return verbatim output.
- Keeps the original command execution, exit code, and side effects untouched.
- Applies to PI embedded runs and OpenClaw dynamic tools in the Codex app-server harness.

## Verify It Is Working

1. Enable the plugin.
2. Start a session that can call `exec`.
3. Run a noisy command such as `git status`.
4. Confirm the returned tool result is shorter and more structured than raw shell output.

## Disable the Plugin

```bash
openclaw config set plugins.entries.tokenjuice.enabled false
# Or
openclaw plugins disable tokenjuice
```

## Related

- [Exec tool](/tools/exec)
- [Context engine](/concepts/context-engine)

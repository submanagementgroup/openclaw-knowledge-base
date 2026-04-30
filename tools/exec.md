---
domain: tools
topic: "Exec Tool: Shell Command Execution, Elevated Mode, and Code Execution"
type: procedure
keywords:
  - exec tool
  - shell commands
  - elevated exec
  - code execution
  - exec sandbox
  - system.run
related:
  - gateway/sandboxing
  - tools/exec-approvals
  - tools/tools-overview
source:
  - tools/exec.md
  - tools/elevated.md
  - tools/code-execution.md
---

The `exec` tool runs shell commands. It is the most powerful tool in OpenClaw and the main target for sandboxing. Elevated exec bypasses sandboxing.

## Quick Config

```json5
{
  tools: {
    enabled: ["exec"],
    elevated: ["exec"],   // bypass sandbox for exec (host access)
  }
}
```

## Exec Basics

Run shell commands in the workspace. Supports foreground + background execution via `process`.
If `process` is disallowed, `exec` runs synchronously and ignores `yieldMs`/`background`.
Background sessions are scoped per agent; `process` only sees sessions from the same agent.

## Parameters

<ParamField path="command" type="string" required>
Shell command to run.
</ParamField>

<ParamField path="workdir" type="string" default="cwd">
Working directory for the command.
</ParamField>

<ParamField path="env" type="object">
Key/value environment overrides merged on top of the inherited environment.
</ParamField>

<ParamField path="yieldMs" type="number" default="10000">
Auto-background the command after this delay (ms).
</ParamField>

<ParamField path="background" type="boolean" default="false">
Background the command immediately instead of waiting for `yieldMs`.
</ParamField>

<ParamField path="timeout" type="number" default="tools.exec.timeoutSec">
Override the configured exec timeout for this call. Set `timeout: 0` only when the command should run without the exec process timeout.
</ParamField>

<ParamField path="pty" type="boolean" default="false">
Run in a pseudo-terminal when available. Use for TTY-only CLIs, coding agents, and terminal UIs.
</ParamField>

<ParamField path="host" type="'auto' | 'sandbox' | 'gateway' | 'node'" default="auto">
Where to execute. `auto` resolves to `sandbox` when a sandbox runtime is active and `gateway` otherwise.
</ParamField>

<ParamField path="security" type="'deny' | 'allowlist' | 'full'">
Enforcement mode for `gateway` / `node` execution.
</ParamField>

<ParamField path="ask" type="'off' | 'on-miss' | 'always'">
Approval prompt behavior for `gateway` / `node` execution.
</ParamField>

<ParamField path="node" type="string">
Node id/name when `host=node`.
</ParamField>

<ParamField path="elevated" type="boolean" default="false">
Request elevated mode — escape the sandbox onto the configured host path. `security=full` is forced only when elevated resolves to `full`.
</ParamField>

Notes:

- `host` defaults to `auto`: sandbox when sandbox runtime is active for the session, otherwise gateway.
- `host` only accepts `auto`, `sandbox`, `gateway`, or `node`. It is not a hostname selector; hostname-like values are rejected before the command runs.
- `auto` is the default routing strategy, not a wildcard. Per-call `host=node` is allowed from `auto`; per-call `host=gateway` is only allowed when no sandbox runtime is active.
- With no extra config, `host=auto` still "just works": no sandbox means it resolves to `gateway`; a live sandbox means it stays in the sandbox.
- `elevated` escapes the sandbox onto the configured host path: `gateway` by default, or `node` when `tools.exec.host=node` (or the session default is `host=node`). It is only available when elevated access is enabled for the current session/provider.
- `gateway`/`node` approvals are controlled by `~/.openclaw/exec-approvals.json`.
- `node` requires a paired node (companion app or headless node host).
- If multiple nodes are available, set `exec.node` or `tools.exec.node` to select one.
- `exec host=node` is the only shell-execution path for nodes; the legacy `nodes.run` wrapper has been removed.
- `timeout` applies to foreground, background, `yieldMs`, gateway, sandbox, and node `system.run` execution. If omitted, OpenClaw uses `tools.exec.timeoutSec`; explicit `timeout: 0` disables the exec process timeout for that call.
- On non-Windows hosts, exec uses `SHELL` when set; if `SHELL` is `fish`, it prefers `bash` (or `sh`)
  from `PATH` to avoid fish-incompatible scripts, then falls back to `SHELL` if neither exists.
- On Windows hosts, exec prefers PowerShell 7 (`pwsh`) discovery (Program Files, ProgramW6432, then PATH),
  then falls back to Windows PowerShell 5.1.
- Host execution (`gateway`/`node`) rejects `env.PATH` and loader overrides (`LD_*`/`DYLD_*`) to
  prevent binary hijacking or injected code.
- OpenClaw sets `OPENCLAW_SHELL=exe

## Elevated Mode

When an agent runs inside a sandbox, its `exec` commands are confined to the
sandbox environment. **Elevated mode** lets the agent break out and run commands
outside the sandbox instead, with configurable approval gates.

  Elevated mode only changes behavior when the agent is **sandboxed**. For
  unsandboxed agents, exec already runs on the host.

## Directives

Control elevated mode per-session with slash commands:

| Directive        | What it does                                                           |
| ---------------- | ---------------------------------------------------------------------- |
| `/elevated on`   | Run outside the sandbox on the configured host path, keep approvals    |
| `/elevated ask`  | Same as `on` (alias)                                                   |
| `/elevated full` | Run outside the sandbox on the configured host path and skip approvals |
| `/elevated off`  | Return to sandbox-confined execution                                   |

Also available as `/elev on|off|ask|full`.

Send `/elevated` with no argument to see the current level.

## How it works

    Elevated must be enabled in config and the sender must be on the allowlist:

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

    Send a directive-only message to set the session default:

    ```
    /elevated full
    ```

    Or use it inline (applies to that message only):

    ```
    /elevated on run the deployment script
    ```

    With elevated active, `exec` calls leave the sandbox. The effective host is
    `gateway` by default, or `node` when the configured/session exec target is
    `node`. In `full` mode, exec approvals are skipped. In `on`/`ask` mode,
    configured approval rules still apply.

## Resolution order

1. **Inline directive** on the message (applies only to that message)

## Code Execution

`code_execution` runs sandboxed remote Python analysis on xAI's Responses API.
This is different from local [`exec`](/tools/exec):

- `exec` runs shell commands on your machine or node
- `code_execution` runs Python in xAI's remote sandbox

Use `code_execution` for:

- calculations
- tabulation
- quick statistics
- chart-style analysis
- analyzing data returned by `x_search` or `web_search`

Do **not** use it when you need local files, your shell, your repo, or paired
devices. Use [`exec`](/tools/exec) for that.

## Setup

You need an xAI API key. Any of these work:

- `XAI_API_KEY`
- `plugins.entries.xai.config.webSearch.apiKey`

Example:

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...",
          },
          codeExecution: {
            enabled: true,
            model: "grok-4-1-fast",
            maxTurns: 2,
            timeoutSeconds: 30,
          },
        },
      },
    },
  },
}
```

## How to use it

Ask naturally and make the analysis intent explicit:

```text
Use code_execution to calculate the 7-day moving average for these numbers: ...
```

```text
Use x_search to find posts mentioning OpenClaw this week, then use code_execution to count them by day.
```

```text
Use web_search to gather the latest AI benchmark numbers, then use code_execution to compare percent changes.
```

The tool takes a single `task` parameter internally, so the agent should send
the full analysis request and any inline data in one prompt.

## Limits

- This is remote xAI execution, not local process execution.
- It should be treated as ephemeral analysis, not a persistent notebook.
- Do not assume access to local files or your workspace.
- For fresh X data, use [`x_search`](/tools/web#x_search) first.

## Related

- [Exec tool](/tools/exec)
- [Exec approvals](/tools/exec-approvals)
- [apply_patch tool](/tools/apply-patch)
- [Web tools](/tools/web)
- [xAI](/providers/xai)

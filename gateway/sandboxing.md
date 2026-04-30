---
domain: gateway
topic: "Sandboxing: Docker/Podman Tool Isolation, Elevated Exec, and Sandbox Modes"
type: concept
keywords:
  - sandbox
  - docker sandbox
  - podman sandbox
  - tool isolation
  - exec sandbox
  - agents.defaults.sandbox
  - elevated
related:
  - tools/exec
  - tools/elevated
  - gateway/config-tools-reference
source:
  - gateway/sandboxing.md
  - gateway/sandbox-vs-tool-policy-vs-elevated.md
---

OpenClaw sandboxing runs tool execution (exec, read, write, edit, browser) inside an isolated container. Sandboxing is optional and configured via `agents.defaults.sandbox`. The Gateway itself always runs on the host.

## Quick Enable

```json5
{
  agents: {
    defaults: {
      sandbox: {
        enabled: true,
        mode: "docker",            // "docker" | "podman" | "process"
      }
    }
  }
}
```

## Sandbox Modes

| Mode | Description |
|------|-------------|
| `docker` | Docker container per session (most isolated) |
| `podman` | Podman container (rootless Docker alternative) |
| `process` | Process-level isolation (lighter, less secure) |
| off | Tools run directly on host (default) |

## What Gets Sandboxed

- Tool execution: `exec`, `read`, `write`, `edit`, `apply_patch`, `process`
- Optional sandboxed browser: `agents.defaults.sandbox.browser`

**Not sandboxed:**
- The Gateway process itself
- Tools in `tools.elevated` (run on host regardless)

## Elevated vs Sandboxed vs Tool Policy

- **Elevated**: tool runs on host (bypasses sandbox) — use for trusted operations that need host access
- **Sandboxed**: tool runs in isolated container
- **Tool policy**: access control (which tools the model can call), independent of sandbox

OpenClaw can run **tools inside sandbox backends** to reduce blast radius. This is **optional** and controlled by configuration (`agents.defaults.sandbox` or `agents.list[].sandbox`). If sandboxing is off, tools run on the host. The Gateway stays on the host; tool execution runs in an isolated sandbox when enabled.

This is not a perfect security boundary, but it materially limits filesystem and process access when the model does something dumb.

## What gets sandboxed

- Tool execution (`exec`, `read`, `write`, `edit`, `apply_patch`, `process`, etc.).
- Optional sandboxed browser (`agents.defaults.sandbox.browser`).

    - By default, the sandbox browser auto-starts (ensures CDP is reachable) when the browser tool needs it. Configure via `agents.defaults.sandbox.browser.autoStart` and `agents.defaults.sandbox.browser.autoStartTimeoutMs`.
    - By default, sandbox browser containers use a dedicated Docker network (`openclaw-sandbox-browser`) instead of the global `bridge` network. Configure with `agents.defaults.sandbox.browser.network`.
    - Optional `agents.defaults.sandbox.browser.cdpSourceRange` restricts container-edge CDP ingress with a CIDR allowlist (for example `172.21.0.1/32`).
    - noVNC observer access is password-protected by default; OpenClaw emits a short-lived token URL that serves a local bootstrap page and opens noVNC with password in URL fragment (not query/header logs).
    - `agents.defaults.sandbox.browser.allowHostControl` lets sandboxed sessions target the host browser explicitly.
    - Optional allowlists gate `target: "custom"`: `allowedControlUrls`, `allowedControlHosts`, `allowedControlPorts`.

Not sandboxed:

- The Gateway process itself.
- Any tool explicitly allowed to run outside the sandbox (e.g. `tools.elevated`).
  - **Elevated exec bypasses sandboxing and uses the configured escape path (`gateway` by default, or `node` when the exec target is `node`).**
  - If sandboxing is off, `tools.elevated` does not change execution (already on host). See [Elevated Mode](/tools/elevated).

## Modes

`agents.defaults.sandbox.mode` controls **when** sandboxing is used:

    No sandboxing.

    Sandbox only **non-main** sessions (default if you want normal chats on host).

    `"non-main"` is based on `session.mainKey` (default `"main"`), not agent id. Group/channel sessions use their own keys, so they count as non-main and will be sandboxed.

    Every session runs in a sandbox.

## Scope

`agents.defaults.sandbox.scope` controls **how many containers** are created:

- `"agent"` (default): one container per agent.
- `"session"`: one container per session.
- `"shared"`: one container shared by all sandboxed sessions.

## Backend

`agents.defaults.sandbox.backend` controls **which runtime** provides the sandbox:

- `"docker"` (default when sandboxing is enabled): local Docker-backed sandbox runtime.
- `"ssh"`: generic SSH-backed remote sandbox runtime.
- `"openshell"`: OpenShell-backed sandbox runtime.

SSH-specific config lives under `agents.defaults.sandbox.ssh`. OpenShell-specific config lives under `plugins.entries.openshell.config`.

### Choosing a backend

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **Where it runs**   | Local container                  | Any SSH-accessible host        | OpenShell managed sandbox                           |
| **Setup**           | `scripts/sandbox-setup.sh`       | SSH key + target host          | OpenShell plugin enabled                            |
| **Workspace model** | Bind-mount or copy               | Remote-canonical (seed once)   | `mirror` or `remote`                                |
| **Network control** | `docker.network` (default: none) | Depends on remote host         | Depends on OpenShell                 

OpenClaw has three related (but different) controls:

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.list[].sandbox.*`) decides **where tools run** (sandbox backend vs host).
2. **Tool policy** (`tools.*`, `tools.sandbox.tools.*`, `agents.list[].tools.*`) decides **which tools are available/allowed**.
3. **Elevated** (`tools.elevated.*`, `agents.list[].tools.elevated.*`) is an **exec-only escape hatch** to run outside the sandbox when you’re sandboxed (`gateway` by default, or `node` when the exec target is configured to `node`).

## Quick debug

Use the inspector to see what OpenClaw is _actually_ doing:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

It prints:

- effective sandbox mode/scope/workspace access
- whether the session is currently sandboxed (main vs non-main)
- effective sandbox tool allow/deny (and whether it came from agent/global/default)
- elevated gates and fix-it key paths

## Sandbox: where tools run

Sandboxing is controlled by `agents.defaults.sandbox.mode`:

- `"off"`: everything runs on the host.
- `"non-main"`: only non-main sessions are sandboxed (common “surprise” for groups/channels).
- `"all"`: everything is sandboxed.

See [Sandboxing](/gateway/sandboxing) for the full matrix (scope, workspace mounts, images).

### Bind mounts (security quick check)

- `docker.binds` _pierces_ the sandbox filesystem: whatever you mount is visible inside the container with the mode you set (`:ro` or `:rw`).
- Default is read-write if you omit the mode; prefer `:ro` for source/secrets.
- `scope: "shared"` ignores per-agent binds (only global binds apply).
- OpenClaw validates bind sources twice: first on the normalized source path, then again after resolving through the deepest existing ancestor. Symlink-parent escapes do not bypass blocked-path or allowed-root checks.
- Non-existent leaf paths are still checked safely. If `/workspace/a

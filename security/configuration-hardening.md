---
domain: security
topic: "Security Configuration Hardening: File Permissions, Secrets, Sandboxing, Per-Agent Profiles, and Insecure Flags"
type: reference
keywords:
  - file permissions
  - secrets on disk
  - sandboxing
  - per-agent access profiles
  - insecure flags
  - dangerous flags
  - workspace env
  - browser SSRF
  - read-only mode
  - incident response
  - secret scanning
source: gateway/security/index.md
related:
  - security/security-model
  - security/network-hardening
  - security/prompt-injection
  - gateway/sandboxing
  - gateway/secrets
  - gateway/logging
  - tools/exec-approvals
---

Security hardening configuration covers file permissions, secrets-on-disk hygiene, insecure/dangerous flag tracking, workspace `.env` restrictions, sandboxing, per-agent access profiles, browser SSRF policy, and incident response.

## File Permissions

Keep config + state private on the gateway host:
- `~/.openclaw/openclaw.json`: `600` (user read/write only)
- `~/.openclaw`: `700` (user only)

`openclaw doctor` can warn and offer to tighten these permissions.

## Secrets on Disk

Assume anything under `~/.openclaw/` (or `$OPENCLAW_STATE_DIR/`) may contain secrets or private data:

- `openclaw.json`: config may include tokens (gateway, remote gateway), provider settings, and allowlists.
- `credentials/**`: channel credentials (e.g., WhatsApp creds), pairing allowlists, legacy OAuth imports.
- `agents/<agentId>/agent/auth-profiles.json`: API keys, token profiles, OAuth tokens, optional `keyRef`/`tokenRef`.
- `agents/<agentId>/agent/codex-home/**`: per-agent Codex app-server account, config, skills, plugins, native thread state, and diagnostics.
- `secrets.json` (optional): file-backed secret payload used by `file` SecretRef providers.
- `agents/<agentId>/sessions/**`: session transcripts (`*.jsonl`) + routing metadata that can contain private messages and tool output.

Hardening tips:
- Keep permissions tight (`700` on dirs, `600` on files).
- Use full-disk encryption on the gateway host.
- Prefer a dedicated OS user account for the Gateway if the host is shared.

## Workspace `.env` File Restrictions

OpenClaw loads workspace-local `.env` files for agents and tools, but never lets those files override gateway runtime controls:

- Any key starting with `OPENCLAW_*` is blocked from untrusted workspace `.env` files.
- Channel endpoint settings for Matrix, Mattermost, IRC, and Synology Chat are also blocked from workspace `.env` overrides. Endpoint env keys (`MATRIX_HOMESERVER`, `MATTERMOST_URL`, `IRC_HOST`, `SYNOLOGY_CHAT_INCOMING_URL`) must come from the gateway process environment or `env.shellEnv`.
- The block is fail-closed: new runtime-control variables added in future releases cannot be inherited from a checked-in or attacker-supplied `.env`.
- Trusted process/OS environment variables (the gateway's own shell, launchd/systemd unit, app bundle) still apply.

## Insecure or Dangerous Flags

`openclaw security audit` raises `config.insecure_or_dangerous_flags` when known insecure/dangerous debug switches are enabled. Keep these unset in production:

- `gateway.controlUi.allowInsecureAuth=true`
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`
- `gateway.controlUi.dangerouslyDisableDeviceAuth=true`
- `hooks.gmail.allowUnsafeExternalContent=true`
- `hooks.mappings[<index>].allowUnsafeExternalContent=true`
- `tools.exec.applyPatch.workspaceOnly=false`
- `plugins.entries.acpx.config.permissionMode=approve-all`

Full list of `dangerous*` / `dangerously*` config keys includes:

- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback`
- `gateway.controlUi.dangerouslyDisableDeviceAuth`
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `channels.<platform>.dangerouslyAllowNameMatching` (Discord, Slack, GoogleChat, MSTeams, Synology Chat, ZaloUser, IRC, Mattermost, etc.)
- `channels.telegram.network.dangerouslyAllowPrivateNetwork`
- `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets`
- `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources`
- `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin`

## Sandboxing (Recommended)

Two complementary approaches:

- **Run the full Gateway in Docker** (container boundary): see the Docker installation guide.
- **Tool sandbox** (`agents.defaults.sandbox`, host gateway + sandbox-isolated tools; Docker is the default backend).

To prevent cross-agent access, keep `agents.defaults.sandbox.scope` at `"agent"` (default) or `"session"` for stricter per-session isolation. `scope: "shared"` uses a single container or workspace.

Agent workspace access inside the sandbox:
- `sandbox.workspaceAccess: "none"` (default) — keeps the agent workspace off-limits.
- `sandbox.workspaceAccess: "ro"` — mounts the agent workspace read-only at `/agent`.
- `sandbox.workspaceAccess: "rw"` — mounts the agent workspace read/write at `/workspace`.

Extra `sandbox.docker.binds` are validated against normalized and canonicalized source paths. Parent-symlink tricks and canonical home aliases fail closed if they resolve into blocked roots such as `/etc`, `/var/run`, or credential directories.

## Browser SSRF Policy (Strict by Default)

OpenClaw's browser navigation policy is strict by default: private/internal destinations stay blocked unless you explicitly opt in.

- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` is unset by default — private/internal/special-use destinations are blocked.
- Use `hostnameAllowlist` (patterns like `*.example.com`) and `allowedHostnames` (exact host exceptions) for explicit exceptions.

Example strict policy:
```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"],
    },
  },
}
```

## Logs and Transcripts (Redaction and Retention)

- Keep log and transcript redaction on (`logging.redactSensitive: "tools"`; default).
- Add custom patterns via `logging.redactPatterns` (tokens, hostnames, internal URLs).
- When sharing diagnostics, prefer `openclaw status --all` (pasteable, secrets redacted) over raw logs.
- Prune old session transcripts and log files if you don't need long retention.

## Per-Agent Access Profiles

With multi-agent routing, each agent can have its own sandbox + tool policy.

**Full access (no sandbox):**
```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

**Read-only tools + read-only workspace:**
```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "ro",
        },
        tools: {
          allow: ["read"],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

**No filesystem/shell access (provider messaging allowed):**
```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "none",
        },
        tools: {
          sessions: { visibility: "tree" }, // self | tree | agent | all
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

## Read-Only Mode

Build a read-only profile by combining:
- `agents.defaults.sandbox.workspaceAccess: "ro"` (or `"none"`)
- Tool allow/deny lists blocking `write`, `edit`, `apply_patch`, `exec`, `process`, etc.
- `tools.exec.applyPatch.workspaceOnly: true` (default): ensures `apply_patch` cannot write/delete outside the workspace directory.
- `tools.fs.workspaceOnly: true` (optional): restricts `read`/`write`/`edit`/`apply_patch` paths to the workspace directory.

## Incident Response

### Contain
1. Stop the macOS app (if it supervises the Gateway) or terminate your `openclaw gateway` process.
2. Set `gateway.bind: "loopback"` (or disable Tailscale Funnel/Serve) until you understand what happened.
3. Switch risky DMs/groups to `dmPolicy: "disabled"` / require mentions, and remove `"*"` allow-all entries.

### Rotate (assume compromise if secrets leaked)
1. Rotate Gateway auth (`gateway.auth.token` / `OPENCLAW_GATEWAY_PASSWORD`) and restart.
2. Rotate remote client secrets (`gateway.remote.token` / `.password`) on any machine that can call the Gateway.
3. Rotate provider/API credentials (WhatsApp creds, Slack/Discord tokens, model/API keys in `auth-profiles.json`, encrypted secrets payload values).

### Audit
1. Check Gateway logs: `/tmp/openclaw/openclaw-YYYY-MM-DD.log` (or `logging.file`).
2. Review relevant transcript(s): `~/.openclaw/agents/<agentId>/sessions/*.jsonl`.
3. Review recent config changes (especially: `gateway.bind`, `gateway.auth`, dm/group policies, `tools.elevated`, plugin changes).
4. Re-run `openclaw security audit --deep` and confirm critical findings are resolved.

### Collect for a Report
- Timestamp, gateway host OS + OpenClaw version
- The session transcript(s) + a short log tail (after redacting)
- What the attacker sent + what the agent did
- Whether the Gateway was exposed beyond loopback (LAN/Tailscale Funnel/Serve)

## Sub-Agent Delegation Guardrail

If you allow session tools, treat delegated sub-agent runs as another boundary decision:
- Deny `sessions_spawn` unless the agent truly needs delegation.
- Keep `agents.defaults.subagents.allowAgents` and per-agent `agents.list[].subagents.allowAgents` overrides restricted to known-safe target agents.
- For any workflow that must remain sandboxed, call `sessions_spawn` with `sandbox: "require"` (default is `inherit`). `sandbox: "require"` fails fast when the target child runtime is not sandboxed.

## Secret Scanning (CI)

CI runs the pre-commit `detect-private-key` hook over the repository. If it fails, remove or rotate the committed key material, then reproduce locally:

```bash
pre-commit run --all-files detect-private-key
```

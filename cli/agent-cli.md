---
domain: cli
topic: "Agent CLI: openclaw agent and openclaw agents Commands"
type: reference
keywords:
  - openclaw agent
  - agent CLI
  - agent run
  - openclaw agents
  - agent management
related:
  - cli/infer-cli
  - concepts/agent-loop
  - cli/cli-overview
source:
  - cli/agent.md
  - cli/agents.md
---

The `openclaw agent` command runs a full agent turn in an existing session.

# `openclaw agent`

Run an agent turn via the Gateway (use `--local` for embedded).
Use `--agent <id>` to target a configured agent directly.

Pass at least one session selector:

- `--to <dest>`
- `--session-id <id>`
- `--agent <id>`

Related:

- Agent send tool: [Agent send](/tools/agent-send)

## Options

- `-m, --message <text>`: required message body
- `-t, --to <dest>`: recipient used to derive the session key
- `--session-id <id>`: explicit session id
- `--agent <id>`: agent id; overrides routing bindings
- `--model <id>`: model override for this run (`provider/model` or model id)
- `--thinking <level>`: agent thinking level (`off`, `minimal`, `low`, `medium`, `high`, plus provider-supported custom levels such as `xhigh`, `adaptive`, or `max`)
- `--verbose <on|off>`: persist verbose level for the session
- `--channel <channel>`: delivery channel; omit to use the main session channel
- `--reply-to <target>`: delivery target override
- `--reply-channel <channel>`: delivery channel override
- `--reply-account <id>`: delivery account override
- `--local`: run the embedded agent directly (after plugin registry preload)
- `--deliver`: send the reply back to the selected channel/target
- `--timeout <seconds>`: override agent timeout (default 600 or config value)
- `--json`: output JSON

## Examples

```bash
openclaw agent --to +15555550123 --message "status update" --deliver
openclaw agent --agent ops --message "Summarize logs"
openclaw agent --agent ops --model openai/gpt-5.4 --message "Summarize logs"
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json
openclaw agent --agent ops --message "Generate report" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "Run locally" --local
```

## Notes

- Gateway mode falls back to the embedded agent when the Gateway request fails. Use `--local` to force embedded execution up front.
- `--local` still preloads the plugin registry first, so plugin-provided providers, tools, and channels stay available during embedded runs.
- `--local` and embedded fallback runs are treated as one-shot runs. Bundled MCP loopback resources and warm Claude stdio sessions opened for that local process are retired after the reply, so scripted invocations do not keep local child processes alive.
- Gateway-backed runs leave Gateway-owned MCP loopback resources under the running Gateway process; older clients may still send the historical cleanup flag, but the Gateway accepts it as a compatibility no-op.
- `--channel`, `--reply-channel`, and `--reply-account` affect reply delivery, not session routing.
- `--json` keeps stdout reserved for the JSON response. Gateway, plugin, and embedded-fallback diagnostics are routed to stderr so scripts can parse stdout directly.
- Embedded fallback JSON includes `meta.transport: "embedded"` and `meta.fallbackFrom: "gateway"` so scripts can distinguish fallback runs from Gateway runs.
- If the Gateway accepts an agent run but the CLI times out waiting for the final reply, embedded fallback uses a fresh explicit `gateway-fallback-*` session/run id and reports `meta.fallbackReason: "gateway_timeout"` plus the fallback session fields. This avoids racing the Gateway-owned transcript lock or silently replacing the original routed conversation session.
- When this command triggers `models.json` regeneration, SecretRef-managed provider credentials are persisted as non-secret markers (for example env var names, `secretref-env:ENV_VAR_NAME`, or `secretref-managed`), not resolved secret plaintext.
- Marker writes are source-authoritative: OpenClaw persists markers from the active source config snapshot, not from resolved runtime secret values.

## Related

- [CLI reference](/cli)
- [Agent runtime](/concepts/agent)

## Managing Agents (openclaw agents)

# `openclaw agents`

Manage isolated agents (workspaces + auth + routing).

Related:

- [Multi-agent routing](/concepts/multi-agent)
- [Agent workspace](/concepts/agent-workspace)
- [Skills config](/tools/skills-config): skill visibility configuration.

## Examples

```bash
openclaw agents list
openclaw agents list --bindings
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add ops --workspace ~/.openclaw/workspace-ops --bind telegram:ops --non-interactive
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## Routing bindings

Use routing bindings to pin inbound channel traffic to a specific agent.

If you also want different visible skills per agent, configure `agents.defaults.skills` and `agents.list[].skills` in `openclaw.json`. See [Skills config](/tools/skills-config) and [Configuration reference](/gateway/config-agents#agents-defaults-skills).

List bindings:

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

Add bindings:

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

If you omit `accountId` (`--bind <channel>`), OpenClaw resolves it from channel defaults and plugin setup hooks when available.

If you omit `--agent` for `bind` or `unbind`, OpenClaw targets the current default agent.

### Binding scope behavior

- A binding without `accountId` matches the channel default account only.
- `accountId: "*"` is the channel-wide fallback (all accounts) and is less specific than an explicit account binding.
- If the same agent already has a matching channel binding without `accountId`, and you later bind with an explicit or resolved `accountId`, OpenClaw upgrades that existing binding in place instead of adding a duplicate.

Example:

```bash
# initial channel-only binding
openclaw agents bind --agent work --bind telegram

# later upgrade to account-scoped binding
openclaw agents bind --agent work --bind telegram:ops
```

After the upgrade, routing for that binding is scoped to `telegram:ops`. If you also want default-account routing, add it explicitly (for example `--bind telegram:default`).

Remove bindings:

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

`unbind` accepts either `--all` or one or more `--bind` values, not both.

## Command surface

### `agents`

Running `openclaw agents` with no subcommand is equivalent to `openclaw agents list`.

### `agents list`

Options:

- `--json`
- `--bindings`: include full routing rules, not only per-agent counts/summaries

### `agents add [name]`

Options:

- `--workspace <dir>`
- `--model <id>`
- `--agent-dir <dir>`
- `--bind <channel[:accountId

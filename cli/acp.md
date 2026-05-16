---
domain: cli
topic: "CLI: openclaw acp Commands"
type: reference
keywords:
  - ACP
  - openclaw acp
  - acp CLI
  - agent communication
  - ACP protocol CLI
source: cli/acp.md
---

Run the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) bridge that talks to an OpenClaw Gateway.

This command speaks ACP over stdio for IDEs and forwards prompts to the Gateway
over WebSocket. It keeps ACP sessions mapped to Gateway session keys.

`openclaw acp` is a Gateway-backed ACP bridge, not a full ACP-native editor
runtime. It focuses on session routing, prompt delivery, and basic streaming
updates.

If you want an external MCP client to talk directly to OpenClaw channel
conversations instead of hosting an ACP harness session, use
[`openclaw mcp serve`](/cli/mcp) instead.

## What this is not

This page is often confused with ACP harness sessions.

`openclaw acp` means:

- OpenClaw acts as an ACP server
- an IDE or ACP client connects to OpenClaw
- OpenClaw forwards that work into a Gateway session

This is different from [ACP Agents](/tools/acp-agents), where OpenClaw runs an
external harness such as Codex or Claude Code through `acpx`.

Quick rule:

- editor/client wants to talk ACP to OpenClaw: use `openclaw acp`
- OpenClaw should launch Codex/Claude/Gemini as an ACP harness: use `/acp spawn` and [ACP Agents](/tools/acp-agents)

## Compatibility Matrix

| ACP area                                                              | Status      | Notes                                                                                                                                                                                                                                            |
| --------------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `initialize`, `newSession`, `prompt`, `cancel`                        | Implemented | Core bridge flow over stdio to Gateway chat/send + abort.                                                                                                                                                                                        |
| `listSessions`, slash commands                                        | Implemented | Session list works against Gateway session state with bounded cursor pagination and `cwd` filtering where Gateway session rows carry workspace metadata; commands are advertised via `available_commands_update`.                                |
| Session lineage metadata                                              | Implemented | Session listings and session info snapshots include OpenClaw parent and child lineage in `_meta` so ACP clients can render subagent graphs without private Gateway side channels.                                                                |
| `resumeSession`, `closeSession`                                       | Implemented | Resume rebinds an ACP session to an existing Gateway session without replaying history. Close cancels active bridge work, resolves pending prompts as cancelled, and releases bridge session state.                                              |
| `loadSession`                                                         | Partial     | Rebinds the ACP session to a Gateway session key and replays ACP event-ledger history for bridge-created sessions. Older/no-ledger sessions fall back to stored user/assistant text.                                                             |
| Prompt content (`text`, embedded `resource`, images)                  | Partial     | Text/resources are flattened into chat input; images become Gateway attachments.                                                                                                                                                                 |
| Session modes                                                         | Partial     | `session/set_mode` is supported and the bridge exposes initial Gateway-backed session controls for thought level, tool verbosity, reasoning, usage detail, and elevated actions. Broader ACP-native mode/config surfaces are still out of scope. |
| Session info and usage updates                                        | Partial     | The bridge emits `session_info_update` and best-effort `usage_update` notifications from cached Gateway session snapshots. Usage is approximate and only sent when Gateway token totals are marked fresh.                                        |
| Tool streaming                                                        | Partial     | `tool_call` / `tool_call_update` events include raw I/O, text content, and best-effort file locations when Gateway tool args/results expose them. Embedded terminals and richer diff-native output are still not exposed.                        |
| Exec approvals                                                        | Partial     | Gateway exec approval prompts during active ACP prompt turns are relayed to the ACP client with `session/request_permission`.                                                                                                                    |
| Per-session MCP servers (`mcpServers`)                                | Unsupported | Bridge mode rejects per-session MCP server requests. Configure MCP on the OpenClaw gateway or agent instead.                                                                                                                                     |
| Client filesystem methods (`fs/read_text_file`, `fs/write_text_file`) | Unsupported | The bridge does not call ACP client filesystem methods.                                                                                                                                                                                          |
| Client terminal methods (`terminal/*`)                                | Unsupported | The bridge does not create ACP client terminals or stream terminal ids through tool calls.                                                                                                                                                       |
| Session plans / thought streaming                                     | Unsupported | The bridge currently emits output text and tool status, not ACP plan or thought updates.                                                                                                                                                         |

## Known Limitations

- `loadSession` can replay complete ACP event-ledger history only for
  bridge-created sessions. Older/no-ledger sessions still use transcript
  fallback and do not reconstruct historic tool calls or system notices.
- If multiple ACP clients share the same Gateway session key, event and cancel
  routing are best-effort rather than strictly isolated per client. Prefer the
  default isolated `acp:<uuid>` sessions when you need clean editor-local
  turns.
- Gateway stop states are translated into ACP stop reasons, but that mapping is
  less expressive than a fully ACP-native runtime.
- Initial session controls currently surface a focused subset of Gateway knobs:
  thought level, tool verbosity, reasoning, usage detail, and elevated
  actions. Model selection and exec-host controls are not yet exposed as ACP
  config options.
- `session_info_update` and `usage_update` are derived from Gateway session
  snapshots, not live ACP-native runtime accounting. Usage is approximate,
  carries no cost data, and is only emitted when the Gateway marks total token
  data as fresh.
- Tool follow-along data is best-effort. The bridge can surface file paths that
  appear in known tool args/results, but it does not yet emit ACP terminals or
  structured file diffs.
- Exec approval relay is scoped to the active ACP prompt turn; approvals from
  other Gateway sessions are ignored.

## Usage

```bash
openclaw acp

# Remote Gateway
openclaw acp --url wss://gateway-host:18789 --token <token>

# Remote Gateway (token from file)
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Attach to an existing session key
openclaw acp --session agent:main:main

# Attach by label (must already exist)
openclaw acp --session-label "support inbox"

# Reset the session key before the first prompt
openclaw acp --session agent:main:main --reset-session
```

## ACP client (debug)

Use the built-in ACP client to sanity-check the bridge without an IDE.
It spawns the ACP bridge and lets you type prompts interactively.

```bash
openclaw acp client

# Point the spawned bridge at a remote Gateway
openclaw acp client --server-args --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Override the server command (default: openclaw)
openclaw acp client --server "node" --server-args openclaw.mjs acp --url ws://127.0.0.1:19001
```

Permission model (client debug mode):

- Auto-approval is allowlist-based and only applies to trusted core tool IDs.
- `read` auto-approval is scoped to the current working directory (`--cwd` when set).
- ACP only auto-approves narrow readonly classes: scoped `read` calls under the active cwd plus readonly search tools (`search`, `web_search`, `memory_search`). Unknown/non-core tools, out-of-scope reads, exec-capable tools, control-plane tools, mutating tools, and interactive flows always require explicit prompt approval.
- Server-provided `toolCall.kind` is treated as untrusted metadata (not an authorization source).
- This ACP bridge policy is separate from ACPX harness permissions. If you run OpenClaw through the `acpx` backend, `plugins.entries.acpx.config.permissionMode=approve-all` is the break-glass "yolo" switch for that harness session.

## Protocol smoke testing

For protocol-level debugging, start a Gateway with isolated state and drive
`openclaw acp` over stdio with an ACP JSON-RPC client. Cover `initialize`,
`session/new`, `session/list` with an absolute `cwd`, `session/resume`,
`session/close`, duplicate close, and missing resume.

The proof should include the advertised lifecycle capabilities, a Gateway-backed
session row, update notifications, and the Gateway `sessions.list` log:

```json
{
  "initialize": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "sessionCapabilities": {
        "list": {},
        "resume": {},
        "close": {}
      }
    }
  },
  "listSessions": {
    "sessions": [
      {
        "sessionId": "agent:main:acp-smoke",
        "cwd": "/path/to/workspace",
        "_meta": {
          "sessionKey": "agent:main:acp-smoke",
          "kind": "direct"
        }
      }
    ],
    "nextCursor": null
  },
  "notifications": ["session_info_update", "available_commands_update", "usage_update"],
  "gatewayLogTail": ["[gateway] ready", "[ws] ⇄ res ✓ sessions.list 305ms"]
}
```

Avoid using `openclaw gateway call sessions.list` as the only ACP proof. That
CLI path may request a fresh-token operator scope upgrade; ACP bridge
correctness is proven by ACP stdio frames plus the Gateway `sessions.list` log.

## How to use this

Use ACP when an IDE (or other client) speaks Agent Client Protocol and you want
it to drive an OpenClaw Gateway session.

1. Ensure the Gateway is running (local or remote).
2. Configure the Gateway target (config or flags).
3. Point your IDE to run `openclaw acp` over stdio.

Example config (persisted):

```bash
openclaw config set gateway.remote.url wss://gateway-host:18789
openclaw config set gateway.remote.token <token>
```

Example direct run (no config write):

```bash
openclaw acp --url wss://gateway-host:18789 --token <token>
# preferred for local process safety
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
```

## Selecting agents

ACP does not pick agents directly. It routes by the Gateway session key.

Use agent-scoped session keys to target a specific agent:

```bash
openclaw acp --session agent:main:main
openclaw acp --session agent:design:m
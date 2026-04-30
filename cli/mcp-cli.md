---
domain: cli
topic: "MCP CLI: Managing Model Context Protocol Servers with openclaw mcp"
type: reference
keywords:
  - MCP
  - openclaw mcp
  - MCP servers
  - Model Context Protocol
  - MCP CLI
related:
  - gateway/config-tools-reference
  - cli/cli-overview
source: cli/mcp.md
---

The `openclaw mcp` command manages MCP (Model Context Protocol) servers. Add, list, and test MCP server connections.

`openclaw mcp` has two jobs:

- run OpenClaw as an MCP server with `openclaw mcp serve`
- manage OpenClaw-owned outbound MCP server definitions with `list`, `show`, `set`, and `unset`

In other words:

- `serve` is OpenClaw acting as an MCP server
- `list` / `show` / `set` / `unset` is OpenClaw acting as an MCP client-side registry for other MCP servers its runtimes may consume later

Use [`openclaw acp`](/cli/acp) when OpenClaw should host a coding harness session itself and route that runtime through ACP.

## OpenClaw as an MCP server

This is the `openclaw mcp serve` path.

### When to use `serve`

Use `openclaw mcp serve` when:

- Codex, Claude Code, or another MCP client should talk directly to OpenClaw-backed channel conversations
- you already have a local or remote OpenClaw Gateway with routed sessions
- you want one MCP server that works across OpenClaw's channel backends instead of running separate per-channel bridges

Use [`openclaw acp`](/cli/acp) instead when OpenClaw should host the coding runtime itself and keep the agent session inside OpenClaw.

### How it works

`openclaw mcp serve` starts a stdio MCP server. The MCP client owns that process. While the client keeps the stdio session open, the bridge connects to a local or remote OpenClaw Gateway over WebSocket and exposes routed channel conversations over MCP.

    The MCP client spawns `openclaw mcp serve`.

    The bridge connects to the OpenClaw Gateway over WebSocket.

    Routed sessions become MCP conversations and transcript/history tools.

    Live events are queued in memory while the bridge is connected.

    If Claude channel mode is enabled, the same session can also receive Claude-specific push notifications.

    - live queue state starts when the bridge connects
    - older transcript history is read with `messages_read`
    - Claude push notifications only exist while the MCP session is alive
    - when the client disconnects, the bridge exits and the live queue is gone
    - one-shot agent entry points such as `openclaw agent` and `openclaw infer model run` retire any bundled MCP runtimes they open when the reply completes, so repeated scripted runs do not accumulate stdio MCP child processes
    - stdio MCP servers launched by OpenClaw (bundled or user-configured) are torn down as a process tree on shutdown, so child subprocesses started by the server do not survive after the parent stdio client exits
    - deleting or resetting a session disposes that session's MCP clients through the shared runtime cleanup path, so there are no lingering stdio connections tied to a removed session

### Choose a client mode

Use the same bridge in two different ways:

    Standard MCP tools only. Use `conversations_list`, `messages_read`, `events_poll`, `events_wait`, `messages_send`, and the approval tools.

    Standard MCP tools plus the Claude-specific channel adapter. Enable `--claude-channel-mode on` or leave the default `auto`.

Today, `auto` behaves the same as `on`. There is no client capability detection yet.

### What `serve` exposes

The bridge uses existing Gateway session route metadata to expose channel-backed conversations. A conversation appears when OpenClaw already has session state with a known route such as:

- `channel`
- recipient or destination metadata
- optional `accountId`
- optional `threadId`

This gives MCP clients one place to:

- list recent routed conversations
- read recent transcript history
- wait for new inbound events
- send a reply back through the same route
- see approval requests that arrive while the bridge is connected

### Usage

    ```bash
    openclaw mcp serve
    ```

    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
    ```

    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password
    ```

    ```bash
    openclaw mcp serve --verbose
    openclaw mcp serve --claude-channel-mode off
    ```

### Bridge tools

The current bridge exposes these MCP tools:

    Lists recent session-backed conversations that already have route metadata in Gateway session state.

    Useful filters:

    - `limit`
    - `search`
    - `channel`
    - `includeDerivedTitles`
    - `includeLastMessage`

    Returns one conversation by `session_key`.

    Reads recent transcript messages for one session-backed conversation.

    Extracts non-text message content blocks from one transcript message. This is a metadata view over transcript content, not a standalone durable attachment blob store.

    Reads queued live events since a numeric cursor.

    Long-polls until the next matching queued event arrives or a timeout expires.

    Use this when a generic MCP client needs near-real-time delivery without a Claude-specific push protocol.

    Sends text back through the same route already recorded on the session.

    Current behavior:

    - requires an existing conversation route
    - uses the session's channel, recipient, account id, and thread id
    - sends text only

    Lists pending exec/plugin approval requests the bridge has observed since it connected to the Gateway.

    Resolves one pending exec/plugin approval request with:

    - `allow-once`
    - `allow-always`
    - `deny`

### Event model

The bridge keeps an in-memory event queue while it is connected.

Current event types:

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

- the queue is live-only; it starts when the MCP bridge starts
- `events_poll` and `events_wait` do not replay older Gateway history by themselves
- durable backlog should be read with `messages_read`

### Claude channel notifications

The bridge can also expose Claude-specific channel notifications. This is the OpenClaw equivalent of a Claude Code channel adapter: standard MCP tools remain available, but live inbound messages can also arrive as Claude-specific MCP notifications.

    `--claude-channel-mode off`: standard MCP tools only.

    `--claude-channel-mode on`: enable Claude channel notifications.

    `--claude-channel-mode auto`: current default; same bridge behavior as `on`.

When Claude channel mode is enabled, the server advertises Claude experimental capabilities and can emit:

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

Current bridge behavior:

- inbound `user` transcript messages are forwarded as `notifications/claude/channel`
- Claude permission requests received over MCP are tracked in-memory
- if the linked conversation later sends `yes abcde` or `no abcde`, the bridge converts that to `notifications/claude/channel/permission`
- these notifications are live-session only; if the MCP client disconnects, there is no push target

This is intentionally client-specific. Generic MCP clients should rely on the standard polling tools.

### MCP client config

Example stdio client config:

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

For most generic MCP clients, start with the standard tool surface and ignore Claude mode. Turn Claude mode on only for clients that actually understand the Claude-specific notification methods.

### Options

`openclaw mcp serve` supports:

<ParamField path="--url" type="string">
  Gateway WebSocket URL.
</ParamField>
<ParamField path="--token" type="string">
  Gateway token.
</ParamField>
<ParamField path="--token-file" type="string">
  Read token from file.
</ParamField>
<ParamField path="--password" type="string">
  Gateway password.
</ParamField>
<ParamField path="--password-file" type="string">
  Read password from file.
</ParamField>
<ParamField path="--claude-channel-mode" type='"auto" | "on" | "off"'>
  Claude notification mode.
</ParamField>
<ParamField path="-v, --verbose" type="boolean">
  Verbose logs on stderr.
</ParamField>

Prefer `--token-file` or `--password-file` over inline secrets when possible.

### Security and trust boundary

The bridge does not invent routing. It only exposes conversations that Gateway already knows how to route.

That means:

- sender allowlists, pairing, and channel-level trust still belong to the underlying OpenClaw channel configuration
- `messages_send` can only reply through an existing stored route
- approval state is live/in-memory only for the current bridge session
- bridge auth should use the same Gateway token or password controls you would trust for any other remote Gateway client

If a conversation is missing from `conversations_list`, the usual cause is not MCP configuration. It is missing or incomplete route metadata in the underlying Gateway session.

### Testing

OpenClaw ships a deterministic Docker smoke for this bridge:

```bash
pnpm test:docke

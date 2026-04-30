---
domain: gateway
topic: "Tools Configuration Reference: Enabling Tools, MCP Servers, Elevated Exec, Web Search"
type: reference
keywords:
  - tools config
  - MCP servers
  - elevated tools
  - web search config
  - exec config
  - tool enable
  - tool disable
related:
  - gateway/configuration-overview
  - tools/exec
  - tools/skills
  - gateway/sandboxing
source: gateway/config-tools.md
---

Tools are configured under `tools` in `openclaw.json`. Enable or disable individual tools, set tool-level policies, configure MCP servers, and control exec/sandbox behavior.

## Enabling and Disabling Tools

```json5
{
  tools: {
    enabled: ["exec", "read", "write", "browser", "web_search"],  // allowlist
    disabled: ["exec"],  // or blocklist specific tools
  }
}
```

## Elevated Tools (Host-Level Exec)

Elevated tools bypass sandboxing and run on the host:
```json5
{
  tools: {
    elevated: ["exec"],   // allow exec to run outside sandbox
  }
}
```

## MCP Server Configuration

```json5
{
  mcp: {
    servers: {
      "my-server": {
        command: "npx",
        args: ["-y", "@my-scope/my-mcp-server"],
        env: { API_KEY: "{{secret:MY_API_KEY}}" },
      },
      "remote-server": {
        url: "https://my-mcp.example.com/sse",
      }
    }
  }
}
```

## Web Search Tool Config

```json5
{
  tools: {
    webSearch: {
      provider: "perplexity",  // or "brave", "duck", "exa", "tavily"
      apiKey: "YOUR_KEY",
    }
  }
}
```

`tools.*` config keys and custom provider / base-URL setup. For agents, channels, and other top-level config keys, see [Configuration reference](/gateway/configuration-reference).

## Tools

### Tool profiles

`tools.profile` sets a base allowlist before `tools.allow`/`tools.deny`:

Local onboarding defaults new local configs to `tools.profile: "coding"` when unset (existing explicit profiles are preserved).

| Profile     | Includes                                                                                                                        |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | `session_status` only                                                                                                           |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                       |
| `full`      | No restriction (same as unset)                                                                                                  |

### Tool groups

| Group              | Tools                                                                                                                   |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` is accepted as an alias for `exec`)                                         |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                  |
| `group:sessions`   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                           |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                   |
| `group:ui`         | `browser`, `canvas`                                                                                                     |
| `group:automation` | `cron`, `gateway`                                                                                                       |
| `group:messaging`  | `message`                                                                                                               |
| `group:nodes`      | `nodes`                                                                                                                 |
| `group:agents`     | `agents_list`                                                                                                           |
| `group:media`      | `image`, `image_generate`, `video_generate`, `tts`                                                                      |
| `group:openclaw`   | All built-in tools (excludes provider plugins)                                                                          |

### `tools.allow` / `tools.deny`

Global tool allow/deny policy (deny wins). Case-insensitive, supports `*` wildcards. Applied even when Docker sandbox is off.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

Further restrict tools for specific providers or models. Order: base profile → provider profile → allow/deny.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.elevated`

Controls elevated exec access outside the sandbox:

```json5
{
  tools: {
    elevat

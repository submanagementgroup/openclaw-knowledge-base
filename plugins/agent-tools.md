---
domain: plugins
topic: "Plugin Agent Tools: Registering and Providing Tools to the Agent"
type: reference
keywords:
  - agent tools
  - plugin tools
  - tool registration
  - plugin-provided tools
  - tool schema
  - registerTool
related:
  - plugins/plugin-architecture
  - tools/plugin-tool
  - plugins/building-plugins
source: plugins/agent-tools.md
---

Plugin agent tools allow plugins to register new tools that the agent can call during a session. Plugins declare available tools in their manifest and implement them using the SDK's `registerTool` API.

## Registering a Tool in a Plugin

```typescript
import { definePlugin } from "@openclaw/sdk";

export default definePlugin({
  id: "my-plugin",
  tools: [
    {
      name: "my_tool",
      description: "Does something useful",
      inputSchema: {
        type: "object",
        properties: {
          input: { type: "string", description: "Input text" },
        },
        required: ["input"],
      },
      handler: async (params, context) => {
        const { input } = params;
        // Do work
        return { result: `Processed: ${input}` };
      },
    },
  ],
});
```

## Tool Schema

Tool schemas follow JSON Schema with these extensions:
- `name`: tool identifier (snake_case, unique within plugin)
- `description`: shown to the model; determines when the tool is called
- `inputSchema`: JSON Schema for the tool's input object
- `handler`: async function receiving `(params, context)` and returning result

## Tool Context

The `context` object provides:
- `context.config` — current plugin config
- `context.session` — current session metadata
- `context.gateway` — gateway API for sending messages, reading config, etc.

## Declaring Tools in the Manifest

Tools must also be declared in `plugin.json` for discovery:

```json
{
  "id": "my-plugin",
  "tools": ["my_tool"]
}
```

## Plugin-Provided vs. Built-In Tools

Plugin tools extend the agent's tool set beyond the built-in tools (exec, browser, search, etc.). They appear alongside built-in tools in the model's tool list and are controlled by the same `tools.enabled`/`tools.disabled` config.

## Related

- [Building Plugins](/plugins/building-plugins) — full plugin development guide
- [Plugin Tool](/tools/plugin-tool) — how the agent interacts with plugin tools
- [Plugin Manifest](/plugins/plugin-manifest) — tool declaration in manifests

---
domain: plugins
topic: "Codex Computer Use Plugin: Desktop Control via Computer Use APIs"
type: reference
keywords:
  - computer use
  - Codex computer use
  - desktop control
  - computer vision
  - UI automation
related:
  - plugins/codex-harness
  - tools/browser
source: plugins/codex-computer-use.md
---

Codex computer use plugin: enables Codex to control the desktop via computer use APIs.

Computer Use is a Codex-native MCP plugin for local desktop control. OpenClaw
does not vendor the desktop app, execute desktop actions itself, or bypass
Codex permissions. The bundled `codex` plugin only prepares Codex app-server:
it enables Codex plugin support, finds or installs the configured Codex
Computer Use plugin, checks that the `computer-use` MCP server is available, and
then lets Codex own the native MCP tool calls during Codex-mode turns.

Use this page when OpenClaw is already using the native Codex harness. For the
runtime setup itself, see [Codex harness](/plugins/codex-harness).

## OpenClaw.app and Peekaboo

OpenClaw.app's Peekaboo integration is separate from Codex Computer Use. The
macOS app can host a PeekabooBridge socket so the `peekaboo` CLI can reuse the
app's local Accessibility and Screen Recording grants for Peekaboo's own
automation tools. That bridge does not install or proxy Codex Computer Use, and
Codex Computer Use does not call through the PeekabooBridge socket.

Use [Peekaboo bridge](/platforms/mac/peekaboo) when you want OpenClaw.app to be
a permission-aware host for Peekaboo CLI automation. Use this page when a
Codex-mode OpenClaw agent should have Codex's native `computer-use` MCP plugin
available before the turn starts.

## iOS app

The iOS app is separate from Codex Computer Use. It does not install or proxy
the Codex `computer-use` MCP server and it is not a desktop-control backend.
Instead, the iOS app connects as an OpenClaw node and exposes mobile
capabilities through node commands such as `canvas.*`, `camera.*`, `screen.*`,
`location.*`, and `talk.*`.

Use [iOS](/platforms/ios) when you want an agent to drive an iPhone node through
the gateway. Use this page when a Codex-mode agent should control the local
macOS desktop through Codex's native Computer Use plugin.

## Direct cua-driver MCP

Codex Computer Use is not the only way to expose desktop control. If you want
OpenClaw-managed runtimes to call TryCua's driver directly, use the upstream
`cua-driver mcp` server through OpenClaw's MCP registry instead of the
Codex-specific marketplace flow.

After installing `cua-driver`, either ask it for the OpenClaw command:

```bash
cua-driver mcp-config --client openclaw
```

or register the stdio server yourself:

```bash
openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
```

That path keeps the upstream MCP tool surface intact, including the driver
schemas and structured MCP responses. Use it when you want the CUA driver
available as a normal OpenClaw MCP server. Use the Codex Computer Use setup on
this page when Codex app-server should own plugin installation, MCP reloads,
and native tool calls inside Codex-mode turns.

CUA's driver is macOS-specific and still requires the local macOS permissions
that its app prompts for, such as Accessibility and Screen Recording. OpenClaw
does not install `cua-driver`, grant those permissions, or bypass the upstream
driver's safety model.

## Quick setup

Set `plugins.entries.codex.config.computerUse` when Codex-mode turns must have
Computer Use available before a thread starts:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          computerUse: {
            autoInstall: true,
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.5",
      agentRuntime: {
        id: "codex",
        fallback: "none",
      },
    },
  },
}
```

With this config, OpenClaw checks Codex app-server before each Codex-mode turn.
If Computer Use is missing but Codex app-server has already discovered an
installable marketplace, OpenClaw asks Codex app-server to install or re-enable
the plugin and reload MCP servers. On macOS, when no matching marketplace is
registered and the standard Codex app bundle exists, OpenClaw also tries to
register the bundled Codex marketplace from
`/Applications/Codex.app/Contents/Resources/plugins/openai-bundled` before it
fails. If setup still cannot make the MCP server available, the turn fails
before the thread starts.

Existing sessions keep their runtime and Codex thread binding. After changing
`agentRuntime` or Computer Use config, use `/new` or `/reset` in the affected
chat before testing.

## Commands

Use the `/codex computer-use` commands from any chat surface where the `codex`
plugin command surface is available. These are OpenClaw chat/runtime commands,
not `openclaw codex ...` CLI subcommands:

```text
/codex computer-use status
/codex computer-use install
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
/codex computer-use install --marketplace <name>
```

`status` is read-only. It does not add marketplace sources, install plugins, or
enable Codex plugin support.

`install` enables Codex app-server plugin support, optionally adds a configured
marketplace source, installs or re-enables the configured plugin through Codex
app-server, reloads MCP servers, and verifies that the MCP server exposes tools.

## Marketplace choices

OpenClaw uses the same app-server API that Codex itself exposes. The
marketplace fields choose where Codex should find `computer-use`.

| Field                | Use when                                                        | Install support                                          |
| -------------------- | --------------------------------------------------------------- | -------------------------------------------------------- |
| No marketplace field | You want Codex app-server to use marketplaces it already knows. | Yes, when app-server returns a local marketplace.        |
| `marketplaceSource`  | You have a Codex marketplace source app-server can add.         | Yes, for explicit `/codex computer-use install`.         |
| `marketplacePath`    | You already know the local marketplace file path on the host.   | Yes, for explicit install and turn-start auto-install.   |
| `marketplaceName`    | You want to select one already registered marketplace by name.  | Yes only when the selected marketplace has a local path. |

Fresh Codex homes may need a short moment to seed their official marketplaces.
During install, OpenClaw polls `plugin/list` for up to
`marketplaceDiscoveryTimeoutMs` milliseconds. The default is 60 seconds.

If multiple known marketplaces contain Computer Use, OpenClaw prefers
`openai-bundled`, then `openai-curated`, then `local`. Unknown ambiguous matches
fail closed and ask you to set `marketplaceName` or `marketplacePath`.

## Bundled macOS marketplace

Recent Codex desktop builds bundle Computer Use here:

```text
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
```

When `computerUse.autoInstall` is true and no marketplace containing
`computer-use` is registered, OpenClaw tries to add the standard bundled
marketplace root automatically:

```text
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

You can also register it explicitly from a shell with Codex:

```bash
codex plugin marketplace add /Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

If you use a nonstandard Codex app path, set `computerUse.marketplacePath` to a
local marketplace file path or run `/codex computer-use install --source
<marketplace-source>` once.

## Remote catalog limit

Codex app-server can list and read remote-only catalog entries, but it does not
currently support remote `plugin/install`. That means `marketplaceName` can
select a remote-only marketplace for status checks, but installs and re-enables
still need a local marketplace via `marketplaceSource` or `marketplacePath`.

If status says the plugin is available in a remote Codex marketplace but remote
install is unsupported, run install with a local source or path:

```text
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
```

## Configuration reference

| Field

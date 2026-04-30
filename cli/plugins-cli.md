---
domain: cli
topic: "Plugins CLI: openclaw plugins install, list, update, and info Commands"
type: reference
keywords:
  - openclaw plugins
  - plugins CLI
  - plugin install
  - plugin list
  - plugin management CLI
related:
  - plugins/plugin-architecture
  - cli/cli-overview
source: cli/plugins.md
---

The `openclaw plugins` command manages plugin installation, listing, and configuration.

```bash
openclaw plugins list              # list installed plugins
openclaw plugins install <id>      # install a plugin
openclaw plugins uninstall <id>    # remove a plugin
openclaw plugins update            # update all plugins
openclaw plugins info <id>         # plugin details
```

Manage Gateway plugins, hook packs, and compatible bundles.

    End-user guide for installing, enabling, and troubleshooting plugins.

    Bundle compatibility model.

    Manifest fields and config schema.

    Security hardening for plugin installs.

## Commands

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins install <path-or-spec>
openclaw plugins inspect <id>
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
openclaw plugins info <id>
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins uninstall <id>
openclaw plugins deps
openclaw plugins deps --repair
openclaw plugins deps --prune
openclaw plugins deps --json
openclaw plugins doctor
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins marketplace list <marketplace>
openclaw plugins marketplace list <marketplace> --json
```

For slow install, inspect, uninstall, or registry-refresh investigation, run the
command with `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1`. The trace writes phase timings
to stderr and keeps JSON output parseable. See [Debugging](/help/debugging#plugin-lifecycle-trace).

Bundled plugins ship with OpenClaw. Some are enabled by default (for example bundled model providers, bundled speech providers, and the bundled browser plugin); others require `plugins enable`.

Native OpenClaw plugins must ship `openclaw.plugin.json` with an inline JSON Schema (`configSchema`, even if empty). Compatible bundles use their own bundle manifests instead.

`plugins list` shows `Format: openclaw` or `Format: bundle`. Verbose list/info output also shows the bundle subtype (`codex`, `claude`, or `cursor`) plus detected bundle capabilities.

### Install

```bash
openclaw plugins install <package>                      # ClawHub first, then npm
openclaw plugins install clawhub:<package>              # ClawHub only
openclaw plugins install npm:<package>                  # npm only
openclaw plugins install <package> --force              # overwrite existing install
openclaw plugins install <package> --pin                # pin version
openclaw plugins install <package> --dangerously-force-unsafe-install
openclaw plugins install <path>                         # local path
openclaw plugins install <plugin>@<marketplace>         # marketplace
openclaw plugins install <plugin> --marketplace <name>  # marketplace (explicit)
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
```

Bare package names are checked against ClawHub first, then npm. Treat plugin installs like running code. Prefer pinned versions.

ClawHub is the primary distribution and discovery surface for most plugins. Npm
remains a supported fallback and direct-install path. During the migration to
ClawHub, OpenClaw still ships some OpenClaw-owned `@openclaw/*` plugin packages
on npm; those package versions can lag the bundled source between plugin release
trains. If npm reports an OpenClaw-owned plugin package as deprecated, that
published version is an old external artifact; use the plugin bundled with
current OpenClaw or a local checkout until a newer npm package is published.

    If your `plugins` section is backed by a single-file `$include`, `plugins install/update/enable/disable/uninstall` write through to that included file and leave `openclaw.json` untouched. Root includes, include arrays, and includes with sibling overrides fail closed instead of flattening. See [Config includes](/gateway/configuration) for the supported shapes.

    If config is invalid during install, `plugins install` normally fails closed and tells you to run `openclaw doctor --fix` first. During Gateway startup, invalid config for one plugin is isolated to that plugin so other channels and plugins can keep running; `openclaw doctor --fix` can quarantine the invalid plugin entry. The only documented install-time exception is a narrow bundled-plugin recovery path for plugins that explicitly opt into `openclaw.install.allowInvalidConfigRecovery`.

    `--force` reuses the existing install target and overwrites an already-installed plugin or hook pack in place. Use it when you are intentionally reinstalling the same id from a new local path, archive, ClawHub package, or npm artifact. For routine upgrades of an already tracked npm plugin, prefer `openclaw plugins update <id-or-npm-spec>`.

    If you run `plugins install` for a plugin id that is already installed, OpenClaw stops and points you at `plugins update <id-or-npm-spec>` for a normal upgrade, or at `plugins install <package> --force` when you genuinely want to overwrite the current install from a different source.

    `--pin` applies to npm installs only. It is not supported with `--marketplace`, because marketplace installs persist marketplace source metadata instead of an npm spec.

    `--dangerously-force-unsafe-install` is a break-glass option for false positives in the built-in dangerous-code scanner. It allows the install to continue even when the built-in scanner reports `critical` findings, but it does **not** bypass plugin `before_install` hook policy blocks and does **not** bypass scan failures.

    This CLI flag applies to plugin install/update flows. Gateway-backed skill dependency installs use the matching `dangerouslyForceUnsafeInstall` request override, while `openclaw skills install` remains a separate ClawHub skill download/install flow.

    If a plugin you published on ClawHub is blocked by a registry scan, use the publisher steps in [ClawHub](/tools/clawhub).

    `plugins install` is also the install surface for hook packs that expose `openclaw.hooks` in `package.json`. Use `openclaw hooks` for filtered hook visibility and per-hook enablement, not package installation.

    Npm specs are **registry-only** (package name + optional **exact version** or **dist-tag**). Git/URL/file specs and semver ranges are rejected. Dependency installs run project-local with `--ignore-scripts` for safety, even when your shell has global npm install settings.

    Use `npm:<package>` when you want to skip ClawHub lookup and install directly from npm. Bare package specs still prefer ClawHub and only fall back to npm when ClawHub does not have that package or version.

    Bare specs and `@latest` stay on the stable track. If npm resolves either of those to a prerelease, OpenClaw stops and asks you to opt in explicitly with a prerelease tag such as `@beta`/`@rc` or an exact prerelease version such as `@1.2.3-beta.4`.

    If a bare install spec matches a bundled plugin id (for example `diffs`), OpenClaw installs the bundled plugin directly. To install an npm package with the same name, use an explicit scoped spec (for example `@scope/diffs`).

    Supported archives: `.zip`, `.tgz`, `.tar.gz`, `.tar`. Native OpenClaw plugin archives must contain a valid `openclaw.plugin.json`

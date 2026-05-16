---
domain: cli
topic: "CLI: openclaw plugins Commands"
type: reference
keywords:
  - openclaw plugins
  - plugins CLI
  - plugin install
  - plugin list
  - plugin update
source: cli/plugins.md
---

Manage Gateway plugins, hook packs, and compatible bundles.

**Plugin system:** End-user guide for installing, enabling, and troubleshooting plugins.
  

**Manage plugins:** Quick examples for install, list, update, uninstall, and publishing.
  

**Plugin bundles:** Bundle compatibility model.
  

**Plugin manifest:** Manifest fields and config schema.
  

**Security:** Security hardening for plugin installs.
  

## Commands

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins search <query>
openclaw plugins search <query> --limit 20
openclaw plugins search <query> --json
openclaw plugins install <path-or-spec>
openclaw plugins inspect <id>
openclaw plugins inspect <id> --runtime
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
openclaw plugins info <id>
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins uninstall <id>
openclaw plugins doctor
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins marketplace list <marketplace>
openclaw plugins marketplace list <marketplace> --json
```

For slow install, inspect, uninstall, or registry-refresh investigation, run the
command with `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1`. The trace writes phase timings
to stderr and keeps JSON output parseable. See [Debugging](/help/debugging#plugin-lifecycle-trace).

> **Note:** In Nix mode (`OPENCLAW_NIX_MODE=1`), plugin lifecycle mutators are disabled. Use the Nix source for this install instead of `plugins install`, `plugins update`, `plugins uninstall`, `plugins enable`, or `plugins disable`; for nix-openclaw, use the agent-first [Quick Start](https://github.com/openclaw/nix-openclaw#quick-start).


> **Note:** Bundled plugins ship with OpenClaw. Some are enabled by default (for example bundled model providers, bundled speech providers, and the bundled browser plugin); others require `plugins enable`.

Native OpenClaw plugins must ship `openclaw.plugin.json` with an inline JSON Schema (`configSchema`, even if empty). Compatible bundles use their own bundle manifests instead.

`plugins list` shows `Format: openclaw` or `Format: bundle`. Verbose list/info output also shows the bundle subtype (`codex`, `claude`, or `cursor`) plus detected bundle capabilities.


### Install

```bash
openclaw plugins search "calendar"                   # search ClawHub plugins
openclaw plugins install <package>                      # npm by default
openclaw plugins install clawhub:<package>              # ClawHub only
openclaw plugins install npm:<package>                  # npm only
openclaw plugins install npm-pack:<path.tgz>            # local npm pack through npm install semantics
openclaw plugins install git:github.com/<owner>/<repo>  # git repo
openclaw plugins install git:github.com/<owner>/<repo>@<ref>
openclaw plugins install <package> --force              # overwrite existing install
openclaw plugins install <package> --pin                # pin version
openclaw plugins install <package> --dangerously-force-unsafe-install
openclaw plugins install <path>                         # local path
openclaw plugins install <plugin>@<marketplace>         # marketplace
openclaw plugins install <plugin> --marketplace <name>  # marketplace (explicit)
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
```

Maintainers testing setup-time installs can override automatic plugin install
sources with guarded environment variables. See
[Plugin install overrides](/plugins/install-overrides).

> **Note:** Bare package names install from npm by default during the launch cutover. Use `clawhub:<package>` for ClawHub. Treat plugin installs like running code. Prefer pinned versions.


`plugins search` queries ClawHub for installable plugin packages and prints
install-ready package names. It searches code-plugin and bundle-plugin packages,
not skills. Use `openclaw skills search` for ClawHub skills.

> **Note:** ClawHub is the primary distribution and discovery surface for most plugins. Npm
remains a supported fallback and direct-install path. OpenClaw-owned
`@openclaw/*` plugin packages are published on npm again; see the current list
on [npmjs.com/org/openclaw](https://www.npmjs.com/org/openclaw) or the
[plugin inventory](/plugins/plugin-inventory). Stable installs use `latest`.
Beta-channel installs and updates prefer the npm `beta` dist-tag when that tag
is available, then fall back to `latest`.


### Config includes and invalid-config repair

If your `plugins` section is backed by a single-file `$include`, `plugins install/update/enable/disable/uninstall` write through to that included file and leave `openclaw.json` untouched. Root includes, include arrays, and includes with sibling overrides fail closed instead of flattening. See [Config includes](/gateway/configuration) for the supported shapes.

    If config is invalid during install, `plugins install` normally fails closed and tells you to run `openclaw doctor --fix` first. During Gateway startup and hot reload, invalid plugin config fails closed like any other invalid config; `openclaw doctor --fix` can quarantine the invalid plugin entry. The only documented install-time exception is a narrow bundled-plugin recovery path for plugins that explicitly opt into `openclaw.install.allowInvalidConfigRecovery`.

  ### --force and reinstall vs update

`--force` reuses the existing install target and overwrites an already-installed plugin or hook pack in place. Use it when you are intentionally reinstalling the same id from a new local path, archive, ClawHub package, or npm artifact. For routine upgrades of an already tracked npm plugin, prefer `openclaw plugins update <id-or-npm-spec>`.

    If you run `plugins install` for a plugin id that is already installed, OpenClaw stops and points you at `plugins update <id-or-npm-spec>` for a normal upgrade, or at `plugins install <package> --force` when you genuinely want to overwrite the current install from a different source.

  ### --pin scope

`--pin` applies to npm installs only. It is not supported with `git:` installs; use an explicit git ref such as `git:github.com/acme/plugin@v1.2.3` when you want a pinned source. It is not supported with `--marketplace`, because marketplace installs persist marketplace source metadata instead of an npm spec.
  ### --dangerously-force-unsafe-install

`--dangerously-force-unsafe-install` is a break-glass option for false positives in the built-in dangerous-code scanner. It allows the install to continue even when the built-in scanner reports `critical` findings, but it does **not** bypass plugin `before_install` hook policy blocks and does **not** bypass scan failures.

    This CLI flag applies to plugin install/update flows. Gateway-backed skill dependency installs use the matching `dangerouslyForceUnsafeInstall` request override, while `openclaw skills install` remains a separate ClawHub skill download/install flow.

    If a plugin you published on ClawHub is hidden or blocked by a registry scan, use the publisher steps in [ClawHub publishing](/clawhub/publishing). `--dangerously-force-unsafe-install` only affects installs on your own machine; it does not ask ClawHub to rescan the plugin or make a blocked release public.

  ### Hook packs and npm specs

`plugins install` is also the install surface for hook packs that expose `openclaw.hooks` in `package.json`. Use `openclaw hooks` for filtered hook visibility and per-hook enablement, not package installation.

    Npm specs are **registry-only** (package name + optional **exact version** or **dist-tag**). Git/URL/file specs and semver ranges are rejected. Dependency installs run project-local with `--ignore-scripts` for safety, even when your shell has global npm install settings. Managed plugin npm roots inherit OpenClaw's package-level npm `overrides`, so host security pins apply to hoisted plugin dependencies too.

    Use `npm:<package>` when you want to make npm resolution explicit. Bare package specs also install directly from npm during the launch cutover.

    Bare specs and `@latest` stay on the stable track. OpenClaw date-stamped correction versions such as `2026.5.3-1` are stable releases for this check. If npm resolves either of those to a prerelease, OpenClaw stops and asks you to opt in explicitly with a prerelease tag such as `@beta`/`@rc` or an exact prerelease version such as `@1.2.3-beta.4`.

    If a bare install spec matches an official plugin id (for example `diffs`), OpenClaw installs the catalog entry directly. To install an npm package with the same name, use an explicit scoped spec (for example `@scope/diffs`).

  ### Git repositories

Use `git:<repo>` to install directly from a git repository. Supported forms include `git:github.com/owner/repo`, `git:owner/repo`, full `https://`, `ssh://`, `git://`, `file://`, and `git@host:owner/repo.git` clone URLs. Add `@<ref>` or `#<ref>` to check out a branch, tag, or commit before install.

    Git installs clone into a temporary directory, check out the requested ref when present, then use the normal plugin directory installer. That means manifest validation, dangerous-code scanning, package-manager install work, and install records behave like npm installs. Recorded git installs include the source URL/ref plus the resolved commit so `openclaw plugins update` can re-resolve the source later.

    After installing from git, use `openclaw plugins inspect <id> --runtime --json` to verify runtime registrations such as gateway methods and CLI commands. If the plugin registered a CLI root with `api.registerCli`, execute that command directly through the OpenClaw root CLI, for example `openclaw demo-plugin ping`.

  ### Archives

Supported archives: `.zip`, `.tgz`, `.tar.gz`, `.tar`. Native OpenClaw plugin archives must contain a valid `openclaw.plugin.json` at the extracted plugin root; archives that only contain `package.json` are rejected before OpenClaw writes install records.

    Use `npm-pack:<path.tgz>` when the file is an npm-pack tarball and you want
    to test the same managed npm-root install path used by registry installs,
    including `package-lock.json` verification, hoisted dependency scanning, and
    npm install records. Plain archive paths still install as local archives
    under the plugin extensions root.

    Claude marketplace installs are also supported.

  ClawHub installs use an explicit `clawhub:<package>` locator:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

Bare npm-safe plugin specs install from npm by default during the launch cutover:

```bash
openclaw plugins install openclaw-codex-app-server
```

Use `npm:` to make npm-only resolution explicit:

```bash
openclaw plugins install npm:openclaw-codex-app-server
openclaw plugins install npm:@scope/plugin-name@1.0.1
```

OpenClaw checks the advertised plugin API / minimum gateway compatibility before install. When the selected ClawHub version publishes a ClawPack artifact, OpenClaw downloads the versioned npm-pack `.tgz`, verifies the ClawHub digest header and the artifact digest, then installs it through the normal archive path. Older ClawHub versions without ClawPack metadata still install through the legacy package archive verification path. Recorded installs keep their ClawHub source metadata, artifact kind, npm integrity, npm shasum, tarball name, and ClawPack digest facts for later updates.
Unversioned ClawHub installs keep an unversioned recorded spec so `openclaw plugins update` can follow newer ClawHub releases; explicit version or tag selectors such as `clawhub:pkg@1.2.3` and `clawhub:pkg@beta` remain pinned to that selector.

#### Marketplace shorthand

Use `plugin@marketplace` shorthand when the marketplace name exists in Claude's local registry cache at `~/.claude/plugins/known_marketplaces.json`:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

Use `--marketplace` when you want to pass the marketplace source explicitly:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

**Marketplace sources:**

- a Claude known-marketplace name from `~/.claude/plugins/known_marketplaces.json`
    - a local marketplace root or `marketplace.json` path
    - a GitHub repo shorthand such as `owner/repo`
    - a GitHub repo URL such as `https://github.com/owner/repo`
    - a git URL

  
**Remote marketplace rules:**

For remote marketplaces loaded from GitHub or git, plugin entries must stay inside the cloned marketplace repo. OpenClaw accepts relative path sources from that repo and rejects HTTP(S), absolute-path, git, GitHub, and other non-path plugin sources from remote manifests.
  
For local paths and archives, OpenClaw auto-detects:

- native OpenClaw plugins (`openclaw.plugin.json`)
- Codex-compatible bundles (`.codex-plugin/plugin.json`)
- Claude-compatible bundles (`.claude-plugin/plugin.json` or the default Claude component layout)
- Cursor-compatible bundles (`.cursor-plugin/plugin.json`)

> **Note:** Compatible bundles install into the normal plugin root and participate in the same list/info/enable/disable flow. Today, bundle skills, Claude command-skills, Claude `settings.json` defaults, Claude `.lsp.json` / manifest-declared `lspServers` defaults, Cursor command-skills, and compatible Codex hook directories are supported; other detected bundle capabilities are shown in diagnostics/info but are not yet wired into runtime execution.


### List

```bash
openclaw plugins list
openclaw 
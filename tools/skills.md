---
domain: tools
topic: "Skills: Reusable Agent Instructions"
type: concept
keywords:
  - skills
  - skill
  - agent skills
  - skill files
  - skill directory
  - reusable prompts
source: tools/skills.md
---

OpenClaw uses **[AgentSkills](https://agentskills.io)-compatible** skill
folders to teach the agent how to use tools. Each skill is a directory
containing a `SKILL.md` with YAML frontmatter and instructions. OpenClaw
loads bundled skills plus optional local overrides, and filters them at
load time based on environment, config, and binary presence.

## Locations and precedence

OpenClaw loads skills from these sources, **highest precedence first**:

| #   | Source                | Path                             |
| --- | --------------------- | -------------------------------- |
| 1   | Workspace skills      | `<workspace>/skills`             |
| 2   | Project agent skills  | `<workspace>/.agents/skills`     |
| 3   | Personal agent skills | `~/.agents/skills`               |
| 4   | Managed/local skills  | `~/.openclaw/skills`             |
| 5   | Bundled skills        | shipped with the install         |
| 6   | Extra skill folders   | `skills.load.extraDirs` (config) |

If a skill name conflicts, the highest source wins.

Codex CLI's native `$CODEX_HOME/skills` directory is not one of these OpenClaw
skill roots. In Codex harness mode, local app-server launches use isolated
per-agent Codex homes, so personal Codex CLI skills are not loaded implicitly.
Use `openclaw migrate codex --dry-run` to inventory them and
`openclaw migrate codex` to choose skill directories with an interactive
checkbox prompt before copying them into the current OpenClaw agent workspace.
For non-interactive runs, repeat `--skill <name>` for the exact skills to copy.

## Per-agent vs shared skills

In **multi-agent** setups each agent has its own workspace:

| Scope                | Path                                        | Visible to                  |
| -------------------- | ------------------------------------------- | --------------------------- |
| Per-agent            | `<workspace>/skills`                        | Only that agent             |
| Project-agent        | `<workspace>/.agents/skills`                | Only that workspace's agent |
| Personal-agent       | `~/.agents/skills`                          | All agents on that machine  |
| Shared managed/local | `~/.openclaw/skills`                        | All agents on that machine  |
| Shared extra dirs    | `skills.load.extraDirs` (lowest precedence) | All agents on that machine  |

Same name in multiple places → highest source wins. Workspace beats
project-agent, beats personal-agent, beats managed/local, beats bundled,
beats extra dirs.

## Agent skill allowlists

Skill **location** and skill **visibility** are separate controls.
Location/precedence decides which copy of a same-named skill wins; agent
allowlists decide which skills an agent can actually use.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"],
    },
    list: [
      { id: "writer" }, // inherits github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults
      { id: "locked-down", skills: [] }, // no skills
    ],
  },
}
```

### Allowlist rules

- Omit `agents.defaults.skills` for unrestricted skills by default.
    - Omit `agents.list[].skills` to inherit `agents.defaults.skills`.
    - Set `agents.list[].skills: []` for no skills.
    - A non-empty `agents.list[].skills` list is the **final** set for that
      agent - it does not merge with defaults.
    - The effective allowlist applies across prompt building, skill
      slash-command discovery, sandbox sync, and skill snapshots.
  ## Plugins and skills

Plugins can ship their own skills by listing `skills` directories in
`openclaw.plugin.json` (paths relative to the plugin root). Plugin skills
load when the plugin is enabled. This is the right place for tool-specific
operating guides that are too long for the tool description but should be
available whenever the plugin is installed - for example, the browser
plugin ships a `browser-automation` skill for multi-step browser control.

Plugin skill directories are merged into the same low-precedence path as
`skills.load.extraDirs`, so a same-named bundled, managed, agent, or
workspace skill overrides them. You can gate them via
`metadata.openclaw.requires.config` on the plugin's config entry.

See [Plugins](/tools/plugin) for discovery/config and [Tools](/tools) for
the tool surface those skills teach.

## Skill Workshop

The optional, experimental **Skill Workshop** plugin can create or update
workspace skills from reusable procedures observed during agent work. It
is disabled by default and must be explicitly enabled via
`plugins.entries.skill-workshop`.

Skill Workshop writes only to `<workspace>/skills`, scans generated
content, supports pending approval or automatic safe writes, quarantines
unsafe proposals, and refreshes the skill snapshot after successful
writes so new skills become available without a Gateway restart.

Use it for corrections such as _"next time, verify GIF attribution"_ or
hard-won workflows such as media QA checklists. Start with pending
approval; use automatic writes only in trusted workspaces after reviewing
its proposals. Full guide: [Skill Workshop plugin](/plugins/skill-workshop).

## ClawHub (install and sync)

[ClawHub](https://clawhub.ai) is the public skills registry for OpenClaw.
Use native `openclaw skills` commands for discover/install/update, or the
separate `clawhub` CLI for publish/sync workflows. Full guide:
[ClawHub](/clawhub).

| Action                             | Command                                |
| ---------------------------------- | -------------------------------------- |
| Install a skill into the workspace | `openclaw skills install <skill-slug>` |
| Update all installed skills        | `openclaw skills update --all`         |
| Sync (scan + publish updates)      | `clawhub sync --all`                   |

Native `openclaw skills install` installs into the active workspace
`skills/` directory. The separate `clawhub` CLI also installs into
`./skills` under your current working directory (or falls back to the
configured OpenClaw workspace). OpenClaw picks that up as
`<workspace>/skills` on the next session.
Configured skill roots also support one grouping level, such as
`skills/<group>/<skill>/SKILL.md`, so related third-party skills can be
kept under a shared folder without broad recursive scanning.

Gateway clients that need private, non-ClawHub delivery can stage a zip skill
archive with `skills.upload.begin`, `skills.upload.chunk`, and
`skills.upload.commit`, then install the committed upload with
`skills.install({ source: "upload", uploadId, slug, force?, sha256? })`. This is
an explicit admin upload path for trusted clients, not the normal
`openclaw skills install <slug>` or ClawHub install flow. It is off by default
and only works when `skills.install.allowUploadedArchives: true` is set in
`openclaw.json`. Upload mode still installs into the default agent workspace
`skills/<slug>` directory; the archive's internal folder name is ignored for the
final install target.

ClawHub skill pages expose the latest security scan state before install,
with scanner detail pages for VirusTotal, ClawScan, and static analysis.
`openclaw skills install <slug>` remains only the install path; publishers
recover false positives through the ClawHub dashboard or
`clawhub skill rescan <slug>`.

## Security

> **Note:** Treat third-party skills as **untrusted code**. Read them before enabling.
Prefer sandboxed runs for untrusted inputs and risky tools. See
[Sandboxing](/gateway/sandboxing) for the agent-side controls.


- Workspace and extra-dir skill discovery only accepts skill roots and `SKILL.md` files whose resolved realpath stays inside the configured root.
- Gateway private archive installs are off by default. When explicitly enabled,
  they require a committed zip upload containing `SKILL.md` and reuse the same
  archive extraction, path traversal, symlink, force, and rollback protections as
  ClawHub skill installs. They are gated by
  `skills.install.allowUploadedArchives`; normal ClawHub installs do not require
  that setting.
- Gateway-backed skill dependency installs (`skills.install`, onboarding, and the Skills settings UI) run the built-in dangerous-code scanner before executing installer metadata. `critical` findings block by default unless the caller explicitly sets the dangerous override; suspicious findings still warn only.
- `openclaw skills install <slug>` is different - it downloads a ClawHub skill folder into the workspace and does not use the installer-metadata path above.
- `skills.entries.*.env` and `skills.entries.*.apiKey` inject secrets into the **host** process for that agent turn (not the sandbox). Keep secrets out of prompts and logs.

For a broader threat model and checklists, see [Security](/gateway/security).

## SKILL.md format

`SKILL.md` must include at least:

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
---
```

OpenClaw follows the AgentSkills spec for layout/intent. The parser used
by the embedded agent supports **single-line** frontmatter keys only;
`metadata` should be a **single-line JSON object**. Use `{baseDir}` in
instructions to reference the skill folder path.

### Optional frontmatter keys


  URL surfaced as "Website" in the macOS Skills UI. Also supported via `metadata.openclaw.homepage`.


  When `true`, the skill is exposed as a user slash command.


  When `true`, OpenClaw keeps the skill's instructions out of the agent's normal
  prompt. The skill is still installed and can still be run explicitly as a
  slash command when `user-invocable` is also `true`.


  When set to `tool`, the slash command bypasses the model and dispatches directly to a tool.


  Tool name to invoke when `command-dispatch: tool` is set.


  For tool dispatch, forwards the raw args string to the tool (no core parsing). The tool is invoked with `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`.


## Gating (load-time filters)

OpenClaw filters skills at load time using `metadata` (single-line JSON):

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

Fields under `metadata.openclaw`:


  When `true`, always include the skill (skip other gates).


  Optional emoji used by the macOS Skills UI.


  Optional URL shown as "Website" in the macOS Skills UI.


  Optional list of platforms. If set, the skill is only eligible on those OSes.


  Each must exist on `PATH`.


  At least one must exist on `PATH`.


  Env var must exist or be provided in config.


  List of `openclaw.json` paths that must be truthy.


  Env var name associated with `skills.entries.<name>.apiKey`.


  Optional installer specs used by the macOS Skills UI (brew/node/go/uv/download).


If no `metadata.openclaw` is present, the skill is always eligible (unless
disabled in config or blocked by `skills.allowBundled` for bundled skills).

> **Note:** Legacy `metadata.clawdbot` blocks are still accepted when
`metadata.openclaw` is absent, so older installed skills keep their
dependency gates and installer hints. New and updated skills should use
`metadata.openclaw`.


### Sandboxing notes

- `requires.bins` is checked on the **host** at skill load time.
- If an agent is sandboxed, the binary must also exist **inside the container**. Install it via `agents.defaults.sandbox.docker.setupCommand` (or a custom image). `setupCommand` runs once after the container is created. Package installs also require network egress, a writable root FS, and a root user in the sandbox.
- Example: the `summarize` skill (`skills/summarize/SKILL.md`) needs the `summarize` CLI in the sandbox container to run there.

### Installer specs

```markdown
---
name:
---
domain: tools
topic: "Skills: Contextual Instruction Files, Loading Skills, and Creating Custom Skills"
type: procedure
keywords:
  - skills
  - skill files
  - context files
  - skill loading
  - creating skills
  - skill workshop
  - agent skills
related:
  - tools/tools-overview
  - plugins/skill-workshop
  - gateway/config-tools-reference
source:
  - tools/skills.md
  - tools/skills-config.md
  - tools/creating-skills.md
---

Skills are contextual instruction files that the agent loads on demand. They extend agent capabilities for specific domains without permanently increasing the system prompt size.

## What Skills Are

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

    - Omit `agents.defaults.skills` for unrestricted skills by default.
    - Omit `agents.list[].skills` to inherit `agents.defaults.skills`.
    - Set `agents.list[].skills: []` for no skills.
    - A non-empty `agents.list[].skills` list is the **final** set for that
      agent — it does not merge with defaults.
    - The effective allowlist applies across prompt building, skill
      slash-command discovery, sandbox sync, and skill snapshots.

## Plugins and skills

Plugins can ship their own skills by listing `skills` directories in
`openclaw.plugin.json` (paths relative to the plugin root). Plugin skills
load when the plugin is enabled. This is the right place for tool-specific
operating guides that are too long for the tool description but should be
available whenever the plugin is installed — for example, the browser
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
is disabled by default and mu

## Skills Configuration

Most skills loader/install configuration lives under `skills` in
`~/.openclaw/openclaw.json`. Agent-specific skill visibility lives under
`agents.defaults.skills` and `agents.list[].skills`.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills", "~/Projects/oss/some-skill-pack/skills"],
      watch: true,
      watchDebounceMs: 250,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun (Gateway runtime still Node; bun not recommended)
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

For built-in image generation/editing, prefer `agents.defaults.imageGenerationModel`
plus the core `image_generate` tool. `skills.entries.*` is only for custom or
third-party skill workflows.

If you select a specific image provider/model, also configure that provider's
auth/API key. Typical examples: `GEMINI_API_KEY` or `GOOGLE_API_KEY` for
`google/*`, `OPENAI_API_KEY` for `openai/*`, and `FAL_KEY` for `fal/*`.

Examples:

- Native Nano Banana Pro-style setup: `agents.defaults.imageGenerationModel.primary: "google/gemini-3-pro-image-preview"`
- Native fal setup: `agents.defaults.imageGenerationModel.primary: "fal/fal-ai/flux/dev"`

## Agent skill allowlists

Use agent config when you want the same machine/workspace skill roots, but a
different visible skill set per agent.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"],
    },
    list: [
      { id: "writer" }, // inherits defaults -> github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults
      { id: "locked-down", skills: [] }, // no skills
    ],
  },
}
```

Rules:

- `agents.defau

## Creating Skills

Skills teach the agent how and when to use tools. Each skill is a directory
containing a `SKILL.md` file with YAML frontmatter and markdown instructions.

For how skills are loaded and prioritized, see [Skills](/tools/skills).

## Create your first skill

    Skills live in your workspace. Create a new folder:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

    Create `SKILL.md` inside that directory. The frontmatter defines metadata,
    and the markdown body contains instructions for the agent.

    ```markdown
    ---
    name: hello-world
    description: A simple skill that says hello.
    ---

    # Hello World Skill

    When the user asks for a greeting, use the `echo` tool to say
    "Hello from your custom skill!".
    ```

    Use hyphen-case with lowercase letters, digits, and hyphens for the skill
    `name`. Keep the folder name and frontmatter `name` aligned.

    You can define custom tool schemas in the frontmatter or instruct the agent
    to use existing system tools (like `exec` or `browser`). Skills can also
    ship inside plugins alongside the tools they document.

    Start a new session so OpenClaw picks up the skill:

    ```bash
    # From chat
    /new

    # Or restart the gateway
    openclaw gateway restart
    ```

    Verify the skill loaded:

    ```bash
    openclaw skills list
    ```

    Send a message that should trigger the skill:

    ```bash
    openclaw agent --message "give me a greeting"
    ```

    Or just chat with the agent and ask for a greeting.

## Skill metadata reference

The YAML frontmatter supports these fields:

| Field                               | Required | Description                                                    |
| ----------------------------------- | -------- | -------------------------------------------------------------- |
| `name`                              | Yes      | Unique identifier using lowercase letters, digits, and hyphens |
| `description`

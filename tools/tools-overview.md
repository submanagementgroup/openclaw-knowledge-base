---
domain: tools
topic: "Agent Tools Overview: Exec, File, Search, Browser, Media, and Communication Tools"
type: reference
keywords:
  - tools overview
  - agent tools
  - tool list
  - exec tool
  - browser tool
  - search tool
  - tools.enabled
related:
  - tools/exec
  - tools/browser
  - tools/skills
  - gateway/config-tools-reference
source: tools/index.md
---

OpenClaw provides a rich set of agent tools across execution, file, search, media, and communication categories. Tools are controlled via `tools.enabled`, `tools.disabled`, and sandboxing config.

## Tool Categories

### Execution Tools
- `exec` — run shell commands
- `apply_patch` — apply unified diffs
- `code_execution` — run code in sandbox

### File Tools  
- `read`, `write`, `edit` — file operations
- `pdf` — read PDF files

### Search Tools
- `web_search` — search via Perplexity, Brave, Exa, DuckDuckGo, Tavily
- `memory_search` — semantic memory search
- Various provider-specific search tools

### Browser Tools
- `browser` — Chromium browser control via CDP

### Media Tools
- `tts` — text to speech
- `image_generation` — generate images
- `video_generation` — generate video
- `music_generation` — generate music

### Communication Tools
- `agent_send` — send message to channel
- `reactions` — add reactions to messages
- `slash_commands` — slash command handling

### Skills and Knowledge
- `skills` — load/use skill files
- `memory_search` / `memory_get` — memory tools

Everything the agent does beyond generating text happens through **tools**.
Tools are how the agent reads files, runs commands, browses the web, sends
messages, and interacts with devices.

## Tools, skills, and plugins

OpenClaw has three layers that work together:

    A tool is a typed function the agent can invoke (e.g. `exec`, `browser`,
    `web_search`, `message`). OpenClaw ships a set of **built-in tools** and
    plugins can register additional ones.

    The agent sees tools as structured function definitions sent to the model API.

    A skill is a markdown file (`SKILL.md`) injected into the system prompt.
    Skills give the agent context, constraints, and step-by-step guidance for
    using tools effectively. Skills live in your workspace, in shared folders,
    or ship inside plugins.

    [Skills reference](/tools/skills) | [Creating skills](/tools/creating-skills)

    A plugin is a package that can register any combination of capabilities:
    channels, model providers, tools, skills, speech, realtime transcription,
    realtime voice, media understanding, image generation, video generation,
    web fetch, web search, and more. Some plugins are **core** (shipped with
    OpenClaw), others are **external** (published on npm by the community).

    [Install and configure plugins](/tools/plugin) | [Build your own](/plugins/building-plugins)

## Built-in tools

These tools ship with OpenClaw and are available without installing any plugins:

| Tool                                       | What it does                                                          | Page                                                         |
| ------------------------------------------ | --------------------------------------------------------------------- | ------------------------------------------------------------ |
| `exec` / `process`                         | Run shell commands, manage background processes                       | [Exec](/tools/exec), [Exec Approvals](/tools/exec-approvals) |
| `code_execution`                           | Run sandboxed remote Python analysis                                  | [Code Execution](/tools/code-execution)                      |
| `browser`                                  | Control a Chromium browser (navigate, click, screenshot)              | [Browser](/tools/browser)                                    |
| `web_search` / `x_search` / `web_fetch`    | Search the web, search X posts, fetch page content                    | [Web](/tools/web), [Web Fetch](/tools/web-fetch)             |
| `read` / `write` / `edit`                  | File I/O in the workspace                                             |                                                              |
| `apply_patch`                              | Multi-hunk file patches                                               | [Apply Patch](/tools/apply-patch)                            |
| `message`                                  | Send messages across all channels                                     | [Agent Send](/tools/agent-send)                              |
| `canvas`                                   | Drive node Canvas (present, eval, snapshot)                           |                                                              |
| `nodes`                                    | Discover and target paired devices                                    |                                                              |
| `cron` / `gateway`                         | Manage scheduled jobs; inspect, patch, restart, or update the gateway |                                                              |
| `image` / `image_generate`                 | Analyze or generate images                                            | [Image Generation](/tools/image-generation)                  |
| `music_generate`                           | Generate music tracks                                                 | [Music Generation](/tools/music-gen

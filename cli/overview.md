---
domain: cli
topic: "CLI Overview: All openclaw Commands"
type: reference
keywords:
  - CLI overview
  - openclaw CLI
  - command reference
  - CLI commands list
source: cli/index.md
---

`openclaw` is the main CLI entry point. Each core command has either a
dedicated reference page or is documented with the command it aliases; this
index lists the commands, the global flags, and the output styling rules that
apply across the CLI.

Use the setup commands by intent:

- `openclaw setup` creates the baseline config and workspace without walking the full guided onboarding flow.
- `openclaw onboard` is the full guided first-run path for gateway, model auth, workspace, channels, skills, and health.
- `openclaw configure` changes targeted parts of an existing setup, such as model auth, gateway, channels, plugins, or skills.
- `openclaw channels add` configures channel accounts after the baseline exists; run it without flags for guided channel setup or with channel-specific flags for scripts.

## Command pages

| Area                 | Commands                                                                                                                                                                                                                                  |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Setup and onboarding | [`crestodian`](/cli/crestodian) · [`setup`](/cli/setup) · [`onboard`](/cli/onboard) · [`configure`](/cli/configure) · [`config`](/cli/config) · [`completion`](/cli/completion) · [`doctor`](/cli/doctor) · [`dashboard`](/cli/dashboard) |
| Reset and uninstall  | [`backup`](/cli/backup) · [`reset`](/cli/reset) · [`uninstall`](/cli/uninstall) · [`update`](/cli/update)                                                                                                                                 |
| Messaging and agents | [`message`](/cli/message) · [`agent`](/cli/agent) · [`agents`](/cli/agents) · [`acp`](/cli/acp) · [`mcp`](/cli/mcp)                                                                                                                       |
| Health and sessions  | [`status`](/cli/status) · [`health`](/cli/health) · [`sessions`](/cli/sessions)                                                                                                                                                           |
| Gateway and logs     | [`gateway`](/cli/gateway) · [`logs`](/cli/logs) · [`system`](/cli/system)                                                                                                                                                                 |
| Models and inference | [`models`](/cli/models) · [`infer`](/cli/infer) · `capability` (alias for [`infer`](/cli/infer)) · [`memory`](/cli/memory) · [`commitments`](/cli/commitments) · [`wiki`](/cli/wiki)                                                      |
| Network and nodes    | [`directory`](/cli/directory) · [`nodes`](/cli/nodes) · [`devices`](/cli/devices) · [`node`](/cli/node)                                                                                                                                   |
| Runtime and sandbox  | [`approvals`](/cli/approvals) · `exec-policy` (see [`approvals`](/cli/approvals)) · [`sandbox`](/cli/sandbox) · [`tui`](/cli/tui) · `chat`/`terminal` (aliases for [`tui --local`](/cli/tui)) · [`browser`](/cli/browser)                 |
| Automation           | [`cron`](/cli/cron) · [`tasks`](/cli/tasks) · [`hooks`](/cli/hooks) · [`webhooks`](/cli/webhooks)                                                                                                                                         |
| Discovery and docs   | [`dns`](/cli/dns) · [`docs`](/cli/docs)                                                                                                                                                                                                   |
| Pairing and channels | [`pairing`](/cli/pairing) · [`qr`](/cli/qr) · [`channels`](/cli/channels)                                                                                                                                                                 |
| Security and plugins | [`security`](/cli/security) · [`secrets`](/cli/secrets) · [`skills`](/cli/skills) · [`plugins`](/cli/plugins) · [`proxy`](/cli/proxy)                                                                                                     |
| Legacy aliases       | [`daemon`](/cli/daemon) (gateway service) · [`clawbot`](/cli/clawbot) (namespace)                                                                                                                                                         |
| Plugins (optional)   | [`path`](/cli/path) · [`voicecall`](/cli/voicecall) (if installed)                                                                                                                                                                        |

## Global flags

| Flag                    | Purpose                                                               |
| ----------------------- | --------------------------------------------------------------------- |
| `--dev`                 | Isolate state under `~/.openclaw-dev` and shift default ports         |
| `--profile <name>`      | Isolate state under `~/.openclaw-<name>`                              |
| `--container <name>`    | Target a named container for execution                                |
| `--no-color`            | Disable ANSI colors (`NO_COLOR=1` is also respected)                  |
| `--update`              | Shorthand for [`openclaw update`](/cli/update) (source installs only) |
| `-V`, `--version`, `-v` | Print version and exit                                                |

## Output modes

- ANSI colors and progress indicators render only in TTY sessions.
- OSC-8 hyperlinks render as clickable links where supported; otherwise the
  CLI falls back to plain URLs.
- `--json` (and `--plain` where supported) disables styling for clean output.
- Long-running commands show a progress indicator (OSC 9;4 when supported).

Palette source of truth: `src/terminal/palette.ts`.

## Command tree

### Full command tree

```
openclaw [--dev] [--profile <name>] <command>
  crestodian
  setup
  onboard
  configure
  config
    get
    set
    unset
    file
    schema
    validate
  completion
  doctor
  dashboard
  backup
    create
    verify
  security
    audit
  secrets
    reload
    audit
    configure
    apply
  reset
  uninstall
  update
    wizard
    status
  channels
    list
    status
    capabilities
    resolve
    logs
    add
    remove
    login
    logout
  directory
    self
    peers list
    groups list|members
  skills
    search
    install
    update
    list
    info
    check
  plugins
    list
    inspect
    install
    uninstall
    update
    enable
    disable
    doctor
    marketplace list
  memory
    status
    index
    search
  path
    resolve
    find
    set
    validate
    emit
  commitments
    list
    dismiss
  wiki
    status
    doctor
    init
    ingest
    compile
    lint
    search
    get
    apply
    bridge import
    unsafe-local import
    obsidian status|search|open|command|daily
  message
    send
    broadcast
    poll
    react
    reactions
    read
    edit
    delete
    pin
    unpin
    pins
    permissions
    search
    thread create|list|reply
    emoji list|upload
    sticker send|upload
    role info|add|remove
    channel info|list
    member info
    voice status
    event list|create
    timeout
    kick
    ban
  agent
  agents
    list
    add
    delete
    bindings
    bind
    unbind
    set-identity
  acp
  mcp
    serve
    list
    show
    set
    unset
  status
  health
  sessions
    cleanup
  tasks
    list
    audit
    maintenance
    show
    notify
    cancel
    flow list|show|cancel
  gateway
    call
    usage-cost
    health
   
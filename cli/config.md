---
domain: cli
topic: "CLI: openclaw config Commands"
type: reference
keywords:
  - openclaw config
  - config CLI
  - config get
  - config set
  - config unset
  - openclaw.json
source: cli/config.md
---

Config helpers for non-interactive edits in `openclaw.json`: get/set/patch/unset/file/schema/validate values by path and print the active config file. Run without a subcommand to open the configure wizard (same as `openclaw configure`).

> **Note:** When `OPENCLAW_NIX_MODE=1`, OpenClaw treats `openclaw.json` as immutable. Read-only commands such as `config get`, `config file`, `config schema`, and `config validate` still work, but config writers refuse. Agents should edit the Nix source for the install instead; for the first-party nix-openclaw distribution, use [nix-openclaw Quick Start](https://github.com/openclaw/nix-openclaw#quick-start) and set values under `programs.openclaw.config` or `instances.<name>.config`.


## Root options

" type="string">
  Repeatable guided-setup section filter when you run `openclaw config` without a subcommand.


Supported guided sections: `workspace`, `model`, `web`, `gateway`, `daemon`, `channels`, `plugins`, `skills`, `health`.

## Examples

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### `config schema`

Print the generated JSON schema for `openclaw.json` to stdout as JSON.

### What it includes

- The current root config schema, plus a root `$schema` string field for editor tooling.
    - Field `title` and `description` docs metadata used by the Control UI.
    - Nested object, wildcard (`*`), and array-item (`[]`) nodes inherit the same `title` / `description` metadata when matching field documentation exists.
    - `anyOf` / `oneOf` / `allOf` branches inherit the same docs metadata too when matching field documentation exists.
    - Best-effort live plugin + channel schema metadata when runtime manifests can be loaded.
    - A clean fallback schema even when the current config is invalid.

  ### Related runtime RPC

`config.schema.lookup` returns one normalized config path with a shallow schema node (`title`, `description`, `type`, `enum`, `const`, common bounds), matched UI hint metadata, and immediate child summaries. Use it for path-scoped drill-down in Control UI or custom clients.
  ```bash
openclaw config schema
```

Pipe it into a file when you want to inspect or validate it with other tools:

```bash
openclaw config schema > openclaw.schema.json
```

### Paths

Paths use dot or bracket notation:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.list[0].id
```

Use the agent list index to target a specific agent:

```bash
openclaw config get agents.list
openclaw config set agents.list[1].tools.exec.node "node-id-or-name"
```

## Values

Values are parsed as JSON5 when possible; otherwise they are treated as strings. Use `--strict-json` to require JSON5 parsing. `--json` remains supported as a legacy alias.

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

`config get <path> --json` prints the raw value as JSON instead of terminal-formatted text.

> **Note:** Object assignment replaces the target path by default. Protected map/list paths that commonly hold user-added entries, such as `agents.defaults.models`, `models.providers`, `models.providers.<id>.models`, `plugins.entries`, and `auth.profiles`, refuse replacements that would remove existing entries unless you pass `--replace`.


Use `--merge` when adding entries to those maps:

```bash
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set models.providers.ollama.models '[{"id":"llama3.2","name":"Llama 3.2"}]' --strict-json --merge
```

Use `--replace` only when you intentionally want the provided value to become the complete target value.

## `config set` modes

`openclaw config set` supports four assignment styles:

**Value mode:**

```bash
    openclaw config set <path> <value>
    ```
  
**SecretRef builder mode:**

```bash
    openclaw config set channels.discord.token \
      --ref-provider default \
      --ref-source env \
      --ref-id DISCORD_BOT_TOKEN
    ```
  
**Provider builder mode:**

Provider builder mode targets `secrets.providers.<alias>` paths only:

    ```bash
    openclaw config set secrets.providers.vault \
      --provider-source exec \
      --provider-command /usr/local/bin/openclaw-vault \
      --provider-arg read \
      --provider-arg openai/api-key \
      --provider-timeout-ms 5000
    ```

  
**Batch mode:**

```bash
    openclaw config set --batch-json '[
      {
        "path": "secrets.providers.default",
        "provider": { "source": "env" }
      },
      {
        "path": "channels.discord.token",
        "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
      }
    ]'
    ```

    ```bash
    openclaw config set --batch-file ./config-set.batch.json --dry-run
    ```

  
> **Note:** SecretRef assignments are rejected on unsupported runtime-mutable surfaces (for example `hooks.token`, `commands.ownerDisplaySecret`, Discord thread-binding webhook tokens, and WhatsApp creds JSON). See [SecretRef Credential Surface](/reference/secretref-credential-surface).


Batch parsing always uses the batch payload (`--batch-json`/`--batch-file`) as the source of truth. `--strict-json` / `--json` do not change batch parsing behavior.

## `config patch`

Use `config patch` when you want to paste or pipe a config-shaped patch instead of running many path-based `config set` commands. The input is a JSON5 object. Objects merge recursively, arrays and scalar values replace the target value, and `null` deletes the target path.

```bash
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config patch --file ./openclaw.patch.json5
```

You can also pipe a patch over stdin, which is useful for remote setup scripts:

```bash
ssh openclaw-host 'openclaw config patch --stdin --dry-run' < ./openclaw.patch.json5
ssh openclaw-host 'openclaw config patch --stdin' < ./openclaw.patch.json5
```

Example patch:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.5" },
      models: {
        "openai/gpt-5.5": { params: { fastMode: true } },
      },
    },
  },
}
```

Use `--replace-path <path>` when one object or array must become exactly the provided value instead of being recursively patched:

```bash
openclaw config patch --file ./discord.patch.json5 --replace-path 'channels.discord.guilds["123"].channels'
```

`--dry-run` runs schema and SecretRef resolvability checks without writing. Exec-backed SecretRefs are skipped by default during dry-run; add `--allow-exec` when you intentionally want dry-run to execute provider commands.

JSON path/value mode remains supported for both SecretRefs and providers:

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

## Provider builder flags

Provider builder targets must use `secrets.providers.<alias>` as the path.

### Common flags

- `--provider-source <env|file|exec>`
    - `--provider-timeout-ms <ms>` (`file`, `exec`)

  ### Env provider (--provider-source env)

- `--provider-allowlist ` (repeatable)

  ### File provider (--provider-source file)

- `--provider-path <path>` (required)
    - `--provider-mode <singleValue|json>`
    - `--provider-max-bytes <bytes>`
    - `--provider-allow-insecure-path`

  ### Exec provider (--provider-source exec)

- `--provider-command <path>` (required)
    - `--provider-arg <arg>` (repeatable)
    - `--provider-no-output-timeout-ms <ms>`
    - `--provider-max-output-bytes <bytes>`
    - `--provider-json-only`
    - `--provider-env ` (repeatable)
    - `--provider-pass-env ` (repeatable)
    - `--provider-trusted-dir <path>` (repeatable)
    - `--provider-allow-insecure-path`
    - `--provider-allow-symlink-command`

  Hardened exec provider example:

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## Dry run

Use `--dry-run` to validate changes without writing `openclaw.json`.

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run

openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

### Dry-run behavior

- Builder mode: runs SecretRef resolvability checks for changed refs/providers.
    - JSON mode (`--strict-json`, `--json`, or batch mode): runs schema validation plus SecretRef resolvability checks.
    - Policy validation also runs for known unsupported SecretRef target surfaces.
    - Policy checks evaluate the full post-change config, so parent-object writes (for example setting `hooks` as an object) cannot bypass unsupported-surface validation.
    - Exec SecretRef checks are skipped by default during dry-run to avoid command side effects.
    - Use `--allow-exec` with `--dry-run` to opt in to exec SecretRef checks (this may execute provider commands).
    - `--allow-exec` is dry-run only and errors if used without `--dry-run`.

  ### --dry-run --json fields

`--dry-run --json` prints a machine-readable report:

    - `ok`: whether dry-run passed
    - `operations`: number of assignments evaluated
    - `checks`: whether schema/resolvability checks ran
    - `checks.resolvabilityComplete`: whether resolvability checks ran to completion (false when exec refs are skipped)
    - `refsChecked`: number of refs actually resolved during dry-run
    - `skippedExecRefs`: number of exec refs skipped because `--allow-exec` was not set
    - `errors`: structured schema/resolvability failures when `ok=false`

  ### JSON output shape

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder", ...],
  checks: {
    schema: boolean,
    r
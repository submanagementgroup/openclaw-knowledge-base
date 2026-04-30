---
domain: cli
topic: "Config CLI: openclaw config get, set, unset, validate, and show Commands"
type: reference
keywords:
  - openclaw config
  - config CLI
  - config get
  - config set
  - config unset
  - configuration CLI
related:
  - gateway/configuration-overview
  - cli/cli-overview
source: cli/config.md
---

The `openclaw config` command reads and writes configuration values. Use it to get, set, unset, and validate config fields without editing the file directly.

```bash
openclaw config get agents.defaults.model
openclaw config set agents.defaults.model "openai/gpt-4o"
openclaw config unset agents.defaults.heartbeat.every
openclaw config validate
openclaw config show
```

Config helpers for non-interactive edits in `openclaw.json`: get/set/patch/unset/file/schema/validate values by path and print the active config file. Run without a subcommand to open the configure wizard (same as `openclaw configure`).

## Root options

<ParamField path="--section <section>" type="string">
  Repeatable guided-setup section filter when you run `openclaw config` without a subcommand.
</ParamField>

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

    - The current root config schema, plus a root `$schema` string field for editor tooling.
    - Field `title` and `description` docs metadata used by the Control UI.
    - Nested object, wildcard (`*`), and array-item (`[]`) nodes inherit the same `title` / `description` metadata when matching field documentation exists.
    - `anyOf` / `oneOf` / `allOf` branches inherit the same docs metadata too when matching field documentation exists.
    - Best-effort live plugin + channel schema metadata when runtime manifests can be loaded.
    - A clean fallback schema even when the current config is invalid.

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

Object assignment replaces the target path by default. Protected map/list paths that commonly hold user-added entries, such as `agents.defaults.models`, `models.providers`, `models.providers.<id>.models`, `plugins.entries`, and `auth.profiles`, refuse replacements that would remove existing entries unless you pass `--replace`.

Use `--merge` when adding entries to those maps:

```bash
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set models.providers.ollama.models '[{"id":"llama3.2","name":"Llama 3.2"}]' --strict-json --merge
```

Use `--replace` only when you intentionally want the provided value to become the complete target value.

## `config set` modes

`openclaw config set` supports four assignment styles:

    ```bash
    openclaw config set <path> <value>
    ```

    ```bash
    openclaw config set channels.discord.token \
      --ref-provider default \
      --ref-source env \
      --ref-id DISCORD_BOT_TOKEN
    ```

    Provider builder mode targets `secrets.providers.<alias>` paths only:

    ```bash
    openclaw config set secrets.providers.vault \
      --provider-source exec \
      --provider-command /usr/local/bin/openclaw-vault \
      --provider-arg read \
      --provider-arg openai/api-key \
      --provider-timeout-ms 5000
    ```

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

SecretRef assignments are rejected on unsupported runtime-mutable surfaces (for example `hooks.token`, `commands.ownerDisplaySecret`, Discord thread-binding webhook tokens, and WhatsApp creds JSON). See [SecretRef Credential Surface](/reference/secretref-credential-surface).

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
      groupPolicy: "allowl

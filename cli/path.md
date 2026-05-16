---
domain: cli
topic: "CLI: openclaw path Commands"
type: reference
keywords:
  - openclaw path
  - path CLI
  - binary path
  - PATH management
source: cli/path.md
---

# `openclaw path`

Plugin-provided shell access to the `oc://` addressing substrate: one
kind-dispatched path scheme for inspecting and editing addressable workspace
files (markdown, jsonc, jsonl). Self-hosters, plugin authors, and editor
extensions use it to read, find, or update a narrow location without
hand-rolling per-file parsers.

The CLI mirrors the substrate's public verbs:

- `resolve` is concrete and single-match.
- `find` is the multi-match verb for wildcards, unions, predicates, and
  positional expansion.
- `set` only accepts concrete paths or insertion markers; wildcard patterns are
  rejected before writing.

`path` is provided by the bundled optional `oc-path` plugin. Enable it before
first use:

```bash
openclaw plugins enable oc-path
```

## Why use it

OpenClaw state is spread across human-edited markdown, commented JSONC config,
and append-only JSONL logs. Shell scripts, hooks, and agents often need one
small value from those files: a frontmatter key, a plugin setting, a log record
field, or a bullet item under a named section.

`openclaw path` gives those callers a stable address instead of a one-off grep,
regex, or parser for each file kind. The same `oc://` path can be validated,
resolved, searched, dry-run, and written from the terminal, which makes narrow
automation easier to review and safer to replay. It is especially useful when
you want to update one leaf while preserving the rest of the file's comments,
line endings, and surrounding formatting.

Use it when the thing you want has a logical address, but the physical file
shape varies:

- A hook wants to read one setting from commented JSONC without losing comments
  when it writes the value back.
- A maintenance script wants to find every matching event field in a JSONL log
  without loading the whole log into a custom parser.
- An editor extension wants to jump to a markdown section or bullet item by
  slug, then render the exact line it resolved to.
- An agent wants to dry-run a tiny workspace edit before applying it, with the
  changed bytes visible in review.

You probably do not need `openclaw path` for ordinary whole-file edits, rich
config migrations, or memory-specific writes. Those should use the owner
command or plugin. `path` is for small, addressable file operations where a
repeatable terminal command is clearer than another bespoke parser.

## How it is used

Read one value from a human-edited config file:

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

Preview a write without touching disk:

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

Find matching records in an append-only JSONL log:

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

Address an instruction in markdown by section and item instead of by line
number:

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

Validate a path in CI or a preflight script before the script reads or writes:

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

Those commands are meant to be copyable into shell scripts. Use `--json` when a
caller needs structured output and `--human` when a person is inspecting the
result.

## How it works

`openclaw path` does four things:

1. Parses the `oc://` address into slots: file, section, item, field, and
   optional session.
2. Chooses the file-kind adapter from the target extension (`.md`, `.jsonc`,
   `.jsonl`, and related aliases).
3. Resolves the slots against that file kind's AST: markdown headings/items,
   JSONC object keys/array indexes, or JSONL line records.
4. For `set`, emits edited bytes through the same adapter so the untouched
   parts of the file keep their comments, line endings, and nearby formatting
   where the kind supports it.

`resolve` and `set` require one concrete target. `find` is the exploratory
verb: it expands wildcards, unions, predicates, and ordinals into the concrete
matches you can inspect before choosing one to write.

## Subcommands

| Subcommand              | Purpose                                                                      |
| ----------------------- | ---------------------------------------------------------------------------- |
| `resolve <oc-path>`     | Print the concrete match at the path (or "not found").                       |
| `find <pattern>`        | Enumerate matches for a wildcard / union / predicate path.                   |
| `set <oc-path> <value>` | Write a leaf or insertion target at a concrete path. Supports `--dry-run`.   |
| `validate <oc-path>`    | Parse-only; print structural breakdown (file / section / item / field).      |
| `emit <file>`           | Round-trip a file through `parseXxx` + `emitXxx` (byte-fidelity diagnostic). |

## Global flags

| Flag            | Purpose                                                                  |
| --------------- | ------------------------------------------------------------------------ |
| `--cwd <dir>`   | Resolve the file slot against this directory (default: `process.cwd()`). |
| `--file <path>` | Override the file slot's resolved path (absolute access).                |
| `--json`        | Force JSON output (default when stdout is not a TTY).                    |
| `--human`       | Force human output (default when stdout is a TTY).                       |
| `--dry-run`     | (only on `set`) print the bytes that would be written without writing.   |

## `oc://` syntax

```
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

Slot rules: `field` requires `item`, and `item` requires `section`. Across all
four slots:

- **Quoted segments** — `"a/b.c"` survives `/` and `.` separators.
  Content is byte-literal; `"` and `\` are not allowed inside quotes.
  The file slot is also quote-aware: `oc://"skills/email-drafter"/Tools/$last`
  treats `skills/email-drafter` as a single file path.
- **Predicates** — `[k=v]`, `[k!=v]`, `[k<v]`, `[k<=v]`, `[k>v]`,
  `[k>=v]`. Numeric ops require both sides to coerce to finite numbers.
- **Unions** — `{a,b,c}` matches any of the alternatives.
- **Wildcards** — `*` (single sub-segment) and `**` (zero-or-more,
  recursive). `find` accepts these; `resolve` and `set` reject them as
  ambiguous.
- **Positional** — `$last` resolves to the last index / last-declared key.
- **Ordinal** — `#N` for Nth match by document order.
- **Insertion markers** — `+`, `+key`, `+nnn` for keyed / indexed
  insertion (use with `set`).
- **Session scope** — `?session=cron-daily` etc. Orthogonal to slot
  nesting. Session values are raw, not percent-decoded; they may not contain
  control characters or reserved query delimiters (`?`, `&`, `%`).

Reserved characters (`?`, `&`, `%`) outside quoted, predicate, or union
segments are rejected. Control characters (U+0000-U+001F, U+007F) are rejected
anywhere, including the `session` query value.

`formatOcPath(parseOcPath(path)) === path` is guaranteed for canonical paths.
Non-canonical query parameters are ignored except for the first non-empty
`session=` value.

## Addressing by file kind

| Kind       | Addressing model                                                                          |
| ---------- | ----------------------------------------------------------------------------------------- |
| Markdown   | H2 sections by slug, bullet items by slug or `#N`, frontmatter via `[frontmatter]`.       |
| JSONC/JSON | Object keys and array indexes; dots split nested sub-segments unless quoted.              |
| JSONL      | Top-level line addresses (`L1`, `L2`, `$last`), then JSONC-style descent inside the line. |

`resolve` returns a structured match: `root`, `node`, `leaf`, or
`insertion-point`, with a 1-based line number. Leaf values are surfaced as text
plus a `leafType` so plugin authors can render previews without depending on
the per-kind AST shape.

## Mutation contract

`set` writes one concrete target:

- Markdown frontmatter values and `- key: value` item fields are string leaves.
  Markdown insertions append sections, frontmatter keys, or section items and
  render a canonical markdown shape for the changed file.
- JSONC leaf writes coerce the string value to the existing leaf type
  (`string`, finite `number`, `true`/`false`, or `null`). JSONC object and array
  insertions parse `<value>` as JSON and use the `jsonc-parser` edit path for
  ordinary leaf writes, preserving comments and nearby formatting.
- JSONL leaf writes coerce like JSONC inside a line. Whole-line replacement and
  append parse `<value>` as JSON. Rendered JSONL preserves the file's dominant
  LF/CRLF line-ending convention.

Use `--dry-run` before user-visible writes when the exact bytes matter. The
substrate preserves byte-identical output for parse/emit round-trips, but a
mutation can canonicalize the edited region or file depending on kind.

## Examples

```bash
# Validate a path (no filesystem access)
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# Read a leaf
openclaw path resolve 'oc://gateway.jsonc/version'

# Wildcard search
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# Dry-run a write
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# Apply the write
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# Byte-fidelity round-trip (diagnostic)
openclaw path emit ./AGENTS.md
```

More grammar examples:

```bash
# Quote keys containing / or .
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# Predicate search over JSONC children
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# Insert into a JSONC array
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# Insert a JSONC object key
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# Append a JSONL event
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file
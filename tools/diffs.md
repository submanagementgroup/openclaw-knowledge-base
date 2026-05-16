---
domain: tools
topic: "Diffs Tool: Code Patching and File Editing"
type: reference
keywords:
  - diffs
  - diff tool
  - patch
  - file editing
  - code diffs
  - apply patch
source: 
  - tools/diffs.md
  - tools/apply-patch.md
---

`diffs` is an optional plugin tool with short built-in system guidance and a companion skill that turns change content into a read-only diff artifact for agents.

It accepts either:

- `before` and `after` text
- a unified `patch`

It can return:

- a gateway viewer URL for canvas presentation
- a rendered file path (PNG or PDF) for message delivery
- both outputs in one call

When enabled, the plugin prepends concise usage guidance into system-prompt space and also exposes a detailed skill for cases where the agent needs fuller instructions.

## Quick start

**Install the plugin**

```bash
    openclaw plugins install diffs
    ```
  
**Enable the plugin**

```json5
    {
      plugins: {
        entries: {
          diffs: {
            enabled: true,
          },
        },
      },
    }
    ```
  
**Pick a mode**

**view:**

Canvas-first flows: agents call `diffs` with `mode: "view"` and open `details.viewerUrl` with `canvas present`.
      
**file:**

Chat file delivery: agents call `diffs` with `mode: "file"` and send `details.filePath` with `message` using `path` or `filePath`.
      
**both:**

Combined: agents call `diffs` with `mode: "both"` to get both artifacts in one call.
      

## Disable built-in system guidance

If you want to keep the `diffs` tool enabled but disable its built-in system-prompt guidance, set `plugins.entries.diffs.hooks.allowPromptInjection` to `false`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

This blocks the diffs plugin's `before_prompt_build` hook while keeping the plugin, tool, and companion skill available.

If you want to disable both the guidance and the tool, disable the plugin instead.

## Typical agent workflow

**Call diffs**

Agent calls the `diffs` tool with input.
  
**Read details**

Agent reads `details` fields from the response.
  
**Present**

Agent either opens `details.viewerUrl` with `canvas present`, sends `details.filePath` with `message` using `path` or `filePath`, or does both.
  
## Input examples

**Before and after:**

```json
    {
      "before": "# Hello\n\nOne",
      "after": "# Hello\n\nTwo",
      "path": "docs/example.md",
      "mode": "view"
    }
    ```
  
**Patch:**

```json
    {
      "patch": "diff --git a/src/example.ts b/src/example.ts\n--- a/src/example.ts\n+++ b/src/example.ts\n@@ -1 +1 @@\n-const x = 1;\n+const x = 2;\n",
      "mode": "both"
    }
    ```
  
## Tool input reference

All fields are optional unless noted.


  Original text. Required with `after` when `patch` is omitted.


  Updated text. Required with `before` when `patch` is omitted.


  Unified diff text. Mutually exclusive with `before` and `after`.


  Display filename for before and after mode.


  Language override hint for before and after mode. Unknown values fall back to plain text.


  Viewer title override.


  Output mode. Defaults to plugin default `defaults.mode`. Deprecated alias: `"image"` behaves like `"file"` and is still accepted for backward compatibility.


  Viewer theme. Defaults to plugin default `defaults.theme`.


  Diff layout. Defaults to plugin default `defaults.layout`.


  Expand unchanged sections when full context is available. Per-call option only (not a plugin default key).


  Rendered file format. Defaults to plugin default `defaults.fileFormat`.


  Quality preset for PNG or PDF rendering.


  Device scale override (`1`-`4`).


  Max render width in CSS pixels (`640`-`2400`).


  Artifact TTL in seconds for viewer and standalone file outputs. Max 21600.


  Viewer URL origin override. Overrides plugin `viewerBaseUrl`. Must be `http` or `https`, no query/hash.


### Legacy input aliases

Still accepted for backward compatibility:

    - `format` -> `fileFormat`
    - `imageFormat` -> `fileFormat`
    - `imageQuality` -> `fileQuality`
    - `imageScale` -> `fileScale`
    - `imageMaxWidth` -> `fileMaxWidth`

  ### Validation and limits

- `before` and `after` each max 512 KiB.
    - `patch` max 2 MiB.
    - `path` max 2048 bytes.
    - `lang` max 128 bytes.
    - `title` max 1024 bytes.
    - Patch complexity cap: max 128 files and 120000 total lines.
    - `patch` and `before` or `after` together are rejected.
    - Rendered file safety limits (apply to PNG and PDF):
      - `fileQuality: "standard"`: max 8 MP (8,000,000 rendered pixels).
      - `fileQuality: "hq"`: max 14 MP (14,000,000 rendered pixels).
      - `fileQuality: "print"`: max 24 MP (24,000,000 rendered pixels).
      - PDF also has a max of 50 pages.

  ## Output details contract

The tool returns structured metadata under `details`.

### Viewer fields

Shared fields for modes that create a viewer:

    - `artifactId`
    - `viewerUrl`
    - `viewerPath`
    - `title`
    - `expiresAt`
    - `inputKind`
    - `fileCount`
    - `mode`
    - `context` (`agentId`, `sessionId`, `messageChannel`, `agentAccountId` when available)

  ### File fields

File fields when PNG or PDF is rendered:

    - `artifactId`
    - `expiresAt`
    - `filePath`
    - `path` (same value as `filePath`, for message tool compatibility)
    - `fileBytes`
    - `fileFormat`
    - `fileQuality`
    - `fileScale`
    - `fileMaxWidth`

  ### Compatibility aliases

Also returned for existing callers:

    - `format` (same value as `fileFormat`)
    - `imagePath` (same value as `filePath`)
    - `imageBytes` (same value as `fileBytes`)
    - `imageQuality` (same value as `fileQuality`)
    - `imageScale` (same value as `fileScale`)
    - `imageMaxWidth` (same value as `fileMaxWidth`)

  Mode behavior summary:

| Mode     | What is returned                                                                                                       |
| -------- | ---------------------------------------------------------------------------------------------------------------------- |
| `"view"` | Viewer fields only.                                                                                                    |
| `"file"` | File fields only, no viewer artifact.                                                                                  |
| `"both"` | Viewer fields plus file fields. If file rendering fails, viewer still returns with `fileError` and `imageError` alias. |

## Collapsed unchanged sections

- The viewer can show rows like `N unmodified lines`.
- Expand controls on those rows are conditional and not guaranteed for every input kind.
- Expand controls appear when the rendered diff has expandable context data, which is typical for before and after input.
- For many unified patch inputs, omitted context bodies are not available in the parsed patch hunks, so the row can appear without expand controls. This is expected behavior.
- `expandUnchanged` applies only when expandable context exists.

## Plugin defaults

Set plugin-wide defaults in `~/.openclaw/openclaw.json`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
            ttlSeconds: 21600,
          },
        },
      },
    },
  },
}
```

Supported defaults:

- `fontFamily`
- `fontSize`
- `lineSpacing`
- `layout`
- `showLineNumbers`
- `diffIndicators`
- `wordWrap`
- `background`
- `theme`
- `fileFormat`
- `fileQuality`
- `fileScale`
- `fileMaxWidth`
- `mode`
- `ttlSeconds`

Explicit tool parameters override these defaults.

### Persistent viewer URL config


  Plugin-owned fallback for returned viewer links when a tool call does not pass `baseUrl`. Must be `http` or `https`, no query/hash.


```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## Security config


  `false`: non-loopback requests to viewer routes are denied. `true`: remote viewers are allowed if tokenized path is valid.


```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## Artifact lifecycle and storage

- Artifacts are stored under the temp subfolder: `$TMPDIR/openclaw-diffs`.
- Viewer artifact metadata contains:
  - random artifact ID (20 hex chars)
  - random token (48 hex chars)
  - `createdAt` and `expiresAt`
  - stored `viewer.html` path
- Default artifact TTL is 30 minutes when not specified.
- Maximum accepted viewer TTL is 6 hours.
- Cleanup runs opportunistically after artifact creation.
- Expired artifacts are deleted.
- Fallback cleanup removes stale folders older than 24 hours when metadata is missing.

## Viewer URL and network behavior

Viewer route:

- `/plugins/diffs/view/{artifactId}/{token}`

Viewer assets:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`

The viewer document resolves those assets relative to the viewer URL, so an optional `baseUrl` path prefix is preserved for both asset requests too.

URL construction behavior:

- If tool-call `baseUrl` is provided, it is used after strict validation.
- Else if plugin `viewerBaseUrl` is configured, it is used.
- Without either override, viewer URL defaults to loopback `127.0.0.1`.
- If gateway bind mode is `custom` and `gateway.customBindHost` is set, that host is used.

`baseUrl` rules:

- Must be `http://` or `https://`.
- Query and hash are rejected.
- Origin plus optional base path is allowed.

## Security model

### Viewer hardening

- Lo

## Apply Patch

Apply file changes using a structured patch format. This is ideal for multi-file
or multi-hunk edits where a single `edit` call would be brittle.

The tool accepts a single `input` string that wraps one or more file operations:

```
*** Begin Patch
*** Add File: path/to/file.txt
+line 1
+line 2
*** Update File: src/app.ts
@@
-old line
+new line
*** Delete File: obsolete.txt
*** End Patch
```

## Parameters

- `input` (required): Full patch contents including `*** Begin Patch` and `*** End Patch`.

## Notes

- Patch paths support relative paths (from the workspace directory) and absolute paths.
- `tools.exec.applyPatch.workspaceOnly` defaults to `true` (workspace-contained). Set it to `false` only if you intentionally want `apply_patch` to write/delete outside the workspace directory.
- Use `*** Move to:` within an `*** Update File:` hunk to rename files.
- `*** End of File` marks an EOF-only insert when needed.
- Available by default for OpenAI and OpenAI Codex models. Set
  `tools.exec.applyPatch.enabled: false` to disable it.
- Optionally gate by model via
  `tools.exec.applyPatch.allowModels`.
- Config is only under `tools.exec`.

## Example

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```

## Related

**Diffs:** Read-only diff viewer for change presentation.
  

**Exec tool:** Shell command execution from the agent.
  

**Code execution:** Sandboxed remote Python analysis with xAI.
  


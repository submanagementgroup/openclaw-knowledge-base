---
domain: tools
topic: "File Patching: apply_patch Tool and Diffs for Code Modification"
type: procedure
keywords:
  - apply_patch
  - diffs
  - patching
  - unified diff
  - file editing
related:
  - tools/exec
  - tools/tools-overview
source:
  - tools/apply-patch.md
  - tools/diffs.md
---

File patching and diff tools for OpenClaw agents. Apply unified diffs or generate/apply patches.

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

- [Diffs](/tools/diffs)
- [Exec tool](/tools/exec)
- [Code execution](/tools/code-execution)

## Diffs

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

        Canvas-first flows: agents call `diffs` with `mode: "view"` and open `details.viewerUrl` with `canvas present`.

        Chat file delivery: agents call `diffs` with `mode: "file"` and send `details.filePath` with `message` using `path` or `filePath`.

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

    Agent calls the `diffs` tool with input.

    Agent reads `details` fields from the response.

    Agent either opens `details.viewerUrl` with `canvas present`, sends `details.filePath` with `message` using `path` or `filePath`, or does both.

## Input examples

    ```json
    {
      "before": "# Hello\n\nOne",
      "after": "# Hello\n\nTwo",
      "path": "docs/example.md",
      "mode": "view"
    }
    ```

    ```json
    {
      "patch": "diff --git a/src/example.ts b/src/example.ts\n--- a/src/example.ts\n+++ b/src/example.ts\n@@ -1 +1 @@\n-const x = 1;\n+const x = 2;\n",
      "mode": "both"
    }
    ```

## Tool input reference

All fields are optional unless noted.

<ParamField path="before" type="string">
  Original text. Required with `after` when `patch` is omitted.
</ParamField>
<ParamField path="after" type="string">
  Updated text. Required with `before` when `patch` is omitted.
</ParamField>
<ParamField path="patch" type="string">
  Unified diff text. Mutually exclusive with `before` and `after`.
</ParamField>
<ParamField path="path" type="string">
  Display filename for before and after mode.
</ParamField>
<ParamField path="lang" type="string">
  Language override hint for before and after mode. Unknown values fall back to plain text.
</ParamField>
<ParamField path="title" type="string">
  Viewer title override.
</ParamField>
<ParamField path="mode" type='"view" | "file" | "both"'>
  Output mode. Defaults to plugin default `defaults.mode`. Deprecated alias: `"image"` behaves like `"file"` and is still accepted for backward compatibility.
</ParamField>
<ParamField path="theme" type='"light" | "dark"'>
  Viewer theme. Defaults to plugin default `defaults.theme`.
</ParamField>
<ParamField path="layout" type='"unified" | "split"'>
  Diff layout. Defaults to plugin default `defaults.layout`.
</ParamField>
<ParamField path="expandUnchanged" type="boolean">
  Expand unchanged sections when full context is available. Per-call option only (not a plugin default key).
</ParamField>
<ParamField path="fileFormat" type='"png" | "pdf"'>
  Rendered file format. Defaults to plugin default `defaults.fileFormat`.
</ParamField>
<ParamField path="fileQuality" type='"standard" | "hq" | "print"'>
  Quality preset for PNG or PDF rendering.
</ParamField>
<ParamField path="fileScale" type="number">
  Device scale override (`1`-`4`).
</ParamField>
<ParamField path="fileMaxWidth" type="number">
  Max render width in CSS pixels (`640`-`2400`).
</ParamField>
<ParamField path="ttlSeconds" type="number" default="1800">
  Artifact TTL in seconds for viewer and standalone file outputs. Max 21600.
</ParamField>
<ParamField path="baseUrl" type="string">
  Viewer URL origin override. Overrides plugin `viewerBaseUrl`. Must be `http` or `https`, no query/hash.
</ParamField>

    Still accepted for backward compatibility:

    - `format` -> `fileFormat`
    - `imageFormat` -> `fileFormat`
    - `imageQuality` -> `fileQuality`
    - `imageScale` -> `fileScale`
    - `imageMaxWidth` -> `fileMaxWidth`

    - `before` and `after` each max 512 KiB.
    - `patch` max 2 MiB.
    - `path` max 2048 bytes.
    - `lang` max 128 bytes.
    - `title` max 1024 bytes.
    - Patch complexity cap: max 128 files and 120000 total lines.
    - `patch` and `before` or `after` together are rejected.

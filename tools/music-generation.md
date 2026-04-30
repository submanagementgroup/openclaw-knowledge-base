---
domain: tools
topic: "Music Generation Tool: Audio Generation and Provider Configuration"
type: procedure
keywords:
  - music generation
  - audio generation
  - music tool
  - generate music
related:
  - tools/tts
  - tools/media-overview
source: tools/music-generation.md
---

Music generation tool for OpenClaw agents.

The `music_generate` tool lets the agent create music or audio through the
shared music-generation capability with configured providers — Google,
MiniMax, and workflow-configured ComfyUI today.

For session-backed agent runs, OpenClaw starts music generation as a
background task, tracks it in the task ledger, then wakes the agent again
when the track is ready so the agent can post the finished audio back into
the original channel.

The built-in shared tool only appears when at least one music-generation
provider is available. If you do not see `music_generate` in your agent's
tools, configure `agents.defaults.musicGenerationModel` or set up a
provider API key.

## Quick start

        Set an API key for at least one provider — for example
        `GEMINI_API_KEY` or `MINIMAX_API_KEY`.

        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```

        _"Generate an upbeat synthpop track about a night drive through a
        neon city."_

        The agent calls `music_generate` automatically. No tool
        allow-listing needed.

    For direct synchronous contexts without a session-backed agent run,
    the built-in tool still falls back to inline generation and returns
    the final media path in the tool result.

        Configure `plugins.entries.comfy.config.music` with a workflow
        JSON and prompt/output nodes.

        For Comfy Cloud, set `COMFY_API_KEY` or `COMFY_CLOUD_API_KEY`.

        ```text
        /tool music_generate prompt="Warm ambient synth loop with soft tape texture"
        ```

Example prompts:

```text
Generate a cinematic piano track with soft strings and no vocals.
```

```text
Generate an energetic chiptune loop about launching a rocket at sunrise.
```

## Supported providers

| Provider | Default model          | Reference inputs | Supported controls                                        | Auth                                   |
| -------- | ---------------------- | ---------------- | --------------------------------------------------------- | -------------------------------------- |
| ComfyUI  | `workflow`             | Up to 1 image    | Workflow-defined music or audio                           | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| Google   | `lyria-3-clip-preview` | Up to 10 images  | `lyrics`, `instrumental`, `format`                        | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax  | `music-2.6`            | None             | `lyrics`, `instrumental`, `durationSeconds`, `format=mp3` | `MINIMAX_API_KEY` or MiniMax OAuth     |

### Capability matrix

The explicit mode contract used by `music_generate`, contract tests, and the
shared live sweep:

| Provider | `generate` | `edit` | Edit limit | Shared live lanes                                                         |
| -------- | :--------: | :----: | ---------- | ------------------------------------------------------------------------- |
| ComfyUI  |     ✓      |   ✓    | 1 image    | Not in the shared sweep; covered by `extensions/comfy/comfy.live.test.ts` |
| Google   |     ✓      |   ✓    | 10 images  | `generate`, `edit`                                                        |
| MiniMax  |     ✓      |   —    | None       | `generate`                                                                |

Use `action: "list"` to inspect available shared providers and models at
runtime:

```text
/tool music_generate action=list
```

Use `action: "status"` to inspect the active session-backed music task:

```text
/tool music_generate action=status
```

Direct generation example:

```text
/tool music_generate prompt="Dreamy lo-fi hip hop with vinyl texture and gentle rain" instrumental=true
```

## Tool parameters

<ParamField path="prompt" type="string" required>
  Music generation prompt. Required for `action: "generate"`.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` returns the current session task; `"list"` inspects providers.
</ParamField>
<ParamField path="model" type="string">
  Provider/model override (e.g. `google/lyria-3-pro-preview`,
  `comfy/workflow`).
</ParamField>
<ParamField path="lyrics" type="string">
  Optional lyrics when the provider supports explicit lyric input.
</ParamField>
<ParamField path="instrumental" type="boolean">
  Request instrumental-only output when the provider supports it.
</ParamField>
<ParamField path="image" type="string">
  Single reference image path or URL.
</ParamField>
<ParamField path="images" type="string[]">
  Multiple reference images (up to 10 on supporting providers).
</ParamField>
<ParamField path="durationSeconds" type="number">
  Target duration in seconds when the provider supports duration hints.
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  Output format hint when the provider supports it.
</ParamField>
<ParamField path="filename" type="string">Output filename hint.</ParamField>
<ParamField path="timeoutMs" type="number">Optional provider request timeout in milliseconds.</ParamField>

Not all providers support all parameters. OpenClaw still validates hard
limits such as input counts before submission. When a provider supports
duration but uses a shorter maximum than the requested value, OpenClaw
clamps to the closest supported duration. Truly unsupported optional hints
are ignored with a warning when the selected provider or model cannot honor
them. Tool results report applied settings; `details.normalization`
captures any requested-to-applied mapping.

## Async behavior

Session-backed music generation runs as a background task:

- **Background task:** `music_generate` creates a background task, returns a
  started/task response immediately, and posts the finished track later in
  a follow-up agent message.
- **Duplicate prevention:** while a task is `queued` or `running`, later
  `music_generate` calls in the same session return task status instead of
  starting another generation. Use `action: "status"` to check explicitly.
- **Status lookup:** `openclaw tasks list` or `openclaw tasks show <taskId>`
  inspects queued, running, and terminal status.
- **Completion wake:** OpenClaw injects an internal completion event back
  into the same session so the model can write the user-facing follow-up
  itself.
- **Prompt hint:** later user/manual turns in the same session get a small
  runtime hint when a music task is already in flight, so the model does
  not blindly call `music_generate` again.
- **No-session fallback:** direct/local contexts without a real agent
  session run inline and return the final audio result in the same turn.

### Task lifecycle

| State       | Meaning                                                                                        |
| ----------- | ----------------------------------------------------------------------------

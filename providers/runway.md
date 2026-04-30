---
domain: providers
topic: "Runway Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - Runway
  - runway provider
  - AI provider
  - model config
  - API key
  - Runway
  - video generation
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/runway.md
---

Runway provider setup, configuration, and model reference for OpenClaw.

OpenClaw ships a bundled `runway` provider for hosted video generation.

| Property    | Value                                                             |
| ----------- | ----------------------------------------------------------------- |
| Provider id | `runway`                                                          |
| Auth        | `RUNWAYML_API_SECRET` (canonical) or `RUNWAY_API_KEY`             |
| API         | Runway task-based video generation (`GET /v1/tasks/{id}` polling) |

## Getting started

    ```bash
    openclaw onboard --auth-choice runway-api-key
    ```

    ```bash
    openclaw config set agents.defaults.videoGenerationModel.primary "runway/gen4.5"
    ```

    Ask the agent to generate a video. Runway will be used automatically.

## Supported modes

| Mode           | Model              | Reference input         |
| -------------- | ------------------ | ----------------------- |
| Text-to-video  | `gen4.5` (default) | None                    |
| Image-to-video | `gen4.5`           | 1 local or remote image |
| Video-to-video | `gen4_aleph`       | 1 local or remote video |

Local image and video references are supported via data URIs. Text-only runs
currently expose `16:9` and `9:16` aspect ratios.

Video-to-video currently requires `runway/gen4_aleph` specifically.

## Configuration

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## Advanced configuration

    OpenClaw recognizes both `RUNWAYML_API_SECRET` (canonical) and `RUNWAY_API_KEY`.
    Either variable will authenticate the Runway provider.

    Runway uses a task-based API. After submitting a generation request, OpenClaw
    polls `GET /v1/tasks/{id}` until the video is ready. No additional
    configuration is needed for the polling behavior.

## Related

    Shared tool parameters, provider selection, and async behavior.

    Agent default settings including video generation model.

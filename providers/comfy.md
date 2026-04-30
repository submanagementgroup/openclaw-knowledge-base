---
domain: providers
topic: "ComfyUI Provider: Setup, Configuration, and Model Reference"
type: procedure
keywords:
  - ComfyUI
  - comfy provider
  - AI provider
  - model config
  - API key
  - ComfyUI
  - image generation
  - local Stable Diffusion
related:
  - concepts/models
  - gateway/configuration-overview
source: providers/comfy.md
---

ComfyUI provider setup, configuration, and model reference for OpenClaw.

OpenClaw ships a bundled `comfy` plugin for workflow-driven ComfyUI runs. The plugin is entirely workflow-driven, so OpenClaw does not try to map generic `size`, `aspectRatio`, `resolution`, `durationSeconds`, or TTS-style controls onto your graph.

| Property        | Detail                                                                           |
| --------------- | -------------------------------------------------------------------------------- |
| Provider        | `comfy`                                                                          |
| Models          | `comfy/workflow`                                                                 |
| Shared surfaces | `image_generate`, `video_generate`, `music_generate`                             |
| Auth            | None for local ComfyUI; `COMFY_API_KEY` or `COMFY_CLOUD_API_KEY` for Comfy Cloud |
| API             | ComfyUI `/prompt` / `/history` / `/view` and Comfy Cloud `/api/*`                |

## What it supports

- Image generation from a workflow JSON
- Image editing with 1 uploaded reference image
- Video generation from a workflow JSON
- Video generation with 1 uploaded reference image
- Music or audio generation through the shared `music_generate` tool
- Output download from a configured node or all matching output nodes

## Getting started

Choose between running ComfyUI on your own machine or using Comfy Cloud.

    **Best for:** running your own ComfyUI instance on your machine or LAN.

        Make sure your local ComfyUI instance is running (defaults to `http://127.0.0.1:8188`).

        Export or create a ComfyUI workflow JSON file. Note the node IDs for the prompt input node and the output node you want OpenClaw to read from.

        Set `mode: "local"` and point at your workflow file. Here is a minimal image example:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "local",
                  baseUrl: "http://127.0.0.1:8188",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```

        Point OpenClaw at the `comfy/workflow` model for the capability you configured:

        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```

        ```bash
        openclaw models list --provider comfy
        ```

    **Best for:** running workflows on Comfy Cloud without managing local GPU resources.

        Sign up at [comfy.org](https://comfy.org) and generate an API key from your account dashboard.

        Provide your key through one of these methods:

        ```bash
        # Environment variable (preferred)
        export COMFY_API_KEY="your-key"

        # Alternative environment variable
        export COMFY_CLOUD_API_KEY="your-key"

        # Or inline in config
        openclaw config set plugins.entries.comfy.config.apiKey "your-key"
        ```

        Export or create a ComfyUI workflow JSON file. Note the node IDs for the prompt input node and the output node.

        Set `mode: "cloud"` and point at your workflow file:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "cloud",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```

        Cloud mode defaults `baseUrl` to `https://cloud.comfy.org`. You only need to set `baseUrl` if you use a custom cloud endpoint.

        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```

        ```bash
        openclaw models list --provider comfy
        ```

## Configuration

Comfy supports shared top-level connection settings plus per-capability workflow sections (`image`, `video`, `music`):

```json5
{
  plugins: {
    entries: {
      comfy: {
        config: {
          mode: "local",
          baseUrl: "http://127.0.0.1:8188",
          image: {
            workflowPath: "./workflows/flux-api.json",
            promptNodeId: "6",
            outputNodeId: "9",
          },
          video: {
            workflowPath: "./workflows/video-api.json",
            promptNodeId: "12",
            outputNodeId: "21",
          },
          music: {
            workflowPath: "./workflows/music-api.json",
            promptNodeId: "3",
            outputNodeId: "18",
          },
        },
      },
    },
  },
}
```

### Shared keys

| Key                   | Type                   | Description                                                                           |
| --------------------- | ---------------------- | ------------------------------------------------------------------------------------- |
| `mode`                | `"local"` or `"cloud"` | Connection mode.                                                                      |
| `baseUrl`             | string                 | Defaults to `http://127.0.0.1:8188` for local or `https://cloud.comfy.org` for cloud. |
| `apiKey`              | string                 | Optional inline key, alternative to `COMFY_API_KEY` / `COMFY_CLOUD_API_KEY` env vars. |
| `allowPrivateNetwork` | boolean                | Allow a private/LAN `baseUrl` in cloud mode.                                          |

### Per-capability keys

These keys apply inside the `image`, `video`, or `music` sections:

| Key                          | Required | Default  | Description                                                                  |
| ---------------------------- | -------- | -------- | ---------------------------------------------------------------------------- |
| `workflow` or `workflowPath` | Yes      | --       | Path to the ComfyUI workflow JSON file.                                      |
| `promptNodeId`               | Yes      | --       | Node ID that receives the text prompt.                                       |
| `promptInputName`            | No       | `"text"` | Input name on the prompt node.                                               |
| `outputNodeId`               | No       | --       | Node ID to read output from. If omitted, all matching output nodes are used. |
| `pollIntervalMs`             | No       | --       | Polling interval in milliseconds for job completion.                         |
| `timeoutMs`                  | No       | --       | Timeout in milliseconds for the workflow run.                                |

The `image` and `video` sections also support:

| Key                   | Required                             | Default   | Description                                         |
| --------------------- | ------------------------------------ | --------- | --------------------------------------------------- |
| `inputImageNodeId`    | Yes (when passing a reference image) | --        | Node ID that receives the uploaded reference image. |
| `inputImageInputName` | No                                   | `"image"` | Input name on the image node.                       |

## Workflow details

    Set the default image model to `comfy/workflow`:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    **Reference-image editing example:**

    To enable image editing with an uploaded referenc

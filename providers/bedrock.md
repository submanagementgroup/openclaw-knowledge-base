---
domain: providers
topic: "AWS Bedrock Provider: Claude on Bedrock, Mantle Multi-Region, and IAM Setup"
type: procedure
keywords:
  - AWS Bedrock
  - Bedrock
  - bedrock provider
  - Claude Bedrock
  - IAM
  - AWS credentials
  - Bedrock Mantle
related:
  - providers/anthropic
  - providers/openai
source:
  - providers/bedrock.md
  - providers/bedrock-mantle.md
---

AWS Bedrock provider for OpenClaw. Supports Claude, Llama, Titan, and other models via Amazon Bedrock.

OpenClaw can use **Amazon Bedrock** models via pi-ai's **Bedrock Converse**
streaming provider. Bedrock auth uses the **AWS SDK default credential chain**,
not an API key.

| Property | Value                                                       |
| -------- | ----------------------------------------------------------- |
| Provider | `amazon-bedrock`                                            |
| API      | `bedrock-converse-stream`                                   |
| Auth     | AWS credentials (env vars, shared config, or instance role) |
| Region   | `AWS_REGION` or `AWS_DEFAULT_REGION` (default: `us-east-1`) |

## Getting started

Choose your preferred auth method and follow the setup steps.

    **Best for:** developer machines, CI, or hosts where you manage AWS credentials directly.

        ```bash
        export AWS_ACCESS_KEY_ID="AKIA..."
        export AWS_SECRET_ACCESS_KEY="..."
        export AWS_REGION="us-east-1"
        # Optional:
        export AWS_SESSION_TOKEN="..."
        export AWS_PROFILE="your-profile"
        # Optional (Bedrock API key/bearer token):
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```

        No `apiKey` is required. Configure the provider with `auth: "aws-sdk"`:

        ```json5
        {
          models: {
            providers: {
              "amazon-bedrock": {
                baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
                api: "bedrock-converse-stream",
                auth: "aws-sdk",
                models: [
                  {
                    id: "us.anthropic.claude-opus-4-6-v1:0",
                    name: "Claude Opus 4.6 (Bedrock)",
                    reasoning: true,
                    input: ["text", "image"],
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                    contextWindow: 200000,
                    maxTokens: 8192,
                  },
                ],
              },
            },
          },
          agents: {
            defaults: {
              model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1:0" },
            },
          },
        }
        ```

        ```bash
        openclaw models list
        ```

    With env-marker auth (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE`, or `AWS_BEARER_TOKEN_BEDROCK`), OpenClaw auto-enables the implicit Bedrock provider for model discovery without extra config.

    **Best for:** EC2 instances with an IAM role attached, using the instance metadata service for authentication.

        When using IMDS, OpenClaw cannot detect AWS auth from env markers alone, so you must opt in:

        ```bash
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1
        ```

        If you also want the env-marker auto-detection path to work (for example, for `openclaw status` surfaces):

        ```bash
        export AWS_PROFILE=default
        export AWS_REGION=us-east-1
        ```

        You do **not** need a fake API key.

        ```bash
        openclaw models list
        ```

    The IAM role attached to your EC2 instance must have the following permissions:

    - `bedrock:InvokeModel`
    - `bedrock:InvokeModelWithResponseStream`
    - `bedrock:ListFoundationModels` (for automatic discovery)
    - `bedrock:ListInferenceProfiles` (for inference profile discovery)

    Or attach the managed policy `AmazonBedrockFullAccess`.

    You only need `AWS_PROFILE=default` if you specifically want an env marker for auto mode or status surfaces. The actual Bedrock runtime auth path uses the AWS SDK default chain, so IMDS instance-role auth works even without env markers.

## Automatic model discovery

OpenClaw can automatically discover Bedrock models that support **streaming**
and **text output**. Discovery uses `bedrock:ListFoundationModels` and
`bedrock:ListInferenceProfiles`, and results are cached (default: 1 hour).

How the implicit provider is enabled:

- If `plugins.entries.amazon-bedrock.config.discovery.enabled` is `true`,
  OpenClaw will try discovery even when no AWS env marker is present.
- If `plugins.entries.amazon-bedrock.config.discovery.enabled` is unset,
  OpenClaw only auto-adds the
  implicit Bedrock provider when it sees one of these AWS auth markers:
  `AWS_BEARER_TOKEN_BEDROCK`, `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY`, or `AWS_PROFILE`.
- The actual Bedrock runtime auth path still uses the AWS SDK default chain, so
  shared config, SSO, and IMDS instance-role auth can work even when discovery
  needed `enabled: true` to opt in.

For explicit `models.providers["amazon-bedrock"]` entries, OpenClaw can still resolve Bedrock env-marker auth early from AWS env markers such as `AWS_BEARER_TOKEN_BEDROCK` without forcing full runtime auth loading. The actual model-call auth path still uses the AWS SDK default chain.

    Config options live under `plugins.entries.

## Bedrock Mantle (Multi-Region)

OpenClaw includes a bundled **Amazon Bedrock Mantle** provider that connects to
the Mantle OpenAI-compatible endpoint. Mantle hosts open-source and
third-party models (GPT-OSS, Qwen, Kimi, GLM, and similar) through a standard
`/v1/chat/completions` surface backed by Bedrock infrastructure.

| Property       | Value                                                                                       |
| -------------- | ------------------------------------------------------------------------------------------- |
| Provider ID    | `amazon-bedrock-mantle`                                                                     |
| API            | `openai-completions` (OpenAI-compatible) or `anthropic-messages` (Anthropic Messages route) |
| Auth           | Explicit `AWS_BEARER_TOKEN_BEDROCK` or IAM credential-chain bearer-token generation         |
| Default region | `us-east-1` (override with `AWS_REGION` or `AWS_DEFAULT_REGION`)                            |

## Getting started

Choose your preferred auth method and follow the setup steps.

    **Best for:** environments where you already have a Mantle bearer token.

        ```bash
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```

        Optionally set a region (defaults to `us-east-1`):

        ```bash
        export AWS_REGION="us-west-2"
        ```

        ```bash
        openclaw models list
        ```

        Discovered models appear under the `amazon-bedrock-mantle` provider. No
        additional config is required unless you want to override defaults.

    **Best for:** using AWS SDK-compatible credentials (shared config, SSO, web identity, instance or task roles).

        Any AWS SDK-compatible auth source works:

        ```bash
        export AWS_PROFILE="default"
        export AWS_REGION="us-west-2"
        ```

        ```bash
        openclaw models list
        ```

        OpenClaw generates a Mantle bearer token from the credential chain automatically.

    When `AWS_BEARER_TOKEN_BEDR

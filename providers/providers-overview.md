---
domain: providers
topic: "AI Providers Overview: All Supported Providers by Category"
type: reference
keywords:
  - providers
  - AI providers
  - provider list
  - embedding providers
  - TTS providers
  - image providers
related:
  - concepts/models
  - concepts/model-failover
  - memory/memory-search
source: providers/index.md
---

Index of all OpenClaw AI providers. OpenClaw supports dozens of providers for text, image, video, audio, and embeddings.

## Provider Categories

### Chat/Text Providers
OpenAI, Anthropic, Google Gemini, Ollama (local), LM Studio (local), vLLM (local), AWS Bedrock, Groq, Together AI, DeepSeek, Mistral, XAI Grok, Perplexity, OpenRouter, HuggingFace, GitHub Copilot, Cerebras, Fireworks, DeepInfra, Cloudflare AI Gateway, LiteLLM

### Image Generation
OpenAI DALL-E, fal.ai, ComfyUI (local Stable Diffusion), Runway

### Video Generation
Runway, OpenAI Sora (via OpenAI provider)

### Audio / TTS / STT
ElevenLabs (TTS), Azure Speech (TTS+STT), Deepgram (STT), Azure OpenAI TTS, OpenAI TTS

### Embedding Providers (for memory search)
OpenAI, Google Gemini, Voyage, Mistral, DeepInfra, local/ollama

### Chinese/Regional Providers
MiniMax, Alibaba Qwen, Moonshot Kimi, Baidu Qianfan, GLM/Z.AI, Stepfun, Volcengine, Xiaomi, Tencent Hunyuan

# Model Providers

OpenClaw can use many LLM providers. Pick a provider, authenticate, then set the
default model as `provider/model`.

Looking for chat channel docs (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/etc.)? See [Channels](/channels).

## Quick start

1. Authenticate with the provider (usually via `openclaw onboard`).
2. Set the default model:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Provider docs

- [Alibaba Model Studio](/providers/alibaba)
- [Amazon Bedrock](/providers/bedrock)
- [Amazon Bedrock Mantle](/providers/bedrock-mantle)
- [Anthropic (API + Claude CLI)](/providers/anthropic)
- [Arcee AI (Trinity models)](/providers/arcee)
- [Azure Speech](/providers/azure-speech)
- [BytePlus (International)](/concepts/model-providers#byteplus-international)
- [Cerebras](/providers/cerebras)
- [Chutes](/providers/chutes)
- [Cloudflare AI Gateway](/providers/cloudflare-ai-gateway)
- [ComfyUI](/providers/comfy)
- [DeepSeek](/providers/deepseek)
- [ElevenLabs](/providers/elevenlabs)
- [fal](/providers/fal)
- [Fireworks](/providers/fireworks)
- [GitHub Copilot](/providers/github-copilot)
- [GLM models](/providers/glm)
- [Google (Gemini)](/providers/google)
- [Gradium](/providers/gradium)
- [Groq (LPU inference)](/providers/groq)
- [Hugging Face (Inference)](/providers/huggingface)
- [inferrs (local models)](/providers/inferrs)
- [Kilocode](/providers/kilocode)
- [LiteLLM (unified gateway)](/providers/litellm)
- [LM Studio (local models)](/providers/lmstudio)
- [MiniMax](/providers/minimax)
- [Mistral](/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/providers/moonshot)
- [NVIDIA](/providers/nvidia)
- [Ollama (cloud + local models)](/providers/ollama)
- [OpenAI (API + Codex)](/providers/openai)
- [OpenCode](/providers/opencode)
- [OpenCode Go](/providers/opencode-go)
- [OpenRouter](/providers/openrouter)
- [Perplexity (web search)](/providers/perplexity-provider)
- [Qianfan](/providers/qianfan

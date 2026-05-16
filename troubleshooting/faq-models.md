---
domain: troubleshooting
topic: "Help: Faq Models"
type: troubleshooting
keywords:
  - faq-models
  - troubleshooting
  - help
  - FAQ
source: help/faq-models.md
---

Model- and auth-profile Q&A. For setup, sessions, gateway, channels, and
troubleshooting, see the main [FAQ](/help/faq).

## Models: defaults, selection, aliases, switching


    OpenClaw's default model is whatever you set as:

    ```
    agents.defaults.model.primary
    ```

    Models are referenced as `provider/model` (example: `openai/gpt-5.5` or `anthropic/claude-sonnet-4-6`). If you omit the provider, OpenClaw first tries an alias, then a unique configured-provider match for that exact model id, and only then falls back to the configured default provider as a deprecated compatibility path. If that provider no longer exposes the configured default model, OpenClaw falls back to the first configured provider/model instead of surfacing a stale removed-provider default. You should still **explicitly** set `provider/model`.

  ### What model do you recommend?

**Recommended default:** use the strongest latest-generation model available in your provider stack.
    **For tool-enabled or untrusted-input agents:** prioritize model strength over cost.
    **For routine/low-stakes chat:** use cheaper fallback models and route by agent role.

    MiniMax has its own docs: [MiniMax](/providers/minimax) and
    [Local models](/gateway/local-models).

    Rule of thumb: use the **best model you can afford** for high-stakes work, and a cheaper
    model for routine chat or summaries. You can route models per agent and use sub-agents to
    parallelize long tasks (each sub-agent consumes tokens). See [Models](/concepts/models) and
    [Sub-agents](/tools/subagents).

    Strong warning: weaker/over-quantized models are more vulnerable to prompt
    injection and unsafe behavior. See [Security](/gateway/security).

    More context: [Models](/concepts/models).

  ### How do I switch models without wiping my config?

Use **model commands** or edit only the **model** fields. Avoid full config replaces.

    Safe options:

    - `/model` in chat (quick, per-session)
    - `openclaw models set ...` (updates just model config)
    - `openclaw configure --section model` (interactive)
    - edit `agents.defaults.model` in `~/.openclaw/openclaw.json`

    Avoid `config.apply` with a partial object unless you intend to replace the whole config.
    For RPC edits, inspect with `config.schema.lookup` first and prefer `config.patch`. The lookup payload gives you the normalized path, shallow schema docs/constraints, and immediate child summaries.
    for partial updates.
    If you did overwrite config, restore from backup or re-run `openclaw doctor` to repair.

    Docs: [Models](/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/gateway/doctor).

  ### Can I use self-hosted models (llama.cpp, vLLM, Ollama)?

Yes. Ollama is the easiest path for local models.

    Quickest setup:

    1. Install Ollama from `https://ollama.com/download`
    2. Pull a local model such as `ollama pull gemma4`
    3. If you want cloud models too, run `ollama signin`
    4. Run `openclaw onboard` and choose `Ollama`
    5. Pick `Local` or `Cloud + Local`

    Notes:

    - `Cloud + Local` gives you cloud models plus your local Ollama models
    - cloud models such as `kimi-k2.5:cloud` do not need a local pull
    - for manual switching, use `openclaw models list` and `openclaw models set ollama/<model>`

    Security note: smaller or heavily quantized models are more vulnerable to prompt
    injection. We strongly recommend **large models** for any bot that can use tools.
    If you still want small models, enable sandboxing and strict tool allowlists.

    Docs: [Ollama](/providers/ollama), [Local models](/gateway/local-models),
    [Model providers](/concepts/model-providers), [Security](/gateway/security),
    [Sandboxing](/gateway/sandboxing).

  ### What do OpenClaw, Flawd, and Krill use for models?

- These deployments can differ and may change over time; there is no fixed provider recommendation.
    - Check the current runtime setting on each gateway with `openclaw models status`.
    - For security-sensitive/tool-enabled agents, use the strongest latest-generation model available.

  ### How do I switch models on the fly (without restarting)?

Use the `/model` command as a standalone message:

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    These are the built-in aliases. Custom aliases can be added via `agents.defaults.models`.

    You can list available models with `/model`, `/model list`, or `/model status`.

    `/model` (and `/model list`) shows a compact, numbered picker. Select by number:

    ```
    /model 3
    ```

    You can also force a specific auth profile for the provider (per session):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Tip: `/model status` shows which agent is active, which `auth-profiles.json` file is being used, and which auth profile will be tried next.
    It also shows the configured provider endpoint (`baseUrl`) and API mode (`api`) when available.

    **How do I unpin a profile I set with @profile?**

    Re-run `/model` **without** the `@profile` suffix:

    ```
    /model anthropic/claude-opus-4-6
    ```

    If you want to return to the default, pick it from `/model` (or send `/model <default provider/model>`).
    Use `/model status` to confirm which auth profile is active.

  ### If two providers expose the same model id, which one does /model use?

`/model provider/model` selects that exact provider route for the session.

    For example, `qianfan/deepseek-v4-flash` and `deepseek/deepseek-v4-flash` are different model refs even though both contain `deepseek-v4-flash`. OpenClaw should not silently switch from one provider to the other just because the bare model id matches.

    A user-selected `/model` ref is also strict for fallback policy. If that selected provider/model is unavailable, the reply fails visibly instead of answering from `agents.defaults.model.fallbacks`. Configured fallback chains still apply to configured defaults, cron job primaries, and auto-selected fallback state.

    If a run that started from a non-session override is allowed to use fallback, OpenClaw tries the requested provider/model first, then configured fallbacks, and only then the configured primary. That prevents duplicate bare model ids from jumping directly back to the default provider.

    See [Models](/concepts/models) and [Model failover](/concepts/model-failover).

  ### Can I use GPT 5.5 for daily tasks and Codex 5.5 for coding?

Yes. Treat model choice and runtime choice separately:

    - **Native Codex coding agent:** set `agents.defaults.model.primary` to `openai/gpt-5.5`. Sign in with `openclaw models auth login --provider openai-codex` when you want ChatGPT/Codex subscription auth.
    - **Direct OpenAI API tasks outside the agent loop:** configure `OPENAI_API_KEY` for images, embeddings, speech, realtime, and other non-agent OpenAI API surfaces.
    - **OpenAI agent API-key auth:** use `/model openai/gpt-5.5` with an ordered `openai-codex` API-key profile.
    - **Sub-agents:** route coding tasks to a Codex-focused agent with its own `openai/gpt-5.5` model.

    See [Models](/concepts/models) and [Slash commands](/tools/slash-commands).

  ### How do I configure fast mode for GPT 5.5?

Use either a session toggle or a config default:

    - **Per session:** send `/fast on` while the session is using `openai/gpt-5.5`.
    - **Per model default:** set `agents.defaults.models["openai/gpt-5.5"].params.fastMode` to `true`.

    Example:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    For OpenAI, fast mode maps to `service_tier = "priority"` on supported native Responses requests. Session `/fast` overrides beat config defaults.

    See [Thinking and fast mode](/tools/thinking) and [OpenAI fast mode](/providers/openai#fast-mode).

  
    If `agents.defaults.models` is set, it becomes the **allowlist** for `/model` and any
    session overrides. Choosing a model that isn't in that list returns:

    ```
    Model "provider/model" is not allowed. Use /models to list providers, or /models <provider> to list models.
    Add it with: openclaw config set agents.defaults.models '{"provider/model":{}}' --strict-json --merge
    ```

    That error is returned **instead of** a normal reply. Fix: add the exact model to
    `agents.defaults.models`, add a provider wildcard such as `"provider/*": {}` for dynamic provider catalogs, remove the allowlist, or pick a model from `/model list`.
    If the command also included `--runtime codex`, update the allowlist first and then retry
    the same `/model provider/model --runtime codex` command.

  
    This means the **provider isn't configured** (no MiniMax provider config or auth
    profile was found), so the model can't be resolved.

    Fix checklist:

    1. Upgrade to a current OpenClaw release (or run from source `main`), then restart the gateway.
    2. Make sure MiniMax is configured (wizard or JSON), or that MiniMax auth
       exists in env/auth profiles so the matching provider can be injected
       (`MINIMAX_API_KEY` for `minimax`, `MINIMAX_OAUTH_TOKEN` or stored MiniMax
       OAuth for `minimax-portal`).
    3. Use the exact model id (case-sensitive) for your auth path:
       `minimax/MiniMax-M2.7` or `minimax/MiniMax-M2.7-highspeed` for API-key
       setup, or `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` for OAuth setup.
    4. Run:

       ```bash
       openclaw models list
       ```

       and pick from the list (or `/model list` in chat).

    See [MiniMax](/providers/minimax) and [Models](/concepts/models).

  ### Can I use MiniMax as
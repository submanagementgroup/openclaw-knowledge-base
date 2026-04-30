---
domain: gateway
topic: "Gateway Configuration Examples: Copy-Paste openclaw.json Configs"
type: reference
keywords:
  - configuration examples
  - openclaw.json examples
  - sample config
  - copy-paste config
related:
  - gateway/configuration-overview
  - gateway/configuration-reference
source: gateway/configuration-examples.md
---

Ready-to-use `openclaw.json` configuration examples for common setups. Copy and adapt these for your use case.

Examples below are aligned with the current config schema. For the exhaustive reference and per-field notes, see [Configuration](/gateway/configuration).

## Quick start

### Absolute minimum

```json5
{
  agent: { workspace: "~/.openclaw/workspace" },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

Save to `~/.openclaw/openclaw.json` and you can DM the bot from that number.

### Recommended starter

```json5
{
  identity: {
    name: "Clawd",
    theme: "helpful assistant",
    emoji: "🦞",
  },
  agent: {
    workspace: "~/.openclaw/workspace",
    model: { primary: "anthropic/claude-sonnet-4-6" },
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  messages: {
    visibleReplies: "automatic",
    groupChat: {
      visibleReplies: "message_tool", // default; use "automatic" for legacy room replies
    },
  },
}
```

## Expanded example (major options)

> JSON5 lets you use comments and trailing commas. Regular JSON works too.

```json5
{
  // Environment + shell
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },

  // Auth profile metadata (secrets live in auth-profiles.json)
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:default": { provider: "openai", mode: "api_key" },
      "openai-codex:personal": { provider: "openai-codex", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:default"],
      "openai-codex": ["openai-codex:personal"],
    },
  },

  // Identity
  identity: {
    name: "Samantha",
    theme: "helpful sloth",
    emoji: "🦥",
  },

  // Logging
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty",
    redactSensitive: "tools",
  },

  // Message formatting
  messages: {
    messagePrefix: "[openclaw]",
    visibleReplies: "automatic",
    responsePrefix: ">",
    ackReaction: "👀",
    ackReactionScope: "group-mentions",
    groupChat: {
      historyLimit: 50,
      visibleReplies: "message_tool", // normal final replies stay private in groups/channels
    },
    queue: {
      mode: "steer",
      debounceMs: 500,
      cap: 20,
      drop: "summarize",
      byChannel: {
        whatsapp: "steer",
        telegram: "steer",
        discord: "steer",
        slack: "steer",
        signal: "steer",
        imessage: "steer",
        webchat: "steer",
      },
    },
  },

  // Tooling
  tools: {
    media: {
      audio: {
        enabled: true,
        maxBytes: 20971520,
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          // Optional CLI fallback (Whisper binary):
          // { type: "cli", command: "whisper", args: ["--model", "base", "{{MediaPath}}"] }
        ],
        timeoutSeconds: 120,
      },
      video: {
        enabled: true,
        maxBytes: 52428800,
        models: [{ provider: "google", model: "gemini-3-flash-preview" }],
      },
    },
  },

  // Session behavior
  session: {
    scope: "per-sender",
    dmScope: "per-channel-peer", // recommended for multi-user inboxes
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 60,
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/default/sessions/sessions.json",
    maintenance: {
      mode: "warn",
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // duration or false
      maxDiskBytes: "500mb", // optional
      highWaterBytes: "400mb", // optional (defaults to 80% of maxDiskBytes)
    },
    typingIntervalSeconds: 5,
    sendPolicy: {
      default: "allow",
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
    },
  },

  // Channels
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15555550123"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },

    telegram: {
      enabled: true,
      botToken: "YOUR_TELEGRAM_BOT_TOKEN",
      allowFrom: ["123456789"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["123456789"],
      groups: { "*": { requireMention: true } },
    },

    discord: {
      enabled: true,
      token: "YOUR_DISCORD_BOT_TOKEN",
      dm: { enabled: true, allowFrom: ["123456789012345678"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },

    slack: {
      enabled: true,
      botToken: "xoxb-REPLACE_ME",
      appToken: "xapp-REPLACE_ME",
      channels: {
        "#general": { allow: true, requireMention: true },
      },
      dm: { enabled: true, allowFrom: ["U123"] },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
    },
  },

  // Agent runtime
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      userTimezone: "America/Chicago",
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["anthropic/claude-opus-4-6", "openai/gpt-5.4"],
      },
      imageModel: {
        primary: "openrouter/anthropic/claude-sonnet-4-6",
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
        "openai/gpt-5.4": { alias: "gpt" },
      },
      skills: ["github", "weather"], // inherited by agents that omit list[].skills
      thinkingDefault: "low",
      verboseDefault: "off",
      reasoningDefault: "off",
      elevatedDefault: "on",
      blockStreamingDefault: "off",
      blockStreamingBreak: "text_end",
      blockStreamingChunk: {
        minChars: 800,
        maxChars: 1200,
        breakPreference: "paragraph",
      },
      blockStreamingCoalesce: {
        idleMs: 1000,
      },
      humanDelay: {
        mode: "natural",
      },
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      typingIntervalSeconds: 5,
      maxConcurrent: 3,
      heartbeat: {
        every: "30m",
        model: "anthropic/claude-sonnet-4-6",
        target: "last",
        directPolicy: "allow", // allow (default) | block
        to: "+15555550123",
        prompt: "HEARTBEAT",
        ackMaxChars: 300,
      },
      memorySearch: {
        provider: "gemini",
        model: "gemini-embedding-001",
        remote: {
          apiKey: "${GEMINI_API_KEY}",
        },
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
      sandbox: {
        mode: "non-main",
        scope: "session", // preferred over legacy perSession: true
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
        },
        browser: {
          enabled: false,
        },
      },
    },
    list: [
      {
        id: "main",
        default: true,
        // inherits defaults.skills -> github, weather
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw"],
        },
        thinkingDefault: "high", // per-agent thinking override
        reasoningDefault: "on", // per-agent reasoning visibility
        fastModeDefault: false, // per-agent fast mode
      },
      {
        id: "quick",
        skills: [], // no skills for this agent
        fas

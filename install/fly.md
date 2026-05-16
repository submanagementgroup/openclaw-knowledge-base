---
domain: install
topic: "Fly.io Installation"
type: procedure
keywords:
  - Fly.io
  - fly install
  - fly deploy
source: install/fly.md
---

**Goal:** OpenClaw Gateway running on a [Fly.io](https://fly.io) machine with persistent storage, automatic HTTPS, and Discord/channel access.

## What you need

- [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/) installed
- Fly.io account (free tier works)
- Model auth: API key for your chosen model provider
- Channel credentials: Discord bot token, Telegram token, etc.

## Beginner quick path

1. Clone repo → customize `fly.toml`
2. Create app + volume → set secrets
3. Deploy with `fly deploy`
4. SSH in to create config or use Control UI

**Create the Fly app**

```bash
    # Clone the repo
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw

    # Create a new Fly app (pick your own name)
    fly apps create my-openclaw

    # Create a persistent volume (1GB is usually enough)
    fly volumes create openclaw_data --size 1 --region iad
    ```

    **Tip:** Choose a region close to you. Common options: `lhr` (London), `iad` (Virginia), `sjc` (San Jose).

  
**Configure fly.toml**

Edit `fly.toml` to match your app name and requirements.

    **Security note:** The default config exposes a public URL. For a hardened deployment with no public IP, see [Private Deployment](#private-deployment-hardened) or use `deploy/fly.private.toml`.

    ```toml
    app = "my-openclaw"  # Your app name
    primary_region = "iad"

    [build]
      dockerfile = "Dockerfile"

    [env]
      NODE_ENV = "production"
      OPENCLAW_PREFER_PNPM = "1"
      OPENCLAW_STATE_DIR = "/data"
      NODE_OPTIONS = "--max-old-space-size=1536"

    [processes]
      app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

    [http_service]
      internal_port = 3000
      force_https = true
      auto_stop_machines = false
      auto_start_machines = true
      min_machines_running = 1
      processes = ["app"]

    [[vm]]
      size = "shared-cpu-2x"
      memory = "2048mb"

    [mounts]
      source = "openclaw_data"
      destination = "/data"
    ```

    The OpenClaw Docker image uses `tini` as its entrypoint. Fly process commands replace Docker `CMD` without replacing `ENTRYPOINT`, so the process still runs under `tini`.

    **Key settings:**

    | Setting                        | Why                                                                         |
    | ------------------------------ | --------------------------------------------------------------------------- |
    | `--bind lan`                   | Binds to `0.0.0.0` so Fly's proxy can reach the gateway                     |
    | `--allow-unconfigured`         | Starts without a config file (you'll create one after)                      |
    | `internal_port = 3000`         | Must match `--port 3000` (or `OPENCLAW_GATEWAY_PORT`) for Fly health checks |
    | `memory = "2048mb"`            | 512MB is too small; 2GB recommended                                         |
    | `OPENCLAW_STATE_DIR = "/data"` | Persists state on the volume                                                |

  
**Set secrets**

```bash
    # Required: Gateway token (for non-loopback binding)
    fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

    # Model provider API keys
    fly secrets set ANTHROPIC_API_KEY=sk-ant-...

    # Optional: Other providers
    fly secrets set OPENAI_API_KEY=sk-...
    fly secrets set GOOGLE_API_KEY=...

    # Channel tokens
    fly secrets set DISCORD_BOT_TOKEN=MTQ...
    ```

    **Notes:**

    - Non-loopback binds (`--bind lan`) require a valid gateway auth path. This Fly.io example uses `OPENCLAW_GATEWAY_TOKEN`, but `gateway.auth.password` or a correctly configured non-loopback `trusted-proxy` deployment also satisfy the requirement.
    - Treat these tokens like passwords.
    - **Prefer env vars over config file** for all API keys and tokens. This keeps secrets out of `openclaw.json` where they could be accidentally exposed or logged.

  
**Deploy**

```bash
    fly deploy
    ```

    First deploy builds the Docker image (~2-3 minutes). Subsequent deploys are faster.

    After deployment, verify:

    ```bash
    fly status
    fly logs
    ```

    You should see:

    ```
    [gateway] listening on ws://0.0.0.0:3000 (PID xxx)
    [discord] logged in to discord as xxx
    ```

  
**Create config file**

SSH into the machine to create a proper config:

    ```bash
    fly ssh console
    ```

    Create the config directory and file:

    ```bash
    mkdir -p /data
    cat > /data/openclaw.json << 'EOF'
    {
      "agents": {
        "defaults": {
          "model": {
            "primary": "anthropic/claude-opus-4-6",
            "fallbacks": ["anthropic/claude-sonnet-4-6", "openai/gpt-5.4"]
          },
          "maxConcurrent": 4
        },
        "list": [
          {
            "id": "main",
            "default": true
          }
        ]
      },
      "auth": {
        "profiles": {
          "anthropic:default": { "mode": "token", "provider": "anthropic" },
          "openai:default": { "mode": "token", "provider": "openai" }
        }
      },
      "bindings": [
        {
          "agentId": "main",
          "match": { "channel": "discord" }
        }
      ],
      "channels": {
        "discord": {
          "enabled": true,
          "groupPolicy": "allowlist",
          "guilds": {
            "YOUR_GUILD_ID": {
              "channels": { "general": { "allow": true } },
              "requireMention": false
            }
          }
        }
      },
      "gateway": {
        "mode": "local",
        "bind": "auto",
        "controlUi": {
          "allowedOrigins": [
            "https://my-openclaw.fly.dev",
            "http://localhost:3000",
            "http://127.0.0.1:3000"
          ]
        }
      },
      "meta": {}
    }
    EOF
    ```

    **Note:** With `OPENCLAW_STATE_DIR=/data`, the config path is `/data/openclaw.json`.

    **Note:** Replace `https://my-openclaw.fly.dev` with your real Fly app
    origin. Gateway startup seeds local Control UI origins from the runtime
    `--bind` and `--port` values so first boot can proceed before config exists,
    but browser access through Fly still needs the exact HTTPS origin listed in
    `gateway.controlUi.allowedOrigins`.

    **Note:** The Discord token can come from either:

    - Environment variable: `DISCORD_BOT_TOKEN` (recommended for secrets)
    - Config file: `channels.discord.token`

    If using env var, no need to add token to config. The gateway reads `DISCORD_BOT_TOKEN` automatically.

    Restart to apply:

    ```bash
    exit
    fly machine restart <machine-id>
    ```

  
**Access the Gateway**

### Control UI

    Open in browser:

    ```bash
    fly open
    ```

    Or visit `https://my-openclaw.fly.dev/`

    Authenticate with the configured shared secret. This guide uses the gateway
    token from `OPENCLAW_GATEWAY_TOKEN`; if you switched to password auth, use
    that password instead.

    ### Logs

    ```bash
    fly logs              # Live logs
    fly logs --no-tail    # Recent logs
    ```

    ### SSH Console

    ```bash
    fly ssh console
    ```

  
## Troubleshooting

### "App is not listening on expected address"

The gateway is binding to `127.0.0.1` instead of `0.0.0.0`.

**Fix:** Add `--bind lan` to your process command in `fly.toml`.

### Health checks failing / connection refused

Fly can't reach the gateway on the configured port.

**Fix:** Ensure `internal_port` matches the gateway port (set `--port 3000` or `OPENCLAW_GATEWAY_PORT=3000`).

### OOM / Memory Issues

Container keeps restarting or getting killed. Signs: `SIGABRT`, `v8::internal::Runtime_AllocateInYoungGeneration`, or silent restarts.

**Fix:** Increase memory in `fly.toml`:

```toml
[[vm]]
  memory = "2048mb"
```

Or update an existing machine:

```bash
fly machine update <machine-id> --vm-memory 2048 -y
```

**Note:** 512MB is too small. 1GB may work but can OOM under load or with verbose logging. **2GB is recommended.**

### Gatew
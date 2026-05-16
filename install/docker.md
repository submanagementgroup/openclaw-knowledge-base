---
domain: install
topic: "Docker Installation"
type: procedure
keywords:
  - docker
  - docker install
  - container
  - docker-compose
  - Docker setup
source: 
  - install/docker.md
  - install/clawdock.md
---

## Docker

Docker is **optional**. Use it only if you want a containerized gateway or to validate the Docker flow.

## Is Docker right for me?

- **Yes**: you want an isolated, throwaway gateway environment or to run OpenClaw on a host without local installs.
- **No**: you are running on your own machine and just want the fastest dev loop. Use the normal install flow instead.
- **Sandboxing note**: the default sandbox backend uses Docker when sandboxing is enabled, but sandboxing is off by default and does **not** require the full gateway to run in Docker. SSH and OpenShell sandbox backends are also available. See [Sandboxing](/gateway/sandboxing).

## Prerequisites

- Docker Desktop (or Docker Engine) + Docker Compose v2
- At least 2 GB RAM for image build (`pnpm install` may be OOM-killed on 1 GB hosts with exit 137)
- Enough disk for images and logs
- If running on a VPS/public host, review
  [Security hardening for network exposure](/gateway/security),
  especially Docker `DOCKER-USER` firewall policy.

## Containerized gateway

**Build the image**

From the repo root, run the setup script:

    ```bash
    ./scripts/docker/setup.sh
    ```

    This builds the gateway image locally. To use a pre-built image instead:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    Pre-built images are published at the
    [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw).
    Common tags: `main`, `latest`, `<version>` (e.g. `2026.2.26`).

  
**Complete onboarding**

The setup script runs onboarding automatically. It will:

    - prompt for provider API keys
    - generate a gateway token and write it to `.env`
    - start the gateway via Docker Compose

    During setup, pre-start onboarding and config writes run through
    `openclaw-gateway` directly. `openclaw-cli` is for commands you run after
    the gateway container already exists.

  
**Open the Control UI**

Open `http://127.0.0.1:18789/` in your browser and paste the configured
    shared secret into Settings. The setup script writes a token to `.env` by
    default; if you switch the container config to password auth, use that
    password instead.

    Need the URL again?

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  
**Configure channels (optional)**

Use the CLI container to add messaging channels:

    ```bash
    # WhatsApp (QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    Docs: [WhatsApp](/channels/whatsapp), [Telegram](/channels/telegram), [Discord](/channels/discord)

  
### Manual flow

If you prefer to run each step yourself instead of using the setup script:

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

> **Note:** Run `docker compose` from the repo root. If you enabled `OPENCLAW_EXTRA_MOUNTS`
or `OPENCLAW_HOME_VOLUME`, the setup script writes `docker-compose.extra.yml`;
include it with `-f docker-compose.yml -f docker-compose.extra.yml`.


> **Note:** Because `openclaw-cli` shares `openclaw-gateway`'s network namespace, it is a
post-start tool. Before `docker compose up -d openclaw-gateway`, run onboarding
and setup-time config writes through `openclaw-gateway` with
`--no-deps --entrypoint node`.


### Environment variables

The setup script accepts these optional environment variables:

| Variable                                   | Purpose                                                         |
| ------------------------------------------ | --------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                           | Use a remote image instead of building locally                  |
| `OPENCLAW_DOCKER_APT_PACKAGES`             | Install extra apt packages during build (space-separated)       |
| `OPENCLAW_EXTENSIONS`                      | Include selected bundled plugin helpers at build time           |
| `OPENCLAW_EXTRA_MOUNTS`                    | Extra host bind mounts (comma-separated `source:target[:opts]`) |
| `OPENCLAW_HOME_VOLUME`                     | Persist `/home/node` in a named Docker volume                   |
| `OPENCLAW_SANDBOX`                         | Opt in to sandbox bootstrap (`1`, `true`, `yes`, `on`)          |
| `OPENCLAW_SKIP_ONBOARDING`                 | Skip the interactive onboarding step (`1`, `true`, `yes`, `on`) |
| `OPENCLAW_DOCKER_SOCKET`                   | Override Docker socket path                                     |
| `OPENCLAW_DISABLE_BONJOUR`                 | Disable Bonjour/mDNS advertising (defaults to `1` for Docker)   |
| `OPENCLAW_DISABLE_BUNDLED_SOURCE_OVERLAYS` | Disable bundled plugin source bind-mount overlays               |
| `OTEL_EXPORTER_OTLP_ENDPOINT`              | Shared OTLP/HTTP collector endpoint for OpenTelemetry export    |
| `OTEL_EXPORTER_OTLP_*_ENDPOINT`            | Signal-specific OTLP endpoints for traces, metrics, or logs     |
| `OTEL_EXPORTER_OTLP_PROTOCOL`              | OTLP protocol override. Only `http/protobuf` is supported today |
| `OTEL_SERVICE_NAME`                        | Service name used for OpenTelemetry resources                   |
| `OTEL_SEMCONV_STABILITY_OPT_IN`            | Opt in to latest experimental GenAI semantic attributes         |
| `OPENCLAW_OTEL_PRELOADED`                  | Skip starting a second OpenTelemetry SDK when one is preloaded  |

Maintainers can test bundled plugin source against a packaged image by mounting
one plugin source directory over its packaged source path, for example
`OPENCLAW_EXTRA_MOUNTS=/path/to/fork/extensions/synology-chat:/app/extensions/synology-chat:ro`.
That mounted source directory overrides the matching compiled
`/app/dist/extensions/synology-chat` bundle for the same plugin id.

### Observability

OpenTelemetry export is outbound from the Gateway container to your OTLP
collector. It does not require a published Docker port. If you build the image
locally and want the bundled OpenTelemetry exporter available inside the image,
include its runtime dependencies:

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_SERVICE_NAME="openclaw-gateway"
./scripts/docker/setup.sh
```

Install the official `@openclaw/diagnostics-otel` plugin from ClawHub in
packaged Docker installs before enabling export. Custom source-built images can
still include the local plugin source with
`OPENCLAW_EXTENSIONS=diagnostics-otel`. To enable export, allow and enable the
`diagnostics-otel` plugin in config, then set
`diagnostics.otel.enabled=true` or use the config example in [OpenTelemetry
export](/gateway/opentelemetry). Collector auth headers are configured through
`diagnostics.otel.headers`, not through Docker environment variables.

Prometheus metrics use the already-published Gateway port. Install
`clawhub:@openclaw/diagnostics-prometheus`, enable the
`diagnostics-prometheus` plugin, then scrape:

```text
http://<gateway-host>:18789/api/diagnostics/prometheus
```

The route is protected by Gateway authentication. Do not expose a separate
public `/metrics` port or unauthenticated reverse-proxy path. See
[Prometheus metrics](/gateway/prometheus).

### Health checks

Container probe endpoints (no auth required):

```bash
curl -f

## ClawDock

ClawDock is a small shell-helper layer for Docker-based OpenClaw installs.

It gives you short commands like `clawdock-start`, `clawdock-dashboard`, and `clawdock-fix-token` instead of longer `docker compose ...` invocations.

If you have not set up Docker yet, start with [Docker](/install/docker).

## Install

Use the canonical helper path:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

If you previously installed ClawDock from `scripts/shell-helpers/clawdock-helpers.sh`, reinstall from the new `scripts/clawdock/clawdock-helpers.sh` path. The old raw GitHub path was removed.

## What you get

### Basic operations

| Command            | Description            |
| ------------------ | ---------------------- |
| `clawdock-start`   | Start the gateway      |
| `clawdock-stop`    | Stop the gateway       |
| `clawdock-restart` | Restart the gateway    |
| `clawdock-status`  | Check container status |
| `clawdock-logs`    | Follow gateway logs    |

### Container access

| Command                   | Description                                   |
| ------------------------- | --------------------------------------------- |
| `clawdock-shell`          | Open a shell inside the gateway container     |
| `clawdock-cli <command>`  | Run OpenClaw CLI commands in Docker           |
| `clawdock-exec <command>` | Execute an arbitrary command in the container |

### Web UI and pairing

| Command                 | Description                  |
| ----------------------- | ---------------------------- |
| `clawdock-dashboard`    | Open the Control UI URL      |
| `clawdock-devices`      | List pending device pairings |
| `clawdock-approve <id>` | Approve a pairing request    |

### Setup and maintenance

| Command              | Description                                      |
| -------------------- | ------------------------------------------------ |
| `clawdock-fix-token` | Configure the gateway token inside the container |
| `clawdock-update`    | Pull, rebuild, and restart                       |
| `clawdock-rebuild`   | Rebuild the Docker image only                    |
| `clawdock-clean`     | Remove containers and volumes                    |

### Utilities

| Command                | Description                             |
| ---------------------- | --------------------------------------- |
| `clawdock-health`      | Run a gateway health check              |
| `clawdock-token`       | Print the gateway token                 |
| `clawdock-cd`          | Jump to the OpenClaw project directory  |
| `clawdock-config`      | Open `~/.openclaw`                      |
| `clawdock-show-config` | Print config files with redacted values |
| `clawdock-workspace`   | Open the workspace directory            |

## First-time flow

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

If the browser says pairing is required:

```bash
clawdock-devices
clawdock-approve <request-id>
```

## Config and secrets

ClawDock works with the same Docker config split described in [Docker](/install/docker):

- `<project>/.env` for Docker-specific values like image name, ports, and the gateway token
- `~/.openclaw/.env` for env-backed provider keys and bot tokens
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` for stored provider OAuth/API-key auth
- `~/.openclaw/openclaw.json` for behavior config

Use `clawdock-show-config` when you want to inspect the `.env` files and `openclaw.json` quickly. It redacts `.env` values in its printed output.

## Related

**Docker:** Canonical Docker install for OpenClaw.
  

**Docker VM runtime:** Docker-managed VM runtime for hardened isolation.
  

**Updating:** Updating the OpenClaw package and managed services.
  


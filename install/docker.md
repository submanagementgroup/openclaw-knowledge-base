---
domain: install
topic: "Docker and Podman: Container Installation, docker-compose, ClawDock, and Volume Mounts"
type: procedure
keywords:
  - Docker
  - Podman
  - docker-compose
  - ClawDock
  - container
  - Docker install
related:
  - install/kubernetes
  - install/installation-methods
source:
  - install/docker.md
  - install/clawdock.md
  - install/podman.md
---

Running OpenClaw in Docker or Podman. Includes docker-compose setup, volume mounts, and the ClawDock wrapper.

## Docker Setup

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

    The setup script runs onboarding automatically. It will:

    - prompt for provider API keys
    - generate a gateway token and write it to `.env`
    - start the gateway via Docker Compose

    During setup, pre-start onboarding and config writes run through
    `openclaw-gateway` directly. `openclaw-cli` is for commands you run after
    the gateway container already exists.

    Open `http://127.0.0.1:18789/` in your browser and paste the configured
    shared secret into Settings. The setup script writes a token to `.env` by
    default; if you switch the container config to password auth, use that
    password instead.

    Need the URL again?

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

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

Run `docker compose` from the repo root. If you enabled `OPENCLAW_EXTRA_MOUNTS`
or `OPENCLAW_HOME_VOLUME`, the setup script writes `docker-compose.extra.yml`;
include it with `-f docker-compose.yml -f docker-compose.extra.yml`.

Because `openclaw-cli` shares `openclaw-gateway`'s network namespace, it is a
post-start tool. Before `docker compose up -d openclaw-gateway`, run onboarding
and setup-time config writes through `openclaw-gateway` with
`--no-deps --entrypoint node`.

### Environment variables

The setup script accepts these optional environment variables:

| Variable                                   | Purpose                                                         |
| ------------------------------------------ | --------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                           | Use a remote image instead of building locally                  |
| `OPENCLAW_DOCKER_APT_PACKAGES`             | Install extra apt packages during build (space-separated)       |
| `OPENCLAW_EXTENSIONS`                      | Pre-install plugin deps at build time (space-separated names)   |
| `OPENCLAW_EXTRA_MOUNTS`                    | Extra host bind mounts (comma-separated `source:target[:opts]`) |
| `OPENCLAW_HOME_VOLUME`                     | Persist `/home/node` in a named Docker volume                   |
| `OPENCLAW_PLUGIN_STAGE_DIR`                | Container path for generated bundled plugin deps and mirrors    |
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
| `OPENCLAW_OTEL_PRELOADED`          

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

| Command              | Description                                      

## Podman

Run the OpenClaw Gateway in a rootless Podman container, managed by your current non-root user.

The intended model is:

- Podman runs the gateway container.
- Your host `openclaw` CLI is the control plane.
- Persistent state lives on the host under `~/.openclaw` by default.
- Day-to-day management uses `openclaw --container <name> ...` instead of `sudo -u openclaw`, `podman exec`, or a separate service user.

## Prerequisites

- **Podman** in rootless mode
- **OpenClaw CLI** installed on the host
- **Optional:** `systemd --user` if you want Quadlet-managed auto-start
- **Optional:** `sudo` only if you want `loginctl enable-linger "$(whoami)"` for boot persistence on a headless host

## Quick start

    From the repo root, run `./scripts/podman/setup.sh`.

    Start the container with `./scripts/run-openclaw-podman.sh launch`.

    Run `./scripts/run-openclaw-podman.sh launch setup`, then open `http://127.0.0.1:18789/`.

    Set `OPENCLAW_CONTAINER=openclaw`, then use normal `openclaw` commands from the host.

Setup details:

- `./scripts/podman/setup.sh` builds `openclaw:local` in your rootless Podman store by default, or uses `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` if you set one.
- It creates `~/.openclaw/openclaw.json` with `gateway.mode: "local"` if missing.
- It creates `~/.openclaw/.env` with `OPENCLAW_GATEWAY_TOKEN` if missing.
- For manual launches, the helper reads only a small allowlist of Podman-related keys from `~/.openclaw/.env` and passes explicit runtime env vars to the container; it does not hand the full env file to Podman.

Quadlet-managed setup:

```bash
./scripts/podman/setup.sh --quadlet
```

Quadlet is a Linux-only option because it depends on systemd user services.

You can also set `OPENCLAW_PODMAN_QUADLET=1`.

Optional build/setup env vars:

- `OPENCLAW_IMAGE` or `OPENCLAW_PODMAN_IMAGE` -- use an existing/pulled image instead of building `openclaw:local`
- `OPENCLAW_DOCKER_APT_PACKAGES` -- install extra apt packages during image build
- `OPENCLAW_EXTENSIONS` -- pre-install plugin dependencies at build time
- `OPENCLAW_INSTALL_BROWSER` -- pre-install Chromium and Xvfb for browser automation (set to `1` to enable)

Container start:

```bash
./scripts/run-openclaw-podman.sh launch
```

The script starts the container as your current uid/gid with `--userns=keep-id` and bind-mounts your OpenClaw state into the container.

Onboarding:

```bash
./scripts/run-openclaw-podman.sh launch setup
```

Then open `http://127.0.0.1:18789/` and use the token from `~/.openclaw/.env`.

Host CLI default:

```bash
export OPENCLAW_CONTAINER=openclaw
```

Then commands such as these will run inside that container automatically:

```bash
openclaw dashboard --no-open
openclaw gateway status --deep   # includes extra service scan
openclaw doctor
openclaw channels login
```

On macOS, Podman machine may make the browser appear non-local to the gateway.
If the Control UI reports device-auth errors after launch, use the Tailscale guidance in
[Pod

## Docker VM Runtime

Shared runtime steps for VM-based Docker installs such as GCP, Hetzner, and similar VPS providers.

## Bake required binaries into the image

Installing binaries inside a running container is a trap.
Anything installed at runtime will be lost on restart.

All external binaries required by skills must be installed at image build time.

The examples below show three common binaries only:

- `gog` (from `gogcli`) for Gmail access
- `goplaces` for Google Places
- `wacli` for WhatsApp

These are examples, not a complete list.
You may install as many binaries as needed using the same pattern.

If you add new skills later that depend on additional binaries, you must:

1. Update the Dockerfile
2. Rebuild the image
3. Restart the containers

**Example Dockerfile**

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# Example binary 1: Gmail CLI (gogcli — installs as `gog`)
# Copy the current Linux asset URL from https://github.com/steipete/gogcli/releases
RUN curl -L https://github.com/steipete/gogcli/releases/latest/download/gogcli_linux_amd64.tar.gz \
  | tar -xzO gog > /usr/local/bin/gog; \
  chmod +x /usr/local/bin/gog

# Example binary 2: Google Places CLI
# Copy the current Linux asset URL from https://github.com/steipete/goplaces/releases
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_linux_amd64.tar.gz \
  | tar -xzO goplaces > /usr/local/bin/goplaces; \
  chmod +x /usr/local/bin/goplaces

# Example binary 3: WhatsApp CLI
# Copy the current Linux asset URL from https://github.com/steipete/wacli/releases
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli-linux-amd64.tar.gz \
  | tar -xzO wacli > /usr/local/bin/wacli; \
  chmod +x /usr/local/bin/wacli

# Add more binaries below using the same pattern

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN co

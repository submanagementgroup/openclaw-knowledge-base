---
domain: install
topic: "Google Cloud Platform Installation"
type: procedure
keywords:
  - GCP
  - Google Cloud
  - gcp install
source: install/gcp.md
---

Run a persistent OpenClaw Gateway on a GCP Compute Engine VM using Docker, with durable state, baked-in binaries, and safe restart behavior.

If you want "OpenClaw 24/7 for ~$5-12/mo", this is a reliable setup on Google Cloud.
Pricing varies by machine type and region; pick the smallest VM that fits your workload and scale up if you hit OOMs.

## What are we doing (simple terms)?

- Create a GCP project and enable billing
- Create a Compute Engine VM
- Install Docker (isolated app runtime)
- Start the OpenClaw Gateway in Docker
- Persist `~/.openclaw` + `~/.openclaw/workspace` on the host (survives restarts/rebuilds)
- Access the Control UI from your laptop via an SSH tunnel

That mounted `~/.openclaw` state includes `openclaw.json`, per-agent
`agents/<agentId>/agent/auth-profiles.json`, and `.env`.

The Gateway can be accessed via:

- SSH port forwarding from your laptop
- Direct port exposure if you manage firewalling and tokens yourself

This guide uses Debian on GCP Compute Engine.
Ubuntu also works; map packages accordingly.
For the generic Docker flow, see [Docker](/install/docker).

---

## Quick path (experienced operators)

1. Create GCP project + enable Compute Engine API
2. Create Compute Engine VM (e2-small, Debian 12, 20GB)
3. SSH into the VM
4. Install Docker
5. Clone OpenClaw repository
6. Create persistent host directories
7. Configure `.env` and `docker-compose.yml`
8. Bake required binaries, build, and launch

---

## What you need

- GCP account (free tier eligible for e2-micro)
- gcloud CLI installed (or use Cloud Console)
- SSH access from your laptop
- Basic comfort with SSH + copy/paste
- ~20-30 minutes
- Docker and Docker Compose
- Model auth credentials
- Optional provider credentials
  - WhatsApp QR
  - Telegram bot token
  - Gmail OAuth

---

**Install gcloud CLI (or use Console)**

**Option A: gcloud CLI** (recommended for automation)

    Install from [https://cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install)

    Initialize and authenticate:

    ```bash
    gcloud init
    gcloud auth login
    ```

    **Option B: Cloud Console**

    All steps can be done via the web UI at [https://console.cloud.google.com](https://console.cloud.google.com)

  
**Create a GCP project**

**CLI:**

    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    ```

    Enable billing at [https://console.cloud.google.com/billing](https://console.cloud.google.com/billing) (required for Compute Engine).

    Enable the Compute Engine API:

    ```bash
    gcloud services enable compute.googleapis.com
    ```

    **Console:**

    1. Go to IAM & Admin > Create Project
    2. Name it and create
    3. Enable billing for the project
    4. Navigate to APIs & Services > Enable APIs > search "Compute Engine API" > Enable

  
**Create the VM**

**Machine types:**

    | Type      | Specs                    | Cost               | Notes                                        |
    | --------- | ------------------------ | ------------------ | -------------------------------------------- |
    | e2-medium | 2 vCPU, 4GB RAM          | ~$25/mo            | Most reliable for local Docker builds        |
    | e2-small  | 2 vCPU, 2GB RAM          | ~$12/mo            | Minimum recommended for Docker build         |
    | e2-micro  | 2 vCPU (shared), 1GB RAM | Free tier eligible | Often fails with Docker build OOM (exit 137) |

    **CLI:**

    ```bash
    gcloud compute instances create openclaw-gateway \
      --zone=us-central1-a \
      --machine-type=e2-small \
      --boot-disk-size=20GB \
      --image-family=debian-12 \
      --image-project=debian-cloud
    ```

    **Console:**

    1. Go to Compute Engine > VM instances > Create instance
    2. Name: `openclaw-gateway`
    3. Region: `us-central1`, Zone: `us-central1-a`
    4. Machine type: `e2-small`
    5. Boot disk: Debian 12, 20GB
    6. Create

  
**SSH into the VM**

**CLI:**

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    **Console:**

    Click the "SSH" button next to your VM in the Compute Engine dashboard.

    Note: SSH key propagation can take 1-2 minutes after VM creation. If connection is refused, wait and retry.

  
**Install Docker (on the VM)**

```bash
    sudo apt-get update
    sudo apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
    ```

    Log out and back in for the group change to take effect:

    ```bash
    exit
    ```

    Then SSH back in:

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    Verify:

    ```bash
    docker --version
    docker compose version
    ```

  
**Clone the OpenClaw repository**

```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    This guide assumes you will build a custom image to guarantee binary persistence.

  
**Create persistent host directories**

Docker containers are ephemeral.
    All long-lived state must live on the host.

    ```bash
    mkdir -p ~/.openclaw
    mkdir -p ~/.openclaw/workspace
    ```

  
**Configure environment variables**

Create `.env` in the repository root.

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
    OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    Set `OPENCLAW_GATEWAY_TOKEN` when you want to manage the stable gateway
    token through `.env`; otherwise configure `gateway.auth.token` before
    relying on clients across restarts. If neither source exists, OpenClaw uses
    a runtime-only token for that startup. Generate a keyring password and paste
    it into `GOG_KEYRING_PASSWORD`:

    ```bash
    openssl rand -hex 32
    ```

    **Do not commit this file.**

    This `.env` file is for container/runtime env such as `OPENCLAW_GATEWAY_TOKEN`.
    Stored provider OAuth/API-key auth lives in the mounted
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`.

  
**Docker Compose configuration**

Create or update `docker-compose.yml`.

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # Recommended: keep the Gateway loopback-only on the VM; access via SSH tunnel.
          # To expose it publicly, remove the `127.0.0.1:` prefix and firewall accordingly.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` is only for bootstrap convenience, it is not a replacement for a proper gateway configuration. Still set auth (`gateway.auth.token` or password) and use safe bind settings for your deployment.

  
**Shared Docker VM runtime steps**

Use the shared runtime guide for the common Docker host flow:

  
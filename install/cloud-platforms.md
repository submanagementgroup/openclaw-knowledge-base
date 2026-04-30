---
domain: install
topic: "Cloud Platform Deployment: DigitalOcean, GCP, Azure, Fly.io, Oracle, and Hetzner"
type: procedure
keywords:
  - DigitalOcean
  - GCP
  - Azure
  - Fly.io
  - Oracle Cloud
  - Hetzner
  - cloud deployment
  - VPS
related:
  - install/installation-methods
  - install/docker
source:
  - install/digitalocean.md
  - install/gcp.md
  - install/azure.md
  - install/fly.md
  - install/oracle.md
---

Cloud platform deployment guides for OpenClaw: DigitalOcean, GCP, Azure, Fly.io, Oracle Cloud, and Hetzner.

## DigitalOcean

Run a persistent OpenClaw Gateway on a DigitalOcean Droplet.

## Prerequisites

- DigitalOcean account ([signup](https://cloud.digitalocean.com/registrations/new))
- SSH key pair (or willingness to use password auth)
- About 20 minutes

## Setup

    Use a clean base image (Ubuntu 24.04 LTS). Avoid third-party Marketplace 1-click images unless you have reviewed their startup scripts and firewall defaults.

    1. Log into [DigitalOcean](https://cloud.digitalocean.com/).
    2. Click **Create > Droplets**.
    3. Choose:
       - **Region:** Closest to you
       - **Image:** Ubuntu 24.04 LTS
       - **Size:** Basic, Regular, 1 vCPU / 1 GB RAM / 25 GB SSD
       - **Authentication:** SSH key (recommended) or password
    4. Click **Create Droplet** and note the IP address.

    ```bash
    ssh root@YOUR_DROPLET_IP

    apt update && apt upgrade -y

    # Install Node.js 24
    curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
    apt install -y nodejs

    # Install OpenClaw
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw --version
    ```

    ```bash
    openclaw onboard --install-daemon
    ```

    The wizard walks you through model auth, channel setup, gateway token generation, and daemon installation (systemd).

    ```bash
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ```

    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```

    The gateway binds to loopback by default. Pick one of these options.

    **Option A: SSH tunnel (simplest)**

    ```bash
    # From your local machine
    ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
    ```

    Then open `http://localhost:18789`.

    **Option B: Tailscale Serve**

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    tailscale up
    openclaw config set gateway.tailscale.mode serve
    openclaw gateway restart
    ```

    Then open `https://<magicdns>/` from any device on your tailnet.

    **Option C: Tailnet bind (no Serve)**

    ```bash
    openclaw config set gateway.bind tailnet
    openclaw gateway restart
    ```

    Then open `http://<tailscale-ip>:18789` (token required).

## Troubleshooting

**Gateway will not start** -- Run `openclaw doctor --non-interactive` and check logs with `journalctl --user -u openclaw-gateway.service -n 50`.

**Port already

## Google Cloud Platform (GCP)

# OpenClaw on GCP Compute Engine (Docker, Production VPS Guide)

## Goal

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

    **Option A: gcloud CLI** (recommended for automation)

    Install from [https://cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install)

    Initialize and authenticate:

    ```bash
    gcloud init
    gcloud auth login
    ```

    **Option B: Cloud Console**

    All steps can be done via the web UI at [https://console.cloud.google.com](https://console.cloud.google.com)

    **CLI:**

    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    ```

    Enable billing at [https://console.cloud.google.com/billing]

## Azure

# OpenClaw on Azure Linux VM

This guide sets up an Azure Linux VM with the Azure CLI, applies Network Security Group (NSG) hardening, configures Azure Bastion for SSH access, and installs OpenClaw.

## What you will do

- Create Azure networking (VNet, subnets, NSG) and compute resources with the Azure CLI
- Apply Network Security Group rules so VM SSH is allowed only from Azure Bastion
- Use Azure Bastion for SSH access (no public IP on the VM)
- Install OpenClaw with the installer script
- Verify the Gateway

## What you need

- An Azure subscription with permission to create compute and network resources
- Azure CLI installed (see [Azure CLI install steps](https://learn.microsoft.com/cli/azure/install-azure-cli) if needed)
- An SSH key pair (the guide covers generating one if needed)
- ~20-30 minutes

## Configure deployment

    ```bash
    az login
    az extension add -n ssh
    ```

    The `ssh` extension is required for Azure Bastion native SSH tunneling.

    ```bash
    az provider register --namespace Microsoft.Compute
    az provider register --namespace Microsoft.Network
    ```

    Verify registration. Wait until both show `Registered`.

    ```bash
    az provider show --namespace Microsoft.Compute --query registrationState -o tsv
    az provider show --namespace Microsoft.Network --query registrationState -o tsv
    ```

    ```bash
    RG="rg-openclaw"
    LOCATION="westus2"
    VNET_NAME="vnet-openclaw"
    VNET_PREFIX="10.40.0.0/16"
    VM_SUBNET_NAME="snet-openclaw-vm"
    VM_SUBNET_PREFIX="10.40.2.0/24"
    BASTION_SUBNET_PREFIX="10.40.1.0/26"
    NSG_NAME="nsg-openclaw-vm"
    VM_NAME="vm-openclaw"
    ADMIN_USERNAME="openclaw"
    BASTION_NAME="bas-openclaw"
    BASTION_PIP_NAME="pip-openclaw-bastion"
    ```

    Adjust names and CIDR ranges to fit your environment. The Bastion subnet must be at least `/26`.

    Use your existing public key if you have one:

    ```bash
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

    If you don't have an SSH key yet, generate one:

    ```bash
    ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519 -C "you@example.com"
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

    ```bash
    VM_SIZE="Standard_B2as_v2"
    OS_DISK_SIZE_GB=64
    ```

    Choose a VM size and OS disk size available in your subscription and region:

    - Start smaller for light usage and scale up later
    - Use more vCPU/RAM/disk for heavier automation, more channels, or larger model/tool workloads
    - 

## Fly.io

# Fly.io Deployment

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

    Edit `fly.toml` to match your app name and requirements.

    **Security note:** The default config exposes a public URL. For a hardened deployment with no public IP, see [Private Deployment](#private-deployment-hardened) or use `fly.private.toml`.

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

    **Key settings:**

    | Setting                        | Why                                                                         |
    | ------------------------------ | --------------------------------------------------------------------------- |
    | `--bind lan`                   | Binds to `0.0.0.0` so Fly's proxy can reach the gateway                     |
    | `--allow-unconfigured`         | Starts without a config file (you'll create one after)                      |
    | `internal_port = 3000

## Oracle Cloud (Always Free)

Run a persistent OpenClaw Gateway on Oracle Cloud's **Always Free** ARM tier (up to 4 OCPU, 24 GB RAM, 200 GB storage) at no cost.

## Prerequisites

- Oracle Cloud account ([signup](https://www.oracle.com/cloud/free/)) -- see [community signup guide](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd) if you hit issues
- Tailscale account (free at [tailscale.com](https://tailscale.com))
- An SSH key pair
- About 30 minutes

## Setup

    1. Log into [Oracle Cloud Console](https://cloud.oracle.com/).
    2. Navigate to **Compute > Instances > Create Instance**.
    3. Configure:
       - **Name:** `openclaw`
       - **Image:** Ubuntu 24.04 (aarch64)
       - **Shape:** `VM.Standard.A1.Flex` (Ampere ARM)
       - **OCPUs:** 2 (or up to 4)
       - **Memory:** 12 GB (or up to 24 GB)
       - **Boot volume:** 50 GB (up to 200 GB free)
       - **SSH key:** Add your public key
    4. Click **Create** and note the public IP address.

    If instance creation fails with "Out of capacity", try a different availability domain or retry later. Free tier capacity is limited.

    ```bash
    ssh ubuntu@YOUR_PUBLIC_IP

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y build-essential
    ```

    `build-essential` is required for ARM compilation of some dependencies.

    ```bash
    sudo hostnamectl set-hostname openclaw
    sudo passwd ubuntu
    sudo loginctl enable-linger ubuntu
    ```

    Enabling linger keeps user services running after logout.

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up --ssh --hostname=openclaw
    ```

    From now on, connect via Tailscale: `ssh ubuntu@openclaw`.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    source ~/.bashrc
    ```

    When prompted "How do you want to hatch your bot?", select **Do this later**.

    Use token auth with Tailscale Serve for secure remote access.

    ```bash
    openclaw config set gateway.bind loopback
    openclaw

## Hetzner

# OpenClaw on Hetzner (Docker, Production VPS Guide)

## Goal

Run a persistent OpenClaw Gateway on a Hetzner VPS using Docker, with durable state, baked-in binaries, and safe restart behavior.

If you want “OpenClaw 24/7 for ~$5”, this is the simplest reliable setup.
Hetzner pricing changes; pick the smallest Debian/Ubuntu VPS and scale up if you hit OOMs.

Security model reminder:

- Company-shared agents are fine when everyone is in the same trust boundary and the runtime is business-only.
- Keep strict separation: dedicated VPS/runtime + dedicated accounts; no personal Apple/Google/browser/password-manager profiles on that host.
- If users are adversarial to each other, split by gateway/host/OS user.

See [Security](/gateway/security) and [VPS hosting](/vps).

## What are we doing (simple terms)?

- Rent a small Linux server (Hetzner VPS)
- Install Docker (isolated app runtime)
- Start the OpenClaw Gateway in Docker
- Persist `~/.openclaw` + `~/.openclaw/workspace` on the host (survives restarts/rebuilds)
- Access the Control UI from your laptop via an SSH tunnel

That mounted `~/.openclaw` state includes `openclaw.json`, per-agent
`agents/<agentId>/agent/auth-profiles.json`, and `.env`.

The Gateway can be accessed via:

- SSH port forwarding from your laptop
- Direct port exposure if you manage firewalling and tokens yourself

This guide assumes Ubuntu or Debian on Hetzner.  
If you are on another Linux VPS, map packages accordingly.
For the generic Docker flow, see [Docker](/install/docker).

---

## Quick path (experienced operators)

1. Provision Hetzner VPS
2. Install Docker
3. Clone OpenClaw repository
4. Create persistent host directories
5. Configure `.env` and `docker-compose.yml`
6. Bake required binaries into the image
7. `docker compose up -d`
8. Verify persistence and Gateway access

---

## What you need

- Hetzner VPS with root access
- SSH access from your laptop
- Basic comfort with SSH + copy/paste
- ~20 minutes
- Docker and Docker Compose
- M

## Hostinger

Run a persistent OpenClaw Gateway on [Hostinger](https://www.hostinger.com/openclaw) via a **1-Click** managed deployment or a **VPS** install.

## Prerequisites

- Hostinger account ([signup](https://www.hostinger.com/openclaw))
- About 5-10 minutes

## Option A: 1-Click OpenClaw

The fastest way to get started. Hostinger handles infrastructure, Docker, and automatic updates.

    1. From the [Hostinger OpenClaw page](https://www.hostinger.com/openclaw), choose a Managed OpenClaw plan and complete checkout.

    During checkout you can select **Ready-to-Use AI** credits that are pre-purchased and integrated instantly inside OpenClaw -- no external accounts or API keys from other providers needed. You can start chatting right away. Alternatively, provide your own key from Anthropic, OpenAI, Google Gemini, or xAI during setup.

    Choose one or more channels to connect:

    - **WhatsApp** -- scan the QR code shown in the setup wizard.
    - **Telegram** -- paste the bot token from [BotFather](https://t.me/BotFather).

    Click **Finish** to deploy the instance. Once ready, access the OpenClaw dashboard from **OpenClaw Overview** in hPanel.

## Option B: OpenClaw on VPS

More control over your server. Hostinger deploys OpenClaw via Docker on your VPS and you manage it through the **Docker Manager** in hPanel.

    1. From the [Hostinger OpenClaw page](https://www.hostinger.com/openclaw), choose an OpenClaw on VPS plan and complete checkout.

    You can select **Ready-to-Use

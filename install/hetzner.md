---
domain: install
topic: "Hetzner: Production VPS Deployment with Docker and Durable State"
type: procedure
keywords:
  - Hetzner
  - hetzner VPS
  - docker openclaw
  - openclaw production VPS
  - persistent openclaw
  - docker-compose openclaw
  - bake binaries docker
  - SSH tunnel control UI
  - openclaw docker compose
source: install/hetzner.md
related:
  - install/docker
  - install/vps-hosting
  - platforms/digitalocean
  - platforms/oracle
  - gateway/remote-access
---

Run a persistent OpenClaw Gateway on a Hetzner VPS using Docker, with durable state, baked-in binaries, and safe restart behavior. This is the recommended approach for "OpenClaw 24/7 for ~$5/month."

## Security Model Reminder

- Company-shared agents are fine when everyone is in the same trust boundary and the runtime is business-only.
- Keep strict separation: dedicated VPS/runtime + dedicated accounts; no personal Apple/Google/browser/password-manager profiles on that host.
- If users are adversarial to each other, split by gateway/host/OS user.

## Quick Path (Experienced Operators)

1. Provision Hetzner VPS (Ubuntu or Debian)
2. Install Docker and Docker Compose
3. Create persistent host directories for `~/.openclaw` and `~/.openclaw/workspace`
4. Configure `.env` and `docker-compose.yml`
5. Bake required binaries into the Docker image at build time
6. `docker compose up -d`
7. Verify persistence and Gateway access

## What You Need

- Hetzner VPS with root access + SSH access from your laptop
- Docker and Docker Compose
- Model auth credentials
- Optional provider credentials (WhatsApp QR, Telegram bot token, Gmail OAuth)

## What Are We Doing?

- Rent a small Linux server (Hetzner VPS)
- Install Docker (isolated app runtime)
- Start the OpenClaw Gateway in Docker
- Persist `~/.openclaw` + `~/.openclaw/workspace` on the host (survives restarts/rebuilds)
- Access the Control UI from your laptop via SSH tunnel

The mounted `~/.openclaw` state includes `openclaw.json`, per-agent `agents/<agentId>/agent/auth-profiles.json`, and `.env`.

## Install Steps

### Step 1: Provision the VPS

Create an Ubuntu or Debian VPS in Hetzner. Connect via SSH:

```bash
ssh root@YOUR_HETZNER_IP
```

### Step 2: Install Docker

```bash
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker
```

### Step 3: Create Persistent Host Directories

```bash
mkdir -p /home/node/.openclaw
mkdir -p /home/node/.openclaw/workspace
```

### Step 4: Configure .env

```bash
cat > /home/node/.openclaw/.env << 'EOF'
ANTHROPIC_API_KEY=sk-ant-...
# Add other provider keys as needed
EOF
chmod 600 /home/node/.openclaw/.env
```

### Step 5: Bake Binaries into the Docker Image

All external binaries required by skills must be installed **at image build time** — anything installed at runtime is lost on container restart.

Example Dockerfile:

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# Example: Gmail CLI (gogcli — installs as `gog`)
RUN curl -L https://github.com/steipete/gogcli/releases/latest/download/gogcli_linux_amd64.tar.gz \
  | tar -xzO gog > /usr/local/bin/gog && chmod +x /usr/local/bin/gog

# Add more binaries with the same pattern (use arm64 assets for ARM VMs)
# For reproducible builds, pin versioned release URLs

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
RUN corepack enable && pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

ENV NODE_ENV=production
CMD ["node","dist/index.js"]
```

If you add skills later that need additional binaries: update the Dockerfile, rebuild the image, and restart containers.

### Step 6: docker-compose.yml

```yaml
services:
  openclaw-gateway:
    build: .
    restart: unless-stopped
    volumes:
      - /home/node/.openclaw:/home/node/.openclaw
    ports:
      - "127.0.0.1:18789:18789"
    env_file:
      - /home/node/.openclaw/.env
```

Binding to `127.0.0.1` keeps the port loopback-only; access via SSH tunnel.

### Step 7: Start

```bash
docker compose up -d
docker compose logs -f openclaw-gateway
# Expected: [gateway] listening on ws://0.0.0.0:18789
```

### Step 8: Access the Control UI

From your local machine:

```bash
ssh -L 18789:localhost:18789 root@YOUR_HETZNER_IP
# Then open: http://localhost:18789
```

## What Persists Where

| Component | Location | Persistence |
|-----------|----------|-------------|
| Gateway config | `/home/node/.openclaw/` | Host volume mount |
| Model auth profiles | `/home/node/.openclaw/agents/` | Host volume mount |
| Agent workspace | `/home/node/.openclaw/workspace/` | Host volume mount |
| WhatsApp session | `/home/node/.openclaw/` | Host volume mount |
| External binaries | `/usr/local/bin/` | Docker image (baked at build time) |
| Plugin runtime deps | `/var/lib/openclaw/plugin-runtime-deps/` | Docker named volume |
| Container filesystem | Ephemeral | Safe to destroy and recreate |

## Updates

```bash
git pull
docker compose build
docker compose up -d
```

## Related

- [Docker install](/install/docker)
- [VPS hosting overview](/install/vps-hosting)
- [DigitalOcean guide](/platforms/digitalocean)
- [Docker VM runtime (shared steps)](/install/docker-vm-runtime)

---
domain: install
topic: "Docker VM Runtime: Shared Steps for Cloud VM OpenClaw Deployments (GCP, Hetzner, VPS)"
type: reference
keywords:
  - docker VM runtime
  - bake binaries docker
  - dockerfile openclaw
  - persistent openclaw docker
  - docker-compose persistence
  - openclaw state docker
  - plugin runtime deps
  - docker update openclaw
  - docker binary baking
source: install/docker-vm-runtime.md
related:
  - install/docker
  - install/hetzner
  - install/vps-hosting
---

Shared runtime steps for VM-based Docker installs (GCP, Hetzner, and similar VPS providers). The core principle: all external binaries must be baked into the Docker image at build time — anything installed at runtime is lost on container restart.

## Bake Required Binaries Into the Image

Installing binaries inside a running container is a trap. All external binaries required by skills must be installed **at image build time**.

The examples below show three common binaries (gog, goplaces, wacli) as a pattern only — not a complete list. Use the same `RUN curl | tar` pattern for any additional binaries.

If you add new skills later that depend on additional binaries, you must:
1. Update the Dockerfile
2. Rebuild the image
3. Restart the containers

### Example Dockerfile

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# Example binary 1: Gmail CLI (gogcli — installs as `gog`)
# Copy the current Linux asset URL from https://github.com/steipete/gogcli/releases
RUN curl -L https://github.com/steipete/gogcli/releases/latest/download/gogcli_linux_amd64.tar.gz \
  | tar -xzO gog > /usr/local/bin/gog && chmod +x /usr/local/bin/gog

# Example binary 2: Google Places CLI
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_linux_amd64.tar.gz \
  | tar -xzO goplaces > /usr/local/bin/goplaces && chmod +x /usr/local/bin/goplaces

# Example binary 3: WhatsApp CLI
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli-linux-amd64.tar.gz \
  | tar -xzO wacli > /usr/local/bin/wacli && chmod +x /usr/local/bin/wacli

# Add more binaries here using the same pattern
# For ARM-based VMs, use arm64 assets
# For reproducible builds, pin versioned release URLs

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production
CMD ["node","dist/index.js"]
```

## Verify Binaries and Gateway

After starting containers, verify binaries are accessible:

```bash
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
docker compose exec openclaw-gateway which wacli
```

Verify Gateway is running:

```bash
docker compose logs -f openclaw-gateway
# Expected output: [gateway] listening on ws://0.0.0.0:18789
```

## What Persists Where

| Component | Location | Persistence Mechanism | Notes |
|-----------|----------|-----------------------|-------|
| Gateway config | `/home/node/.openclaw/` | Host volume mount | Includes `openclaw.json`, `.env` |
| Model auth profiles | `/home/node/.openclaw/agents/` | Host volume mount | `agents/<agentId>/agent/auth-profiles.json` |
| Skill configs | `/home/node/.openclaw/skills/` | Host volume mount | Skill-level state |
| Agent workspace | `/home/node/.openclaw/workspace/` | Host volume mount | Code and agent artifacts |
| WhatsApp session | `/home/node/.openclaw/` | Host volume mount | Preserves QR login |
| Gmail keyring | `/home/node/.openclaw/` | Host volume + password | Requires `GOG_KEYRING_PASSWORD` |
| Plugin runtime deps | `/var/lib/openclaw/plugin-runtime-deps/` | Docker named volume | Generated bundled plugin deps |
| External binaries | `/usr/local/bin/` | Docker image | **Must be baked at build time** |
| Node runtime | Container filesystem | Docker image | Rebuilt every image build |
| OS packages | Container filesystem | Docker image | Do not install at runtime |
| Docker container | Ephemeral | Restartable | Safe to destroy |

## Updates

```bash
git pull
docker compose build
docker compose up -d
```

## Related

- [Docker install](/install/docker)
- [Hetzner guide](/install/hetzner)
- [VPS hosting overview](/install/vps-hosting)

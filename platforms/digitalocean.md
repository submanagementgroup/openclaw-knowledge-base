---
domain: platforms
topic: "DigitalOcean: Running OpenClaw on a Droplet ($6/month)"
type: procedure
keywords:
  - DigitalOcean
  - droplet
  - VPS hosting
  - openclaw DigitalOcean
  - Ubuntu 24.04 droplet
  - openclaw onboard daemon
  - SSH tunnel control UI
  - Tailscale Serve VPS
  - swap memory
source: platforms/digitalocean.md
related:
  - platforms/oracle
  - install/vps-hosting
  - install/hetzner
  - gateway/remote-access
---

Run a persistent OpenClaw Gateway on DigitalOcean for $6/month (or $4/month with reserved pricing). For a free alternative, see the Oracle Cloud guide.

## Cost Comparison (2026)

| Provider | Plan | Specs | Price/mo | Notes |
|----------|------|-------|----------|-------|
| Oracle Cloud | Always Free ARM | up to 4 OCPU, 24GB RAM | $0 | ARM, limited signup |
| Hetzner | CX22 | 2 vCPU, 4GB RAM | ~$4 | Best price/perf |
| DigitalOcean | Basic | 1 vCPU, 1GB RAM | $6 | Simplest UI |
| Vultr | Cloud Compute | 1 vCPU, 1GB RAM | $6 | Many locations |
| Linode | Nanode | 1 vCPU, 1GB RAM | $5 | Akamai-owned |

## Prerequisites

- DigitalOcean account (signup with $200 free credit available)
- SSH key pair
- ~20 minutes

## 1) Create a Droplet

Use a clean base image (Ubuntu 24.04 LTS). Avoid third-party Marketplace 1-click images.

1. Log into DigitalOcean → **Create → Droplets**
2. Choose: Region (closest to you), Image (Ubuntu 24.04 LTS), Size ($6/mo — 1 vCPU, 1GB RAM, 25GB SSD), Authentication (SSH key recommended)
3. Click **Create Droplet** and note the IP address.

## 2) Connect and Install

```bash
ssh root@YOUR_DROPLET_IP

# Update system
apt update && apt upgrade -y

# Install Node.js 24
curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
apt install -y nodejs

# Install OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash

# Verify
openclaw --version
```

## 3) Run Onboarding

```bash
openclaw onboard --install-daemon
```

The wizard configures: model auth, channel setup, gateway token, and systemd daemon installation.

## 4) Verify the Gateway

```bash
openclaw status
systemctl --user status openclaw-gateway.service
journalctl --user -u openclaw-gateway.service -f
```

## 5) Access the Dashboard

The gateway binds to loopback by default.

**Option A: SSH Tunnel (recommended)**

```bash
# From your local machine
ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
# Then open: http://localhost:18789
```

**Option B: Tailscale Serve (HTTPS, loopback-only)**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
openclaw config set gateway.tailscale.mode serve
openclaw gateway restart
# Open: https://<magicdns>/
```

Tailscale Serve keeps the Gateway loopback-only and authenticates Control UI/WebSocket traffic via Tailscale identity headers. To require explicit credentials instead, set `gateway.auth.allowTailscale: false` and use `gateway.auth.mode: "token"` or `"password"`.

**Option C: Tailnet bind (no Serve)**

```bash
openclaw config set gateway.bind tailnet
openclaw gateway restart
# Open: http://<tailscale-ip>:18789 (token required)
```

## Optimizations for 1GB RAM

### Add Swap (Recommended)

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Use API-Based Models

If hitting OOMs with local models, use API-based models (Claude, GPT) or set a smaller primary model in `agents.defaults.model.primary`.

## Persistence

All state lives in `~/.openclaw/` (config, channel state, sessions) and `~/.openclaw/workspace/` (SOUL.md, memory). Back up periodically:

```bash
openclaw backup create
```

## Troubleshooting

```bash
# Gateway won't start
openclaw gateway status
openclaw doctor --non-interactive
journalctl --user -u openclaw-gateway.service --no-pager -n 50

# Port already in use
lsof -i :18789
```

## Related

- [Oracle Cloud guide](/platforms/oracle)
- [Hetzner guide](/install/hetzner)
- [VPS hosting overview](/install/vps-hosting)
- [Tailscale remote access](/gateway/remote-access)

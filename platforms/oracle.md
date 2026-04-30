---
domain: platforms
topic: "Oracle Cloud: Running OpenClaw on Always Free ARM (24GB RAM, $0/month)"
type: procedure
keywords:
  - Oracle Cloud
  - OCI
  - Always Free ARM
  - Ampere A1
  - free VPS OpenClaw
  - oracle arm ubuntu
  - Tailscale Serve oracle
  - VCN security
  - token auth oracle
  - openclaw oci
source: platforms/oracle.md
related:
  - platforms/digitalocean
  - install/vps-hosting
  - gateway/remote-access
---

Run a persistent OpenClaw Gateway on Oracle Cloud's Always Free ARM tier — up to 4 OCPUs and 24GB RAM at no cost. ARM architecture; most things work, but some binaries may be x86-only. Signup can be finicky.

## Cost Comparison (2026)

| Provider | Plan | Specs | Price/mo | Notes |
|----------|------|-------|----------|-------|
| Oracle Cloud | Always Free ARM | up to 4 OCPU, 24GB RAM | $0 | ARM, limited capacity |
| Hetzner | CX22 | 2 vCPU, 4GB RAM | ~$4 | Best paid price/perf |
| DigitalOcean | Basic | 1 vCPU, 1GB RAM | $6 | Simplest UI |
| Vultr | Cloud Compute | 1 vCPU, 1GB RAM | $6 | Many locations |
| Linode | Nanode | 1 vCPU, 1GB RAM | $5 | Akamai-owned |

## Prerequisites

- Oracle Cloud account (signup at [oracle.com/cloud/free](https://www.oracle.com/cloud/free/))
- Tailscale account (free at [tailscale.com](https://tailscale.com))
- ~30 minutes

## 1) Create an OCI Instance

1. Log into [Oracle Cloud Console](https://cloud.oracle.com/)
2. Navigate to **Compute → Instances → Create Instance**
3. Configure:
   - **Name:** `openclaw`
   - **Image:** Ubuntu 24.04 (aarch64)
   - **Shape:** `VM.Standard.A1.Flex` (Ampere ARM)
   - **OCPUs:** 2 (or up to 4)
   - **Memory:** 12 GB (or up to 24 GB)
   - **Boot volume:** 50 GB (up to 200 GB free)
   - **SSH key:** add your public key
4. Click **Create** and note the public IP address.

**Tip:** If creation fails with "Out of capacity", try a different availability domain or retry later. Free tier capacity is limited.

## 2) Connect and Update

```bash
ssh ubuntu@YOUR_PUBLIC_IP

sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential  # required for ARM compilation of some dependencies
```

## 3) Configure User and Hostname

```bash
sudo hostnamectl set-hostname openclaw
sudo passwd ubuntu

# Enable lingering (keeps user services running after logout)
sudo loginctl enable-linger ubuntu
```

## 4) Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=openclaw
tailscale status
```

Tailscale SSH lets you connect via `ssh ubuntu@openclaw` from any device on your tailnet — no public IP needed. From now on, prefer Tailscale for connectivity.

## 5) Install OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
source ~/.bashrc
```

When prompted "How do you want to hatch your bot?", select **"Do this later"**.

## 6) Configure Gateway and Enable Tailscale Serve

```bash
# Keep Gateway private on the VM
openclaw config set gateway.bind loopback

# Require auth
openclaw config set gateway.auth.mode token
openclaw doctor --generate-gateway-token

# Expose over Tailscale Serve (HTTPS + tailnet access)
openclaw config set gateway.tailscale.mode serve
openclaw config set gateway.trustedProxies '["127.0.0.1"]'

systemctl --user restart openclaw-gateway.service
```

`gateway.trustedProxies=["127.0.0.1"]` is for the local Tailscale Serve proxy only. It is **not** `gateway.auth.mode: "trusted-proxy"` — gateway auth remains token-based.

## 7) Verify

```bash
openclaw --version
systemctl --user status openclaw-gateway.service
tailscale serve status
curl http://localhost:18789
```

## 8) Lock Down VCN Security

Block all traffic except Tailscale at the network edge (OCI Virtual Cloud Network):

1. **Networking → Virtual Cloud Networks** → click your VCN → **Security Lists** → Default Security List
2. **Remove** all ingress rules except: `0.0.0.0/0 UDP 41641` (Tailscale)
3. Keep default egress rules (allow all outbound)

This blocks SSH port 22, HTTP, HTTPS, and everything else at the network level. Connect only via Tailscale from now on.

## Access the Control UI

From any device on your Tailscale network:

```
https://openclaw.<tailnet-name>.ts.net/
```

Replace `<tailnet-name>` with your tailnet name (visible in `tailscale status`).

## Connect Your Channels

```bash
# Telegram
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>

# WhatsApp
openclaw channels login whatsapp
# Scan QR code
```

See [Channels](/channels/channels-overview) for other providers.

## After Applying

```bash
openclaw doctor
```

## Related

- [DigitalOcean guide](/platforms/digitalocean)
- [VPS hosting overview](/install/vps-hosting)
- [Tailscale remote access](/gateway/remote-access)

---
domain: install
topic: "Hostinger: 1-Click Managed OpenClaw or VPS Docker Install"
type: procedure
keywords:
  - Hostinger
  - Hostinger OpenClaw
  - 1-click openclaw
  - managed openclaw
  - hostinger VPS
  - docker manager hPanel
  - Ready-to-Use AI credits
  - hostinger WhatsApp
  - hostinger Telegram
source: install/hostinger.md
related:
  - install/docker
  - install/vps-hosting
  - platforms/digitalocean
---

Run a persistent OpenClaw Gateway on Hostinger via a 1-Click managed deployment or a VPS Docker install. Hostinger offers a dedicated OpenClaw hosting page at [hostinger.com/openclaw](https://www.hostinger.com/openclaw).

## Prerequisites

- Hostinger account
- About 5–10 minutes

## Option A: 1-Click Managed OpenClaw (Fastest)

Hostinger handles infrastructure, Docker, and automatic updates.

**Steps:**

1. From the [Hostinger OpenClaw page](https://www.hostinger.com/openclaw), choose a Managed OpenClaw plan and complete checkout.

   During checkout you can select **Ready-to-Use AI** credits — pre-purchased credits integrated instantly inside OpenClaw, so you can start chatting without external accounts or API keys. Alternatively, provide your own key from Anthropic, OpenAI, Google Gemini, or xAI during setup.

2. **Select messaging channels:**
   - **WhatsApp** — scan the QR code shown in the setup wizard.
   - **Telegram** — paste the bot token from BotFather.

3. Click **Finish** to deploy. Once ready, access the OpenClaw dashboard from **OpenClaw Overview** in hPanel.

## Option B: OpenClaw on VPS (More Control)

Hostinger deploys OpenClaw via Docker on your VPS; you manage it through the **Docker Manager** in hPanel.

**Steps:**

1. From the [Hostinger OpenClaw page](https://www.hostinger.com/openclaw), choose an OpenClaw on VPS plan and complete checkout.

   Ready-to-Use AI credits are also available here.

2. **Configure OpenClaw:**
   - **Gateway token** — auto-generated; save it.
   - **WhatsApp number** — your number with country code (optional).
   - **Telegram bot token** — from BotFather (optional).
   - **API keys** — only needed if you did not select Ready-to-Use AI credits.

3. Click **Deploy**. Once running, open the OpenClaw dashboard from hPanel → **Open**.

Logs, restarts, and updates are managed from the Docker Manager interface in hPanel. To update, press **Update** in Docker Manager to pull the latest image.

## Verify Your Setup

Send "Hi" to your assistant on the channel you connected. OpenClaw will reply and walk you through initial preferences.

## Related

- [Docker install](/install/docker)
- [VPS hosting overview](/install/vps-hosting)
- [DigitalOcean guide](/platforms/digitalocean)

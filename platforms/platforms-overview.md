---
domain: platforms
topic: "Platforms Overview: OS Support, VPS Hosting, and Gateway Service Install"
type: concept
keywords:
  - platforms overview
  - OS support
  - macOS Linux Windows
  - VPS hosting
  - gateway service install
  - openclaw runtime
  - Node runtime
  - companion apps
  - systemd LaunchAgent scheduled task
source: platforms/index.md
related:
  - platforms/macos-app
  - platforms/ios-android
  - install/vps-hosting
  - install/installation-methods
  - gateway/gateway-runbook
---

OpenClaw core is written in TypeScript. **Node is the recommended runtime**. Bun is not recommended for the Gateway — known issues with WhatsApp and Telegram channels.

Companion apps exist for macOS (menu bar app) and mobile nodes (iOS/Android). Windows and Linux companion apps are planned; the Gateway is fully supported on those platforms today. Native Windows companion apps are also planned; the Gateway is recommended via WSL2.

## Choose Your OS

- macOS: [macOS](/platforms/macos-app)
- iOS: [iOS/Android nodes](/platforms/ios-android)
- Android: [iOS/Android nodes](/platforms/ios-android)
- Windows: [Other platforms](/platforms/other-platforms)
- Linux: [Other platforms](/platforms/other-platforms)

## VPS and Hosting

- VPS overview: [VPS hosting](/install/vps-hosting)
- Fly.io: [Fly.io](/install/cloud-platforms)
- Hetzner (Docker): [Hetzner](/install/hetzner)
- GCP (Compute Engine): [Cloud platforms](/install/cloud-platforms)
- Azure (Linux VM): [Cloud platforms](/install/cloud-platforms)
- DigitalOcean: [DigitalOcean](/platforms/digitalocean)
- Oracle Cloud (free ARM): [Oracle Cloud](/platforms/oracle)

## Gateway Service Install

Use any of these methods (all supported):

- **Wizard (recommended):** `openclaw onboard --install-daemon`
- **Direct:** `openclaw gateway install`
- **Configure flow:** `openclaw configure` → select Gateway service
- **Repair/migrate:** `openclaw doctor` (offers to install or fix the service)

The service target depends on OS:

| OS | Service Manager | Service Name |
|----|----------------|--------------|
| macOS | LaunchAgent | `ai.openclaw.gateway` (or `ai.openclaw.<profile>`) |
| Linux/WSL2 | systemd user service | `openclaw-gateway[-<profile>].service` |
| Native Windows | Scheduled Task | `OpenClaw Gateway` (or `OpenClaw Gateway (<profile>)`) with startup-folder fallback |

## Related

- [Install overview](/install/installation-methods)
- [macOS app](/platforms/macos-app)
- [iOS/Android nodes](/platforms/ios-android)
- [Gateway runbook](/gateway/gateway-runbook)

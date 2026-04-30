---
domain: install
topic: "Raspberry Pi Installation: ARM Setup and macOS VM"
type: procedure
keywords:
  - Raspberry Pi
  - ARM
  - RPi
  - macOS VM
  - Raspberry Pi install
related:
  - platforms/raspberry-pi
  - install/installation-methods
source:
  - install/raspberry-pi.md
  - install/macos-vm.md
---

Installing OpenClaw on Raspberry Pi and running in macOS VMs.

## Raspberry Pi

Run a persistent, always-on OpenClaw Gateway on a Raspberry Pi. Since the Pi is just the gateway (models run in the cloud via API), even a modest Pi handles the workload well.

## Prerequisites

- Raspberry Pi 4 or 5 with 2 GB+ RAM (4 GB recommended)
- MicroSD card (16 GB+) or USB SSD (better performance)
- Official Pi power supply
- Network connection (Ethernet or WiFi)
- 64-bit Raspberry Pi OS (required -- do not use 32-bit)
- About 30 minutes

## Setup

    Use **Raspberry Pi OS Lite (64-bit)** -- no desktop needed for a headless server.

    1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
    2. Choose OS: **Raspberry Pi OS Lite (64-bit)**.
    3. In the settings dialog, pre-configure:
       - Hostname: `gateway-host`
       - Enable SSH
       - Set username and password
       - Configure WiFi (if not using Ethernet)
    4. Flash to your SD card or USB drive, insert it, and boot the Pi.

    ```bash
    ssh user@gateway-host
    ```

    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl build-essential

    # Set timezone (important for cron and reminders)
    sudo timedatectl set-timezone America/Chicago
    ```

    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt install -y nodejs
    node --version
    ```

    ```bash
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # Reduce swappiness for low-RAM devices
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```

    ```bash
    openclaw onboard --install-daemon
    ```

    Follow the wizard. API keys are recommended over OAuth for headless devices. Telegram is the easiest channel to start with.

    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```

    On your computer, get a dashboard URL from the Pi:

    ```bash
    ssh user@gateway-host 'openclaw dashboard --no-open'
    ```

    Then create an SSH tunnel in another terminal:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
    ```

    Open the printed URL in your local browser. For always-on remote access, see [Tailscale integration](/gateway/tailscale).

## Performance tips

**Use a USB SSD** -- SD cards are slow and wear out. A USB SSD dramatically improves performance. See the [Pi USB boot guide](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot).

**Enable module compile cache** -- Speeds up repeated CLI invocations on lower-power Pi hosts:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

**Reduce memory usage** -- For headless setups, free GPU memory and disable unused services:

```bash
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt
sudo systemctl disable bluetooth
```

## Troubleshooting

**Out of memory** -- Verify swap is active with `free -h`. Disable unused services (`sudo systemctl disable cups bluetooth avahi-daemon`). Use API-based models only.

**Slow performance** -- Use a USB SSD instead of an SD card. Check for CPU throttling with `vcgencmd get_throttled` (should return `0x0`).

**Service will not start** -- Check logs with `journalctl --user -u openclaw-gateway.service --no-pager -n 100` and run `openclaw doctor --non-interactive`. If this is a headless Pi, also verify lingering is enabled: `sudo loginctl enable-linger "$(whoami)"`.

**ARM binary issues** -- If a skill fails with "exec format error", check whether the binary has an ARM64 build. Verify architecture with `uname -m` (should show `aarch64`).

**WiFi drops** -- Disable WiFi power management: `sudo iwconfig wlan0 power off`.

## Next steps

- [Channels](/channels) -- connect Telegram, WhatsApp, Discord, and more
- [Gateway configuration](/gateway/configuration) -- all config options
- [Updating](/install/updating) -- keep OpenClaw up to date

## Related

- [Install overview](/install)
- [Linux server](/vps)
- [Platforms](/platforms)

## macOS VM

# OpenClaw on macOS VMs (Sandboxing)

## Recommended default (most users)

- **Small Linux VPS** for an always-on Gateway and low cost. See [VPS hosting](/vps).
- **Dedicated hardware** (Mac mini or Linux box) if you want full control and a **residential IP** for browser automation. Many sites block data center IPs, so local browsing often works better.
- **Hybrid:** keep the Gateway on a cheap VPS, and connect your Mac as a **node** when you need browser/UI automation. See [Nodes](/nodes) and [Gateway remote](/gateway/remote).

Use a macOS VM when you specifically need macOS-only capabilities (iMessage/BlueBubbles) or want strict isolation from your daily Mac.

## macOS VM options

### Local VM on your Apple Silicon Mac (Lume)

Run OpenClaw in a sandboxed macOS VM on your existing Apple Silicon Mac using [Lume](https://cua.ai/docs/lume).

This gives you:

- Full macOS environment in isolation (your host stays clean)
- iMessage support via BlueBubbles (impossible on Linux/Windows)
- Instant reset by cloning VMs
- No extra hardware or cloud costs

### Hosted Mac providers (cloud)

If you want macOS in the cloud, hosted Mac providers work too:

- [MacStadium](https://www.macstadium.com/) (hosted Macs)
- Other hosted Mac vendors also work; follow their VM + SSH docs

Once you have SSH access to a macOS VM, continue at step 6 below.

---

## Quick path (Lume, experienced users)

1. Install Lume
2. `lume create openclaw --os macos --ipsw latest`
3. Complete Setup Assistant, enable Remote Login (SSH)
4. `lume run openclaw --no-display`
5. SSH in, install OpenClaw, configure channels
6. Done

---

## What you need (Lume)

- Apple Silicon Mac (M1/M2/M3/M4)
- macOS Sequoia or later on the host
- ~60 GB free disk space per VM
- ~20 minutes

---

## 1) Install Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

If `~/.local/bin` isn't in your PATH:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

Verify:

```bash
lume --version
```

Docs: [Lume Installation](https://cua.ai/docs/lume/guide/getting-started/installation)

---

## 2) Create the macOS VM

```bash
lume create openclaw --os macos --ipsw latest
```

This downloads macOS and creates the VM. A VNC window opens automatically.

The download can take a while depending on your connection.

---

## 3) Complete Setup Assistant

In the VNC window:

1. Select language and region
2. Skip Apple ID (or sign in if you want iMessage later)
3. Create a user account (remember the username and password)
4. Skip all optional features

After setup completes, enable SSH:

1. Open System Settings → General → Sharing
2. Enable "Remote Login"

---

## 4) Get the VM IP address

```bash
lume get openclaw
```

Look for the IP address (usually `192.168.64.x`).

---

## 5) SSH into the VM

```bash
ssh youruser@192.168.64.X
```

Replace `youruser` with the account you created, and the IP with your VM's IP.

---

## 6) Install OpenClaw

Inside the VM:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

Follow the onboarding prompts to set up your model provider (Anthropic, OpenAI, etc.).

---

## 7) Configure channels

Edit the config file:

```bash
nano ~/.openclaw/openclaw.json
```

Add your channels:

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
  },
}
```

Then login to WhatsApp (scan QR):

```bash
openclaw channels login
```

---

## 8) Run the VM headlessly

Stop the VM and restart without display:

```bash
lume stop openclaw
lume run openclaw --no-display
```

The VM runs in the background. OpenClaw's daemon keeps the gateway running.

To check status:

```bash
ssh youruser@192.168.64.X "openclaw status"
```

---

## Bonus: iMessage integration

This is the killer feature of running on macOS. Use [BlueBubbles](https://bluebubbles.app) to add

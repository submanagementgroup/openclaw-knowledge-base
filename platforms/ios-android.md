---
domain: platforms
topic: "iOS and Android Nodes: Mobile App Setup, Pairing, and Node Features"
type: procedure
keywords:
  - iOS
  - Android
  - mobile
  - node app
  - mobile app
  - iOS pairing
  - Android node
related:
  - nodes/nodes-overview
  - platforms/macos-app
source:
  - platforms/ios.md
  - platforms/android.md
---

iOS and Android node apps for OpenClaw. These apps pair with the Gateway to provide mobile-native access and node features.

## iOS App

Availability: internal preview. The iOS app is not publicly distributed yet.

## What it does

- Connects to a Gateway over WebSocket (LAN or tailnet).
- Exposes node capabilities: Canvas, Screen snapshot, Camera capture, Location, Talk mode, Voice wake.
- Receives `node.invoke` commands and reports node status events.

## Requirements

- Gateway running on another device (macOS, Linux, or Windows via WSL2).
- Network path:
  - Same LAN via Bonjour, **or**
  - Tailnet via unicast DNS-SD (example domain: `openclaw.internal.`), **or**
  - Manual host/port (fallback).

## Quick start (pair + connect)

1. Start the Gateway:

```bash
openclaw gateway --port 18789
```

2. In the iOS app, open Settings and pick a discovered gateway (or enable Manual Host and enter host/port).

3. Approve the pairing request on the gateway host:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

If the app retries pairing with changed auth details (role/scopes/public key),
the previous pending request is superseded and a new `requestId` is created.
Run `openclaw devices list` again before approval.

Optional: if the iOS node always connects from a tightly controlled subnet, you
can opt in to first-time node auto-approval with explicit CIDRs or exact IPs:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

This is disabled by default. It applies only to fresh `role: node` pairing with
no requested scopes. Operator/browser pairing and any role, scope, metadata, or
public-key change still require manual approval.

4. Verify connection:

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## Relay-backed push for official builds

Official distributed iOS builds use the external push relay instead of publishing the raw APNs
token to the gateway.

Gateway-side requirement:

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

How the flow works:

- The iOS app registers with the relay using App Attest and a StoreKit app transaction JWS.
- The relay returns an opaque relay handle plus a registration-scoped send grant.
- The iOS app fetches the paired gateway identity and includes it in relay registration, so the relay-backed registration is delegated to that specific gateway.
- The app forwards that relay-backed registration to the paired gateway with `push.apns.register`.
- The gateway uses that stored relay handle for `push.test`, background wakes, and wake nudges.
- The gateway relay base URL must match the relay URL baked into the official/TestFlight iOS build.
- If the app later connects to a different gateway or a build with a different relay base URL, it refreshes the relay registration instead of reusing the old binding.

What the gateway does **not** need for this path:

- No deployment-wide relay token.
- No direct APNs key for official/TestFlight relay-backed sends.

Expected operator flow:

1. Install the official/TestFlight iOS build.
2. Set `gateway.push.apns.relay.baseUrl` on the gateway.
3. Pair the app to the gateway and let it finish connecting.
4. The app publishes `push.apns.register` automatically after it has an APNs token, the operator session is connected, and relay registration succeeds.
5. After that, `push.test`, reconnect wakes, and wake nudges can use the stored relay-backed registration.

## Background alive beacons

When iOS wakes the app for a silent push, background refresh, or significant-location event, the app
attempts a short node reconnect and then calls `node.event` with `event: "node.presence.alive"`.
The gateway records this as `lastSeenAtMs`/`lastSeenReason` on the paired node/device metadata only
after the authenticated node device identity is known.

The app treats a background wake as successfully recorded only when the gateway response includes
`handled: true`. Older gateways may acknowledge `node.event` with `{ "ok": true }`; that response is
compatible but does not count as a durable last-seen update.

Compatibility note:

- `OPENCLAW_APNS_RELAY_BASE_URL` still works as a temporary env override for the gateway.

## Authentication and trust flow

The relay exists to enforce two constraints that direct APNs-on-gateway cannot provide for
official iOS builds:

- Only genuine OpenClaw iOS builds distributed through Apple can use the hosted relay.
- A gateway can send relay-backed pushes only for iOS devices that paired with that specific
  gateway.

Hop by hop:

1. `iOS app -> gateway`
   - The app first pairs with the gateway through the normal Gateway auth flow.
   - That gives the app an authenticated node session plus an authenticated operator session.
   - The operator session is used to call `gateway.identity.get`.

2. `iOS app -> relay`
   - The app calls the relay registration endpoints over HTTPS.
   - Registration includes App Attest proof plus a StoreKit app tr

## Android App

The Android app has not been publicly released yet. The source code is available in the [OpenClaw repository](https://github.com/openclaw/openclaw) under `apps/android`. You can build it yourself using Java 17 and the Android SDK (`./gradlew :app:assemblePlayDebug`). See [apps/android/README.md](https://github.com/openclaw/openclaw/blob/main/apps/android/README.md) for build instructions.

## Support snapshot

- Role: companion node app (Android does not host the Gateway).
- Gateway required: yes (run it on macOS, Linux, or Windows via WSL2).
- Install: [Getting Started](/start/getting-started) + [Pairing](/channels/pairing).
- Gateway: [Runbook](/gateway) + [Configuration](/gateway/configuration).
  - Protocols: [Gateway protocol](/gateway/protocol) (nodes + control plane).

## System control

System control (launchd/systemd) lives on the Gateway host. See [Gateway](/gateway).

## Connection runbook

Android node app ⇄ (mDNS/NSD + WebSocket) ⇄ **Gateway**

Android connects directly to the Gateway WebSocket and uses device pairing (`role: node`).

For Tailscale or public hosts, Android requires a secure endpoint:

- Preferred: Tailscale Serve / Funnel with `https://<magicdns>` / `wss://<magicdns>`
- Also supported: any other `wss://` Gateway URL with a real TLS endpoint
- Cleartext `ws://` remains supported on private LAN addresses / `.local` hosts, plus `localhost`, `127.0.0.1`, and the Android emulator bridge (`10.0.2.2`)

### Prerequisites

- You can run the Gateway on the “master” machine.
- Android device/emulator can reach the gateway WebSocket:
  - Same LAN with mDNS/NSD, **or**
  - Same Tailscale tailnet using Wide-Area Bonjour / unicast DNS-SD (see below), **or**
  - Manual gateway host/port (fallback)
- Tailnet/public mobile pairing does **not** use raw tailnet IP `ws://` endpoints. Use Tailscale Serve or another `wss://` URL instead.
- You can run the CLI (`openclaw`) on the gateway machine (or via SSH).

### 1) Start the Gateway

```bash
openclaw gateway --port 18789 --verbose
```

Confirm in logs you see something like:

- `listening on ws://0.0.0.0:18789`

For remote Android access over Tailscale, prefer Serve/Funnel instead of a raw tailnet bind:

```bash
openclaw gateway --tailscale serve
```

This gives Android a secure `wss://` / `https://` endpoint. A plain `gateway.bind: "tailnet"` setup is not enough for first-time remote Android pairing unless you also terminate TLS separately.

### 2) Verify discovery (optional)

From the gateway machine:

```bash
dns-sd -B _openclaw-gw._tcp local.
```

More debugging notes: [Bonjour](/gateway/bonjour).

If you also configured a wide-area discovery domain, compare against:

```bash
openclaw gateway discover --json
```

That shows `local.` plus the configured wide-area domain in one pass and uses the resolved
service endpoint instead of TXT-only hints.

#### Tailnet (Vienna ⇄ London) discovery via unicast DNS-SD

Android NSD/mDNS discovery won’t cross networks. If your Android node and the gateway are on different networks but connected via Tailscale, use Wide-Area Bonjour / unicast DNS-SD instead.

Discovery alone is not sufficient for tailnet/public Android pairing. The discovered route still needs a secure endpoint (`wss://` or Tailscale Serve):

1. Set up a DNS-SD zone (example `openclaw.internal.`) on the gateway host and publish `_openclaw-gw._tcp` records.
2. Configure Tailscale split DNS for your chosen domain pointing at that DNS server.

Details and example CoreDNS config: [Bonjour](/gateway/bonjour).

### 3) Connect from Android

In the Android app:

- The app keeps its gateway connection alive via a **foreground service** (persistent notification).
- Open the **Connect** tab.
- Use **Setup Code** or **Manual** mode.
- If discovery is blocked, use manual host/port in **Advanced controls**. For private LAN hosts, `ws://` still works. For Tailscale/public hosts, turn on TLS and use a `wss://` / Tailscale Serve endpoint.

After the first successful pairing, Android auto-reconnects on launch:

- Manual endpoint (if enabled), otherwise
- The last discovered gateway (best-effort).

### Presence alive beacons

After the authenticated node session connects, and when the app moves to the background while the
foreground service is still connected, Android calls `node.event` with
`event: "node.presence.alive"`. The gateway records this as `lastSeenAtMs`/`lastSeenReason` on the
paired node/device metadata only after the authenticated node device identity is known.

The app counts the beacon as successfully recorded only when the gateway response includes
`handled: true`. Older gateways may acknowledge `node.event` with `{ "ok": true }`; that response is
compatible but does not count as a durable last-seen update.

### 4) Approve pairing (CLI)

On the gateway machine:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Pairing details: [Pairing](/channels/pairing).

Optional: if the Android node al

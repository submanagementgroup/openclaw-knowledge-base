---
domain: nodes
topic: "Location Command: node.invoke location.get, Permissions, and Android Foreground Behavior"
type: reference
keywords:
  - location.get
  - node location
  - location command
  - location permission
  - whileUsing location
  - precise location
  - LOCATION_DISABLED
  - LOCATION_TIMEOUT
  - Android location foreground
  - location_get CLI
source: nodes/location-command.md
related:
  - nodes/nodes-overview
  - nodes/troubleshooting
  - channels/location
---

`location.get` is a node command invoked via `node.invoke`. It is off by default. Android app requires the app to be in the foreground for location access.

## Settings Model (Per Node Device)

- `location.enabledMode`: `off` | `whileUsing`
- `location.preciseEnabled`: bool

**UI behavior:** Selecting `whileUsing` requests foreground permission. If the OS denies the requested level, OpenClaw reverts to the highest granted level and shows status.

## Permissions Mapping

OS permissions are multi-level. The UI selector drives the requested mode; the actual grant lives in OS settings.

- iOS/macOS may expose **While Using** or **Always** in system prompts/Settings.
- Android app currently supports foreground location only.
- Precise location is a separate grant (iOS 14+ "Precise", Android "fine" vs "coarse").

## Command: `location.get`

Called via `node.invoke`:

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

**Response payload:**

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

## Error Codes

| Code | Cause |
|------|-------|
| `LOCATION_DISABLED` | Location mode selector is off |
| `LOCATION_PERMISSION_REQUIRED` | Permission missing for requested mode |
| `LOCATION_BACKGROUND_UNAVAILABLE` | App is backgrounded but only While Using is allowed |
| `LOCATION_TIMEOUT` | No fix obtained within `timeoutMs` |
| `LOCATION_UNAVAILABLE` | System failure / no providers |

## CLI

```bash
openclaw nodes location get --node <id>
```

## Android Background Behavior

The Android app denies `location.get` while backgrounded. Keep OpenClaw open when requesting location on Android. Other node platforms may differ.

## Model/Tooling Integration

- Tool surface: `nodes` tool adds `location_get` action (node required).
- Agent guideline: only call when the user has enabled location and understands the scope.

## Related

- [Nodes overview](/nodes/nodes-overview)
- [Node troubleshooting](/nodes/troubleshooting)
- [Channel location parsing](/channels/location)

---
domain: nodes
topic: "Image and Media Support: Send, Gateway, and Agent Reply Pipeline"
type: reference
keywords:
  - media send
  - image send
  - openclaw message send media
  - WhatsApp media
  - audio voice note
  - document send
  - gif playback
  - media understanding
  - inbound media
  - MediaUrl MediaPath
  - media transcription
source: nodes/images.md
related:
  - nodes/nodes-overview
  - nodes/media-understanding
  - channels/whatsapp
---

Image and media handling rules for send, gateway, and agent replies in the OpenClaw WhatsApp channel (Baileys Web). Covers the `openclaw message send --media` CLI, auto-reply pipeline, inbound media processing, and size limits.

## CLI Surface

```bash
# Send media (with optional caption)
openclaw message send --media <path-or-url> [--message <caption>]

# Dry run (preview payload without sending)
openclaw message send --media photo.jpg --dry-run

# JSON output
openclaw message send --media photo.jpg --json
# Returns: { channel, to, messageId, mediaUrl, caption }
```

## WhatsApp Web Channel Behavior

**Input:** local file path or HTTP(S) URL.

**Flow by media type:**

| Type | Processing | Cap |
|------|-----------|-----|
| Images | Resize and recompress to JPEG (max side 2048px) | `channels.whatsapp.mediaMaxMb` (default: 50 MB) |
| Audio/Voice | Pass-through; sent as voice note (`ptt: true`) | 16 MB |
| Video | Pass-through | 16 MB |
| Documents | Pass-through with filename preserved | 100 MB |

- **GIF-style playback:** send an MP4 with `gifPlayback: true` (CLI: `--gif-playback`) so mobile clients loop inline.
- **MIME detection:** prefers magic bytes, then headers, then file extension.
- **Caption:** from `--message` or `reply.text`; empty caption is allowed.

## Auto-Reply Pipeline

`getReplyFromConfig` returns `{ text?, mediaUrl?, mediaUrls? }`. When media is present, the web sender resolves local paths or URLs using the same pipeline as `openclaw message send`. Multiple media entries are sent sequentially.

## Inbound Media Processing (Pi Commands)

When inbound web messages include media, OpenClaw downloads to a temp file and exposes:

- `{{MediaUrl}}` — pseudo-URL for the inbound media
- `{{MediaPath}}` — local temp path written before running the command

**Docker sandbox:** inbound media is copied into the sandbox workspace and `MediaPath`/`MediaUrl` are rewritten to a relative path like `media/inbound/<filename>`.

**Media understanding** (configured via `tools.media.*`) runs before templating and can insert `[Image]`, `[Audio]`, and `[Video]` blocks into `Body`:

- Audio sets `{{Transcript}}` and uses the transcript for command parsing so slash commands still work.
- Video and image descriptions preserve any caption text for command parsing.
- If the active primary image model already supports vision natively, OpenClaw skips the `[Image]` summary block and passes the original image to the model instead.
- By default only the first matching image/audio/video attachment is processed; set `tools.media.<cap>.attachments` to process multiple attachments.

## Size Limits

**Outbound send caps (WhatsApp web):**

| Type | Cap |
|------|-----|
| Images | `channels.whatsapp.mediaMaxMb` (default: 50 MB after recompression) |
| Audio/Voice/Video | 16 MB |
| Documents | 100 MB |

**Media understanding caps:**

| Type | Default cap |
|------|------------|
| Image | 10 MB (`tools.media.image.maxBytes`) |
| Audio | 20 MB (`tools.media.audio.maxBytes`) |
| Video | 50 MB (`tools.media.video.maxBytes`) |

Oversize or unreadable media → clear error in logs; reply still goes through with the original body.

## Related

- [Nodes overview](/nodes/nodes-overview)
- [Media understanding](/nodes/media-understanding)
- [Camera capture](/nodes/camera)

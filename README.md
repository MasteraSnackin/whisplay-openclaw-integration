# Whisplay AI Chatbot ↔ OpenClaw Integration

This repo contains everything needed to connect a **Whisplay AI Chatbot** (Pi 5 + LLM8850) to **OpenClaw** as a voice I/O channel.

When you speak to the Whisplay device, your voice is transcribed locally (Whisper on LLM8850 NPU), forwarded to OpenClaw, processed by your OpenClaw agent, and the reply is spoken back via MeloTTS — all without the Pi needing to run its own LLM.

## Works on Pi 5 4GB

Because `LLM_SERVER=whisplay-im` offloads all LLM inference to OpenClaw, the Pi only handles voice I/O via the LLM8850 NPU. The bridge service uses ~100MB of Pi RAM.

## Contents

| File | Purpose |
|---|---|
| `whisplay-dotenv-additions.env` | Lines to add to `~/whisplay-ai-chatbot/.env` on the Pi |
| `openclaw.json` | OpenClaw config template (`~/.openclaw/openclaw.json`) |
| `whisplay-im/index.js` | OpenClaw channel plugin |
| `whisplay-im/openclaw.channel.json` | Channel metadata |
| `whisplay-im/openclaw.plugin.json` | Plugin metadata |

## Quick Start

### Step 1 — Configure the Pi

SSH into your Pi:

```bash
ssh pi@<YOUR_PI_IP>
cd ~/whisplay-ai-chatbot
nano .env
```

Add these two lines:

```env
LLM_SERVER=whisplay-im
WHISPLAY_IM_TOKEN=your-secret-token
```

Restart:

```bash
sudo systemctl restart chatbot.service
```

### Step 2 — Install the OpenClaw Plugin

On your OpenClaw machine:

```bash
git clone https://github.com/MasteraSnackin/whisplay-openclaw-integration
openclaw plugins install /absolute/path/to/whisplay-openclaw-integration/whisplay-im --link
```

### Step 3 — Configure OpenClaw

Edit `~/.openclaw/openclaw.json`:

```json
{
  "channels": {
    "whisplay-im": {
      "enabled": true,
      "accounts": {
        "default": {
          "ip": "YOUR_PI_IP:18888",
          "token": "your-secret-token",
          "waitSec": 60
        }
      }
    }
  }
}
```

### Step 4 — Restart OpenClaw Gateway

```bash
openclaw gateway restart
openclaw channels status
```

### Step 5 — Test

```bash
# Send a message to the Pi (should be spoken aloud)
curl -X POST \
  -H "Authorization: Bearer your-secret-token" \
  -H "Content-Type: application/json" \
  -d '{"reply":"Hello from OpenClaw","emoji":"🦞"}' \
  http://YOUR_PI_IP:18888/whisplay-im/send

# Poll for voice messages from the Pi
curl -X GET \
  -H "Authorization: Bearer your-secret-token" \
  "http://YOUR_PI_IP:18888/whisplay-im/poll?waitSec=5"
```

## Architecture

```
[You speak] → Whisplay HAT mic
           → whisper.axcl (LLM8850 NPU)
           → whisplay-im bridge (HTTP, port 18888)
           → OpenClaw polls /whisplay-im/poll
           → OpenClaw agent generates reply
           → OpenClaw POSTs to /whisplay-im/send
           → melotts.axcl (LLM8850 NPU)
           → Speaker plays reply
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `HTTP 401` on poll/send | Token mismatch — check `.env` vs `openclaw.json` |
| `HTTP 404` on poll | `LLM_SERVER` not set to `whisplay-im` — restart chatbot |
| No poll requests in chatbot log | OpenClaw can't reach Pi — check IP and port 18888 |
| `missing accounts.default.ip` error | Set `ip` in `openclaw.json` accounts section |

## License

GPL-3.0 — consistent with [PiSugar/whisplay-ai-chatbot](https://github.com/PiSugar/whisplay-ai-chatbot)
